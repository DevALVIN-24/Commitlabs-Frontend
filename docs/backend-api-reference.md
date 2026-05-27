# Backend API Reference

This document describes the HTTP API surface exposed by the frontend backend
(`src/app/api`).  The routes are intentionally thin stubs in the current code
base; they exist primarily for analytics hooks and development/testing.

Each entry includes the HTTP method, path, expected request body (if any), and
an example response.  All endpoints return JSON.

## CORS Summary

- Public browser routes return wildcard CORS without credentials.
- First-party browser routes echo only trusted Commitlabs origins and may allow
  credentials.
- Implemented routes answer `OPTIONS` preflight requests automatically.

See [docs/backend-cors-policy.md](./backend-cors-policy.md) for the full
origin configuration and route classification.

---

## Standard Response Conventions

All endpoints follow these conventions.

### Success Response

```json
{
  "success": true,
  "data": { ... },
  "meta": { ... }       // optional pagination / additional metadata
}
```

### Error Response

```json
{
  "success": false,
  "error": {
    "code": "TOO_MANY_REQUESTS",
    "message": "Too many requests. Please try again later.",
    "retryAfterSeconds": 60  // present on 429 and 503 only
  }
}
```

### Rate Limited Responses (429 / 503)

When a request is rate-limited, the response includes the `Retry-After` HTTP header:

```
HTTP/1.1 429 Too Many Requests
Retry-After: 60
```

| Status | `retryAfterSeconds` default | Meaning |
|--------|---------------------------|---------|
| 429 | 60 s | Client exceeded rate limit |
| 503 | 30 s | Service temporarily unavailable |

Clients should wait the indicated seconds before retrying. See [error-handling.md](./error-handling.md) for the full client retry strategy (exponential backoff + jitter).

---

## `GET /api/commitments/search`

Searches and filters commitments with rich query parameters. Supports filtering
by asset, `CommitmentStatus`, and risk type, with stable sorting and pagination
via the `pagination.ts` utilities. Common queries are cached with a short TTL
(15 s) for performance.

- **Query parameters**:
    - `ownerAddress`: (string, **required**) The Stellar address of the owner.
    - `asset`: (string, optional) Filter by asset code (e.g. "XLM", "USDC"). Case-insensitive.
    - `status`: (enum, optional) Filter by commitment status. Values: `ACTIVE`, `SETTLED`, `VIOLATED`, `EARLY_EXIT`.
    - `riskType`: (enum, optional) Filter by risk type. Values: `Safe`, `Balanced`, `Aggressive`.
    - `minCompliance`: (number, optional) Minimum compliance score (0–100).
    - `page`: (integer, optional, default: 1) Page number.
    - `pageSize`: (integer, optional, default: 10, max: 100) Items per page.
    - `sortBy`: (enum, optional, default: `createdAt`) Sort field. Values: `createdAt`, `amount`, `complianceScore`, `status`, `asset`.
    - `sortOrder`: (enum, optional, default: `desc`) Sort direction. Values: `asc`, `desc`.

- **Response**:
    - `200 OK`: Filtered, sorted, and paginated list of commitments.
    - `400 Bad Request`: Invalid filter or pagination parameters.
    - `429 Too Many Requests`: Rate limit exceeded.

### Example

```bash
curl "http://localhost:3000/api/commitments/search?ownerAddress=GABC...&asset=USDC&status=ACTIVE&sortBy=amount&sortOrder=desc&page=1&pageSize=10"
```

```json
{
  "success": true,
  "data": {
    "data": [
      {
        "commitmentId": "c4",
        "ownerAddress": "GABC...",
        "asset": "USDC",
        "amount": "8000",
        "status": "ACTIVE",
        "riskType": "Safe",
        "complianceScore": 99,
        "currentValue": "8200",
        "feeEarned": "15",
        "violationCount": 0,
        "createdAt": "2026-04-01T00:00:00.000Z",
        "expiresAt": "2026-10-01T00:00:00.000Z"
      }
    ],
    "meta": {
      "page": 1,
      "pageSize": 10,
      "total": 1,
      "totalPages": 1,
      "hasNextPage": false,
      "hasPrevPage": false
    },
    "filters": {
      "asset": "USDC",
      "status": "ACTIVE",
      "riskType": null,
      "minCompliance": null,
      "sortBy": "amount",
      "sortOrder": "desc"
    }
  }
}
```

---

## `POST /api/commitments`

Creates a new commitment on the Stellar network.

- **Headers**:
    - `Idempotency-Key`: (Optional) A unique string to identify the request and prevent duplicate processing. Recommended for safe retries.
- **Request body**:
    - `ownerAddress`: (string, required) The Stellar address of the owner.
    - `asset`: (string, required) The asset code.
    - `amount`: (string, required) The amount to commit.
    - `durationDays`: (number, required) The duration of the commitment in days.
    - `maxLossBps`: (number, required) Maximum loss in basis points.
    - `metadata`: (object, optional) Additional metadata.
- **Response**:
    - `201 Created`: The commitment was successfully created.
    - `409 Conflict`: A request with the same `Idempotency-Key` is already in progress.
    - `429 Too Many Requests`: Rate limit exceeded.

### Example

```bash
curl -X POST http://localhost:3000/api/commitments \
     -H 'Content-Type: application/json' \
     -d '{"asset":"XLM","amount":100}'
```

```json
{
  "message": "Commitments creation endpoint stub - rate limiting applied",
  "ip": "::1"
}
```

---

## `POST /api/commitments/[id]/settle`

Marks the commitment identified by `id` as settled.  Currently a stub that emits
`CommitmentSettled` events.

- **Path parameter**: `id` (string)
- **Request body**: optional JSON payload with additional details.
- **Response**: stub confirmation message.

### Example

```bash
curl -X POST http://localhost:3000/api/commitments/abc123/settle \
     -H 'Content-Type: application/json' \
     -d '{"finalValue":105}'
```

```json
{
  "message": "Stub settlement endpoint for commitment abc123",
  "commitmentId": "abc123"
}
```

---

## `POST /api/commitments/[id]/early-exit`

Triggers an early exit (with penalty) for the named commitment.  Emits
`CommitmentEarlyExit` events.

- **Path parameter**: `id` (string)
- **Request body**: optional JSON with penalty or reason.
- **Response**: stub message.

### Example

```bash
curl -X POST http://localhost:3000/api/commitments/abc123/early-exit \
     -H 'Content-Type: application/json' \
     -d '{"reason":"user-request"}'
```

```json
{
  "message": "Stub early-exit endpoint for commitment abc123",
  "commitmentId": "abc123"
}
```

---

## `POST /api/attestations`

Records an attestation event.  Stub implementation logs
`AttestationReceived`.

- **Request body**: JSON describing the attestation (e.g. signature,
commitmentId).
- **Response**: stub message with requester IP.

### Example

```bash
curl -X POST http://localhost:3000/api/attestations \
     -H 'Content-Type: application/json' \
     -d '{"commitmentId":"abc123","status":"valid"}'
```

```json
{
  "message": "Attestations recording endpoint stub - rate limiting applied",
  "ip": "::1"
}
```

---

## `GET /api/protocol/constants`

Returns the public protocol constants used by UX copy and calculations, including fee parameters, penalty tiers, and commitment limits. This endpoint is public and includes caching headers.

### Example

```bash
curl http://localhost:3000/api/protocol/constants
```

```json
{
  "success": true,
  "data": {
    "protocolVersion": "v1",
    "network": "Test SDF Network ; September 2015",
    "fees": {
      "networkBaseFeeStroops": 100,
      "platformFeePercent": 0
    },
    "penalties": [...],
    "commitmentLimits": { ... },
    "cachedAt": "2026-02-25T00:00:00.000Z"
  }
}
```

---

## `GET /api/metrics`

Simple health/metrics endpoint used by monitoring tools.

- **Response**: JSON object containing uptime, mock request/error counts, and
current timestamp.

### Example

```bash
curl http://localhost:3000/api/metrics
```

```json
{
  "status": "up",
  "uptime": 123.456,
  "mock_requests_total": 789,
  "mock_errors_total": 2,
  "timestamp": "2026-02-25T00:00:00.000Z"
}
```

---

> 🔧 _This reference will grow as the backend implements real business logic._

```
