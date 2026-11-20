# HANDOVER Package

Everything a new team needs to run, maintain, and fix `<Project Name>` without asking the original developers.

| # | Document | File | What it covers |
|---|---|---|---|
| 1 | Architecture Diagram | [`architecture-diagram.md`](architecture-diagram.md) | Visual map of services, ports, and data flow |
| 2 | API Documentation | [`openapi.yaml`](openapi.yaml) + [`api-sample-requests.md`](api-sample-requests.md) | OpenAPI spec, sample requests, responses, error codes |
| 3 | Operations Runbook | [`RUNBOOK.md`](RUNBOOK.md) | Exact commands for daily ops and live incidents |
| 4 | Known Issues | [`KNOWN_ISSUES.md`](KNOWN_ISSUES.md) | Current limitations and workarounds |

**Deployment plan:** see [`../DEPLOYMENT.md`](../DEPLOYMENT.md).

## Quick Start (for the new team)

```bash
git clone <repo-url>
cd <project-folder>
cp .env.example .env          # then fill in real values
docker compose up -d          # starts postgres → rabbitmq → api → worker
curl http://localhost:3000/health/ready
```

## Contacts

| Role | Name | Contact |
|---|---|---|
| Release Lead | `<Name>` | `<email>` |
| Backend / API | `<Name>` | `<email>` |
| Database | `<Name>` | `<email>` |
