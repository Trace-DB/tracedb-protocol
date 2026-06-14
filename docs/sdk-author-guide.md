---
title: TraceDB SDK Author Guide
tags:
  - tracedb
  - sdk
  - protocol
status: contract-freeze
type: sdk-author-guide
updated: 2026-06-02
---

# TraceDB SDK Author Guide

This guide describes what every TraceDB SDK must implement to be conformant
with `platform-contract-v0`. It extracts the canonical rules from
`docs/api/v1-http.md` and `docs/platform-contract-v0.md`.

## 1. Retry Policies

### Safe Retries

Safe retries apply **only** to routes or operation payloads that are provably
read-only. SDKs must default safe retries **off**.

**Blanket safe-retry routes** (always safe to retry without idempotency key):

| Method | Route |
| --- | --- |
| `GET` | `/v1/health` |
| `GET` | `/v1/ready` |
| `GET` | `/v1/graphql/schema` |
| `POST` | `/v1/records/get` |
| `POST` | `/v1/records/scan` |
| `POST` | `/v1/query` |
| `POST` | `/v1/explain` |
| `POST` | `/v1/graphql/bounded` |

**Polymorphic routes** (retry only when the payload is provably read-only):

| Method | Route | Read-only condition |
| --- | --- | --- |
| `POST` | `/v1/traceql` | Payload contains `GET`, `SCAN`, `QUERY`, `EXPLAIN`, or `JOBS LIST` |
| `POST` | `/v1/graphql` | Payload uses read-only root fields (`get`, `scan`, `query`, `explain`, `jobs`) |

SDKs should expose a configuration option (e.g., `safe_retries` or
`TRACEDB_SAFE_RETRIES` env var) that controls the maximum number of safe
retry attempts.

### Idempotency Retries

Idempotency retries apply to mutation, admin, and polymorphic native operation
routes **only** when the individual request carries a non-empty
`Idempotency-Key` header. SDKs must default idempotency retries **off**.

SDKs should expose a configuration option (e.g., `idempotency_retries` or
`TRACEDB_IDEMPOTENCY_RETRIES` env var) that controls the maximum number of
idempotency retry attempts.

### Backoff Strategy

SDKs should implement exponential backoff with jitter for both safe and
idempotency retries. The recommended base delay is 100 ms, doubling on each
attempt, with a maximum delay of 10 seconds. Jitter should be ±25% of the
current delay.

### Retryable Status Codes

The following HTTP status codes should trigger a retry:

| Status Code | Meaning | Retry Policy |
| --- | --- | --- |
| `429` | Rate limited | Safe or idempotency retry after backoff |
| `502` | Bad gateway | Safe or idempotency retry after backoff |
| `503` | Service unavailable | Safe or idempotency retry after backoff |
| `504` | Gateway timeout | Safe or idempotency retry after backoff |
| Network timeout / connection error | — | Safe or idempotency retry after backoff |

Note: `409 Conflict` (idempotency key body mismatch) must **never** be retried;
it indicates a client logic error.

### SDK Environment Variables

| Variable | Purpose |
| --- | --- |
| `TRACEDB_SAFE_RETRIES` | Max safe retry attempts (default: 0) |
| `TRACEDB_IDEMPOTENCY_RETRIES` | Max idempotency retry attempts (default: 0) |
| `TRACEDB_TIMEOUT_MS` | Per-request timeout in milliseconds |

## 2. Error Envelope Parsing

### Standard Error Shape

All non-2xx responses from TraceDB use the following JSON envelope:

```json
{
  "error": "<human-readable error string>",
  "code": "<optional machine-readable error code>"
}
```

The `error` field is the compatibility field and is always present. The `code`
field is a stable machine-readable value when the server or gateway can
classify the failure.

### SDK Requirements

Every SDK must:

1. **Preserve the raw response body** for debugging.
2. **Parse the `error` string** and expose it as a structured field
   (e.g., `server_error`).
3. **Parse the optional `code` string** and expose it as a structured field
   (e.g., `server_error_code`).
4. **Expose method, path, and status** when present on the error object.

This is current-envelope ergonomics, not a broader RFC 7807/problem-details
contract.

### Example SDK Error Type

```python
# Python example
class TraceDBError(Exception):
    def __init__(self, message, status, method, path,
                 server_error=None, server_error_code=None,
                 raw_body=None):
        self.status = status
        self.method = method
        self.path = path
        self.server_error = server_error
        self.server_error_code = server_error_code
        self.raw_body = raw_body
        super().__init__(message)
```

### GraphQL Error Envelope

The native `POST /v1/graphql` endpoint uses the standard GraphQL response
envelope:

```json
{
  "data": { ... },
  "errors": [
    {
      "message": "...",
      "extensions": {
        "code": "TRACEDB_GRAPHQL_ERROR"
      }
    }
  ]
}
```

SDKs should expose both the `data` and `errors` fields. Unsupported fields
and operation failures return `errors` with `extensions.code =
"TRACEDB_GRAPHQL_ERROR"`.

## 3. Managed Routing

### Overview

Managed routing allows SDKs to target a specific database and branch through
a TraceDB gateway. The mechanism injects `database_id` and `branch_id` fields
into JSON POST request bodies before they are sent to the server.

### When to Set Managed Routing Fields

SDKs should inject managed routing metadata when:

1. The SDK is configured with a `database_id` (and optionally `branch_id`).
2. The route is a POST endpoint that accepts a JSON body.
3. The request body is an object shape (not a scalar or array).

**Do not** inject managed routing fields for:

- GET routes (they send no JSON body).
- Requests where the caller has already explicitly set `database_id` and/or
  `branch_id` in the body (explicit fields always win).

### How Injection Works

```text
1. SDK receives a POST request body from the caller.
2. If the body is an object and `database_id` is configured:
   a. If the body does not already have `database_id`, copy `database_id` into it.
   b. If `branch_id` is configured and the body does not have `branch_id`,
      copy `branch_id` into it.
   c. If `database_id` is configured but `branch_id` is not,
      default `branch_id` to `<database_id>:main`.
3. The caller's original object must not be mutated. Work on a shallow copy.
```

### Endpoints That Support Managed Routing

All POST endpoints that accept object-shaped bodies support managed routing.
The following are the primary endpoints:

| Method | Route | Body Type |
| --- | --- | --- |
| `POST` | `/v1/schema/apply` | `TableSchema` |
| `POST` | `/v1/records/put` | `RecordInput` or `{ "record": RecordInput }` |
| `POST` | `/v1/records/put-batch` | `RecordPutBatchRequest` |
| `POST` | `/v1/records/patch` | `RecordPatchRequest` |
| `POST` | `/v1/records/delete` | `RecordDeleteRequest` |
| `POST` | `/v1/records/get` | `RecordGetRequest` |
| `POST` | `/v1/records/scan` | `RecordScanRequest` |
| `POST` | `/v1/query` | `HybridQuery` |
| `POST` | `/v1/explain` | `HybridQuery` |
| `POST` | `/v1/traceql` | `TraceQlQueryRequest` |
| `POST` | `/v1/graphql` | `GraphQlQueryRequest` |
| `POST` | `/v1/graphql/bounded` | `GraphQlQueryRequest` |
| `POST` | `/v1/admin/compact` | `{}` |
| `POST` | `/v1/admin/snapshot` | `SnapshotRequest` |
| `POST` | `/v1/admin/restore` | `RestoreRequest` |

For `POST /v1/records/put`, SDKs may inject managed-routing fields into either
accepted body shape. The wrapped form is `{ "record": RecordInput,
"database_id"?: string, "branch_id"?: string }`; it is closed to other
wrapper-level fields so generated SDK validators preserve a stable method
signature. Caller-specific data belongs inside the record payload, not next to
`record`.

### Gateway-Only: Bodyless Admin Routes

The following GET routes do not have a JSON body, so managed routing
metadata cannot be injected into the request body:

| Method | Route |
| --- | --- |
| `GET` | `/v1/admin/jobs` |

For these bodyless routes, SDKs must send managed routing metadata as
query parameters using the following canonical names:

| Parameter | Type | Description |
| --- | --- | --- |
| `database_id` | `string` | Canonical managed-routing database identifier |
| `branch_id` | `string` | Canonical managed-routing branch identifier |

SDKs **must** use these exact parameter names (not `db_id`, `databaseId`,
`br_id`, `branchId`, or any other variant) so that all SDKs target the
same gateway routing key. These names match the field names used in JSON
POST body injection (`database_id` and `branch_id`) for consistency
across body and bodyless routes.

When `database_id` is configured and the route is bodyless:

1. Append `database_id` as a query parameter.
2. If `branch_id` is configured, append `branch_id` as a query parameter.
3. If `database_id` is configured but `branch_id` is not, default
   `branch_id` to `<database_id>:main` and append it as a query parameter.
4. Never override query parameters that the caller has already explicitly set.

### Configuration via Environment

SDKs should support the following environment variables for managed routing:

| Variable | Purpose |
| --- | --- |
| `TRACEDB_DATABASE_ID` | Default `database_id` for managed routing |
| `TRACEDB_BRANCH_ID` | Default `branch_id` for managed routing |
| `TRACEDB_URL` | Base URL of the TraceDB endpoint |
| `TRACEDB_TOKEN` | Authentication token |

The `from_env()` / `fromEnv()` pattern should read these variables and
configure the client accordingly.

### SDK Configuration Pattern

Every SDK should provide a configuration object or class that:

1. Accepts `database_id`, `branch_id`, `url`, and `token` as constructor
   arguments.
2. Provides a `from_env()` static/class method that reads from environment
   variables.
3. Applies managed routing body injection transparently on every POST request
   when `database_id` is configured.
4. Never mutates the caller's request body object.
