---
name: spark-stack-ops
description: Exploiter et diagnostiquer la stack Spark (Caddy, Docker/Colima, Cloudflare tunnel, Postgres, secrets). Use when editing docker-compose/Caddyfile/volumes, sizing Colima, managing the Cloudflare tunnel, rotating/handling secrets, resyncing a Postgres password, or diagnosing a container (crashloop, 502, "ma modif n'apparait pas", restart needed). Couvre NC_DB_JSON, bind mounts C1-C6, tunnel pattern A, hygiène secrets.
metadata:
  spark:
    layer: ops
    source: spark-pitfalls-catalog (C1-C6) + INCIDENTS 2026-05-05 + CLAUDE.md
    services: "Caddy, n8n, NocoDB, Postgres16, cloudflared"
---

# Exploitation de la stack Spark — ops & pièges

> Pour **opérer** la stack (pas pour modéliser des données ni construire des endpoints → voir `spark-nocodb-v3-patterns` / `spark-n8n-pseudo-api`).
> Guides de setup détaillés : `docs/caddy.md`, `docs/cloudflared.md`, `docs/cf-access.md`, `spark-kit/SECURITY.md`.

---

## 🚨 NocoDB : toujours `NC_DB_JSON`, jamais `NC_DB` (INC-2026-05-05)

`NC_DB="pg://…&p=XXX&d=…"` **URL-décode** le password. Si `NOCODB_DB_PASSWORD` (généré par `openssl rand -base64`) contient `&`, `=`, `+`, `%`… → **auth fail Postgres en boucle**, `nocodb` en `Restarting`, 502 public.

- **Toujours** `NC_DB_JSON` (objet JSON, pas de parsing URL).
- **Enforcer un alphabet secret URL-safe** à la génération (`spark-bootstrap.sh` / `setup-skeleton/00-generer-secrets-env`).
- **Resync d'un password driftié** : `init-db.sh` ne se rejoue pas après le 1er boot. Resynchroniser **depuis l'intérieur du container postgres** (jamais `source .env` côté host) :
  ```bash
  docker exec -it <prefix>-postgres-1 \
    psql -U postgres -c "ALTER USER nocodb WITH ENCRYPTED PASSWORD '…';"
  ```

---

## 🚨 Bind mounts Caddy — quand faut-il restart ? (C1/C2/C5)

| Action | Propagation | Commande |
|--------|-------------|----------|
| **C2** Ajout/modif d'un **fichier** dans un dossier déjà monté | immédiate | rien |
| **C1** `mv`/remplacement d'un **dossier** monté | ❌ NE propage PAS | `docker-compose -p <prefix> restart <service>` |
| **C5** Ajout d'un **nouveau volume bind** dans le compose | recrée le container | `docker-compose -p <prefix> up -d caddy` (coupure ~3-5 s sur **tous** les fronts) |

- **C1** : symptôme typique = « ma nouvelle page est invisible » après un `mv` de dossier. Restart obligatoire.
- **C5** : **prévenir les builders** avant (coupe WMS/CRM/GRADING/proxy n8n simultanément ~3-5 s).
- **C3** : convention — volumes front en **read-only** (`./front-wms:/srv/wms:ro`). Le container ne doit pas écrire dans le mount.

### Caddyfile : reload vs restart

```bash
# Édition du Caddyfile → reload SANS downtime (préféré)
docker-compose -p <prefix> exec caddy caddy reload --config /etc/caddy/Caddyfile
# Hard restart (rare, si reload ne suffit pas)
docker-compose -p <prefix> restart caddy
```

> Rappel archi : Caddy a `auto_https off` (Cloudflare gère le TLS) et injecte `X-Forwarded-Proto https` ; il bind **`127.0.0.1` uniquement** (jamais `0.0.0.0`). Détail : `docs/caddy.md`.

---

## Cloudflare tunnel — pattern A (local-managed)

- Le YAML vit **côté hôte** (`~/.cloudflared/config-*.yml`), édité par `scripts/tunnel-up.sh` / `tunnel-down.sh` via des blocs marqués `# >>> spark-begin` / `# <<< spark-end`.
- **Ne jamais éditer à la main** les blocs marqués (legacy `kyklos-begin`/`kyklos-end` à migrer si trouvés).
- Cycle : `tunnel-up.sh` après `docker-compose up` → ~10 s plus tard les sous-domaines répondent ; `tunnel-down.sh` au déprovisionnement. Détail : `docs/cloudflared.md`.

---

## Sizing Colima

- Mac dev **8 GB** unified → Colima **≤ 4 GiB** (incompatible avec un LLM local concurrent + la stack complète).
- Kit prod **16 GB+** → 6 GiB.
- Diagnostic mémoire : `docker run --rm alpine free -h`.

---

## Hygiène des secrets

- **Jamais** afficher la valeur d'un secret en sortie de tool/message — surtout `CF_API_TOKEN` (scope DNS), `N8N_ENCRYPTION_KEY`, `NOCODB_API_TOKEN` (très haut privilège). Vérifier **existence/longueur**, pas la valeur.
- `.env` est **gitignored** — vérifier `git status` avant tout commit.
- **Secrets métier** (API des logiciels du client) → **n8n > Settings > Credentials** (chiffrés par `N8N_ENCRYPTION_KEY`). Le `.env` ne porte que les secrets **d'infra**.
- En cas de casse NocoDB/n8n, **réparer par le chemin MCP/CLI sanctionné** (rotation PAT, restart container MCP), pas `docker exec psql` / `curl` brut / `cat .env` ad-hoc.

---

## Accès host aux vhosts derrière CF Access (sans Service Token)

Un script/CLI sur le Mac hôte qui vise un vhost public prendra un **302**. Préférer le **bypass Caddy local** :

```bash
# NocoDB via nocodb.sh (CLI de la skill `nocodb`)
export NOCODB_URL="http://127.0.0.1:<KYKLOS_HOST_HTTP_PORT>"   # ex. 18080
export NOCODB_HOST_HEADER="<prefix>-db.<domain>"               # Caddy route par Host header
```

Le trafic reste local (127.0.0.1), ne traverse jamais Cloudflare → insensible à CF Access, pas de token. (Service Token CF = uniquement off-host / audit du 302 public.) Détail : `docs/cf-access.md`.

### 🚨 C7 — sans en-tête `Host`, Caddy répond **200 avec un corps VIDE**

Caddy route **par Host header**. Une requête sur `http://127.0.0.1:<port>/…` **sans** cet en-tête ne tombe sur aucun vhost et rend un **200 vide** — pas un 404, pas une erreur.

Ce qui rend le piège coûteux, c'est la façon dont on « vérifie » :

```bash
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:18080/webhook/api/wms/pieces   # → 200 … et un corps vide
```

Un script de test qui ne regarde que le **code HTTP** conclut que l'API répond. Vécu 2026-08-05 : un E2E a rendu **29 assertions rouges** sans jamais atteindre n8n — une assertion rouge dit « le code est cassé », alors qu'elle disait en fait « ta requête n'est jamais arrivée ». Une demi-heure perdue à chercher dans le mauvais service.

➡️ **Deux réflexes** :
1. Vérifier un endpoint sur son **corps**, jamais sur son code HTTP seul.
2. Donner à tout script de test un **contrôle [0]** qui s'arrête net si l'API ne rend pas du JSON exploitable, en rappelant la commande correcte :

```bash
if ! call GET /pieces | python3 -c "import json,sys; sys.exit(0 if isinstance(json.load(sys.stdin).get('pieces'), list) else 1)" 2>/dev/null; then
  echo "✗ $API ne rend pas de JSON (302 CF Access, ou Caddy sans Host header ?)"
  echo "  WMS_API_BASE=http://127.0.0.1:18080/webhook/api/wms WMS_HOST_HEADER=<prefix>-app.<domain> bash $0"
  exit 2
fi
```

---

## n8n — accès fichiers du repo

Monter dans le path canonique **`/home/node/.n8n-files/*`** (le node "Read/Write Files from Disk" n'autorise que ça par défaut ; `N8N_RESTRICT_FILE_ACCESS_TO` n'est pas fiable) :

```yaml
volumes:
  - n8n_data:/home/node/.n8n
  - ../discovery:/home/node/.n8n-files/discovery:ro
```

---

## Commandes de diagnostic fréquentes

```bash
docker-compose -p <prefix> ps                  # état de la stack
docker logs <prefix>-<service>-1 --tail 50     # logs d'un service
docker-compose -p <prefix> restart <service>   # restart ciblé
docker run --rm alpine free -h                 # mémoire Colima
```

---

## Réflexe : rebuild > patcher un état cassé

Quand un schéma/état diverge trop (link fields incohérents, IDs driftés), un **full wipe + rebuild via seed idempotent** est souvent plus rapide et plus propre que de patcher. ➡️ Garder les **scripts de création reproductibles et idempotents** : c'est ce qui rend le rebuild sans douleur.

> Tout nouveau piège ops transverse : le promouvoir ici **et** dans `spark-kit/INCIDENTS.md`.
