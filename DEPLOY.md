# DEPLOY.md — Operational Runbook

## Trigger

Le déploiement se déclenche **automatiquement** sur un **push vers `main`**, après que les jobs `test` et `build` ont réussi.

- Une **Pull Request** ne déclenche **jamais** de déploiement (condition `github.event_name == 'push' && github.ref == 'refs/heads/main'`).
- Un **échec** du job `test` **bloque** les jobs `build` et `deploy` (safety gate via `needs:`).

## Target

- **Platform** : Render
- **Service** : `hbtn-devops-pipeline-lab:latest` (Web Service, région Frankfurt)
- **Public URL** : `https://hbtn-devops-pipeline-lab-latest-w14i.onrender.com`
- **Image** : `ghcr.io/madi-spec49/hbtn-devops-pipeline-lab:latest`

> **Note** : la base PostgreSQL est hébergée sur **Neon** (au lieu de Render) en raison d'une limitation du free tier Render qui bloquait la création d'une seconde base gratuite. Le Web Service Render s'y connecte via la variable d'environnement `DATABASE_URL`. L'application reste inchangée — Node.js utilise simplement la connection string fournie.

## Database Configuration

- **Type** : PostgreSQL managée (Neon, région eu-central-1 / Frankfurt)
- **Connection** : injectée dans le Web Service Render via la variable `DATABASE_URL`
- **Migrations** : appliquées automatiquement au démarrage du service (`npm run migrate`)
- **Seed** : `002_seed_items.sql` insère 2 items (Alpha Item, Beta Item) utilisés par `/items`

## Verification

Le job `deploy` vérifie **deux** endpoints avec un retry borné :

| Endpoint | Vérifie | Attendu |
|---|---|---|
| `/health` | Process liveness | HTTP 200 |
| `/items` | DB-backed behavior (Postgres répond) | HTTP 200 |

**Retry** : 30 tentatives × 10 s = 5 minutes maximum. Si les deux endpoints ne répondent pas 200 dans ce délai, le job échoue (`exit 1`).

Vérification manuelle indépendante :

```bash
STAGING_URL="https://hbtn-devops-pipeline-lab-latest-w14i.onrender.com"
curl -s -o /dev/null -w "%{http_code}\n" "$STAGING_URL/health"   # attendu : 200
curl -s -o /dev/null -w "%{http_code}\n" "$STAGING_URL/items"    # attendu : 200