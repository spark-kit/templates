---
name: spark-nocodb-v3-patterns
description: Pièges et patterns empiriques NocoDB v3 (CE 2026.04.5+) sur stack Spark. Use when modeling data, creating NocoDB tables/fields, writing n8n workflows that read/write NocoDB, debugging Links/Lookups/filters, or hitting "le lien ne se crée pas", "where sur FK renvoie 0", "bulk delete silencieux", "Lookup renvoie un array/null", "/links renvoie records vide", "GET record lent", "le record a deux liens", "le Lookup renvoie deux valeurs", "get-or-create a créé un doublon", "ma garde sur un champ Links ne marche que parfois". Complète (ne remplace pas) la skill `nocodb` de référence API.
metadata:
  spark:
    layer: data
    source: spark-pitfalls-catalog (N1-N39)
    nocodb_version: "2026.04.5+ CE"
---

# NocoDB v3 — pièges & patterns Spark

> Couche **empirique** au-dessus de la skill `nocodb` (qui documente l'API v3 brute).
> Ici : ce qui fait perdre 20-60 min à découvrir et 3 s à éviter. Cristallisé sur la chaîne WMS v2 (102 assertions E2E, 6 PRDs) puis enrichi en continu — 2026-07-31 : **N34-N36** (un même chantier de re-liaison, tous les trois silencieux) · 2026-08-05 : **N37-N39** (coût caché d'un lien en lecture, sémantique d'écriture par `relation_type`) · 2026-08-19 : **N40** (where date) + nuances N33/N37 (la falaise se franchit en jours ; `isblank` sur Link peut rendre 42804).
> **Cible : NocoDB Community Edition 2026.04.5+, API v3, PAT (`xc-token`).**

---

## 🚨 Les 4 pièges qui coûtent le plus

### 1. Insert avec FK ≠ création du lien (N3 — vécu 4 fois)

Passer une FK dans le body d'insert **ne crée PAS le lien**. Le champ Link renvoie ensuite `1` (= « 1 lien »), mais ce `1` n'a **aucun référent réel**.

```jsonc
// ❌ NE CRÉE PAS LE LIEN — juste un compteur fantôme
POST /api/v3/data/{base}/{table}/records
{ "fields": { "commande": { "id": 42 } } }
```

```bash
# ✅ Créer le lien APRÈS l'insert, séparément (N4 — pattern universel)
POST /api/v3/data/{base}/{table}/links/{link_field_id}/{record_id}
[ { "id": 42 } ]
```

Dans un workflow n8n : insert → récupérer l'id créé (`{{ $json.records[0].id }}`) → 1 POST `/links` par FK. C'est **le** pattern de toute écriture avec relations.

⚠️ **N34 — 🚨 POST `/links` AJOUTE un lien, il ne REMPLACE jamais — même sur un `belongsTo`.** Le nom de la relation promet 1:1, l'API n'en tient aucun compte :

```jsonc
// dossiers.sku_kyklos est déclaré "relation_type": "belongsTo"
POST /links/{link_field_id}/285   [{"id": 339}]        → 200 OK
GET  /links/{link_field_id}/285                        → records: [{id:214}, {id:339}]   // DEUX
GET  /records/285?fields=Id,sku_kyklos,sku_code_lookup
     → { "sku_kyklos": 2, "sku_code_lookup": ["…-A0M10", "…-A0210"] }   // compteur 2, Lookup à 2
```

Aucune erreur, aucun avertissement. Le compteur de Link passe à `2` sur une relation censée être 1:1 et le Lookup (N20) renvoie **deux** valeurs. Tout consommateur qui fait `champ[0]` continue de lire l'**ancienne** — la correction paraît avoir échoué alors qu'elle a *doublé* la donnée. Pire cas : un écran lit `[0]`, un autre `[1]`, et les deux sont « cohérents avec eux-mêmes ».

**Pattern de re-liaison idempotent** — ne jamais présumer qu'un POST remplace, quel que soit le `relation_type` déclaré :

```
GET  /links/{field}/{id}?fields=Id     → liens actuels
DELETE /links/{field}/{id}  [{id:…}, …]   // tout ce qui n'est pas la cible
POST   /links/{field}/{id}  [{id: cible}] // seulement si absent
GET  /links/{field}/{id}?fields=Id     → RELIRE : exactement 1, et le bon
```

La relecture finale n'est pas du zèle : c'est elle qui transforme un « 80/80 re-liés » mensonger en échec bruyant. (Vécu 2026-07-31, réalignement de 80 dossiers.)

**N38 — 🚨 le comportement de POST `/links` dépend du `relation_type`, et les noms mentent.** N34 ci-dessus est vrai pour un `belongsTo` ; ce n'est **pas** la règle générale. Vérifié en base le 2026-08-05, en lisant `/links` (jamais l'objet expansé — c'est précisément N35 qui rend cette vérification fiable) :

| `relation_type` réel | POST `/links` sur un lien déjà posé | Conséquence pratique |
|---|---|---|
| **`belongsTo`** | **ajoute** (N34) → 2 liens, Lookup à 2 valeurs, `[0]` lit l'ancienne | DELETE des liens actuels → POST → **relire** |
| **`mo`** (many-to-one) | **remplace** — après deux POST successifs, `/links` ne montre qu'**une** entrée | un seul appel suffit pour changer la cible |
| **`mm`** | **idempotent** dans les deux sens (double DELETE → `{success:true}` ; double POST → compteur reste à 1) | rejouable sans garde préalable |

⚠️ Le piège du piège : `mo` et `belongsTo` sonnent tous deux « 1:1 » et se comportent à l'**opposé**. Un champ nommé pareil dans deux tables peut avoir l'un ou l'autre (`pieces.piece_type` = `mo`, `dossiers.sku_kyklos` = `belongsTo`). ➡️ **Lire `relation_type` dans la meta avant d'écrire** — et dans le doute, appliquer le pattern de re-liaison de N34, qui est correct dans les trois cas.

⚠️ **N39 — DELETE d'un *record* inexistant → 404, alors que DELETE d'un *lien* inexistant rend `{success:true}`.** Asymétrie piégeuse : ne supprimer que des ids qu'on vient de **lire**, jamais un id déduit. Corollaire pour une double SoT (jonction explicite **+** m2m natif, cas fréquent) : le m2m se rejoue sans risque, **la ligne de jonction non** — un « lier » naïf rejoué laisse le m2m propre et crée un doublon de jonction. Lire les jonctions existantes avant d'en créer une.

### 2. Lecture Links = compteurs, pas d'objets (N5/N6/N21)

La liste des records renvoie `0/1/2…` sur les champs Link (nombre de liens), **jamais** l'objet expansé (contrairement à v1).

```bash
# Résoudre les FK : GET /links, en explicitant les champs voulus
GET /api/v3/data/{base}/{table}/links/{link_field_id}/{record_id}?fields=Id,nom,statut
```

⚠️ **N21** : sans `?fields=`, `/links` ne renvoie **que le display field** (1er SingleLineText). Tous les autres champs sont *omis* (même pas `null`). Toujours expliciter `?fields=`.
⚠️ **N26 — 🚨 le `?fields=` d'un `/links` doit inclure `Id`** : `?fields=code,nom` (sans `Id`) → `records: []` **vide silencieux** alors que les liens existent. Pire que N21 : on perd les records eux-mêmes, pas juste des champs. Toujours `?fields=Id,…`.
⚠️ **N30 — `/links` d'un lien `bt` (belongsTo) renvoie `{record: {...}}` SINGULIER** (pas `{records: [...]}`) — un consumer qui lit `.records` obtient `[]`/undefined en silence. Gérer les deux formes : `resp.record || (resp.records || [])[0]`. (Vécu 2026-07-17, dossiers.couleur_stock.)
⚠️ **N35 — 🚨 « compteur » n'est vrai qu'en fetch LISTE : en `GET /records/{id}`, un champ Links renvoie l'OBJET EXPANSÉ** — et le `?fields=` n'y change rien, c'est la **route** qui décide :

```jsonc
GET /records?where=(Id,eq,299)&fields=Id,produit_kyklos   → { "produit_kyklos": 1 }          // compteur
GET /records/299?fields=Id,produit_kyklos                 → { "produit_kyklos": { "Id": 1008,
      "code_produit": "…", "_nc_m2m_sku_kyklos_produit_kyklos": [ … ] } }                    // ~4 ko
```

Conséquence directe : toute garde numérique du type `if (champ || 0) !== 0` **passe sur la liste et échoue toujours par id**, l'objet étant systématiquement truthy. Pour lire un compteur, passer par une liste (`?where=(Id,eq,X)&pageSize=1`). C'est le même mécanisme que **N28** (GET par id 10-35× plus lent) vu de l'autre côté : la lenteur et l'expansion sont un seul phénomène. (Vécu 2026-07-31, garde de suppression d'un doublon.)
⚠️ Résolution N+1 par défaut → voir pattern d'agrégat par Lookups plus bas (N24).

### 3. `where` sur un champ Link ne marche pas (N7/N8)

```bash
# ❌ Renvoie 0 résultat même si des liens existent
GET ...?where=(commande,eq,42)
```

Deux contournements (N8) :
- **(a)** interroger le `/links` **inverse** côté table opposée (NocoDB crée auto un Link inverse pour tout `belongsTo`, cf. N13 — son nom est trouvable via `field:list`) ;
- **(b)** ajouter un Link `belongsTo` **dénormalisé direct** vers le parent recherché (ex. `pannes_dossier.dossier` posé en plus de `pannes_dossier.test`) → permet un `where=(dossiers_id,eq,X)` simple. À réserver aux volumes modérés.

> Pour filtrer sur une FK « normale » (belongsTo), le `where` veut le **nom de colonne FK réel** (`dossiers_id`), pas le nom logique (`dossier`) → 400 sinon. Vérifier le vrai nom dans le schéma.

### 4. Bulk insert ET delete cap à 10 — delete échoue *silencieusement* (N2)

`POST`/`DELETE` bulk au-delà de 10 records → `ERR_MAX_PAYLOAD_LIMIT_EXCEEDED` (insert) ou **0 suppression sans aucune erreur** (delete). Toujours **batcher par 10** et **vérifier le retour** (`len(records supprimés)`), sinon boucle infinie possible si le caller re-page sans contrôle.

---

## Lookups (N19/N20/N24/N25/N27/N29/N33)

Les Lookups sont la bonne arme contre le N+1 sur les agrégats — mais 6 chausse-trappes :

- **N19 — schéma de création** : `options.related_field_id` (= ID du **Link** sur la table source) + `options.related_table_lookup_field_id` (= ID du **champ à lookup-er** sur la cible). **PAS** `fk_relation_column_id` / `fk_lookup_column_id` (noms intuitifs mais faux → 400).
- **N20 — réponse = ARRAY** : un Lookup renvoie `["Samsung Galaxy S7"]`, jamais la string nue, **même** pour un belongsTo 1:1. Le consommateur fait `champ[0]`.
- **N24 — agrégat N:N propre** : poser des Lookups sur la **table de jonction** (ex. `piece_id_l` via Link pieces + `loc_type_l` via Link localisations) permet de fetch toutes les jointures avec attributs résolus **en 1 call** → agrégat côté Code node. Évite le N+1×2. **Pattern réutilisable.**
- **N25 (raffiné 2026-07-15) — 🚨 la collision de Lookups est PAR LIEN** : plusieurs Lookups du **même** Link coexistent dans un seul `?fields=` (ex. `sku_code_lookup` + `sku_libelle_complet`, tous deux via le Link `sku_kyklos` → les 2 corrects). C'est un Lookup d'un **autre** Link dans le même `?fields=` qui revient `[null]` — valeur perdue, sur `/records/{id}` comme sur les listes. **Workaround : 1 fetch HTTP par Link porteur de Lookups** (pas par Lookup), combiner dans Build Response. ⚠️ **Faux négatif de test** : sur un record dont le lien est vide, `[]` semble correct → tester la collision sur un record où **tous** les liens concernés sont peuplés avant de "simplifier" des fetchs séparés existants.
- **N27 — 🚨 Lookup sur un lien `mo` (many-to-one) = `null` systématique** : un Lookup posé sur un Link `relation_type: "mo"` renvoie `null` partout, même config identique aux Lookups qui marchent et lien peuplé. Les Lookups sur `belongsTo` et sur `mm` fonctionnent. **Vérifier `relation_type` du Link (meta table) AVANT de créer un Lookup dessus.** Workaround batching sans changement de schéma : joindre via une table intermédiaire belongsTo + colonne FK dénormalisée (ex. `dossiers → ligne_commande` par `/links` inverse par ligne + `lignes_commande.produit_kyklos_id`) → O(intermédiaires) appels au lieu de O(records).
- **N29 — 🔥 Lookup `mm` sur une LISTE = superlinéaire (fondeur de Postgres)** : `records?pageSize=N&fields=…,lookup_mm` → N=50 : 0,5 s · N=150 : 6-8 s · N=500 : 25 s+ **et Postgres à 190 % CPU / +2 GiB** (cause racine d'OOM récurrents sur kyklos — endpoint appelé à chaque ouverture d'écran). Les requêtes zombies survivent aux timeouts client et au restart du client NocoDB — seul un restart Postgres purge. **Fix = inversion de requête** : partir de l'objet demandé (`/links` inverse : « les X compatibles de CE parent », trivial) puis ne fetcher que ces records (`where=(Id,in,…)`). Diagnostiquer par paliers de pageSize (5→50→150) pour isoler le champ toxique sans re-fondre la base.
- **N33 — 🚨 la superlinéarité N29 frappe AUSSI les Lookups `belongsTo`, juste plus tard** : un Lookup bt dans le `?fields=` d'une **liste** tient jusqu'à ~200 rows (≈2 s) puis s'effondre vers ~600 rows : **40-45 s puis `ERR_INTERNAL_SERVER`** (`fields=Id` seul : <1 s — vécu 2026-07-25, table de jonction à 615 rows, les 2 Lookups testés séparément, même verdict). Un fetch qui « marchait » à 200 rows casse silencieusement quand la table grossit. Remèdes par ordre : ① le **compteur de Link suffit souvent** (N5 : le champ Link en liste = nombre de liens — « ce record a-t-il ≥1 lien ? » ne coûte aucun Lookup) ; ② pagination ≤200 rows si les valeurs sont vraiment nécessaires ; ③ `/links` du parent ciblé (peu de rows par parent — le pattern par-parent reste sain). Après l'échec, Postgres retombe seul (pas de zombie type N29) mais garde ~1,6 GiB de caches. ⚠️ **La falaise se franchit en JOURS, pas en mois** (mesuré 2026-08-19 : deux tables +80 et +194 rows en UNE journée d'atelier — 5 crashs OOM Postgres le jour où elles ont passé ~600) : un fetch liste avec Lookup non borné qui « marche » aujourd'hui est une bombe à échéance de semaines. **Fix durable éprouvé : dénormaliser en colonne scalaire** (patcher le writer AVANT le backfill, backfiller, migrer les lecteurs) — un scalaire se lit gratis (bench : Lookup 17,25 s vs scalaire 0,20 s sur la même donnée, ×86). Avant tout backfill de masse, **lister les consommateurs d'`UpdatedAt`** de la table : le backfill bumpe TOUS les rows, et tout filtre « modifié aujourd'hui » ment le jour même.

- **N37 — 🚨 même SANS Lookup, demander un champ `belongsTo` dans le `?fields=` d'une LISTE coûte comme un Lookup** : NocoDB résout la ligne liée **entière** (avec ses propres compteurs de liens) pour chaque row. Mesuré sur 664 rows, pageSize=1000 : `fields=` sans le lien → **0,08 s / 227 KB** ; `+ piece_type` (belongsTo) → **0,88 s / 411 KB** (×11) ; `+ modeles_compatibles` (compteur mm) → **0,08 s** (gratuit). C'est N28 (le coût des expansions) qui frappe en liste, pas seulement sur un GET par id. **Contournement quand on ne veut qu'un booléen « ce record a-t-il ce lien ? »** : une requête dédiée `?fields=Id&where=(champ_link,isblank)` rend les seuls ids concernés — **0,06 s / 4 KB pour 88 ids**. ⚠️ `isblank`/`notblank` **filtrent bien sur une colonne Link** (contrairement à `eq`, cf. N7) ; `(champ,is,null)` renvoie **0 ligne en silence**. (Vécu 2026-08-05, endpoint de liste appelé par 3 écrans atelier.) ⚠️ Nuance 2026-08-14/19 : `where=(lien,isblank)` peut aussi rendre **`ERR_DATABASE_OP_FAILED` 42804** (datatype mismatch Postgres) sur certains liens `bt` — vécu sur 2 liens d'une même table quand d'autres (dont un `mo`) passaient. Repli fiable partout : **`isblank` sur un Lookup du lien** — même sémantique, zéro coût.

---

## Idempotence & concurrence (N36)

**N36 — 🚨 un get-or-create NocoDB n'est pas atomique : appelé en fan-out sur la même clé, il crée des doublons.** Le pattern canonique d'un endpoint « résoudre-ou-créer » est `Find (where=clé)` → `IF Exists` → `Insert`. Correct appelé une fois. Mais un node HTTP n8n **itère sur ses items d'entrée** : N items visant la même clé produisent N `Find` qui ne trouvent rien, puis N `Insert`.

```
item A → clé "APP-IP11-64-MX-C0210"  ┐ les deux Find ne trouvent rien,
item B → clé "APP-IP11-64-MX-C0210"  ┘ les deux insèrent → 2 records, même clé
```

NocoDB **n'a pas de contrainte d'unicité applicative** sur un SingleLineText : rien ne refuse le second insert. Le doublon ne casse rien tout de suite — il casse **au coup suivant**, quand un consommateur résout `clé → id` : un `dict[clé] = id` garde le dernier, et une écriture ciblée (DELETE d'un lien, PATCH) frappe le jumeau. Le symptôme apparaît alors à plusieurs nœuds de la cause, dans une exécution *antérieure*.

**Trois remèdes, cumulatifs** :
1. **Dédupliquer AVANT le fan-out** — un item par clé cible distincte, puis ré-étaler le résultat sur les items d'origine (`byCle[réponse.clé] = id`).
2. **Ne jamais résoudre par clé naturelle ce qu'on peut lire directement** — pour délier, lire les liens réels du record (`GET /links`) plutôt que traduire une clé en id. Robuste aux doublons *et* aux N34 déjà en place.
3. **Un invariant d'unicité dans le harnais E2E** — `SELECT clé HAVING count(*) > 1` sur la table de référence. Sur un cas réel, il a immédiatement sorti un doublon **préexistant** que personne n'avait vu : la classe de bug était déjà là.

> Corollaire de méthode : « risque de concurrence documenté et assumé » sur une écriture unitaire ne l'assume **pas** en fan-out. C'est la même ligne de code, avec N appels au lieu d'un. Re-qualifier le risque quand l'appelant change de forme. (Vécu 2026-07-31.)

---

## CRUD & schéma — réflexes

| # | Règle |
|---|---|
| N1 | **PAT (`xc-token`) sur v3 uniquement** — v1/v2 → 403 sur 2026.04.5+. |
| N9 | **GET single par valeur naturelle** (code, ref) : `?where=(code,eq,X)&pageSize=1` puis check `records.length === 1`. Pas d'endpoint dédié. |
| N10 | **GET single par ID interne** : `GET /records/{id}` (sans `?where`) — plus rapide quand on a l'id. Toujours avec `?fields=` (N28). |
| N11 | **SingleSelect création** : `options.choices: [{title:"X"}, …]` (pas besoin de color/id). |
| N12 | **Update SingleSelect** : PATCH le field avec la liste **complète** des choices (pas un append — sinon on perd les existants). |
| N13 | **belongsTo crée auto un Link inverse** côté cible (`tableSource` ou `tableSource1`). Indispensable pour les agrégats inverses. Trouvable via `field:list`. |
| N14 | **Formula/Rollup pénibles en v3** → préférer un champ direct géré par n8n (ex. `quantite_stock` updaté à chaque mouvement). |
| N15 | **Renommer une base = cosmétique** : `title` change, ID inchangé. Sûr pour l'archivage. |
| N16 | **Append-only sur tables d'audit** : pas de DELETE, correction = mouvement **compensatoire** (`mouvements_stock`, `evenements`). |
| N17 | **PATCH single** : `{id, fields:{…}}` (objet). **PATCH bulk** : `[{id, fields}, …]` (array). |
| N18 | **MCP NocoDB instable** sur 2026.04.5+ → utiliser le **CLI `nocodb.sh`** de la skill `nocodb`. Cf. INC-2026-05-19. |
| N28 | **🚨 GET `/records/{id}` sans `?fields=` = 10-35× plus lent** (150-750 ms vs ~20 ms mesuré) : NocoDB résout **toutes** les expansions (objets belongsTo, m2m imbriqués, jonctions `_nc_m2m_*`) même si le payload final paraît petit. `?fields=` systématique sur tout GET par id dans un workflow. |

---

## Réponses — wrapping & tri (rappels qui coûtent cher en n8n)

- **Response wrapping** : toute réponse POST/PATCH est `{records:[{id, fields:{…}}]}`. Dans n8n : `{{ $json.records[0].id }}`, **jamais** `{{ $json.id }}`. (Bug #1 historique, chaque endpoint l'a eu.)
- **Sort** : `sort=[{"field":"CreatedTime","direction":"desc"}]` (JSON URL-encodé). Le format v1/v2 `-CreatedTime` → 400 au message trompeur.
- **Types de champs date** : `CreatedTime` / `LastModifiedTime` (pas `CreatedAt`).
- **N40 — un `where` sur un champ date exige le sous-opérateur** : `(CreatedAt,gt,exactDate,2026-08-17)` — la forme intuitive `(CreatedAt,gte,2026-08-17)` → `ERR_INVALID_FILTER`. (Vécu 2026-08-19 en bornant un fetch d'événements par jour.)

---

## Check-list avant tout HTTP NocoDB dans un workflow

1. Écriture avec FK ? → insert **puis** `POST /links` (N3/N4).
1bis. **Re**-liaison (le record a déjà un lien) ? → **N34/N38** : le POST **ajoute** sur un `belongsTo` (il ne remplace pas, malgré le nom) mais **remplace** sur un `mo` — lire `relation_type` avant d'écrire, ou appliquer le pattern sûr dans les deux cas : DELETE des liens actuels → POST la cible → **relire** et vérifier qu'il n'en reste qu'un.
2. Lecture qui a besoin des FK ? → `/links?fields=Id,…` explicite (N21 + **Id obligatoire** N26), ou Lookups (N24) en pesant N25/N27 (1 fetch par **lien** ; jamais sur un lien `mo`).
2ter. Garde/compteur sur un champ Links ? → **N35** : compteur en fetch **liste** seulement ; en `GET /records/{id}` c'est un objet expansé, donc toujours truthy.
2bis. Lookup dans un fetch **liste** ? → **N29/N33** : superlinéaire — `mm` interdit au-delà de ~50 rows, `bt` s'effondre vers ~600. Inverser la requête (/links du parent + `(Id,in,…)`), ou **compteur de Link** si seul « ≥1 lien ? » compte.
3. Filtre sur relation ? → **pas** de `where` sur Link (N7) ; /links inverse (N8a) ou dénorm (N8b) ; FK normale = nom de colonne réel.
3bis. Besoin de savoir seulement **si** un lien existe, sur une liste ? → **jamais** le champ belongsTo dans le `?fields=` (N37, ×11 sur 664 rows) : requête dédiée `?fields=Id&where=(champ_link,isblank)`. Un **compteur** de lien mm, lui, est gratuit (N5).
4. Bulk ? → batch de 10 + check du retour (N2).
5. Lecture de l'id créé ? → `records[0].id`.
6. GET par id ? → `?fields=` systématique, sinon 10-35× plus lent (N28).
7. Écriture de lien **rejouable** ? → le POST **ajoute** sur un `belongsTo` (N34) mais **remplace** sur un `mo` — lire `relation_type` avant d'écrire (N38) ; un `mm` est idempotent, une ligne de **jonction** non : la lire avant de la créer, et ne DELETE que des ids lus (N39).
8. Un endpoint qui reçoit un id de ligne « à mettre à jour » de son appelant ? → le **résoudre lui-même** (`where` sur les colonnes scalaires dénormalisées, N8b — ça, ça filtre) plutôt que de faire confiance : deux appelants finissent toujours par diverger, et c'est l'écrivain qui porte l'invariant. (Vécu : 138 lignes dupliquées sur 555.)

> Promouvoir tout nouveau piège confirmé ici **et** dans `spark-kit/INCIDENTS.md` s'il est transverse à plusieurs sites.
