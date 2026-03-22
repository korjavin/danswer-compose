# danswer-compose

Git-ops deployment of [Danswer/Onyx](https://github.com/danswer-ai/danswer) — an AI-powered enterprise search and Q&A platform — via Portainer with Traefik reverse proxy and Authentik OIDC.

## Architecture

| Service | Image | Role |
|---------|-------|------|
| `danswer-web` | `onyx-web-server` | Next.js frontend (port 3000) |
| `danswer-api` | `onyx-backend` | FastAPI backend (port 8080) |
| `danswer-background` | `onyx-backend` | Async workers / connectors (supervisord) |
| `danswer-inference-model` | `onyx-model-server` | Inference model serving |
| `danswer-indexing-model` | `onyx-model-server` | Indexing model serving |
| `danswer-db` | `postgres:15.2-alpine` | Relational data |
| `danswer-index` | `vespaengine/vespa` | Vector search engine |
| `danswer-cache` | `redis:7.4-alpine` | Session cache |
| `danswer-minio` | `minio/minio` | S3-compatible object storage |

Traefik routes:
- `danswer.yourdomain.com` → `danswer-web:3000` (priority 1)
- `danswer.yourdomain.com/api` → `danswer-api:8080` (priority 10)

## Prerequisites

- Traefik running and connected to the Docker network specified in `TRAEFIK_NETWORK_NAME`
- Portainer CE/EE
- Authentik (or other OIDC provider) with a Danswer application configured

## Authentik OIDC Setup

1. In Authentik: **Applications → Create** → type `OAuth2/OIDC`
2. Set redirect URI: `https://danswer.yourdomain.com/auth/oidc/callback`
3. Copy the **Client ID** and **Client Secret** → use in env vars below
4. Your `OPENID_CONFIG_URL` will be:
   `https://auth.yourdomain.com/application/o/<app-slug>/.well-known/openid-configuration`

## Portainer Setup

1. **Create stack** in Portainer:
   - Repository URL: `https://github.com/korjavin/danswer-compose`
   - Branch: `deploy` ← **IMPORTANT: not master**
   - Compose path: `docker-compose.yml`

2. **Set environment variables** — copy all values from `.env.example`, filling in:
   - `SERVICE_HOST` — your domain
   - `POSTGRES_PASSWORD` — strong random password
   - `MINIO_ROOT_PASSWORD` — strong random password
   - `DANSWER_SECRET` — `openssl rand -hex 32`
   - `ENCRYPTION_KEY_SECRET` — `openssl rand -hex 32`
   - `OAUTH_CLIENT_ID` / `OAUTH_CLIENT_SECRET` / `OPENID_CONFIG_URL` — from Authentik

3. **Enable webhook** in Portainer stack settings → copy the webhook URL

4. **Add GitHub secret**:
   Settings → Secrets → Actions → New repository secret
   - Name: `PORTAINER_REDEPLOY_HOOK`
   - Value: webhook URL from step 3

## Deployment

Push to `master` → GitHub Actions creates/updates `deploy` branch → Portainer reloads the stack.

Manual trigger: Actions tab → "Deploy Danswer Stack" → Run workflow.

## GHCR Image Vendoring

The `vendor-images.yml` workflow mirrors upstream images to GHCR weekly:

| Vendored image | Upstream |
|----------------|----------|
| `ghcr.io/korjavin/danswer-backend-vendor` | `onyxdotapp/onyx-backend` |
| `ghcr.io/korjavin/danswer-web-vendor` | `onyxdotapp/onyx-web-server` |
| `ghcr.io/korjavin/danswer-model-vendor` | `onyxdotapp/onyx-model-server` |
| `ghcr.io/korjavin/danswer-vespa-vendor` | `vespaengine/vespa:8.609.39` |

After the first vendor run, set package visibility:
GitHub → Packages → select package → Settings → Visibility (private recommended)

To pin a specific digest, set in Portainer env:
```
API_IMAGE=ghcr.io/korjavin/danswer-backend-vendor@sha256:<digest>
```
(Digests are logged in the Actions run output after each vendor job.)

## Environment Variables Reference

| Variable | Required | Description |
|----------|----------|-------------|
| `SERVICE_HOST` | ✅ | Public domain, e.g. `danswer.example.com` |
| `POSTGRES_PASSWORD` | ✅ | PostgreSQL password |
| `MINIO_ROOT_PASSWORD` | ✅ | MinIO root password |
| `DANSWER_SECRET` | ✅ | App secret key (`openssl rand -hex 32`) |
| `ENCRYPTION_KEY_SECRET` | ✅ | Encryption key (`openssl rand -hex 32`) |
| `OAUTH_CLIENT_ID` | ✅ | Authentik OIDC client ID |
| `OAUTH_CLIENT_SECRET` | ✅ | Authentik OIDC client secret |
| `OPENID_CONFIG_URL` | ✅ | OIDC discovery endpoint URL |
| `AUTH_TYPE` | — | `oidc` (default), `basic`, `disabled` |
| `LOG_LEVEL` | — | `info` (default), `debug`, `warning` |
| `FILE_STORE_BACKEND` | — | `s3` (default via MinIO) |
| `S3_BUCKET_NAME` | — | MinIO bucket name (default: `danswer-files`) |
| `TRAEFIK_NETWORK_NAME` | — | Traefik Docker network (default: `traefik_default`) |
| `TRAEFIK_CERTRESOLVER` | — | Traefik cert resolver name (default: `myresolver`) |

## Connecting to Outline and Mattermost

Connectors are configured via the Danswer web UI after deployment:

**Outline:**
1. Danswer UI → Admin → Connectors → Outline
2. You'll need an Outline API token: Outline Settings → API → Create token

**Mattermost:**
1. Danswer UI → Admin → Connectors → Mattermost
2. Create a Mattermost bot account and copy the token

## Resource Requirements

Minimum: 4 vCPU, 10 GB RAM, 32 GB disk
Recommended: 8+ vCPU, 16+ GB RAM, 100 GB+ disk

Per 1 GB of indexed documents: ~3 GB RAM, ~0.5 vCPU additional.

## References

- [Danswer/Onyx GitHub](https://github.com/danswer-ai/danswer)
- [Onyx Deployment Docs](https://docs.onyx.app/deployment/local/docker)
- [Onyx Configuration Reference](https://docs.onyx.app/deployment/configuration/configuration)
- [Authentik OIDC Setup](https://docs.goauthentik.io/docs/providers/oauth2/)
