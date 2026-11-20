# DEPLOYMENT.md — Deployment Plan

> **System:** `<Project Name>` — Integrated System
> **Version being released:** `v1.5.0` (previous stable: `v1.4.2`)
> **Team:** `<Team Name>` | **Course:** `<Course / Section>`
> **Last updated:** `<YYYY-MM-DD>`

A step-by-step guide for releasing this system to production safely, and for undoing the release if production fails.

> Replace every `<placeholder>` and the example values (versions, ports, names) with your team's real ones.

---

## 1. System Overview

| Service | Technology | Container name | Port | Purpose |
|---|---|---|---|---|
| Database | PostgreSQL 16 | `postgres` | 5432 | Stores all application data |
| Message Queue | RabbitMQ 3.13 | `rabbitmq` | 5672 (AMQP), 15672 (UI) | Passes background jobs between API and worker |
| API | REST API (Node.js / Express) | `api` | 3000 | Public entry point for clients |
| Worker | Background consumer | `worker` | — | Processes jobs from the `jobs` queue |

Full diagram: see [`HANDOVER/architecture-diagram.md`](HANDOVER/architecture-diagram.md).

---

## 2. Team Roles

| Role | Name | Responsibility during launch |
|---|---|---|
| **Flight Director** (Release Lead) | `<Name>` | Runs the checklist, makes the final **GO / NO-GO** and **ABORT** calls |
| Deployment Engineer | `<Name>` | Executes migrations and container deployment commands |
| Database Engineer | `<Name>` | Takes backups, verifies migrations, runs down-migration if needed |
| QA / Validation Engineer | `<Name>` | Runs smoke tests and watches logs/metrics after release |
| Documentation Lead | `<Name>` | Records the deployment log and updates the handover docs |

**Rule:** Only the Flight Director can declare GO or ABORT. Everyone else reports status to them.

---

## 3. Deployment Window

- **Allowed:** Monday–Thursday, 9:00 AM – 2:00 PM (team is online for at least 3 hours after release).
- **Not allowed:** Friday afternoons, weekends, holidays, or right before a deadline. **No Friday 5 PM deploys.**
- The full team must be available for the whole window.

---

## 4. Environment Configuration

Values live in a `.env` file on the server. **`.env` is never committed to Git** — only `.env.example` (names only, no real values) is in the repo.

| Variable | Used by | Example / Description |
|---|---|---|
| `NODE_ENV` | api, worker | `production` |
| `API_PORT` | api | `3000` |
| `DATABASE_URL` | api, worker | `postgres://<user>:<password>@postgres:5432/<db_name>` |
| `POSTGRES_USER` / `POSTGRES_PASSWORD` / `POSTGRES_DB` | postgres | Database credentials (secret) |
| `RABBITMQ_URL` | api, worker | `amqp://<user>:<password>@rabbitmq:5672` |
| `RABBITMQ_DEFAULT_USER` / `RABBITMQ_DEFAULT_PASS` | rabbitmq | Queue credentials (secret) |
| `JWT_SECRET` | api | Secret for signing auth tokens (secret) |
| `APP_VERSION` | api, worker | Image tag being deployed, e.g. `v1.5.0` |

**Service endpoints (production):**

| Endpoint | URL |
|---|---|
| Public API base URL | `https://<your-domain>/api/v1` |
| API health check | `https://<your-domain>/health` |
| API readiness check | `https://<your-domain>/health/ready` |
| RabbitMQ management UI (internal only) | `http://<server-ip>:15672` |

---

## 5. Launch Sequence

### STAGE 01 — Pre-Deployment (T-30 min)

| # | Check | Command / Action | Owner | ✔ |
|---|---|---|---|---|
| 1 | Code is merged, tagged, and CI tests pass | `git tag v1.5.0 && git push origin v1.5.0` | Deployment Eng. | ☐ |
| 2 | New image is built and pushed | `docker build -t <registry>/<project>-api:v1.5.0 . && docker push <registry>/<project>-api:v1.5.0` | Deployment Eng. | ☐ |
| 3 | **Database backup taken** | `docker compose exec postgres pg_dump -U $POSTGRES_USER -d $POSTGRES_DB -F c -f /backups/pre_v1.5.0.dump` | DB Eng. | ☐ |
| 4 | Backup file exists and is not empty | `docker compose exec postgres ls -lh /backups/pre_v1.5.0.dump` | DB Eng. | ☐ |
| 5 | `.env` is complete (all variables in Section 4 are set) | `diff <(grep -o '^[A-Z_]*' .env.example \| sort) <(grep -o '^[A-Z_]*' .env \| sort)` → no output | Deployment Eng. | ☐ |
| 6 | Server has enough disk space (> 20% free) | `df -h` | Deployment Eng. | ☐ |
| 7 | Down-migration was tested on staging | Confirmed in staging log | DB Eng. | ☐ |
| 8 | Previous stable tag recorded for rollback | `PREVIOUS_VERSION=v1.4.2` | Flight Director | ☐ |

**If any item fails → NO-GO. Stop and fix before continuing.**

### STAGE 02 — Deployment Steps (Service Startup Order)

Services must start in this order, because each one depends on the one before it:

```
1. Database (PostgreSQL)  →  2. Message Queue (RabbitMQ)  →  3. API  →  4. Worker
```

| Step | Action | Command | Wait until |
|---|---|---|---|
| 1 | Start / confirm database | `docker compose up -d postgres` | `docker compose exec postgres pg_isready -U $POSTGRES_USER` → `accepting connections` |
| 2 | Start / confirm message queue | `docker compose up -d rabbitmq` | `docker compose exec rabbitmq rabbitmq-diagnostics -q ping` → `Ping succeeded` |
| 3 | **Run DB migrations** (before new code) | `docker compose run --rm api npm run migrate:up` | Command exits with no errors |
| 4 | Deploy new API container | `APP_VERSION=v1.5.0 docker compose up -d --no-deps api` | `curl -f http://localhost:3000/health` → `200` |
| 5 | Deploy new worker container | `APP_VERSION=v1.5.0 docker compose up -d --no-deps worker` | `docker compose logs --tail=20 worker` shows `Connected to queue: jobs` |

**Why migrations come first:** the new code expects the new database schema. If containers start before migrations, the API will crash on missing tables or columns.

### GO / NO-GO Gate — Health Checks Before Public Traffic

All 3 must pass before the system is opened to users:

| # | Health check | Command | Pass condition |
|---|---|---|---|
| 1 | **Database is reachable** | `docker compose exec postgres pg_isready -U $POSTGRES_USER` | `accepting connections` |
| 2 | **Message queue is reachable and the worker is consuming** | `docker compose exec rabbitmq rabbitmqctl list_queues name messages consumers` | `jobs` queue shows `consumers ≥ 1` |
| 3 | **API is ready (connected to DB and queue)** | `curl -s http://localhost:3000/health/ready` | HTTP `200` with `{"status":"ok","db":"up","queue":"up"}` |

- **All 3 pass** → Flight Director declares **GO** → open public traffic (enable the reverse proxy / remove maintenance page).
- **Any check fails** → **NO-GO** → do not open traffic. Fix it, or go to Stage 04 (Rollback).

### STAGE 03 — Post-Deployment Validation (first 30 minutes)

**Smoke tests** (run immediately after GO):

| # | Test | Command | Expected |
|---|---|---|---|
| 1 | Health endpoint | `curl -i https://<your-domain>/health` | `200 OK` |
| 2 | Login works | `curl -i -X POST https://<your-domain>/api/v1/auth/login -H "Content-Type: application/json" -d '{"email":"smoke@test.com","password":"<test-password>"}'` | `200 OK` + token |
| 3 | Read data | `curl -i https://<your-domain>/api/v1/orders -H "Authorization: Bearer <token>"` | `200 OK` + JSON list |
| 4 | Write data + queue flow | `curl -i -X POST https://<your-domain>/api/v1/orders -H "Authorization: Bearer <token>" -H "Content-Type: application/json" -d '{"item":"Smoke Test","quantity":1}'` | `201 Created`, and worker log shows the job processed |
| 5 | Version is correct | `curl -s https://<your-domain>/health` | `"version":"v1.5.0"` |

**Monitoring (telemetry)** — watch for 30 minutes:

```bash
docker compose logs -f --tail=100 api worker          # watch for errors
docker stats                                         # CPU / memory usage
docker compose exec rabbitmq rabbitmqctl list_queues name messages consumers   # queue backlog
```

| Metric | Normal | Warning |
|---|---|---|
| API 5xx error rate | < 1% | ≥ 1% |
| API response time (p95) | < 500 ms | ≥ 1 s |
| `jobs` queue backlog | < 100 messages | Growing for 5+ min |
| Container restarts | 0 | Any restart loop |

### STAGE 04 — Rollback Plan (Launch Abort)

#### Abort Triggers — roll back immediately if ANY of these happens:

| # | Trigger | Exact threshold |
|---|---|---|
| 1 | Health check failure | `/health/ready` fails **3 times in a row** (checked every 30 seconds) |
| 2 | High error rate | API **5xx error rate > 5% for 5 minutes** straight |
| 3 | Smoke test failure | **Any** smoke test in Stage 03 fails and cannot be fixed within **10 minutes** |
| 4 | Crash loop | API or worker container **restarts 3+ times within 10 minutes** |
| 5 | Data problem | Migration fails, or any data loss / corruption is detected |

**The Flight Director declares ABORT. Do not try to debug in production — roll back first, investigate after.**

#### Rollback Procedure

| Step | Action | Command |
|---|---|---|
| 1 | Announce ABORT to the team | Team chat: "ABORT v1.5.0 — rolling back to v1.4.2" |
| 2 | Stop public traffic (optional, if data is at risk) | Enable maintenance page / stop the reverse proxy |
| 3 | Revert API and worker to the stable image | `APP_VERSION=v1.4.2 docker compose up -d --no-deps api worker` |
| 4 | Run the pre-tested down-migration | `docker compose run --rm api npm run migrate:down` |
| 5 | **Only if down-migration fails or data is corrupted:** restore the backup | `docker compose exec postgres pg_restore -U $POSTGRES_USER -d $POSTGRES_DB --clean /backups/pre_v1.5.0.dump` |
| 6 | Re-run the 3 GO/NO-GO health checks | See GO/NO-GO Gate above |
| 7 | Re-run smoke tests 1–3 | See Stage 03 |
| 8 | Confirm version is back | `curl -s http://localhost:3000/health` → `"version":"v1.4.2"` |
| 9 | Re-open traffic and write an incident note | Log what failed, when, and why in `HANDOVER/KNOWN_ISSUES.md` |

**Rollback target time:** system back on `v1.4.2` within **15 minutes** of the ABORT call.

---

## 6. Deployment Log (fill in on launch day)

| Time | Step | Result | By |
|---|---|---|---|
| `<HH:MM>` | Pre-deployment checklist | ☐ Pass ☐ Fail | |
| `<HH:MM>` | Migrations | ☐ Pass ☐ Fail | |
| `<HH:MM>` | API + worker deployed | ☐ Pass ☐ Fail | |
| `<HH:MM>` | GO / NO-GO decision | ☐ GO ☐ NO-GO | Flight Director |
| `<HH:MM>` | Smoke tests | ☐ Pass ☐ Fail | |
| `<HH:MM>` | 30-min monitoring complete | ☐ Stable ☐ Rolled back | |

---

## 7. Related Documents

| Document | Location |
|---|---|
| Architecture diagram | [`HANDOVER/architecture-diagram.md`](HANDOVER/architecture-diagram.md) |
| API documentation (OpenAPI) | [`HANDOVER/openapi.yaml`](HANDOVER/openapi.yaml) |
| Sample requests | [`HANDOVER/api-sample-requests.md`](HANDOVER/api-sample-requests.md) |
| Operations runbook | [`HANDOVER/RUNBOOK.md`](HANDOVER/RUNBOOK.md) |
| Known issues | [`HANDOVER/KNOWN_ISSUES.md`](HANDOVER/KNOWN_ISSUES.md) |
