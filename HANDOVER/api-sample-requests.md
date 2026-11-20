# API Sample Requests

Full spec: [`openapi.yaml`](openapi.yaml) (open it in https://editor.swagger.io to view it nicely).

**Base URL:** `https://<your-domain>/api/v1` (local: `http://localhost:3000/api/v1`)
**Auth:** Send `Authorization: Bearer <token>` on every endpoint except `/health`, `/health/ready`, and `/auth/login`.

---

## Documented Endpoint: `POST /api/v1/orders` — Create an Order

**What it does:** Saves a new order to the database with status `pending`, then publishes an `order.created` job to the `jobs` queue so the worker can process it.

### Request

```http
POST /api/v1/orders HTTP/1.1
Host: <your-domain>
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json

{
  "item": "Wireless Mouse",
  "quantity": 2
}
```

| Field | Type | Required | Rules |
|---|---|---|---|
| `item` | string | ✅ | 1–100 characters |
| `quantity` | integer | ✅ | 1–999 |

**cURL:**

```bash
curl -X POST https://<your-domain>/api/v1/orders \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"item":"Wireless Mouse","quantity":2}'
```

### Success Response — `201 Created`

```json
{
  "id": 101,
  "item": "Wireless Mouse",
  "quantity": 2,
  "status": "pending",
  "createdAt": "2026-11-20T10:15:00Z"
}
```

After a few seconds, `GET /api/v1/orders/101` should show `"status": "processed"` (the worker finished the job).

### Error Responses

| Status | Code | When it happens | Example body |
|---|---|---|---|
| `400` | `VALIDATION_ERROR` | Missing field, or `quantity` < 1 | `{"error":{"code":"VALIDATION_ERROR","message":"Request body is invalid","details":["quantity must be at least 1"]}}` |
| `401` | `UNAUTHORIZED` | No token, wrong token, or expired token | `{"error":{"code":"UNAUTHORIZED","message":"Missing or invalid token"}}` |
| `503` | `SERVICE_UNAVAILABLE` | Database or RabbitMQ is down | `{"error":{"code":"SERVICE_UNAVAILABLE","message":"Message queue is unreachable, try again later"}}` |

**Example error request (invalid quantity):**

```bash
curl -i -X POST https://<your-domain>/api/v1/orders \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"item":"Wireless Mouse","quantity":0}'
```

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request body is invalid",
    "details": ["quantity must be at least 1"]
  }
}
```

---

## Other Sample Requests

**Log in (get a token):**

```bash
curl -X POST https://<your-domain>/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"user@example.com","password":"secret123"}'
# → 200 {"token":"eyJhbGciOi...","expiresIn":3600}
# → 401 {"error":{"code":"UNAUTHORIZED","message":"Invalid email or password"}}
```

**List orders:**

```bash
curl "https://<your-domain>/api/v1/orders?page=1&limit=20" \
  -H "Authorization: Bearer <token>"
# → 200 {"data":[...],"page":1,"total":42}
```

**Get one order:**

```bash
curl https://<your-domain>/api/v1/orders/101 -H "Authorization: Bearer <token>"
# → 200 {"id":101,...}
# → 404 {"error":{"code":"NOT_FOUND","message":"Order 101 not found"}}
```

**Health checks (no token needed):**

```bash
curl https://<your-domain>/health         # → 200 {"status":"ok","version":"v1.5.0"}
curl https://<your-domain>/health/ready   # → 200 {"status":"ok","db":"up","queue":"up"}
```

## Error Code Reference

| HTTP | Code | Meaning |
|---|---|---|
| 400 | `VALIDATION_ERROR` | Bad request body or parameters |
| 401 | `UNAUTHORIZED` | Missing / invalid / expired token |
| 404 | `NOT_FOUND` | Record does not exist |
| 500 | `INTERNAL_ERROR` | Unexpected server error — check API logs |
| 503 | `SERVICE_UNAVAILABLE` | Database or queue is down — see RUNBOOK |
