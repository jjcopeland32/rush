Awesome—here’s a drop-in **README.md** that’s purpose-built to make an AI IDE/agent (e.g., Codex/Cursor/VS Code Copilot) *reliably* extend and maintain this repo with minimal mistakes.

Copy this entire block into your repo root as `README.md`.

---

# Payments Modernization – Unified Stack

**Purpose:** A modern control-plane and data pipeline that unlocks legacy processor data (e.g., Fiserv/TSYS) into clean APIs, webhooks, and an admin UI.

**Audience:** Humans *and* AI agents. This README is written to guide AI tools to produce correct, consistent changes.

---

## 🔭 High-level

* **Adapters**: Poll SFTP, store raw files in MinIO, announce via Kafka.
* **Ingestion**: Consume events, parse CSV/JSON, upsert into Postgres (idempotent).
* **APIs**:

  * `reporting-svc`: settlements & disputes (OpenAPI-validated).
  * `config-svc`: config drift & maker/checker (OpenAPI-validated).
  * `recon-svc`: math checks + alerts (+ optional Slack).
  * `ops-svc`: message hospital (job list/replay).
  * `webhooks-svc`: enqueue+retry outbound events.
* **Admin UI**: Next.js + NextAuth (Keycloak).
* **Security**: OIDC JWT (JWKS verification) + RBAC (roles in Keycloak realm).
* **Infra**: Postgres, Redpanda (Kafka), MinIO (S3), Keycloak, SFTP via Docker Compose.

---

## 🧭 Repo map (monorepo)

```
infra/
  docker/docker-compose.dev.yml        # One command dev stack
  keycloak/realm-export.json           # Clients, roles, user ops/ops
openapi/
  reporting.yaml
  config.yaml
data-model/
  migrations/                          # 001..004.sql
apps/
  reporting-svc/                       # Express, JWKS auth, OpenAPI validator
  config-svc/
  recon-svc/
  ops-svc/
  webhooks-svc/
  ingestion-pipeline/                  # Kafka -> MinIO -> Postgres worker
  adapters/fiserv-batch-adapter/       # SFTP -> MinIO -> Kafka
  admin-next/                          # Next.js + NextAuth
ops/
  vendor-specs/fixtures/               # Sample CSV/JSON for local testing
```

---

## 🚀 Quick start (local)

```bash
# 0) From repo root:
docker compose -f infra/docker/docker-compose.dev.yml up -d --build

# 1) Apply DB migrations
docker exec -it pg psql postgresql://postgres:postgres@localhost:5432/payments \
  -f /app/data-model/migrations/001_init.sql
docker exec -it pg psql postgresql://postgres:postgres@localhost:5432/payments \
  -f /app/data-model/migrations/004_webhook_deliveries.sql

# 2) Seed test files into SFTP
docker cp ops/vendor-specs/fixtures/settlements_sample.csv sftp:/home/dev/upload/
docker cp ops/vendor-specs/fixtures/disputes_sample.csv   sftp:/home/dev/upload/
docker cp ops/vendor-specs/fixtures/config_sample.json    sftp:/home/dev/upload/

# 3) Admin UI -> login with Keycloak
# Keycloak: http://localhost:8080
# Admin UI: http://localhost:3007 (user: ops / pass: ops)
```

---

## 🤖 AI Agent Operating Manual (critical)

> The following rules are **mandatory** for any AI IDE/agent working on this repo.

1. **Never break auth invariants**

   * All services must accept **Bearer JWT** and verify via **JWKS**.
   * Do not bypass `authMiddleware` or `requireAnyRole` except behind `AUTH_DISABLED=true` for local dev.
   * Do not store secrets in code. Use env vars.

2. **Schema-first endpoints**

   * **Do not** modify handlers in `reporting-svc` or `config-svc` without **updating `openapi/*.yaml`** and keeping `express-openapi-validator` happy.
   * Add new endpoints to OpenAPI, then implement. Validation must pass on first run.

3. **Idempotency is non-negotiable**

   * In ingestion, every upsert must be idempotent and safe to reprocess.
   * Enforce **unique indexes** and use `ON CONFLICT DO UPDATE`.

4. **Events & retries**

   * Topic naming: `reports.*`, `alerts.*`.
   * Consumers must be **replay-safe** and tolerate duplicates.
   * For webhooks, enqueue → attempt → backoff → record status. No fire-and-forget.

5. **Logging & errors**

   * Use structured logs: one JSON object per line for non-UI apps.
   * On errors, include `{ service, op, id, err }`. Never log secrets or PII.

6. **Tests & migrations**

   * If a PR changes DB tables, include a migration under `data-model/migrations/`.
   * Provide at least one realistic fixture and a smoke test path.

7. **Performance & pagination**

   * All list endpoints accept `page` and `limit` (max 500) and return `{ items, page, limit }`.
   * Use indexed columns in `WHERE` clauses.

8. **Security**

   * No PII fields beyond minimal demo data.
   * No plaintext secrets; use env.
   * Keep RBAC checks close to route definitions.

9. **Backwards compatibility**

   * Do not change response shapes in a breaking way without bumping version and updating the OpenAPI info block.

10. **Commit hygiene (for AI agents)**

    * Conventional commits: `feat(service): add X`, `fix(ingestion): handle dup key`, `chore(ci)`, etc.
    * Keep diffs minimal and scoped to the task.
    * Update README sections if you introduce new commands or flows.

---

## ✅ Definition of Done (DoD)

A change is **Done** only if:

* ✅ Code compiles and services start via compose.
* ✅ OpenAPI validation passes (for APIs).
* ✅ New/changed endpoints are documented in `openapi/*.yaml`.
* ✅ Database migrations are included & reversible (or safe idempotent).
* ✅ Smoke test steps are documented (copy/paste curl or UI path).
* ✅ Logging is structured; secrets are not printed.
* ✅ RBAC enforced where applicable.

---

## 📋 Acceptance criteria by component

### Adapters (e.g., Fiserv)

* Poll SFTP periodically; move processed files to `/upload/processed/`.
* Write `raw_files` row with checksum; store raw file in MinIO; publish `reports.ingested`.
* **No duplicates**: same checksum must not create duplicate records.

### Ingestion

* On `reports.ingested`, pull file from MinIO, parse by type:

  * `settlements`: upsert into `settlements` with `(merchant_id, business_date, batch_id)` unique key.
  * `disputes`: upsert on `(merchant_id, case_ref)`.
  * `config_snapshot`: store JSON, not parsed fields.
* Record job status in `ingest_jobs` with counts or error.

### Reporting API

* Endpoints:

  * `GET /v1/settlements?merchant_id=&page=&limit=`
  * `GET /v1/disputes?merchant_id=&status=&page=&limit=`
* Must validate with OpenAPI; return `{ items, page, limit }`.

### Config API

* `POST /v1/config/change-requests` (roles: maker/admin)
* `POST /v1/config/change-requests/:id/approve` (roles: checker/admin)
* `GET /v1/config/drift?merchant_id=<uuid>` returns diffs vs external snapshot.

### Recon

* Recompute `net == gross - fees`; for mismatches, insert `alerts`.
* Optional Slack notification if `SLACK_WEBHOOK_URL` set.

### Webhooks

* Create `webhook_deliveries` entries for selected events.
* Deliver with retries and record `status` & `last_error`.

### Ops (Message Hospital)

* `GET /v1/ingest/jobs` lists jobs with status.
* `POST /v1/ingest/jobs/:id/replay` re-emits event.

### Admin UI

* Login via NextAuth/Keycloak; acquire token.
* Display Settlements table.
* (Stretch) Add Disputes, Alerts, and Hospital views.

---

## 🔧 Environment variables (common)

| Var                 | Example                                           | Notes                             |
| ------------------- | ------------------------------------------------- | --------------------------------- |
| `POSTGRES_URI`      | `postgresql://postgres:postgres@pg:5432/payments` | All services                      |
| `KAFKA_BROKERS`     | `redpanda:9092`                                   | Adapter, ingestion, webhooks, ops |
| `MINIO_ENDPOINT`    | `http://minio:9000`                               | Adapter & ingestion               |
| `MINIO_ACCESS_KEY`  | `minioadmin`                                      | local only                        |
| `MINIO_SECRET_KEY`  | `minioadmin`                                      | local only                        |
| `MINIO_BUCKET`      | `raw-files`                                       | create if missing                 |
| `OIDC_ISSUER`       | `http://keycloak:8080/realms/payments`            | APIs                              |
| `OIDC_AUDIENCE`     | `api`                                             | JWT aud claim                     |
| `AUTH_DISABLED`     | `false`                                           | Set `true` for local bypass only  |
| `SLACK_WEBHOOK_URL` | `https://hooks.slack.com/...`                     | optional                          |

---

## 🧪 Manual smoke tests

After seeding fixtures into SFTP:

```bash
# Token: login to http://localhost:8080 (ops/ops) → copy access token from browser storage (NextAuth session) or Keycloak tokens tab.

# Settlements list (replace TOKEN)
curl -H "Authorization: Bearer TOKEN" http://localhost:3002/v1/settlements?limit=5

# Disputes list
curl -H "Authorization: Bearer TOKEN" http://localhost:3002/v1/disputes?limit=5

# Create change request (maker/admin token)
curl -XPOST -H "Authorization: Bearer TOKEN" -H "Content-Type: application/json" \
  -d '{"merchant_id":"00000000-0000-0000-0000-000000000000","operations":[{"path":"/descriptor","to":"NEW DESC"}]}' \
  http://localhost:3003/v1/config/change-requests
```

Expected:

* `items` array with rows in Reporting responses.
* `201` on change request creation.

---

## 🧱 Common pitfalls (AI agent, read carefully)

* **Do not** remove or skip OpenAPI validation; add schemas first.
* **Do not** add blocking I/O or file system writes on API hot paths.
* **Do not** log entire JWTs or secrets.
* **Do not** break pagination or change response shapes without updating OpenAPI.
* **Do** wrap SQL in parameterized queries; never string-concatenate user input.
* **Do** ensure retry logic is bounded and idempotent.
* **Do** maintain unique constraints to avoid duplication on replays.

---

## 🗺️ Roadmap prompts for AI agents

Use these pre-vetted prompts to request changes safely:

* **Add server-side pagination & filters to Admin Next (Settlements):**
  “Implement server-side pagination & merchant filter in `apps/admin-next`. Use the existing `/v1/settlements` API and include `page`/`limit`/`merchant_id` query params. Do not change API shapes. Add a table pager UI.”

* **Extend disputes workflow:**
  “Add `GET /v1/disputes/{id}` and `PATCH /v1/disputes/{id}` to `openapi/reporting.yaml` and implement read/update of status in `apps/reporting-svc`. Keep OpenAPI validator passing. Add minimal Admin UI screen to view/update a dispute.”

* **Webhooks redelivery UI:**
  “Create Admin Next page ‘Webhooks’ to list `webhook_deliveries` (status, attempt, last_error) via a new `webhooks-svc` endpoint `GET /v1/deliveries`. Add a ‘retry’ button calling `POST /v1/deliveries/{id}/retry`. Ensure auth and RBAC.”

* **Observability:**
  “Integrate pino logger with JSON output into all Node services. Add `/metrics` with Prometheus client. Do not log PII or secrets.”

---

## 🧰 Developer commands

> Run from repo root unless noted.

* **Compose up:**
  `docker compose -f infra/docker/docker-compose.dev.yml up -d --build`
* **Tail logs:**
  `docker compose -f infra/docker/docker-compose.dev.yml logs -f reporting-svc`
* **Rebuild one service:**
  `docker compose -f infra/docker/docker-compose.dev.yml up -d --build reporting-svc`
* **Run only APIs locally (bypass JWT):**
  Set `AUTH_DISABLED=true` in shell and start Node services with `npm run dev` inside each app (not recommended for full-stack tests).

---

## 🔐 Security posture (dev)

* JWT/JWKS enabled by default in compose.
* Dev secrets are placeholder only; **never** reuse in non-dev.
* Local data not PCI-scoped; real PCI controls required before production (tokenization, key management, access logging, etc.).

---

## 🆘 Troubleshooting

* **Admin UI won’t auth**: ensure Keycloak is up and realm import succeeded; check `infra/keycloak/realm-export.json`.
* **No settlements appear**: verify adapter processed files (`raw_files`), then ingestion created rows (`settlements`).
* **OpenAPI validation errors**: check request query/body; if endpoints changed, update `openapi/*.yaml`.
* **JWT unauthorized**: verify `OIDC_ISSUER` matches Keycloak realm URL and token audience `api`.

---

## 📄 License / Ownership

Internal prototype for evaluation. Do not deploy to production without security review and a formal license.

---

**You’re good to go.** This README gives both you and your AI agent the rules of the road, the invariants to respect, and the exact steps to run and test the platform. If you want, I can also generate a couple of **issue templates** (feature request, bug) and a **PR checklist** tailored to these acceptance criteria.
