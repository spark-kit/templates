---
name: spark-n8n-pseudo-api
description: Construire un endpoint backend Spark = webhook n8n adossé à NocoDB (pseudo-API derrière Caddy). Use when creating/editing an n8n webhook endpoint, designing a Spark backend route, wiring a front to NocoDB via n8n, or debugging "invalid syntax"/"Unmatched expression brackets" on a jsonBody, IF node validation errors, parallel-branch fan-in returning undefined, or webhook path params. Complète les skills `n8n-*` (qui couvrent un node isolé) par l'assemblage bout-en-bout.
metadata:
  spark:
    layer: backend
    source: spark-pitfalls-catalog (W1-W23, P1-P2)
    arch: "Caddy → n8n (pseudo-API) → NocoDB"
---

# n8n pseudo-API — patterns d'endpoint Spark

> L'architecture Spark : **Caddy** sert un front statique + reverse-proxy → **n8n** expose des webhooks qui orchestrent → **NocoDB** stocke.
> Un « endpoint » Spark = un workflow webhook. Cette skill explique comment **assembler** un endpoint correct du premier coup. Les skills `n8n-node-configuration` / `n8n-expression-syntax` couvrent un node isolé ; ici on couvre la **route entière**.
> Voir aussi `spark-nocodb-v3-patterns` pour la couche data (Links, Lookups, bulk).

---

## 🚨 Le bug n°1 : jsonBody avec struct complexe (W3/W4/W22)

`={{ JSON.stringify({…}) }}` inline → **"invalid syntax"**. Et `={"fields": {"qty": {{ $json.x }}}}` → **"Unmatched expression brackets: 1 opening, 2 closing"** (les `{}` JSON sont confondus avec les `{{}}` d'expression). Vécu 5+ fois.

**Pattern systématique éprouvé** :

```
[Webhook] → [Code node "Prep"] → [HTTP Request]
```

1. **Code node "Prep"** prépare la string body en JS pur :
   ```js
   const body = JSON.stringify({ fields: { qty: $json.body.qty, nom: $json.body.nom } });
   return [{ json: { insert_body: body } }];
   ```
2. **HTTP Request** consomme la string toute prête :
   ```
   jsonBody = ={{ $json.insert_body }}
   ```

> **`={{ $json.X }}` marche systématiquement.** C'est l'imbrication d'objets dans l'expression qui casse, pas l'expression elle-même.
> **W4 (cas plat seulement)** : `={"id": {{ id }}, "fields": {"x": "y"}}` passe pour des structs **plates sans string interpolée**. Au moindre doute → Code node Prep.

---

## Entrée du webhook (W1)

- POST : données dans **`$json.body.X`**.
- GET : query params dans **`$json.query.X`**.
- Jamais `$json.X` direct.
- **W19 — 🚨 pas de path params `:varname`** : `api/grades/:slug/mapping` est enregistré **littéralement** (tout autre URL → 404). Testé n8n 2.51.1. Workaround : `?slug=X` (query) ou body. (Re-tester en 2.56+.)

## Validation & erreurs (W10)

```js
// Dans un Code node : un rejet de validation doit faire un VRAI 500
if (!$json.body.motif) {
  throw new Error("motif requis");        // ✅ → HTTP 500 automatique
}
// ❌ return [{json:{_error:"…"}}]  → CONTINUE la chaîne et casse plus loin
```

## Jamais de branches parallèles (W9)

n8n CE n'a **pas** de merge/wait fiable. Un Code node "Build Response" en aval de branches parallèles s'exécute **avant** que les branches aient fini → `$('NodeName').first().json` = `undefined`.

➡️ Tout en **chaîne séquentielle** : `Fetch A → Fetch B → Fetch C → Build Response`. Légèrement plus lent, 100 % fiable.

**W24 — 🚨 un node qui renvoie 0 item ARRÊTE la chaîne** : les nodes suivants ne s'exécutent pas, `Respond` compris → le webhook répond **200 avec un corps vide**, sans erreur nulle part. C'est le piège qui pousse à remettre des IF (et donc du fan-in, W9). ➡️ Dans une chaîne linéaire, un Code node conditionnel doit **toujours émettre ≥ 1 item** : à défaut de travail, un **no-op inoffensif** (typiquement un GET `?fields=Id` sur le record déjà connu). Une ligne de commentaire suffit à ce que le prochain lecteur ne le « simplifie » pas.

```js
// rien à faire, mais la chaîne doit rester vivante (un node à 0 item tue le Respond)
if (!aFaire.length) {
  return [{ json: { noop: true, method: 'GET', body: '[]',
                    url: `${NOCO}/${TABLE}/records/${id}?fields=Id` } }];
}
```

## Pagination d'un HTTP node (W27/W28)

- **W27 — 🚨 un HTTP node paginé sort UN ITEM PAR PAGE** : le node suivant s'exécute une fois **par page** → duplication en aval (vécu 2026-08-19 : 84 entrées × 5 pages = 420 lignes d'export). Réflexes : le consommateur agrège TOUTES les pages (`$input.all().flatMap(p => p.json.records || [])`) et **déduplique par id** ; placer les fetchs multi-pages en **fin de chaîne** quand c'est possible ; dans l'URL d'un node situé APRÈS un node paginé, référencer `$('Prep')` **explicitement**, jamais `$json` (qui change à chaque item/page).
- **W28 — 🚨 un fetch « toute la table » NON paginé tronque en SILENCE au pageSize** : le jour où la table dépasse le cap, le dict bâti dessus ment — et une idempotence par clé naturelle (get-or-create, « existe déjà ? ») se met à créer des **doublons**. Si on choisit de ne pas paginer (légitime quand W27 mordrait la chaîne), poser une **garde bruyante** dans le consommateur : `if ((p.json.records || []).length >= PAGE_SIZE) throw new Error('fetch tronqué — paginer')`. 3 lignes, posées le 2026-08-19 sur un fetch à 235/1000 rows — la bombe désamorcée avant d'exploser.

## IF node (W5/W21)

- **W5** : `conditions.options = {version:2, leftValue:"", caseSensitive:true, typeValidation:"strict"}` est **obligatoire** (le validator MCP refuse sinon).
- **W21** : en `typeValidation:"strict"` + operator `string`, un `leftValue` numérique (ex. un id) → "Wrong type: '4' is a number…". Forcer en string : `={{ $json.x ? 'yes' : '' }}` ou `.toString()`.

## Method dynamique (upserts) (W17/W20)

```
method = ={{ $json.mode === 'insert' ? 'POST' : 'PATCH' }}
```
**W20** : ça déclenche un faux positif "Invalid value for method" en validation MCP **mais fonctionne au runtime**. Ignorer le warning, activer quand même.

### Le pattern qui en découle : « un item = un appel » (W17 poussé au bout)

Si `method`, `url` **et** `jsonBody` sont tous des expressions, **un seul** node HTTP exécute une **séquence d'appels hétérogènes** — un par item d'entrée. Un Code node en amont décide de la liste des appels ; le node HTTP ne fait que dérouler.

```js
// Code "Prep" : 1 item = 1 appel. Ordre garanti, un item de sortie par item d'entrée.
return calls.map(c => ({ json: c }));   // {quoi, method, url, body}
```
```
method = ={{ $json.method }}   url = ={{ $json.url }}   jsonBody = ={{ $json.body }}
```

Ça remplace des cascades d'IF (donc du fan-in W9) pour tout endpoint « pose N liens de natures différentes » : lien belongsTo + n lignes de jonction + batches m2m dans une seule chaîne linéaire. Quand une étape dépend du **retour** de la précédente (id d'insert), faire **deux tours** : `Prep → Appels tour 1 → Prep tour 2 → Appels tour 2`. ⚠️ Le remapping entre tours se fait **par index** (le node HTTP rend un item de sortie par item d'entrée, dans l'ordre) : vérifier `sorties.length === entrées.length` et **`throw`** sinon — un lien posé sur le mauvais parent ne se voit jamais.

---

## NocoDB depuis n8n (W6/W7/W8)

- **Credential natif** `nocoDbApiToken` (libellé "API Token"), **pas** `httpHeaderAuth` générique (W6).
- HTTP Request node (W7) :
  ```
  authentication      = predefinedCredentialType
  nodeCredentialType  = nocoDbApiToken
  ```
- **URL interne** (W8) : `http://nocodb:8080/api/v3/…` (réseau Docker), **jamais** l'URL publique (qui passe par Cloudflare Access → 302).

## Workflow → workflow comme building blocks (W23)

Un workflow public peut en appeler un autre via HTTP standard sur le réseau interne :

```
HTTP Request → http://n8n:5678/webhook/api/wms/piece/transfer
```

Plus simple que le node `Execute Workflow`, transparent au runtime, permet de réutiliser des endpoints comme briques (`/picking/assign` → `/piece/transfer` ; `/reparation/valider-panne` → `/mouvement-stock`). Latence +~100 ms/hop, acceptable en POC.

---

## Patterns d'architecture (P1/P2)

- **P1 — le front orchestre l'atomicité** : pour une opération multi-tables (commande+lignes, picking+mouvement), faire **N appels séquentiels côté front** avec gestion d'erreur, plutôt qu'un workflow géant atomique. Moins de risque, plus simple à débugger. (POC : la non-atomicité est acceptable.)
- **P2 — résolution FK côté front** : exposer `/{ressource}-detail?id=X` qui résout les FK d'**un** record (via `/links`, cf. `spark-nocodb-v3-patterns`). Le front fait `Promise.all` sur une liste. Tient jusqu'à ~50 records.

---

## Outils MCP n8n — réflexes (W12-W16, W18)

- **Valider avant d'activer** : `n8n_validate_workflow` (errors + warnings).
- **Activer** : `activateWorkflow` via `n8n_update_partial_workflow` (plus pratique que l'UI).
- **Débugger une erreur** : `n8n_executions action=get id=X includeData=true` → payload + erreur node par node. **Outil n°1.**
- **Copier un pattern existant** : `n8n_get_workflow`.
- **W18** : un `n8n_update_partial_workflow` est actif **immédiatement** (pas de reload webhook).
- **W22/N-MCP** : `patchNodeField` ne patche pas `parameters.assignments.assignments` (object) → `updateNode` avec l'array complet. `n8n_update_full_workflow` exige `name` (sinon 422).

---

## Patcher un workflow ACTIF par l'API REST v1 (W25/W26)

Sur un site sans MCP (le canal recommandé depuis 2026-07), on patche en `GET → modifier le JSON en Python → PUT {name, nodes, connections, settings}`. Deux pièges qui coûtent chacun une demi-heure, et le second a mis un endpoint de production par terre.

- **W25 — 🚨 un PUT qui change le GRAPHE n'est PAS pris en compte par l'instance active.** W18 (« actif immédiatement ») ne vaut que pour un changement de **paramètres**. Dès qu'on **ajoute des nodes ou des connexions**, l'instance active continue de servir **l'ancien graphe** : le GET de contrôle montre bien les nouveaux nodes et les bonnes connexions, mais l'exécution n'en enchaîne que les anciens — on croit à un bug de son propre code. ➡️ **`POST /activate` après `POST /deactivate`** à chaque changement de graphe. À mettre dans le script de patch, pas dans la tête.
- **W26 — 🚨 `nodeCredentialType` SEUL ne suffit pas sur un node créé par l'API.** Un node HTTP construit à la main avec `authentication: predefinedCredentialType` + `nodeCredentialType: 'nocoDbApiToken'` part en **`Credentials not found`** au runtime — l'objet `credentials` est un champ **à part**, absent par défaut. Le workflow paraît parfait à la relecture API. ➡️ **Recopier `credentials` d'un node voisin qui marche** : `{'nocoDbApiToken': {'id': '…', 'name': '…'}}`.

- **L'assertion post-PUT doit viser le CHAMP précis, pas le JSON entier** : `assert 'X' not in json.dumps(nodes)` échoue à tort si le **commentaire** posé par le patch mentionne X — vécu deux fois dans le même script le 2026-08-19. Asserter sur `node['parameters']['url']` ou le `jsCode` du node visé. Même classe de piège pour un **auditeur d'URLs** : parser les *expressions* (`={{ … }}`), pas la string brute — une regex qui tronque à la première quote a rendu ~50 % de faux positifs « fetch non borné » (les `where={{ encodeURIComponent(…) }}` invisibles).

**Le garde-fou qui rend ces deux pièges bénins** : sauvegarder le workflow (`GET` → fichier) **avant** le patch, et écrire le script de patch avec des **assertions sur l'état de départ** (nom du workflow, présence des nodes attendus, code exact des Code nodes touchés, connexions attendues) — puis `--dry-run` par défaut, `--execute` explicite. Un patch qui échoue bruyamment sur un état inattendu vaut mieux qu'un patch qui « marche » sur un workflow qui a bougé. Restaurer = un PUT du fichier de sauvegarde.

---

## Squelette d'un endpoint d'écriture type

```
[Webhook POST /api/wms/commande]
        │  $json.body.{…}
        ▼
[Code "Validate"]   throw Error si invalide (W10)
        ▼
[Code "Prep body"]  insert_body = JSON.stringify({fields:{…}})  (W3)
        ▼
[HTTP POST nocodb:8080 /records]   jsonBody = ={{ $json.insert_body }}  (W7/W8)
        ▼
[HTTP POST /links/{field}/{records[0].id}]   [{id: fk}]   (N3/N4)
        ▼
[Code "Build Response"]   séquentiel, pas de parallèle (W9)
```

> Charger **aussi** `spark-nocodb-v3-patterns` avant de construire : 90 % des bugs d'un endpoint Spark viennent de la couche NocoDB (N3, Links, bulk), pas de n8n.
