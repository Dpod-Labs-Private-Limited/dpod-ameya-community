# dpod-ameya-community

Docker Compose deployment for the DPOD (Ameya) community stack. A single
`docker compose up` brings up the frontend, auth/authorization, backend +
Celery, the document-extraction subsystem, the analytics subsystem, and all
supporting infrastructure (DynamoDB, MongoDB, MinIO, Redis, RabbitMQ) behind a
Caddy reverse proxy on `http://localhost:8080`.

## Architecture

Everything is fronted by **Caddy** on port `8080`. Most services get their own
hostname; the extraction and analytics APIs share the `api.localhost` host and
are separated by **path**:

| URL                                                    | Routes to                     |
| ------------------------------------------------------ | ----------------------------- |
| `http://localhost:8080`                                | Frontend SPA (ameya-web)      |
| `http://auth.localhost:8080`                           | Authentication service        |
| `http://backend.localhost:8080`                        | Backend API                   |
| `http://api.localhost:8080/entityextractionagent`      | Extraction service            |
| `http://api.localhost:8080/ameya/analytics/v1`         | Analytics API                 |
| `http://plugin.localhost:8080`                         | Training / schema-builder UI  |
| `http://monitoring.localhost:8080`                     | Monitoring service            |
| `http://minio.localhost:8080`                          | MinIO S3 API                  |
| `http://mongoui.localhost:8080`                        | Mongo Express (DB UI)         |
| `http://dynamoadmin.localhost:8080`                    | DynamoDB Admin UI             |

The browser must resolve `*.localhost` to `127.0.0.1`. Chrome and Firefox do
this automatically; **Safari does not** — see [Host resolution](#host-resolution).

### Services

- **Application** — `ameya-web-service` (frontend), `ameya-authentication-service`,
  `ameya-authorization-service` (gRPC), `ameya-backend-service`,
  `ameya-celery-worker`.
- **Extraction** — `extraction-service`, `extraction-celery-worker`,
  `extraction-rabbitmq`, `training_plugin`.
- **Analytics** — `analytics-api`, `analytics-celery-beat`,
  `analytics-celery-worker`, `analytics-rabbitmq`, `analytics-mongo-server`,
  `analytics-mongo-consumer`, `analytics-excel-server`, `analytics-excel-consumer`,
  `analytics-file-server`, `analytics-file-consumer`.
- **Parser** — PDF/document parsing: `parser-api`, `parser-consumer`,
  `parser-rabbitmq`.
- **Monitoring** — `monitoring-service`, `minio-monitor`, and the one-shot
  `minio-bootstrap` job that wires MinIO bucket events to the monitor.
- **Infrastructure** — `dynamodb` (+ `dynamodb-admin`, `dynamodb-init`),
  `mongodb` (+ `mongo-express`), `minio`, `ameya-redis`, `ameya-rabbitmq`,
  `caddy`.

## Prerequisites

- **Docker** and **Docker Compose v2** (`docker compose`, not `docker-compose`).
- LLM API key(s):
  - `GEMINI_API_KEY` — the extraction service defaults to Google Gemini
    (`gemini-2.5-flash`), so this is required for extraction.
  - `OPENAI_API_KEY` — the analytics subsystem uses OpenAI embeddings
    (`text-embedding-3-small`), so this is required for analytics.
- License credentials for DPOD (`LICENSE_TOKEN`, `LICENSE_PUBLIC_KEY`,
  `ROOT_USER_TOKEN`).

> The `dpod-labs-private-limited` images are published on a **public** GHCR
> registry — no `docker login` is needed to pull them.

## Deployment

### 1. Configure environment

Copy the example env file and fill in real values:

```bash
cp .env.example .env
```

Set the following in `.env`:

| Variable             | Description                                        |
| -------------------- | -------------------------------------------------- |
| `LICENSE_TOKEN`      | DPOD license token                                 |
| `LICENSE_PUBLIC_KEY` | DPOD license public key                            |
| `ROOT_USER_TOKEN`    | Root/admin user bootstrap token                    |
| `GEMINI_API_KEY`     | Google Gemini API key (extraction LLM)             |
| `OPENAI_API_KEY`     | OpenAI API key (analytics embeddings)              |

> Infrastructure credentials (MinIO, DynamoDB, Mongo, RabbitMQ) are baked into
> `docker-compose.yaml` with local-development defaults — change them before any
> non-local use.

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

On first run this pulls all images, then two one-shot jobs run: `dynamodb-init`
creates the DynamoDB table and the MinIO `dpod-aws-s3` bucket, and
`minio-bootstrap` registers the MinIO webhook and wires bucket events to the
monitoring service. Services start in dependency order and wait on healthchecks
— note the backend (`start_period` 5 min) and analytics API (`start_period`
~8 min) warm up slowly, so the full stack can take several minutes to report
healthy.

The parser subsystem (`parser-api`, `parser-consumer`, `parser-rabbitmq`) is
**opt-in** — it's memory-heavy (loads OCR models) and gated behind the `parser`
Compose profile, so a plain `up` skips it. Enable it only if you need PDF/document
parsing for analytics:

```bash
docker compose --profile parser up -d
```

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
127.0.0.1  auth.localhost backend.localhost api.localhost plugin.localhost
127.0.0.1  minio.localhost mongoui.localhost dynamoadmin.localhost
```

## Common operations

```bash
# View logs for one service
docker compose logs -f ameya-backend-service

# Restart a single service
docker compose restart analytics-api

# Pull updated images and recreate
docker compose pull && docker compose up -d

# Start including the opt-in parser subsystem
docker compose --profile parser up -d

# Stop just the parser subsystem (frees memory) without touching the rest
docker compose stop parser-api parser-consumer parser-rabbitmq

# Stop the stack (keeps volumes/data)
docker compose down

# Stop AND delete all data volumes (full reset)
docker compose down -v
```

## Configuration notes

- **`api.localhost` path routing** — Extraction and analytics share one host,
  split by path in the `Caddyfile`: `/entityextractionagent/*` →
  `extraction-service`, `/ameya/analytics/v1/*` → `analytics-api`. Paths are
  preserved (no stripping); each upstream mounts under its own prefix.
- **LLM provider** — Extraction defaults to Google Gemini. To switch providers,
  change `LLM_MODEL` / `LLM_PROVIDER` in the `x-extraction-env` block and supply
  the matching API key.
- **Parser subsystem** *(opt-in)* — `parser-api`, `parser-consumer`, and
  `parser-rabbitmq` carry `profiles: ["parser"]`, so a plain `docker compose up`
  does **not** start them (they load heavy OCR models). Bring them up with
  `docker compose --profile parser up -d`. Nothing else depends on them at
  startup; the only coupling is the runtime `PDF_PARSER_HOST_URL`
  (`http://parser-api:9090`) that analytics calls to parse PDFs — so with the
  profile off, PDF-parsing analytics flows won't work, but everything else does.
- **Parser OCR engines** *(optional)* — When the parser subsystem is enabled it
  can load different OCR/parsing backends, toggled via env in the `x-parser-env`
  block. Defaults load Marker only; set as needed:

  | Variable       | Default   | Engine                  |
  | -------------- | --------- | ----------------------- |
  | `LOAD_MARKER`  | `"True"`  | Marker (PDF → markdown) |
  | `LOAD_EASYOCR` | `"False"` | EasyOCR                 |
  | `LOAD_PADDLE`  | `"False"` | PaddleOCR               |

  Enabling more engines increases the parser image's memory/startup cost, so
  turn on only what you need.
- **MinIO / presigned URLs** — The backend presigns S3 URLs against
  `http://minio.localhost:8080`. S3 SigV4 signs the Host header, so this host
  must match exactly between the browser and the backend; do not change it
  without updating both `S3_ENDPOINT_URL` and the Caddy route.
- **CORS** — The SPA calls the API hosts cross-origin. Caddy adds uniform CORS
  headers on the API hosts (see `Caddyfile`); no per-service CORS config needed.
- **Persistent data** — All stateful data lives in named volumes
  (`mongodb_data`, `dynamodb-data`, `minio-data`, `redis-data`, `uploads_data`,
  etc.). `docker compose down` preserves them; `-v` deletes them.
- **Changing the `8080` port** — Port `8080` is embedded in the browser-facing
  URLs throughout the env anchors. To run on another host port, remap only the
  host side of the Caddy `ports:` (e.g. `"9090:8080"`) and change the
  `*.localhost:8080` / `localhost:8080` URLs to the new port (leave internal
  ports like `parser-api:9090` alone). Caddy matches sites by hostname
  regardless of port, so the `Caddyfile` itself needs no change.

## Troubleshooting

| Symptom                                    | Likely cause / fix                                                             |
| ------------------------------------------ | ------------------------------------------------------------------------------ |
| `network ameya_community_network not found`| Run step 2: `docker network create ameya_community_network`.                   |
| `no matching manifest for linux/arm64`     | Images are amd64-only. Run under emulation on Apple Silicon: `DOCKER_DEFAULT_PLATFORM=linux/amd64 docker compose up -d`. |
| SPA loads but API calls fail in Safari     | Add the `*.localhost` entries to `/etc/hosts` (see Host resolution).           |
| Extraction jobs never complete             | Missing/invalid `GEMINI_API_KEY`; check `docker compose logs extraction-service`. |
| Analytics endpoints error out              | Missing/invalid `OPENAI_API_KEY`, or `analytics-api` still warming up (long start_period). |
| Chatbot replies *"I encountered an issue while processing your request. Please try again."* | `GEMINI_API_KEY` is invalid or its usage/quota limit is exhausted. Verify the key and check your Gemini quota; then `docker compose restart` the affected service. |
| A service is stuck `unhealthy`             | Inspect its logs; dependents wait on healthchecks and won't start.             |
