# Deploy Irmin on Railway (from scratch)

This guide recreates the previous Irmin Railway project from this repository
alone. Per-service build/deploy settings live in config-as-code files; root
directory, networking ports, resource limits, and environment variables are
still set in the Railway dashboard (or Railway IaC).

**Verification note:** the live Railway project is being deleted. Config and
env-name lists below were captured from that project before deletion and cannot
be re-verified against a running Railway deployment from this PR.

## Config-as-code files

| Service | Root directory | Config file in repo | Dashboard **Config file path** |
| --- | --- | --- | --- |
| `irmin` (Core) | `/core` | `core/railway.json` | `/core/railway.json` |
| `irmin-ai` | `/ai` | `ai/railway.json` | `/ai/railway.json` |
| `irmin-console` | `/console` | `console/railway.json` | `/console/railway.json` |
| `irmin-connectors` | `/connectors` | `connectors/railway.json` | `/connectors/railway.json` |

Railway does **not** resolve the config file relative to the service root
directory. Set each service's **Config file path** in the dashboard to the
absolute-from-repo-root path above (for example `/core/railway.json`). See
[Railway monorepo docs](https://docs.railway.com/guides/monorepo) and
[config as code](https://docs.railway.com/guides/config-as-code).

`watchPatterns` in those files are relative to the **repository root** (not the
service root), matching Railway's monorepo watch-path behaviour.

Schema: [`https://railway.com/railway.schema.json`](https://railway.com/railway.schema.json).

Encoded in each file today:

| Service | Builder | Watch patterns | Sleep | Extra deploy |
| --- | --- | --- | --- | --- |
| `irmin` | `NIXPACKS` | `core/**`, `sdks/go/**`, `go.work`, `go.work.sum` | on | — |
| `irmin-ai` | `RAILPACK` | `ai/**` | on | — |
| `irmin-console` | `DOCKERFILE` (`console/Dockerfile`) | `console/**` | on | `drainingSeconds` 15, `overlapSeconds` 10 |
| `irmin-connectors` | `NIXPACKS` | `connectors/**`, `sdks/go/**`, `go.work`, `go.work.sum` | on | — |

Public networking target ports (dashboard / `PORT` env, not in `railway.json`):

| Service | Port |
| --- | --- |
| `irmin` | `8080` |
| `irmin-ai` | `3000` |
| `irmin-console` | `8080` |
| `irmin-connectors` | `8080` |

## Project defaults

- **Region:** `europe-west4` (Railway region id `europe-west4-drams3a`)
- **Replicas:** 1 per service
- **Resource limits (app services):**
  - `irmin`, `irmin-ai`, `irmin-connectors`: 4 vCPU / 4 GB
  - `irmin-console`: 2 vCPU / 2 GB

Sleep was enabled on app and data services in the previous project. Sleep did
**not** idle the data services in practice — Postgres, Qdrant, MinIO, and
LakeFS continued to bill for reserved RAM continuously. Plan cost accordingly
or leave data services without sleep if you prefer predictable always-on
behaviour.

## Service map

### Application services (this repo)

Connect the GitHub repo to each service, set **Root Directory**, **Config file
path**, region, replicas, and resource limits as above.

```
Console ──► Core API ──► Connectors
   │            │
   └──────────► AI ──► Qdrant
                │
Core ──► LakeFS ──► MinIO (S3 blockstore + Irmin bucket)
  │         │
  │         └──► LakeFS Postgres
  ├──► Core Postgres (via Core PgBouncer)
  │
Connectors ──► Connectors Postgres (via Connectors PgBouncer)
AI ──► AI Postgres (via AI PgBouncer)
```

Wire private networking hostnames where possible; expose public domains only
for Console, Core, AI, Connectors, and (if needed) LakeFS / MinIO / Qdrant
admin UIs.

### Infrastructure services

#### Postgres (×4)

Use the Railway `postgres-ssl` template (`ghcr.io/railwayapp-templates/postgres-ssl:16`
or `:17`). Create four databases:

| Service name | Used by |
| --- | --- |
| Core Postgres | Core (via Core PgBouncer) |
| AI Postgres | AI (via AI PgBouncer) |
| Connectors Postgres | Connectors (via Connectors PgBouncer) |
| LakeFS Postgres | LakeFS only (no PgBouncer in the previous setup) |

#### PgBouncer (×3)

Image: `railwayapp/pgbouncer`. One each for Core, AI, and Connectors. Point
each at its Postgres (`POSTGRESQL_HOST`, `POSTGRESQL_USERNAME`,
`POSTGRESQL_PASSWORD`, `POSTGRESQL_DATABASE`, `POSTGRESQL_PORT`) and point the
app's database URL at the PgBouncer `DATABASE_URL` (or equivalent private
URL), not the raw Postgres public URL.

#### Qdrant

- Image: `qdrant/qdrant`
- Volume mount: `/qdrant/storage`
- Public / networking port: `6333`
- API key env: `QDRANT__SERVICE__API_KEY` (reference the same value from AI as
  `QDRANT_API_KEY`)

#### LakeFS

- Image: `treeverse/lakefs:latest`
- Port: `8000`
- Own Postgres: `LAKEFS_DATABASE_TYPE=postgres` +
  `LAKEFS_DATABASE_POSTGRES_CONNECTION_STRING` → LakeFS Postgres
- S3 blockstore on MinIO (`LAKEFS_BLOCKSTORE_TYPE=s3`, endpoint + credentials
  pointing at the MinIO / Bucket service)
- Encrypt secret: `LAKEFS_AUTH_ENCRYPT_SECRET_KEY`

#### MinIO Bucket

- Image: `pgsty/minio` (pin a release tag in production)
- Start command:
  `minio server --address [::]:$MINIO_PRIVATE_PORT $RAILWAY_VOLUME_MOUNT_PATH`
- Healthcheck path: `/minio/health/ready`
- Volume mount: `/data`
- Root credentials: `MINIO_ROOT_USER`, `MINIO_ROOT_PASSWORD`

Create buckets used by LakeFS and Core (historically `lakefs` and `irmin`).

#### Optional: MinIO Console

- Template / repo: `railwayapp-templates/minio-console`
- Start command: `console server --host 0.0.0.0 --port $PORT`
- Healthcheck path: `/login`
- Point `CONSOLE_MINIO_SERVER` at the MinIO private endpoint

## Environment variables (names only)

Never commit values. Prefer Railway reference variables
(`${{Service.VAR}}`) for cross-service wiring.

### Shared secrets that must match

| Purpose | Variables (must be identical values) |
| --- | --- |
| Core ↔ AI ↔ Connectors system auth | Core `TOKEN` = AI `IRMIN_SYSTEM_TOKEN` = Connectors `IRMIN_API_TOKEN` |
| Core ↔ AI service auth | Core `AI_SERVICE_SYSTEM_TOKEN` = AI `AI_API_SYSTEM_TOKEN` |
| Console → Core system calls | Console `API_SYSTEM_TOKEN` (same family as Core `TOKEN` when used for system API access) |

Also keep public URLs aligned (`URL` / `NEXT_PUBLIC_*` / `IRMIN_API_BASE_URL` /
`AI_SERVICE_BASE_URL` / `CONSOLE_URL` / `ALLOWED_ORIGINS` / `CORS_ORIGINS`).

### Generated secrets (create once per environment)

| Area | Names |
| --- | --- |
| Core crypto / IDs | `CREDENTIAL_ENCRYPTION_KEYS`, `SIGNED_URL_SECRET`, `SQID_ALPHABET` |
| Console env gate | `HMAC_SECRET`, `ENV_PASSWORD` |
| LakeFS | `LAKEFS_AUTH_ENCRYPT_SECRET_KEY` |
| MinIO | `MINIO_ROOT_USER`, `MINIO_ROOT_PASSWORD` |
| Qdrant | `QDRANT__SERVICE__API_KEY` (and AI `QDRANT_API_KEY`) |
| Shared system tokens | `TOKEN` / `IRMIN_SYSTEM_TOKEN` / `IRMIN_API_TOKEN`, `AI_SERVICE_SYSTEM_TOKEN` / `AI_API_SYSTEM_TOKEN`, Console `API_SYSTEM_TOKEN` |

### Third-party keys

| Provider | Typical names |
| --- | --- |
| Clerk | Core `CLERK_SECRET_KEY`, `CLERK_SIGNING_KEY`, `CLERK_SIGNING_ALGORITHM`, `CLERK_PUBLIC_KEY`; Console `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`, `CLERK_SECRET_KEY` |
| Polar | `POLAR_API_KEY`, `POLAR_WEBHOOK_SECRET`, `POLAR_PRODUCT_ID`, `POLAR_BASE_URL`, `BILLING_ENABLED` |
| Daytona | `DAYTONA_API_KEY`, `DAYTONA_API_URL`, `DAYTONA_SNAPSHOT_GO` (and optional `DAYTONA_TARGET`) |
| Google Drive OAuth | `GOOGLE_DRIVE_CLIENT_ID`, `GOOGLE_DRIVE_CLIENT_SECRET`, `CONNECTOR_OAUTH_REDIRECT_URI` |
| Novu | Core / Console `NOVU_SECRET_KEY`; Console `NEXT_PUBLIC_NOVU_APP_ID` |
| OpenAI | Core / AI `OPENAI_API_KEY` |
| OpenRouter | AI `OPENROUTER_API_KEY`, `OPENROUTER_SITE_URL`, `OPENROUTER_SITE_NAME`, `OPENROUTER_PROVIDER_ALLOWLIST` |
| LangSmith | AI `LANGSMITH_TRACING`, `LANGSMITH_API_KEY`, `LANGSMITH_PROJECT` |
| Sentry | Core / AI / Console / Connectors `SENTRY_*` / `NEXT_PUBLIC_SENTRY_*` as in each `.env.example` |
| PostHog | Console `NEXT_PUBLIC_POSTHOG_KEY`, `NEXT_PUBLIC_POSTHOG_HOST` |

### Railway references (examples)

Use dashboard references rather than pasting connection strings:

| Consumer | Example reference pattern |
| --- | --- |
| Core DB via PgBouncer | `DATABASE_CONNECTION_STRING=${{Core PgBouncer.DATABASE_URL}}` |
| AI DB via PgBouncer | `DATABASE_URL=${{AI PgBouncer.DATABASE_URL}}` |
| Connectors DB via PgBouncer | `DATABASE_CONNECTION_STRING=${{Connectors PgBouncer.DATABASE_URL}}` |
| LakeFS DB | `LAKEFS_DATABASE_POSTGRES_CONNECTION_STRING=${{LakeFS Postgres.DATABASE_URL}}` |
| MinIO / S3 | Core `S3_ENDPOINT`, `S3_ACCESS_KEY_ID`, `S3_ACCESS_SECRET` ← Bucket / MinIO vars |
| LakeFS blockstore | `LAKEFS_BLOCKSTORE_S3_*` ← same MinIO credentials + private endpoint |
| Qdrant | AI `QDRANT_URL` ← Qdrant private/public host; `QDRANT_API_KEY` ← `QDRANT__SERVICE__API_KEY` |
| Service URLs | `AI_SERVICE_BASE_URL`, `IRMIN_API_BASE_URL`, `NEXT_PUBLIC_API_URL`, `NEXT_PUBLIC_AI_API_URL`, `CONSOLE_URL`, `LAKE_FS_URL` ← `${{Service.RAILWAY_PUBLIC_DOMAIN}}` (with `https://` prefix) |

Exact Railway service display names must match the `${{Name.VAR}}` references
you create.

## Suggested recreate order

1. Create an empty Railway project in `europe-west4`.
2. Provision the four Postgres services, three PgBouncers, MinIO (+ volume),
   optional MinIO Console, Qdrant (+ volume), and LakeFS.
3. Create empty app services `irmin`, `irmin-ai`, `irmin-console`,
   `irmin-connectors`; connect this repo; set root directory + config file path;
   set region, replicas, limits, and ports.
4. Set shared/generated/third-party env vars (names above); use references for
   databases and internal URLs.
5. Deploy infra first, create MinIO buckets, complete LakeFS first-time setup,
   then deploy app services.
6. Confirm health endpoints and that Core / AI / Connectors system tokens match.

## Related local docs

- Root [README.md](../README.md) and [AGENTS.md](../AGENTS.md) (local compose /
  token alignment)
- Per-project `.env.example` files under `core/`, `ai/`, `console/`,
  `connectors/`
