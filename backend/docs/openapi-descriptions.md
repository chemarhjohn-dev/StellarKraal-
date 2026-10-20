# StellarKraal API — Endpoint Reference

This document provides a human-readable reference for all endpoints defined in
`openapi.json`. It mirrors the descriptions in the spec and is intended for
quick navigation during development and code review.

## API Versioning

All endpoints are served under a versioned base path. The current stable
version is **`/api/v1`**, which is the `servers[].url` base path declared in
`openapi.json`. Every path listed below is relative to that base path — e.g.
the health check is reachable at `/api/v1/health`.

### Version router pattern

Routes are mounted through a version router so that new versions can be added
independently without touching existing handlers:

```
/api/v1  ->  v1 router  (all routes in this document)
/api/v2  ->  v2 router  (added independently during a migration period)
```

To introduce `/api/v2`, add a new version router and mount it alongside `v1`;
both versions coexist during the migration window. Existing clients keep using
`/api/v1` unchanged.

### Unversioned requests

Requests to `/api` (without a version segment) are **not** routed to any
handler. They return `404` with a message pointing to the versioned path, e.g.:

```json
{ "error": "API version required. Use /api/v1/...", "code": "API_VERSION_REQUIRED" }
```

## Authentication (`auth`)

| Method | Path | Summary |
|--------|------|---------|
| GET | `/auth/challenge` | Get a one-time challenge |
| POST | `/auth/login` | Login with Stellar signature |
| POST | `/auth/refresh` | Rotate refresh token |

## Health (`health`)

| Method | Path | Summary |
|--------|------|---------|
| GET | `/health` | Service health check |
| GET | `/health/live` | Kubernetes liveness probe |
| GET | `/health/ready` | Kubernetes readiness probe |
| GET | `/health/deep` | Deep infrastructure health check |
| GET | `/health/{loanId}` | Get loan health factor |

## Collateral (`collateral`)

| Method | Path | Summary |
|--------|------|---------|
| POST | `/collateral/register` | Register livestock collateral (on-chain) |
| POST | `/collateral/register/batch` | Batch register livestock collateral (on-chain) |
| POST | `/collateral` | Create collateral record (DB) |
| GET | `/collateral` | List collateral records |
| GET | `/collateral/{id}` | Get collateral by ID |
| PUT | `/collateral/{id}/appraise` | Update appraised value |
| PATCH | `/collateral/{id}` | Partially update collateral fields |
| DELETE | `/collateral/{id}` | Soft-delete a collateral record |
| PATCH | `/collateral/{id}/restore` | Restore soft-deleted collateral |

### `PATCH /collateral/{id}` — Partial update

Partially updates a collateral record. Only the owner of the collateral or an
admin may patch it; other callers receive `403 Forbidden`.

**Patchable fields** (all optional, at least one required):

| Field | Type | Notes |
|-------|------|-------|
| `weight` | number | Changing this queues a new appraisal |
| `healthStatus` | string | Changing this queues a new appraisal |
| `notes` | string | Free-form health notes; does not affect appraisal |

**Behaviour**

- Returns `400 Bad Request` when the request body contains no patchable fields.
- When `weight` or `healthStatus` changes, a new appraisal is queued for the
  collateral record.
- Returns the updated collateral record on success.

## Loans (`loan`)

| Method | Path | Summary |
|--------|------|---------|
| POST | `/loan/request` | Request a new loan |
| POST | `/loan/repay` | Repay a loan |
| POST | `/loan/liquidate` | Liquidate an undercollateralised loan |
| POST | `/loan/repayment-preview` | Preview repayment breakdown |
| GET | `/loan/{id}` | Fetch a loan record |
| DELETE | `/loan/{id}` | Soft-delete a loan record |
| GET | `/loans` | List loans (paginated) |
| GET | `/loans/summary` | Get borrower loan portfolio summary |

### Idempotency (`Idempotency-Key`)

`POST /loan/repay` (and the versioned alias `POST /api/v1/loans/{id}/repay`)
accepts an optional `Idempotency-Key` request header. When supplied, the server
records the response for that key and replays it on any subsequent request that
reuses the same key, preventing duplicate repayments caused by network retries.

| Header | Required | Description |
|--------|----------|-------------|
| `Idempotency-Key` | No | Client-generated unique key (e.g. a UUID). Reusing a key returns the cached response (original status code) instead of creating a new repayment. Keys expire after 24 hours. |

## Oracle (`oracle`)

| Method | Path | Summary |
|--------|------|---------|
| POST | `/oracle/price-update` | Invalidate appraisal cache |

## Webhooks (`webhooks`)

| Method | Path | Summary |
|--------|------|---------|
| POST | `/webhooks` | Register a webhook |
| DELETE | `/webhooks/{id}` | Delete a webhook |

## Admin (`admin`)

| Method | Path | Summary |
|--------|------|---------|
| POST | `/alerts/webhook` | AWS SNS alert receiver |
| GET | `/admin/webhooks` | List registered webhooks |
| GET | `/admin/webhooks/logs` | List webhook delivery logs |
| GET | `/admin/migrations/status` | Migration status |
| GET | `/admin/deleted/collateral` | List soft-deleted collateral |
| POST | `/admin/restore/collateral/{id}` | Restore soft-deleted collateral |
| GET | `/admin/deleted/loans` | List soft-deleted loans |
| POST | `/admin/restore/loans/{id}` | Restore soft-deleted loan |
| GET | `/admin/users` | List borrowers |
| GET | `/admin/moderation-queue` | Pending-review loans |
| GET | `/admin/statistics` | Aggregate platform statistics |

## Transactions (`transactions`)

| Method | Path | Summary |
|--------|------|---------|
| GET | `/transactions` | List transactions |
| GET | `/transactions/{id}` | Get transaction by ID |
| GET | `/transactions/{hash}/status` | Poll transaction status on-chain |

## User Settings (`settings`)

| Method | Path | Summary |
|--------|------|---------|
| GET | `/settings/{wallet}` | Get user settings |
| PUT | `/settings/{wallet}` | Update user settings |

---

### Error Response Envelope

All error responses follow the `{ error, code, details? }` JSON envelope
defined by the `ErrorResponse` schema in `openapi.json`. Component-level error
responses (`ValidationError`, `Unauthorized`, `NotFound`, `RateLimited`,
`InternalError`) include example payloads for use in Swagger UI and
client SDK generation.

### Documentation Source of Truth

- **OpenAPI spec**: [`openapi.json`](../openapi.json) — `servers[].url` is
  `/api/v1`, so all paths are versioned under that base path.
- **Swagger UI**: available at the server root when `NODE_ENV !== 'production'`
- **This file**: mirrors the spec for quick reference; always validate against
  the canonical `openapi.json`.
