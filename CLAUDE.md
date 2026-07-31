# CLAUDE.md — Guide agent pour un deploiement Spark

> Toi (Claude) qui debarques sur un repo Spark : voici l'essentiel avant d'ecrire ou modifier quoi que ce soit.

---

## ⚠️ REGLE D'OR — avant toute action technique

**Obligatoire — avant d'ecrire le moindre workflow, table NocoDB ou page front, charger les 2 skills coeur** (non negociable, c'est le harnais Spark) :

```
/skill spark-nocodb-v3-patterns  # OBLIGATOIRE — pieges data-layer NocoDB v3 (Links, Lookups, bulk)
/skill spark-n8n-pseudo-api      # OBLIGATOIRE — construire un endpoint webhook->NocoDB
```

**Au besoin (trigger situationnel), charger aussi** :

```
/skill spark-frontend-patterns   # si on touche une page front (-app)
/skill spark-stack-ops           # si on touche compose / Caddy / tunnel / secrets
/skill spark-poc-method          # au cadrage d'un POC (fiche, PRD)
/skill nocodb                    # reference API v3 + CLI nocodb.sh (a la demande)
/skill n8n-node-configuration    # config d'un node precis (a la demande)
```

Les deux skills coeur portent la connaissance empirique cristallisee sur les POCs (les ~30 pieges N/W). Elles sont **versionnees dans le kit** (`spark-kit/templates/skills/`), donc presentes sur toute machine ou les skills Spark sont installees. La cheat-sheet ci-dessous est la version **toujours-en-contexte** (le 20% qui evite 80% de la douleur) ; les skills coeur en sont le detail complet a charger en debut de session. Installation : voir `skills/README.md`.

### Cheat-sheet always-on — les ~15 pieges-rois

Distillation toujours-en-contexte (le detail vit dans les skills coeur). Si un seul de ces points est ignore, c'est 20-60 min de debug.

**NocoDB (data)**
- **N3/N4** : insert avec FK ≠ creation du lien (juste un compteur fantome). Apres l'insert, POST `/api/v3/data/{base}/{table}/links/{link_field_id}/{record_id}` body `[{id:X}]`.
- **N34** : POST `/links` **AJOUTE** un lien, il ne remplace jamais — **meme sur un `belongsTo`**. Le record se retrouve a 2 liens, le Lookup renvoie 2 valeurs, et tout lecteur en `[0]` voit l'ancienne. Re-liaison = GET liens actuels → DELETE ce qui n'est pas la cible → POST → **relire et verifier qu'il n'en reste qu'un**.
- **N5/N21/N26/N35** : lecture Links = compteurs, pas d'objets. Resoudre via `/links?fields=Id,nom,...` (sans `?fields=`, seul le display field revient ; sans `Id` dans le fields → `records:[]` vide silencieux). **N35** : le « compteur » n'est vrai qu'en fetch **liste** — en `GET /records/{id}` c'est un objet expanse (donc toujours truthy), `?fields=` n'y change rien. Une garde `if (champ || 0) !== 0` passe sur la liste et echoue toujours par id.
- **N7** : `?where=(champ_link,eq,X)` ne filtre PAS (renvoie 0). Passer par le `/links` inverse, ou un Link `belongsTo` denormalise direct.
- **N2** : bulk insert ET delete cap a **10**/call. Le delete >10 echoue **silencieusement** (0 supprime, aucune erreur). Batcher + verifier le retour.
- **N36** : un endpoint **get-or-create** (Find → IF → Insert) n'est **pas atomique**. Un node HTTP n8n itere sur ses items : N items visant la meme cle → N inserts → **doublons**. NocoDB n'a aucune contrainte d'unicite pour les refuser. **Dedupliquer par cle AVANT le fan-out**, puis re-etaler.
- **N25/N27** : Lookups de liens **differents** dans le meme `?fields=` s'ecrasent (`[null]`) → 1 fetch par **lien** (Lookups d'un meme lien = partageables). Lookup sur un lien **m2m** = null systematique → verifier `relation_type` avant de le creer.
- **N28** : GET `/records/{id}` sans `?fields=` = 10-35× plus lent (NocoDB resout toutes les expansions). `?fields=` systematique sur les GET par id.
- **N29** : Lookup mm dans un fetch **liste** = superlinéaire (150 rows=8s, 500=25s+ et Postgres a 190%). Inverser la requete : `/links` inverse du parent puis `where=(Id,in,...)`.
- **Wrapping** : reponse POST/PATCH = `{records:[{id,fields}]}`. En n8n : `{{ $json.records[0].id }}`, jamais `{{ $json.id }}`.

**n8n (backend)**
- **W3/W22** : jsonBody complexe inline → "invalid syntax" / "Unmatched expression brackets". Preparer la string dans un Code node "Prep", puis `jsonBody = ={{ $json.insert_body }}`.
- **W9** : **jamais de branches paralleles** (fan-in non fiable en CE → `undefined`). Tout en chaine sequentielle.
- **W10** : `throw new Error()` dans un Code node = HTTP 500. Un `return [{json:{_error}}]` CONTINUE la chaine et casse plus loin.
- **W1** : webhook = `$json.body.X` (POST) / `$json.query.X` (GET). Pas de path params `:slug` (pris litteralement).
- **W6/W8** : credential natif `nocoDbApiToken` (pas Header Auth) ; URL interne `http://nocodb:8080` / `http://n8n:5678` (jamais l'URL publique → 302 CF).

**Front**
- **F1** : ne JAMAIS nommer une fonction `confirm`/`alert`/`prompt`/`open`... (proprietes de `window`) → clic inerte, zero erreur console. Prefixer (`confirmX`).
- **CF Access** : appeler les webhooks en **relatif** (`/webhook/api/...`), jamais l'URL absolue n8n (cross-origin → 302). Pas d'iframe NocoDB.

**Caddy / Docker**
- **NC_DB_JSON jamais NC_DB** : `NC_DB` (URL) URL-decode le password → auth fail en boucle si caractere special. Utiliser `NC_DB_JSON` (objet).
- **C1/C5** : `mv` d'un dossier monte → `restart` necessaire ; nouveau volume bind → `up -d caddy` (coupe tous les fronts ~3-5s) ; ajout/modif de fichier → rien. **C6** : cache CF sur `.js` (~4h) → bumper `?v=N`.

**Skipper la regle d'or coute 2-3h de bugs evitables par session** (vecu sur le premier gros build WMS v2). Cette section est en haut volontairement.

Premiere installation sur une nouvelle machine ? → suivre `GETTING-STARTED.md`.

---

## Ce qu'est Spark

Spark est un **side-stack** : un Mac Mini pose a cote des systemes existants de l'entreprise. Il ne remplace rien. Le CRM reste. L'ERP reste. Le fichier Excel qui marche depuis 2012 reste. Spark les fait parler entre eux.

**La source de verite business est toujours le systeme metier de l'entreprise** (CRM, ERP, Google Sheets, facturation, WMS...). NocoDB n'est jamais la source de verite business — c'est un bac a sable, un staging, une surface pour des donnees qui n'existaient nulle part avant. n8n est un pont controle qui ouvre des portes choisies vers les sources metier.

---

## Vocabulaire critique

| Terme | Sens |
|-------|------|
| **Spark** | La kit / methode (org GitHub `spark-kit`). Pas un site specifique. |
| **Site** | Un deploiement Spark concret : 1 Mac Mini, 1 entreprise, 1 domaine, 1 repo. |
| **`SPARK_PREFIX`** | Slug par-site qui forme les hostnames (`<prefix>-<service>.<domain>`). Ce n'est PAS le nom de la kit. |
| **Playbook** | Brique d'integration assemblable (workflow n8n + tables NocoDB + config). |

---

## Stack technique

| Service | Role | Port interne |
|---------|------|-------------|
| n8n | Orchestration, workflows | 5678 |
| NocoDB | Base visuelle, ecrans metier | 8080 |
| PostgreSQL 16 | Base relationnelle partagee (users separes : `n8n`, `nocodb`) | 5432 |
| Caddy | Reverse proxy, route le trafic | 80 (127.0.0.1 uniquement) |
| cloudflared | Tunnel Cloudflare (TLS, acces distant) | host, pas Docker |

Caddy a `auto_https off` — c'est Cloudflare qui gere le TLS. Caddy injecte `X-Forwarded-Proto https` pour que les apps croient etre en HTTPS.

---

## Outillage agent : skills + acces live

Deux outillages distincts. Pour **n8n** : skills (reference) + MCP (action live). Pour **NocoDB** : skill (reference) + **CLI v3** (action live, pas de MCP).

Les skills se repartissent en **deux familles** : le **socle generique tiers** (`nocodb` + 7 `n8n-*` — reference des API et des nodes, detaille ci-dessous) et la **couche Spark** (`spark-*` — pieges empiriques + patterns d'assemblage du combo NocoDB/n8n/Caddy). Les 2 skills coeur `spark-nocodb-v3-patterns` + `spark-n8n-pseudo-api` sont a charger **avant d'ecrire** (cf. Regle d'or) ; `spark-frontend-patterns` / `spark-stack-ops` / `spark-poc-method` se chargent par trigger. Source versionnee dans `spark-kit/templates/skills/` (cf. son `README.md`).

### n8n

**Skills** (7 skills `n8n-*`) :
- `n8n-node-configuration` — configuration des nodes par type et operation
- `n8n-workflow-patterns` — patterns architecturaux prouves (webhook, CRUD, scheduling, AI agents)
- `n8n-expression-syntax` — syntaxe `{{ }}`, `$json`, `$node`, troubleshooting expressions
- `n8n-code-javascript` — Code nodes JS, `$input`, `$helpers`, `DateTime`
- `n8n-code-python` — Code nodes Python, `_input`, `_json`, limitations
- `n8n-validation-expert` — interpretation des erreurs de validation
- `n8n-mcp-tools-expert` — guide d'utilisation des outils MCP n8n

**MCP** (`n8n-mcp`) :
- Lecture/ecriture de workflows, activation, executions
- Se connecte a `http://n8n:5678` en interne Docker
- Script wrapper : `infra/scripts/mcp-n8n.sh`

### NocoDB

**Skill** (`nocodb`) :
- Reference API v3 complete : data CRUD, meta management, links, filters, sorts, attachments
- Inclut le CLI `nocodb.sh` — c'est l'outil d'**action** pour NocoDB
- Syntaxe des filtres `where` : `(field,op,value)~and(field2,op2,value2)`

**Acces live : API v3 via CLI `nocodb.sh`** (pas de MCP). L'ecosysteme MCP NocoDB n'est pas stable contre les versions recentes (`2026.04.5+`) — le package historique `@andrewlwn77/nocodb-mcp@0.2.2` cible v1/v2 que NocoDB rejette pour les PAT. Cf. INC-2026-05-19 dans `spark-kit/INCIDENTS.md`. La stack n'integre donc pas de MCP NocoDB ; on s'en tient au CLI tant qu'un package compatible v3 n'a pas ete valide.

Utilisation type :
```bash
set -a; source infra/.env; set +a
export NOCODB_TOKEN="$NOCODB_API_TOKEN"
export NOCODB_URL="http://127.0.0.1:${SPARK_HOST_HTTP_PORT:-18080}"
export NOCODB_HOST_HEADER="${SPARK_PREFIX}-db.${SPARK_DOMAIN}"
bash ~/.claude/skills/nocodb/scripts/nocodb.sh table:list <base>
unset NOCODB_TOKEN NOCODB_API_TOKEN
```

Le CLI lit le token via env, hit `/api/v3/...` avec le header `xc-token`, n'expose jamais le secret en sortie.

### Regles

1. **n8n : ne jamais `curl` quand le MCP n8n peut le faire.** Le MCP gere auth, pagination, format.
2. **NocoDB : toujours passer par `nocodb.sh`.** Pas de curl ad-hoc, pas de docker exec psql, pas de lecture brute de `.env` — le CLI est l'outil sanctionne.
3. **Skills = reference, MCP/CLI = action.** Consulter la skill pour comprendre la syntaxe, utiliser le MCP n8n ou le CLI NocoDB pour executer.
4. **Charger la skill avant de configurer un node.** `n8n-node-configuration` donne les champs requis par operation — evite les allers-retours.
5. **Valider avec `n8n-validation-expert`** apres avoir modifie un workflow.

---

## Configuration Claude Code

Le repo entreprise contient un `.mcp.json` (gitignored) a la racine qui pointe vers le wrapper n8n :

```json
{
  "mcpServers": {
    "n8n-mcp": {
      "command": "bash",
      "args": ["infra/scripts/mcp-n8n.sh"]
    }
  }
}
```

Le script source `infra/.env` et lance `docker run --network spark_spark` en mode stdio (`name: spark` dans le compose → reseau `spark_spark`). Au demarrage d'une session, le MCP n8n apparait automatiquement comme tool provider. NocoDB s'utilise via le CLI `nocodb.sh` de la skill (cf. plus haut).

### Installation des skills

Les skills sont globales (une seule fois par poste, dans `~/.claude/skills/`). Elles sont embarquees dans le repo templates :

```bash
cd ~/spark/templates/skills
cp -R nocodb spark-* ~/.claude/skills/
```

Les 7 skills n8n (`n8n-workflow-patterns`, `n8n-node-configuration`, etc.) sont fournies automatiquement par le serveur MCP n8n — pas besoin de les installer separement.

### Obtention des cles API

Apres le premier acces aux apps :
1. **n8n** : Settings > API > Create API Key → `N8N_API_KEY` dans `.env`
2. **NocoDB** : Team & Settings > Tokens > Add New Token → `NOCODB_API_TOKEN` dans `.env` (PAT `nc_pat_...`)
3. Relancer `docker-compose up -d` pour que le MCP n8n recharge `N8N_API_KEY`. Le token NocoDB est lu depuis `.env` par le CLI au runtime (pas besoin de restart).

---

## Principes de travail

### Donnees

- NocoDB = bac a sable. Le systeme metier = source de verite.
- Quand une donnee existe deja dans un legacy, NocoDB en est le **cache/staging**.
- Quand une donnee n'existait nulle part (ex: WMS d'un atelier papier), NocoDB devient la **source pour cette donnee precise**.

### Secrets

- Pas de credential en clair dans le repo — jamais.
- `.env` est gitignored. Verifier avec `git status` avant tout commit.
- Les secrets metier (API keys, tokens des logiciels de l'entreprise) vont dans **n8n > Settings > Credentials**, chiffres par `N8N_ENCRYPTION_KEY`.
- Le `.env` ne contient que les secrets d'infrastructure de la stack.

### Securite de l'exposition externe

Tout site Spark expose 3 sous-domaines via Cloudflare Tunnel (`<prefix>-n8n`, `<prefix>-app`, `<prefix>-db`). Le standard de durcissement par defaut :

- **Cloudflare Access devant tout vhost UI / app / webhook** (pattern Entra ID M365 ou Google OAuth, Free tier jusqu'a 50 users). Aucune page login native n8n/NocoDB ne doit etre joignable directement depuis Internet.
- **Headers de securite Caddy** (HSTS, X-Frame-Options, etc.) sur **chaque** vhost — bloc copiable dans `spark-kit/SECURITY.md` §2.2.
- **CORS NocoDB** restreint au sous-domaine `-app` (NocoDB renvoie `*` par defaut, l'override est cote Caddy).
- **Caddy bind `127.0.0.1`** uniquement (pas `0.0.0.0`) pour eviter l'exposition LAN.
- **Pas de Uptime Kuma dans le compose client** — monitoring vit sur un master Spark separe.

Hygiene speciale : `CF_API_TOKEN` (scope DNS), `N8N_ENCRYPTION_KEY` et `NOCODB_API_TOKEN` sont des secrets de tres haut privilege. Ne **jamais** afficher leur valeur en sortie de tool/message. Cf. `spark-kit/SECURITY.md` §5.

Detail complet, modele de menace, recettes copy-paste et procedure d'audit recurrent : **`spark-kit/SECURITY.md`**.

### Workflows n8n

- Un workflow = une responsabilite claire.
- Nommer les workflows avec un prefixe explicite : `[SYNC] CRM → NocoDB`, `[ALERT] Stock bas`, `[WEBHOOK] Commande recue`.
- Tester les workflows en mode manuel avant de les activer.
- Les credentials sont references par nom, pas par ID — facilite la portabilite.

### Modifications infra

- Travailler dans `infra/`.
- Tester localement (`docker-compose up -d`) avant de committer.
- Commit avec prefixe : `infra: ...`, `workflow: ...`, `discovery: ...`.

---

## Structure d'un repo entreprise

```
<entreprise>/
├── infra/
│   ├── .env                  secrets (gitignored)
│   ├── .env.example          template sans secrets
│   ├── docker-compose.yml
│   ├── config/
│   │   ├── Caddyfile
│   │   └── postgres/init-db.sh
│   ├── apps/                    apps metier statiques (servies par Caddy sur -app)
│   └── scripts/
│       ├── tunnel-up.sh      creation routes + CNAMEs CF
│       ├── tunnel-down.sh    suppression routes + CNAMEs
│       └── mcp-n8n.sh        wrapper MCP n8n
├── discovery/
│   ├── onboarding/           questionnaires entreprise
│   ├── fiches/               fiches-logiciel legacy
│   └── prds/                 PRDs des POCs
├── LESSONS-LEARNED.md        notes operationnelles
├── CLAUDE.md                 ce fichier, adapte a l'entreprise
└── .mcp.json                 config MCP (gitignored)
```

---

## Pieges connus

### NocoDB — toujours `NC_DB_JSON`, jamais `NC_DB`

La conf URL `NC_DB="pg://...&p=XXX&d=..."` URL-decode le password. Si le password contient `&`, `=`, `+`, `%` → auth fail en boucle. Utiliser `NC_DB_JSON` (objet JSON, pas de parsing URL).

### Sizing Colima

4 GiB par defaut. Si la stack devient lourde ou si des LLMs locaux tournent en parallele → 6+ GiB, mais seulement sur Mac 16 GB+. Diagnostic memoire : `docker run --rm alpine free -h`.

### Tunnel Cloudflare — pattern A

Le YAML cloudflared vit cote hote (`~/.cloudflared/config-*.yml`), edite par `scripts/tunnel-up.sh` via blocs marques `# >>> spark-begin` / `# <<< spark-end`. Ne pas editer manuellement les blocs marques.

### Endpoints derriere CF Access — ne pas pointer un script au mauvais endroit

Les 3 vhosts publics passent derriere Cloudflare Access (cf. "Securite de l'exposition externe"). Un appel anonyme recoit un `302` vers le login CF. Donc, pour tout nouveau front ou script :

- **Fronts** : appeler les webhooks en **relatif** (`/webhook/api/...`), jamais l'URL absolue `https://<prefix>-n8n.<domain>/...`. Front + API sont same-origin sur `<prefix>-app` → le cookie CF se propage aux XHR. Hardcoder l'hote n8n comme base d'API d'un front est un anti-pattern. Pas d'iframe NocoDB non plus.
- **Scripts / tooling host** (validation, cron, CLI) qui visent un vhost public → ils prendront un `302` sans auth. Trois voies, par ordre de preference :
  1. **Caddy local + Host header** (prefere sur le Mac hote) : `http://127.0.0.1:<port-http-local>` + en-tete `Host: <prefix>-<svc>.<domain>`. Reste en local, ne traverse jamais Cloudflare → pas de token. `nocodb.sh` : passer le header via `NOCODB_HOST_HEADER`.
  2. **Reseau Docker interne** (si le script tourne dans la stack) : viser `http://n8n:5678` / `http://nocodb:8080`.
  3. **Service Token CF** (seulement off-host / audit du 302 public) : en-tetes `CF-Access-Client-Id` / `CF-Access-Client-Secret`.
- **MCP n8n** : non concerne (reseau Docker interne).
- Runbook complet (creation app Access, Service Tokens, verif) : `docs/cf-access.md`.

### NocoDB v3 — vues custom = Enterprise

Sur NocoDB CE (le cas par defaut auto-heberge), l'API v3 des vues (`/api/v3/meta/bases/.../tables/.../views`) renvoie `ERR_LICENSE_REQUIRED`. Le CLI v3 ne peut donc pas creer/lister de vues Kanban / Calendar / Gallery / Form en programmatique. Workarounds : (a) creer les vues via l'UI NocoDB a la main, ou (b) se contenter de la grille par defaut. Les autres operations meta (tables, fields) marchent normalement sur CE.

### n8n — credential NocoDB : utiliser `nocoDbApiToken`, pas Header Auth

Quand un workflow n8n appelle l'API NocoDB via HTTP Request node, configurer le credential en type natif `nocoDbApiToken` (libelle "API Token" dans l'UI n8n). Ne pas creer un Header Auth generique avec `xc-token` — c'est techniquement equivalent mais moins propre et rejete par certains endpoints proteges.

### n8n — lecture de fichiers du repo via volume mount

Le node "Read/Write Files from Disk" n'autorise par defaut que les chemins sous `/home/node/.n8n-files/`. Pour qu'un workflow puisse lire des fichiers du repo (PRDs MD, configs...), monter dans ce path :

```yaml
# infra/docker-compose.yml, service n8n
volumes:
  - n8n_data:/home/node/.n8n
  - ../discovery:/home/node/.n8n-files/discovery:ro
```

Le `:ro` rend la lecture seule (recommande). Setter `N8N_RESTRICT_FILE_ACCESS_TO` pour etendre la whitelist est theoriquement possible mais n8n applique parfois encore la restriction par defaut — preferer le mount dans `/home/node/.n8n-files/*`.

---

## Liens

- Meta Spark : https://github.com/spark-kit/spark-kit
- Templates (gabarits methodologie) : https://github.com/spark-kit/templates
- Wiki Spark (manifeste, archi, concurrence) : `<spark-vault>/wiki/`
