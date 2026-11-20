# Operations Runbook — Flight Crew Manual

A "how-to" guide for the operations team during daily work and live incidents.
Every command is meant to be run from the project folder on the server (where `docker-compose.yml` is).

```bash
cd /opt/<project-folder>
```

---

## Quick Incident Table

| Scenario | Symptom | Exact Action Command |
|---|---|---|
| **Restart primary API** | `/health` times out or returns 5xx; clients get `502 Bad Gateway` | `docker compose restart api` |
| Restart database | Logs show `too many connections` / connection pool exhausted | `docker compose restart postgres` then `docker compose restart api worker` |
| Restart worker | Orders stuck in `pending`; `jobs` queue shows `consumers = 0` | `docker compose restart worker` |
| Restart message queue | `/health/ready` shows `"queue":"down"`; API returns `503` on create | `docker compose restart rabbitmq` then `docker compose restart api worker` |
| Clear stuck queue | Consumer lag exceeds 10,000 messages and is not going down | `docker compose exec rabbitmq rabbitmqctl purge_queue jobs` ⚠️ deletes pending jobs — see 3.3 |
| Disk almost full | `df -h` shows < 10% free | `docker system prune -f` and `docker compose logs` cleanup — see 3.5 |
| Bad release | Error rate > 5% after deploy | Follow **Stage 04 — Rollback Plan** in [`../DEPLOYMENT.md`](../DEPLOYMENT.md) |

---

## 1. Runbook Entry: Restart the Primary API (step-by-step)

**When to use:** The API is not responding, is returning 5xx errors, or is using too much memory.

**Time needed:** ~2 minutes. **Downtime:** ~10–20 seconds.

| Step | Do this | Command | You should see |
|---|---|---|---|
| 1 | Go to the project folder | `cd /opt/<project-folder>` | — |
| 2 | Check the API's current status | `docker compose ps api` | `STATUS` column (e.g. `Up`, `Restarting`, `Exited`) |
| 3 | Save the last logs before restarting (for investigation later) | `docker compose logs --tail=200 api > /tmp/api-$(date +%F-%H%M).log` | No output (file is saved) |
| 4 | Make sure the database is up | `docker compose exec postgres pg_isready -U $POSTGRES_USER` | `accepting connections` (if not → restart DB first, see table above) |
| 5 | Make sure the queue is up | `docker compose exec rabbitmq rabbitmq-diagnostics -q ping` | `Ping succeeded` (if not → restart RabbitMQ first) |
| 6 | **Restart the API** | `docker compose restart api` | `Container api  Started` |
| 7 | Wait 15 seconds, then check liveness | `sleep 15 && curl -i http://localhost:3000/health` | `HTTP/1.1 200 OK` |
| 8 | Check readiness | `curl -s http://localhost:3000/health/ready` | `{"status":"ok","db":"up","queue":"up"}` |
| 9 | Watch logs for 1 minute for new errors | `docker compose logs -f --tail=50 api` (press `Ctrl + C` to exit) | No repeating `ERROR` lines |

**If step 7 still fails after 2 tries:**

```bash
docker compose up -d --force-recreate api      # recreate the container from scratch
```

**If it still fails:** the new release is likely broken → follow the Rollback plan in `DEPLOYMENT.md`, then contact the Release Lead (see `README.md`).

---

## 2. Daily Operations

| Task | When | Command |
|---|---|---|
| Check all services are running | Every morning | `docker compose ps` |
| Check API health | Every morning | `curl -s http://localhost:3000/health/ready` |
| Check queue backlog | Every morning | `docker compose exec rabbitmq rabbitmqctl list_queues name messages consumers` |
| Check dead-letter queue | Every morning | Same command — look at `jobs.dlq`; should be `0` |
| Check disk space | Every morning | `df -h` |
| Daily database backup | Every night (cron, 2:00 AM) | `docker compose exec -T postgres pg_dump -U $POSTGRES_USER -d $POSTGRES_DB -F c > /backups/daily_$(date +%F).dump` |
| Delete backups older than 7 days | Weekly | `find /backups -name "daily_*.dump" -mtime +7 -delete` |
| View recent errors | As needed | `docker compose logs --since=1h api worker \| grep -i error` |

---

## 3. Incident Procedures

### 3.1 Database — Connection Pool Exhausted

**Symptom:** API logs show `sorry, too many clients already` or `timeout acquiring connection`.

```bash
# 1. See how many connections are open
docker compose exec postgres psql -U $POSTGRES_USER -d $POSTGRES_DB -c "SELECT count(*) FROM pg_stat_activity;"
# 2. Restart the database
docker compose restart postgres
# 3. Restart the apps so they reconnect cleanly
docker compose restart api worker
# 4. Verify
curl -s http://localhost:3000/health/ready
```

### 3.2 Worker Not Processing Jobs

**Symptom:** Orders stay `pending`; `jobs` queue messages keep growing; `consumers = 0`.

```bash
docker compose logs --tail=100 worker            # look for the error
docker compose restart worker
docker compose exec rabbitmq rabbitmqctl list_queues name messages consumers   # consumers should be ≥ 1
```

### 3.3 Clear a Stuck Queue

**Symptom:** Consumer lag exceeds **10,000 messages** and is not going down even after restarting the worker.

> ⚠️ **Purging deletes all waiting jobs permanently.** Get approval from the Release Lead first, and export the messages if they matter.

```bash
# 1. Confirm the backlog
docker compose exec rabbitmq rabbitmqctl list_queues name messages consumers
# 2. Stop the worker
docker compose stop worker
# 3. Purge the queue
docker compose exec rabbitmq rabbitmqctl purge_queue jobs
# 4. Start the worker again
docker compose start worker
# 5. See which orders lost their job (still 'pending' in the database)
docker compose exec postgres psql -U $POSTGRES_USER -d $POSTGRES_DB -c "SELECT id, created_at FROM orders WHERE status='pending' ORDER BY created_at;"
# 6. Re-publish a job for each of those orders (script included in the API project)
docker compose exec api npm run requeue:pending
```

### 3.4 Messages in the Dead-Letter Queue (`jobs.dlq`)

**Symptom:** `jobs.dlq` has more than 0 messages.

1. Open the RabbitMQ UI: `http://<server-ip>:15672` → **Queues** → `jobs.dlq` → **Get messages**.
2. Read the message and the matching worker error: `docker compose logs worker | grep <orderId>`.
3. Fix the cause, then move messages back using the UI's **Move messages** (shovel) to `jobs`.

### 3.5 Disk Almost Full

```bash
df -h                                   # confirm
docker system df                        # see what Docker is using
docker system prune -f                  # remove stopped containers and dangling images
find /backups -name "*.dump" -mtime +7 -delete
```

### 3.6 Full System Restart (last resort)

Restart in the correct order — **database → queue → API → worker**:

```bash
docker compose down
docker compose up -d postgres  && sleep 10
docker compose up -d rabbitmq  && sleep 10
docker compose up -d api       && sleep 10
docker compose up -d worker
curl -s http://localhost:3000/health/ready
```

---

## 4. Escalation

| Severity | Example | Action |
|---|---|---|
| Low | One slow request, 1–2 DLQ messages | Log it in `KNOWN_ISSUES.md`, fix during work hours |
| Medium | Worker down, orders delayed | Follow this runbook; notify Release Lead |
| High | API down, data errors, error rate > 5% | Follow this runbook **and** Rollback plan; call the Release Lead immediately |
