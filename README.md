# dpod-ameya-community

Docker Compose deployment for the DPOD (Ameya) community stack. A single
`docker compose up` brings up the frontend, auth/authorization, backend +
Celery, the document-extraction service, and all supporting infrastructure
(DynamoDB, MongoDB, MinIO, Redis, RabbitMQ) behind a Caddy reverse proxy on
`http://localhost:8080`.

## Architecture

Everything is fronted by **Caddy** on port `8080`, which routes one subdomain
per service (no path prefixes):

| URL                                              | Service                       |
| ------------------------------------------------ | ----------------------------- |
| `http://localhost:8080`                          | Frontend SPA (ameya-web)      |
| `http://auth.localhost:8080`                     | Authentication service        |
| `http://api.localhost:8080`                      | Backend API                   |
| `http://extraction.localhost:8080`              | Extraction service            |
| `http://plugin.localhost:8080`                   | Training / schema-builder UI  |
| `http://minio.localhost:8080`                    | MinIO S3 API                  |
| `http://mongoui.localhost:8080`                  | Mongo Express (DB UI)         |
| `http://dynamoadmin.localhost:8080`             | DynamoDB Admin UI             |

The browser must resolve `*.localhost` to `127.0.0.1`. Chrome and Firefox do
this automatically; **Safari does not** — see [Host resolution](#host-resolution).

## Prerequisites

- **Docker** and **Docker Compose v2** (`docker compose`, not `docker-compose`).
- LLM API key(s) — the extraction service defaults to Google Gemini
  (`gemini-2.5-flash`), so a `GEMINI_API_KEY` is required for extraction to work.
- License credentials for DPOD (`LICENSE_TOKEN`, `LICENSE_PUBLIC_KEY`,
  `ROOT_USER_TOKEN`).

## Deployment

### 1. Configure environment

Copy the example env file and fill in real values:

```bash
cp .env.example .env
```

Edit `.env` and set:

| Variable             | Description                                        |
| -------------------- | -------------------------------------------------- |
| `LICENSE_TOKEN`      | DPOD license token                                 |
| `LICENSE_PUBLIC_KEY` | DPOD license public key                            |
| `ROOT_USER_TOKEN`    | Root/admin user bootstrap token                    |
| `GEMINI_API_KEY`     | Google Gemini API key (default extraction LLM)     |
| `OPENAI_API_KEY`     | OpenAI key (only if switching LLM provider)        |

> The compose file reads `.env` automatically. Infrastructure credentials
> (MinIO, DynamoDB, Mongo, RabbitMQ) are baked into `docker-compose.yaml` with
> local-development defaults — change them before any non-local use.

### 2. Create the external network

The stack attaches to an **external** Docker network named
`ameya_community_network`. Compose will not create it for you — create it once:

```bash
docker network create ameya_community_network
```

### 3. Start the stack

```bash
docker compose up -d
```

On first run this pulls all images, then `dynamodb-init` runs once to create
the DynamoDB table and the MinIO `dpod-aws-s3` bucket. Services start in
dependency order and wait on healthchecks, so the full stack may take a couple
of minutes to become ready.

Watch progress with:

```bash
docker compose ps
docker compose logs -f
```

### 4. Verify

Open **http://localhost:8080** in Chrome or Firefox. If the SPA loads and can
reach the API, the deployment is up.

## Host resolution

Chrome and Firefox resolve `*.localhost` → `127.0.0.1` automatically (RFC 6761).
**Safari and some other clients do not.** Add these entries to `/etc/hosts`:

```
127.0.0.1  auth.localhost api.localhost extraction.localhost plugin.localhost
127.0.0.1  minio.localhost mongoui.localhost dynamoadmin.localhost
```

## Common operations

```bash
# View logs for one service
docker compose logs -f ameya-backend-service

# Restart a single service
docker compose restart ameya-backend-service

# Pull updated images and recreate
docker compose pull && docker compose up -d

# Stop the stack (keeps volumes/data)
docker compose down

# Stop AND delete all data volumes (full reset)
docker compose down -v
```

## Configuration notes

- **LLM provider** — Extraction defaults to Google Gemini. To switch providers,
  change `LLM_MODEL` / `LLM_PROVIDER` in the `x-extraction-env` block of
  `docker-compose.yaml` and supply the matching API key.
- **MinIO / presigned URLs** — The backend presigns S3 URLs against
  `http://minio.localhost:8080`. S3 SigV4 signs the Host header, so this host
  must match exactly between the browser and the backend; do not change it
  without updating both `S3_ENDPOINT_URL` and the Caddy route.
- **CORS** — The SPA calls the API subdomains cross-origin. Caddy adds uniform
  CORS headers on the API hosts (see `Caddyfile`); no per-service CORS config is
  needed.
- **Persistent data** — All stateful data lives in named volumes
  (`mongodb_data`, `dynamodb-data`, `minio-data`, `redis-data`, etc.).
  `docker compose down` preserves them; `-v` deletes them.

## Troubleshooting

| Symptom                                   | Likely cause / fix                                                        |
| ----------------------------------------- | ------------------------------------------------------------------------- |
| `network ameya_community_network not found`| Run step 2: `docker network create ameya_community_network`.              |
| SPA loads but API calls fail in Safari    | Add the `*.localhost` entries to `/etc/hosts` (see Host resolution).      |
| Extraction jobs never complete            | Missing/invalid `GEMINI_API_KEY`; check `docker compose logs extraction-service`. |
| A service is stuck `unhealthy`            | Inspect its logs; dependents wait on healthchecks and won't start.        |
