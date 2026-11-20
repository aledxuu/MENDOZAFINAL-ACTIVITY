# Known Issues

Current limitations of `<Project Name>` (as of `v1.5.0`) and how to work around them.

| ID | Issue | Impact | Workaround | Status |
|---|---|---|---|---|
| KI-01 | **Single server, no load balancing.** All services run on one machine. | If the server goes down, the whole system is down. | Daily backups (see RUNBOOK §2). Restart the server, then run RUNBOOK §3.6. | Open — planned: second instance behind load balancer |
| KI-02 | **API restart causes ~10–20 seconds of downtime.** No rolling/blue-green deploy yet. | Users may see `502 Bad Gateway` briefly during restarts or deploys. | Deploy only during the allowed window (Mon–Thu, 9 AM–2 PM). Clients should retry once after 5 seconds. | Open |
| KI-03 | **Purging the `jobs` queue deletes pending jobs.** | Orders can stay `pending` forever after a purge. | After purging, re-queue affected orders (RUNBOOK §3.3, steps 5–6). | Open |
| KI-04 | **JWT tokens cannot be revoked before they expire** (1 hour). | A logged-out or stolen token stays valid until expiry. | Keep expiry short (1 hour). In an emergency, change `JWT_SECRET` and restart the API — this logs out **all** users. | Open |
| KI-05 | **No automatic alerts.** Monitoring is manual (logs + commands). | Problems may go unnoticed outside working hours. | Run the daily checks in RUNBOOK §2 every morning. | Open — planned: uptime monitor on `/health/ready` |
| KI-06 | **Down-migrations that drop columns lose data.** | Rolling back a migration may delete data written after the release. | Always take a backup before deploying (DEPLOYMENT.md Stage 01). Restore with `pg_restore` if needed. | By design |
| KI-07 | **List endpoint is capped at 100 records per page.** | Clients cannot fetch all orders in one call. | Use `page` and `limit` query parameters to paginate. | By design |
| KI-08 | **RabbitMQ management UI uses the default port (15672).** | Security risk if the port is exposed publicly. | Keep port 15672 blocked in the firewall; access it through SSH tunnel only: `ssh -L 15672:localhost:15672 <user>@<server-ip>` | Open |

## Reporting a New Issue

Add a new row with the next ID (`KI-09`, …) and include:

- What happened and when
- How to reproduce it
- The workaround (if any)
- Who is assigned to fix it
