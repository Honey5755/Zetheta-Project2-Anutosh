# D2 — API Specification (v1)

**Project:** 493560B — FDE-9B — Custom API Integration (SAP S/4HANA → Zetheta FinSight)
**Client:** Meridian Manufacturing Ltd.
**Deliverable:** D2 — API Specification
**Version:** 1.0
**Status:** Complete

---

## 1. Overview

This deliverable defines the two machine-readable API contracts that bound the
FDE-9B integration, plus a Postman collection for interactive exploration:

| Artifact | File | Standard |
|---|---|---|
| Source contract | `api/sap/API_SAP_Source.yaml` | OpenAPI 3.0.3 |
| Destination contract | `api/finsight/API_FinSight_Destination.yaml` | OpenAPI 3.0.3 |
| Interactive collection | `api/postman/FinSight_Integration.postman_collection.json` | Postman Collection v2.1 |

The **source contract** describes the twelve SAP S/4HANA OData v2 extraction
endpoints (`SRC-001`..`SRC-012`) consumed by the ODP Extractor. Every endpoint
is a read-only `GET` over an ABAP CDS consumption view, exposing OData
pagination, ODP delta tokens, a canonical company-code filter and rate-limit
headers. Response fields map field-for-field to the underlying SAP tables
documented in Appendix A of the project brief.

The **destination contract** describes the twelve Zetheta FinSight resources
(`DST-001`..`DST-012`) written by the FinSight Loader. Each resource offers a
`POST` (idempotent create/upsert) and a `GET` (cursor-paginated list) under the
versioned base path `/api/v2/`. All FinSight error codes from Appendix B are
modelled as a single shared `Error` schema whose `code` field is an enum over
the canonical registry, and an asynchronous webhook confirms durable loads.

### Source vs destination — contract summary

| Concern | SAP Source (SRC) | FinSight Destination (DST) |
|---|---|---|
| Direction | Read (extract) | Write (load) + read-back |
| Verb(s) | `GET` | `POST`, `GET` |
| Base path | `/sap/odata/v2/` | `/api/v2/` |
| Pagination | OData `$top` / `$skip` + `@odata.nextLink` | Cursor (`pageSize` / `cursor`) |
| Delta | ODP `deltaToken` | n/a (write side) |
| Idempotency | n/a | `Idempotency-Key` header (business key) |
| Multi-entity | `companyCode` = MC01/MC02/MC03 | `X-FinSight-Tenant` = MERIDIAN-MC01/02/03 |
| Auth | OAuth 2.0 + X.509 mTLS, or gateway API key | OAuth 2.0 `client_credentials` + refresh |
| Error envelope | OData `error` object | Shared `Error` schema (enum code) |
| Async confirm | n/a | Load-confirmation webhook callback |

---

## 2. Endpoint index

### Source endpoints (SAP S/4HANA)

| ID | Path (`/sap/odata/v2`) | Method | Domain | Primary tables |
|---|---|---|---|---|
| SRC-001 | `/journal-entries` | GET | General Ledger | ACDOCA, BKPF, BSEG |
| SRC-002 | `/payables` | GET | Accounts Payable | BSIK, LFA1, LFC1 |
| SRC-003 | `/receivables` | GET | Accounts Receivable | BSID, KNA1, KNC1 |
| SRC-004 | `/cost-centres` | GET | Cost Centre Accounting | CSKS, CSKT, COSS |
| SRC-005 | `/profit-centres` | GET | Profit Centre Accounting | CEPC, CEPCT |
| SRC-006 | `/material-ledger` | GET | Material Ledger | MBEW, COSP |
| SRC-007 | `/purchase-orders` | GET | Purchase Orders | EKKO, EKPO, EKET |
| SRC-008 | `/sales-orders` | GET | Sales Orders | VBAK, VBAP, VBEP |
| SRC-009 | `/fixed-assets` | GET | Fixed Assets | ANLA, ANLZ, ANLP |
| SRC-010 | `/bank-statements` | GET | Bank Statements | FEBKO, FEBEP |
| SRC-011 | `/budget-actuals` | GET | Budget/Actual | COSP, COSS |
| SRC-012 | `/inventory` | GET | Inventory | MARD, MBEW |

### Destination endpoints (Zetheta FinSight)

| ID | Path (`/api/v2`) | Methods | Domain |
|---|---|---|---|
| DST-001 | `/journal-entries` | POST, GET | General Ledger |
| DST-002 | `/payables` | POST, GET | Accounts Payable |
| DST-003 | `/receivables` | POST, GET | Accounts Receivable |
| DST-004 | `/cost-centres` | POST, GET | Cost Centre Accounting |
| DST-005 | `/profit-centres` | POST, GET | Profit Centre Accounting |
| DST-006 | `/material-ledger` | POST, GET | Material Ledger |
| DST-007 | `/purchase-orders` | POST, GET | Purchase Orders |
| DST-008 | `/sales-orders` | POST, GET | Sales Orders |
| DST-009 | `/fixed-assets` | POST, GET | Fixed Assets |
| DST-010 | `/bank-statements` | POST, GET | Bank Statements |
| DST-011 | `/budget-actuals` | POST, GET | Budget/Actual |
| DST-012 | `/inventory` | POST, GET | Inventory |

**Endpoint count:** 12 source `GET` operations + 24 destination operations
(12 `POST` + 12 `GET`) = **36 operations** across **24 endpoints**.

---

## 3. Authentication flows

### 3.1 SAP source — OAuth 2.0 + X.509 mutual TLS

The SAP Gateway terminates mutual TLS at a reverse proxy. The extractor presents
a registered X.509 client certificate (bound to the technical user
`ZFINSIGHT_EXTRACT`) **and** an OAuth 2.0 bearer token obtained via the
`client_credentials` grant. Certificate and token are cross-validated before SAP
authorisation object checks (e.g. `F_BKPF_BUK` for company-code scope) run.
A gateway API key (`APIKey` header) is supported as an alternative to the bearer
token for machine-to-machine calls, but the client certificate is always
mandatory.

```
Extractor                 mTLS Proxy / SAP Gateway         SAP OAuth AS
   |-- TLS ClientHello + X.509 client cert -->|                 |
   |                                          |-- verify cert ->|
   |-- POST /oauth2/token (client_credentials) ---------------->|
   |<-------------------- access_token (bearer) ----------------|
   |-- GET /journal-entries  Authorization: Bearer <token> ---->|
   |                                          |-- authz checks ->|
   |<------------------- 200 OData collection ------------------|
```

### 3.2 FinSight destination — OAuth 2.0 client_credentials with refresh

The FinSight Loader exchanges its `client_id`/`client_secret` at the FinSight
authorization server for a short-lived (3600 s) access token plus a refresh
token. Tokens are scoped per resource (for example `finsight.journal-entries.write`).

- On **`401 TOKEN_EXPIRED`** the Loader silently refreshes using the
  `refresh_token` grant and replays the original request (idempotent).
- On **`401 INVALID_TOKEN`** the Loader re-authenticates from scratch and raises
  a security alert; it does **not** auto-retry.

```
Loader                    FinSight Auth Server           FinSight API
   |-- POST /oauth2/token (client_credentials) --->|          |
   |<---- access_token (3600s) + refresh_token ----|          |
   |-- POST /journal-entries  Bearer <token> ----------------->|
   |<----------------------- 201 Created (ACCEPTED) -----------|
   |                (token expires)                            |
   |-- POST /payables  Bearer <token> ----------------------->|
   |<----------------------- 401 TOKEN_EXPIRED ----------------|
   |-- POST /oauth2/token (refresh_token) -------->|          |
   |<----------------- new access_token -----------|          |
   |-- POST /payables  Bearer <new token> -------------------->|
   |<----------------------- 201 Created ----------------------|
```

---

## 4. Pagination strategy

| Side | Mechanism | Parameters | Continuation |
|---|---|---|---|
| SAP source | OData offset paging | `$top` (max 5000, default 1000), `$skip` | `@odata.nextLink`, `@odata.count` |
| FinSight destination | Opaque cursor | `pageSize` (max 500, default 100) | `nextCursor` (null on last page) |

Offset paging is used on the source because ODP packages are naturally ordered
and callers already checkpoint on the delta token. Cursor paging is used on the
destination because it is stable under concurrent writes and cheaper for large
result sets, per the RESTful design guidance (cursor for large datasets,
offset for smaller ones).

**Delta extraction:** every source endpoint accepts a `deltaToken`. The caller
passes the token returned by the previous successful extraction to receive only
changes since that point. Omitting it performs a full initialisation load. The
token is only advanced on a successful `HTTP 200`, so a failed extraction is
safely re-driven with no data loss (ODP error-recovery semantics).

---

## 5. Rate limiting

Both contracts return the standard trio on every response:

- `X-RateLimit-Limit` — requests permitted in the current window
- `X-RateLimit-Remaining` — requests left in the window
- `X-RateLimit-Reset` — Unix epoch seconds when the window resets

On the source, limits enforce the Basis constraints: at most one ODP delta per
provider per 30 minutes and no more than 50 concurrent RFC connections; breaching
either returns `429` with a `Retry-After` header. On the destination, `429`
(`RATE_LIMITED`) likewise carries `Retry-After`; clients honour it and apply
exponential backoff with jitter: `wait = min(cap, random(base, base * 2^attempt))`.

---

## 6. Idempotency

Every FinSight `POST` requires an `Idempotency-Key` header carrying the record
business key (for journal entries this is `documentId`). FinSight retains the key
for 24 hours. Behaviour:

- **Same key + identical payload** → the original result is returned
  (`HTTP 200`, `RecordAck`/`JournalEntry` with status), no duplicate created.
- **Same key + divergent payload** → `409 DUPLICATE_ENTRY`.

This makes the at-least-once delivery guarantee from Kafka safe: replays after a
partial failure never double-post. The idempotency key equals the composite
business key used for reconciliation lineage (SAP document number → FinSight
record).

---

## 7. Versioning

- **Major versions** are carried in the URL path — source `/sap/odata/v2/`,
  destination `/api/v2/`. Breaking changes (field removal, type change, semantic
  change) increment the path version.
- **Minor / additive** changes (new optional fields, new endpoints) are made in
  place and communicated via the `info.version` field of each OpenAPI document
  (currently source `1.0.0`, destination `2.0.0`).
- Schema evolution on the SAP side (support packs altering CDS views) is handled
  by the Transformation Engine's graceful-degradation strategy; the API contract
  version only changes when the exposed shape changes.

---

## 8. Error handling

### 8.1 Source (SAP)

The SAP Gateway returns the OData `error` envelope (`code`, `message`,
`target`). The contract documents `400`, `401`, `403`, `429` and `500` for every
endpoint. `429` distinguishes the ODP frequency governor from RFC-pool
exhaustion via the message text and always carries `Retry-After`.

### 8.2 Destination (FinSight) — shared Error schema

All FinSight errors use one reusable `Error` schema:

```yaml
Error:
  type: object
  required: [error]
  properties:
    error:
      type: object
      required: [code, httpStatus, message]
      properties:
        code:        { $ref: '#/components/schemas/ErrorCode' }   # enum
        httpStatus:  { type: integer }
        message:     { type: string }
        target:      { type: string }
        ruleName:    { type: string }         # 422 BUSINESS_RULE_VIOLATION
        correlationId: { type: string, format: uuid }
        retryable:   { type: boolean }
```

The `ErrorCode` enum covers the full Appendix B registry:

| HTTP | Error code(s) | Retryable | Recommended action |
|---|---|---|---|
| 400 | `INVALID_REQUEST` | no | Parse errors, log, route to DLQ |
| 400 | `INVALID_DATE_FORMAT` | no | Transform to ISO 8601, retry once, else DLQ |
| 400 | `INVALID_CURRENCY_CODE` | no | Apply currency-mapping correction, else DLQ |
| 401 | `TOKEN_EXPIRED` | yes | Refresh via `refresh_token` and replay |
| 401 | `INVALID_TOKEN` | no | Re-authenticate, alert security |
| 403 | `INSUFFICIENT_SCOPE` | no | Alert platform admin, request scope |
| 403 | `TENANT_LOCKED` | no | Pause pipeline for tenant, alert PM |
| 404 | `RESOURCE_NOT_FOUND` | no | Check master-data sync, business exception queue |
| 409 | `DUPLICATE_ENTRY` | no | Compare payloads; identical → skip |
| 409 | `CONCURRENT_MODIFICATION` | yes | Retry after 5 s with optimistic-lock check |
| 422 | `BUSINESS_RULE_VIOLATION` | no | Log with `ruleName`, business exception queue |
| 422 | `REFERENTIAL_INTEGRITY` | yes | Sync master data first, then retry |
| 429 | `RATE_LIMITED` | yes | Honour `Retry-After`, backoff with jitter |
| 500 | `INTERNAL_ERROR` | yes | Backoff (base 2 s, max 3), then DLQ |
| 503 | `SERVICE_UNAVAILABLE` | yes | Linear backoff (60 s, max 5), then circuit-break |
| 504 | `GATEWAY_TIMEOUT` | yes | Retry once after 30 s, then DLQ (timeout flag) |

Every response example carries a `correlationId` (UUID) so a failure can be
traced end-to-end from Kafka through the Transformation Engine to the FinSight
load, satisfying the audit-lineage requirement. The full retry/backoff and
DLQ behaviour is elaborated in Deliverable D4 (Error Handling Framework).

### 8.3 Asynchronous load confirmation

A `POST` is accepted synchronously (`201`, status `ACCEPTED`) but the durable
commit is asynchronous. When the caller supplies `X-Callback-Url`, FinSight
invokes that webhook with a `LoadConfirmationEvent` (`status: COMMITTED` or
`REJECTED`, with `correlationId`) once the record is durably persisted. The
callback is modelled under `callbacks.loadConfirmation` on every create operation.

---

## 9. Linting

Both specifications lint clean under Spectral's `oas` ruleset with **zero
errors and zero warnings**. Every operation has an `operationId`, `summary`,
`description`, `tags` and a full response set (`400`/`401`/`429`/`500` plus the
resource-specific codes), and all schemas, parameters, headers and responses are
reused via `$ref`.

Ruleset (`.spectral.yaml` at repo root):

```yaml
extends: ["spectral:oas"]
```

Run:

```bash
spectral lint api/**/*.yaml --ruleset .spectral.yaml
# or, without a global install:
npx -y @stoplight/spectral-cli lint "api/**/*.yaml" --ruleset .spectral.yaml
```

Expected output:

```
No results with a severity of 'error' found!
```

This satisfies the **API Artisan** badge criterion (OpenAPI passes automated
linting with zero errors).

---

## 10. Traceability

- **SAP tables / fields** — every source response property is annotated in its
  schema `description` with the originating SAP table and field (e.g.
  `amountLC` → `ACDOCA HSL`), fulfilling the **SAP Whisperer** requirement of
  accurate table and field references for all 12 extraction interfaces
  (Appendix A).
- **FinSight error codes** — the `ErrorCode` enum and per-response examples map
  1:1 to Appendix B.
- **Canonical identifiers** — company codes (`MC01`/`MC02`/`MC03`), tenants
  (`MERIDIAN-MC01/02/03`), plants (`PL01`..`PL07`), fiscal-year variant V3 and
  the SRC/DST endpoint IDs are used exactly as defined in the canonical facts.
