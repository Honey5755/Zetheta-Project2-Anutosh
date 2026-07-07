# D7 — Integration Test Plan (v1)

**Project:** FDE-9B — Custom API Integration (SAP S/4HANA → Zetheta FinSight)
**Project code:** 493560B · **Client (simulated):** Meridian Manufacturing Ltd.
**Author:** Anutosh Mishra · **Role:** Forward Deployed Engineer
**Deliverable:** D7 — Integration Test Plan
**Status:** Complete (25 mandatory scenarios) · **Version:** 1.0

> Companion artifacts: the ten functional scenarios are also authored as individual one-page files
> under `tests/` (`TST_Functional_001.md` … `TST_Functional_010.md`) following the Zetheta naming
> convention `TST_{Category}_{ID}.md`.

---

## 1. Test strategy

### 1.1 Objectives

Prove that the integration moves financial data from SAP S/4HANA to Zetheta FinSight **completely,
accurately, on time, and resiliently**, across all 10 domains and 3 legal entities (MC01–MC03),
while respecting Meridian's real-world constraints (RFC pool ≤ 50, ODP ≤ once/30 min, bandwidth ≤
25 % business hours, India-only residency). Testing is organised so that every SLA and data-quality
target in Canonical Facts §8 has at least one scenario that verifies it.

### 1.2 Test levels and types

| Level / type | Coverage | Scenarios |
|---|---|---|
| **Functional** | End-to-end correctness of each domain flow and cross-cutting transform (currency, fiscal period, hierarchy, tenant routing, reconciliation) | `TST-FNC-001..010` |
| **Non-functional (NFR)** | Performance, concurrency, latency, scalability, endurance | `TST-NFR-001..005` |
| **Failure injection (FLR)** | Chaos / resilience: SAP, FinSight, Kafka, malformed data, network partition | `TST-FLR-001..005` |
| **Security (SEC)** | Token lifecycle, unauthorised access, encryption | `TST-SEC-001..003` |
| **Reconciliation (REC)** | Break detection and referential integrity | `TST-REC-001..002` |

Total mandatory: **25 scenarios**. Extension ideas to reach 30+ are catalogued in §7.

### 1.3 Entry / exit criteria

- **Entry:** SAP sandbox seeded with representative data; ODP subscriptions active; FinSight sandbox
  tenants `MERIDIAN-MC01/02/03` provisioned; `TCURR` rates loaded; monitoring stack (D6) live so
  test runs are observable; all mappings (D3) and error framework (D4) deployed to the test env.
- **Exit:** 100 % of P1/P2 scenarios pass; ≥ 95 % of P3 pass; zero unresolved data-loss defects; all
  reconciliation scenarios (`TST-FNC-010`, `TST-REC-001/002`) show zero unexplained breaks; SLA
  targets (freshness ≤ 4 h, completeness > 99.5 %, accuracy > 99.9 %, consistency 100 %) met under
  the NFR load runs.

### 1.4 Roles

FDE (test design + automation), SAP Basis (sandbox data, ODP, `RGGBS000` window), Zetheta Platform
Engineer (FinSight sandbox, rate limits), Internal Audit (reconciliation sign-off).

---

## 2. Test environments

| Environment | Purpose | SAP source | FinSight target | Notes |
|---|---|---|---|---|
| **DEV** | Component + mapping unit tests | SAP sandbox (PRD-like client, masked) | FinSight dev tenant | Fast feedback; synthetic data |
| **SIT (System Integration Test)** | The 25 scenarios run here | SAP S/4HANA sandbox, Client 100-equivalent, FY variant V3 | `MERIDIAN-MC01/02/03` sandbox tenants | Full pipeline: ODP → Kafka (3 brokers) → Transform → Loader |
| **NFR / Perf** | Load, endurance, scalability | Scaled sandbox with 500K+ record generators | Rate-limited FinSight perf tenant | Isolated so load tests don't skew SIT metrics |
| **Chaos** | Failure injection (FLR) | SIT clone + fault injectors | SIT clone tenants | Toxiproxy / broker-kill / RFC-drop tooling |

All environments run in AWS `ap-south-1` to keep test data (masked but India-origin) within the RBI
residency boundary. Every run is tagged with a `correlation_id` prefix (`TST-<id>-<run>`) so the D6
dashboards and ELK logs can be filtered per scenario.

---

## 3. Test data requirements

| Data set | Content | Used by |
|---|---|---|
| GL postings | 100 balanced journal entries (ACDOCA/BKPF/BSEG), multi-currency, periods 001–016 | FNC-001, FNC-005, FNC-010 |
| AP open items | 50 items in INR/USD/EUR with ageing spread; vendors in LFA1 | FNC-002 |
| Master data | Cost-centre & profit-centre masters, 7-level SETNODE/SETLEAF hierarchy | FNC-003, FNC-006, REC-002 |
| Multi-entity batch | Mixed MC01/MC02/MC03 documents | FNC-004 |
| P2P chain | Linked PO→GR→Invoice→Payment (EKKO/EKPO, MIGO, BSIK, payment) | FNC-007 |
| Bank statement | FINSTA IDoc, 50 items (matched + unmatched) | FNC-008 |
| Budget vs actual | COSP/COSS budget + actuals for 5 cost centres | FNC-009 |
| Exchange rates | `TCURR` KURST=M, point-in-time by GDATU | FNC-002, FNC-005 |
| Volume generators | 1K→100K→500K record synthetic feeds | NFR-001..005 |
| Fault fixtures | 10 malformed records; deliberate INR 1.50 variance; orphan cost-centre reference | FLR-004, REC-001, REC-002 |

Data is masked/synthetic; no production PII. GST breakdown (CGST/SGST/IGST) preserved on invoice
fixtures so transform tests can assert it survives.

---

## 4. Functional test scenarios (TST-FNC-001 .. 010)

Standardised format: **ID · Category · Title · Preconditions · Steps · Expected result · Priority.**

---

### TST-FNC-001 — Happy Path GL Extraction
- **Category:** Functional
- **Preconditions:** 100 GL entries posted in SAP sandbox; ODP subscription active for GL provider.
- **Steps:**
  1. Trigger ODP delta extraction for the GL domain.
  2. Verify Kafka message count on the `gl` topic equals 100.
  3. Verify FinSight `POST /api/v2/journal-entries` load succeeded for all 100.
  4. Run reconciliation for the GL domain.
- **Expected result:** 100 records in target; checksums match source; reconciliation = **PASS** (zero variance).
- **Priority:** P1

### TST-FNC-002 — AP Open Items Multi-Currency
- **Category:** Functional
- **Preconditions:** 50 AP open items in INR, USD, EUR in SAP; `TCURR` rates (KURST=M) loaded.
- **Steps:**
  1. Extract AP open items.
  2. Verify currency conversion to INR using point-in-time `GDATU` rate.
  3. Check ageing buckets computed correctly.
  4. Validate vendor enrichment (name/master from LFA1).
- **Expected result:** All amounts converted to INR at correct rates; ageing buckets correct; vendor fields enriched.
- **Priority:** P1

### TST-FNC-003 — Master Data Delta Sync
- **Category:** Functional
- **Preconditions:** Existing cost-centre master present in both systems.
- **Steps:**
  1. Create a new cost centre in SAP.
  2. Wait one extraction cycle.
  3. Query the FinSight target system.
- **Expected result:** New cost centre appears in target within one cycle.
- **Priority:** P2

### TST-FNC-004 — Multi-Company-Code Batch
- **Category:** Functional
- **Preconditions:** Entries from 3 company codes (MC01, MC02, MC03) staged.
- **Steps:**
  1. Extract the mixed batch.
  2. Verify tenant routing (MC01→MERIDIAN-MC01, MC02→MERIDIAN-MC02, MC03→MERIDIAN-MC03).
  3. Check no cross-contamination between tenants.
- **Expected result:** Each company code routes to the correct analytics tenant; no cross-tenant leakage.
- **Priority:** P1

### TST-FNC-005 — Fiscal Period Mapping
- **Category:** Functional
- **Preconditions:** Entries in SAP periods 001–016 (FY variant V3, Apr–Mar).
- **Steps:**
  1. Extract all periods.
  2. Verify calendar mapping of 001–012.
  3. Check special-period flag on 013–016.
- **Expected result:** Periods 001–012 map to calendar months correctly; 013–016 map to period 012 with a special-period flag.
- **Priority:** P2

### TST-FNC-006 — 7-Level Hierarchy Flatten
- **Category:** Functional
- **Preconditions:** Cost-centre hierarchy with 7 levels in SAP (SETNODE/SETLEAF).
- **Steps:**
  1. Extract the hierarchy.
  2. Verify the flattened table.
  3. Check all levels populated.
- **Expected result:** Flat table with Level1–Level7 columns, all populated.
- **Priority:** P2

### TST-FNC-007 — P2P Flow Tracing
- **Category:** Functional
- **Preconditions:** Complete P2P chain in SAP: PO, GR, Invoice, Payment.
- **Steps:**
  1. Extract all 4 stages.
  2. Trace the document flow linkage.
  3. Verify status mapping.
- **Expected result:** All stages linked; statuses correctly mapped.
- **Priority:** P2

### TST-FNC-008 — Bank Statement Load
- **Category:** Functional
- **Preconditions:** Bank statement with 50 items (matched and unmatched).
- **Steps:**
  1. Load the statement via FINSTA IDoc.
  2. Transform and load to `bank-statements`.
  3. Verify clearing status.
- **Expected result:** Matched items show **CLEARED**; unmatched show **OPEN**.
- **Priority:** P2

### TST-FNC-009 — Budget vs Actual Variance
- **Category:** Functional
- **Preconditions:** Budget and actuals for 5 cost centres in SAP CO.
- **Steps:**
  1. Extract budget data.
  2. Extract actual postings.
  3. Calculate variances.
- **Expected result:** Variances match the SAP CO report **S_ALR_87013611**.
- **Priority:** P3

### TST-FNC-010 — End-of-Day Full Reconciliation
- **Category:** Functional
- **Preconditions:** Full day of data across all 12 domains/endpoints.
- **Steps:**
  1. Run all 12 extractions.
  2. Execute full reconciliation.
  3. Generate the reconciliation report.
- **Expected result:** **Zero breaks** across all domains and reconciliation dimensions.
- **Priority:** P1

---

## 5. Non-functional, failure, security & reconciliation scenarios (15)

### 5.1 Non-Functional (TST-NFR-001 .. 005)

#### TST-NFR-001 — Peak Load
- **Category:** Performance
- **Preconditions:** 500K GL entries staged for a single batch; NFR env sized to production.
- **Steps:** 1. Trigger a single 500K-entry extraction. 2. Monitor throughput, memory, and load latency (D6 MON-002/003/010). 3. Reconcile at completion.
- **Expected result:** Completes within the 2-hour SLA; no OOM errors; no data loss.
- **Priority:** P1

#### TST-NFR-002 — Concurrent Extraction
- **Category:** Concurrency
- **Preconditions:** All 12 extractions runnable simultaneously; RFC pool ≤ 50 enforced.
- **Steps:** 1. Launch all 12 extractions at once. 2. Watch for deadlocks / resource contention. 3. Confirm each completes.
- **Expected result:** No deadlocks, no resource contention (RFC pool stays ≤ 50), all complete successfully.
- **Priority:** P1

#### TST-NFR-003 — API Latency Under Concurrency
- **Category:** Latency
- **Preconditions:** Load generator able to issue 500 concurrent FinSight API requests.
- **Steps:** 1. Drive 500 concurrent requests to FinSight. 2. Measure P95 latency (MON-003). 3. Track HTTP 429 rate.
- **Expected result:** P95 latency < 5 s; no HTTP 429 beyond expected rate-limit boundaries.
- **Priority:** P2

#### TST-NFR-004 — Volume Scalability
- **Category:** Scalability
- **Preconditions:** Generators for a gradual ramp 1K → 100K records.
- **Steps:** 1. Ramp load stepwise 1K→100K. 2. Plot throughput vs input. 3. Identify the ceiling.
- **Expected result:** Linear throughput scaling; ceiling identified; no degradation below the ceiling.
- **Priority:** P2

#### TST-NFR-005 — Endurance
- **Category:** Endurance
- **Preconditions:** 24-hour continuous run scheduled in NFR env.
- **Steps:** 1. Run continuously for 24 h. 2. Track memory, connections, throughput (MON-010/007/008). 3. Inspect for leaks post-run.
- **Expected result:** No memory leaks; no connection exhaustion; stable throughput throughout.
- **Priority:** P2

### 5.2 Failure Injection (TST-FLR-001 .. 005)

#### TST-FLR-001 — SAP RFC Connection Failure Mid-Batch
- **Category:** Failure
- **Preconditions:** Batch in flight; fault injector can drop the SAP RFC connection.
- **Steps:** 1. Start a batch. 2. Kill the RFC connection mid-batch. 3. Observe circuit breaker and recovery.
- **Expected result:** Circuit breaker activates (ERR-EXT-001/002); partial batch preserved; recovery is automatic (delta token not advanced → no data loss).
- **Priority:** P1

#### TST-FLR-002 — FinSight API 429 Throttling
- **Category:** Failure
- **Preconditions:** FinSight sandbox configured to return HTTP 429.
- **Steps:** 1. Drive load until 429 returned. 2. Observe Retry-After handling. 3. Confirm eventual success.
- **Expected result:** Respects `Retry-After` header (ERR-LOAD-001); backs off with jitter; eventually succeeds; no data loss.
- **Priority:** P1

#### TST-FLR-003 — Kafka Broker Failure (1 of 3)
- **Category:** Failure
- **Preconditions:** 3-broker cluster (RF=3, ISR=2); ability to kill one broker.
- **Steps:** 1. Publish steady traffic. 2. Kill 1 broker. 3. Verify failover and ISR recovery.
- **Expected result:** Automatic failover; zero message loss; ISR recovers (ERR-SYS-001 handled without data loss).
- **Priority:** P1

#### TST-FLR-004 — Malformed Data Handling
- **Category:** Failure
- **Preconditions:** Batch of 100 with 10 deliberately malformed records.
- **Steps:** 1. Extract/transform the batch. 2. Verify 90 clean records load. 3. Inspect DLQ for the 10.
- **Expected result:** 90 succeed; 10 route to DLQ with correct error codes (ERR-MAP-*); pipeline not blocked.
- **Priority:** P1

#### TST-FLR-005 — Network Partition to Target
- **Category:** Failure
- **Preconditions:** Toxiproxy able to sever the link to FinSight for 5 minutes.
- **Steps:** 1. Start steady load. 2. Introduce a 5-minute partition to the target. 3. Restore and observe recovery.
- **Expected result:** Buffering active during partition; recovery post-partition; eventual consistency achieved (no loss/duplication).
- **Priority:** P2

### 5.3 Security (TST-SEC-001 .. 003)

#### TST-SEC-001 — OAuth Token Expiry During Extraction
- **Category:** Security
- **Preconditions:** Short-lived OAuth token so it expires mid-run.
- **Steps:** 1. Begin a long extraction/load. 2. Let the access token expire. 3. Observe refresh behaviour.
- **Expected result:** Automatic refresh via `refresh_token` grant (TOKEN_EXPIRED / ERR-LOAD-005 path) without data loss or manual intervention.
- **Priority:** P1

#### TST-SEC-002 — Unauthorised Access
- **Category:** Security
- **Preconditions:** Test harness able to send invalid/expired/missing tokens.
- **Steps:** 1. Call FinSight with an invalid token. 2. Repeat with expired and missing tokens. 3. Inspect response and audit log.
- **Expected result:** HTTP 401 (INVALID_TOKEN); audit-log entry written; zero data exposure; no auto-retry loop.
- **Priority:** P1

#### TST-SEC-003 — Data Encryption Verification
- **Category:** Security
- **Preconditions:** Access to in-transit and at-rest inspection points.
- **Steps:** 1. Inspect transport between components. 2. Inspect Kafka and DLQ storage. 3. Verify cipher/keys.
- **Expected result:** TLS 1.2+ in transit; AES-256 at rest in Kafka and DLQ.
- **Priority:** P1

### 5.4 Reconciliation (TST-REC-001 .. 002)

#### TST-REC-001 — Deliberate Variance Detection
- **Category:** Reconciliation
- **Preconditions:** Source and target with a deliberate INR 1.50 discrepancy injected.
- **Steps:** 1. Load data with the seeded discrepancy. 2. Run reconciliation. 3. Inspect the alert and break report.
- **Expected result:** Reconciliation detects the break (ERR-RECON-002); generates an alert (D6 rule 6); logs details.
- **Priority:** P1

#### TST-REC-002 — Referential Integrity Orphan
- **Category:** Reconciliation
- **Preconditions:** A GL entry that references a non-existent cost centre.
- **Steps:** 1. Load the orphan GL entry. 2. Run referential-integrity checks. 3. Verify routing of the orphan.
- **Expected result:** Referential-integrity check catches the orphan (ERR-RECON-003); routes it to the business exception queue and triggers master-data resync.
- **Priority:** P1

---

## 6. Traceability matrix

Each test ID is mapped to the requirement / deliverable it verifies — domains, error classes, and
reconciliation dimensions from the Canonical Facts.

| Test ID | Verifies (requirement / deliverable) | Domain(s) / endpoint | Error class / code | Recon dimension | SLA / DQ target |
|---|---|---|---|---|---|
| TST-FNC-001 | Happy-path extract→transform→load (D1/D3) | GL (SRC/DST-001) | — | Completeness, Accuracy | Completeness > 99.5 %, Accuracy > 99.9 % |
| TST-FNC-002 | Currency conversion + ageing + enrichment (D3) | AP (SRC/DST-002) | DATA_QUALITY (ERR-MAP-004/005) | Accuracy | Accuracy > 99.9 % |
| TST-FNC-003 | Master-data delta before transactional (D1 §4) | Cost Centre (SRC/DST-004) | — | Consistency | Freshness ≤ 4 h |
| TST-FNC-004 | Company-code → tenant routing (Canonical §2) | GL, all (MC01/02/03) | — | Consistency | Consistency 100 % |
| TST-FNC-005 | Fiscal period + special-period mapping (D3) | GL (periods 001–016) | — | Validity | Validity > 99 % |
| TST-FNC-006 | Hierarchy flattening L1–L7 (D3) | Cost Centre / Profit Centre | — | Consistency | Consistency 100 % |
| TST-FNC-007 | P2P document-flow linkage (D3) | PO, Material Ledger, AP | — | Consistency | Consistency 100 % |
| TST-FNC-008 | Clearing-status mapping (D3) | Bank Statements (SRC/DST-010) | — | Accuracy | Accuracy > 99.9 % |
| TST-FNC-009 | Budget vs actual variance (D3) | Budget/Actual (SRC/DST-011), Cost Centre | — | Accuracy | Accuracy > 99.9 % |
| TST-FNC-010 | Full zero-break reconciliation (D5) | All 10 domains / 12 EPs | ERR-RECON-* | All 4 dimensions | Zero-break target |
| TST-NFR-001 | Peak throughput within SLA (D1 NFR) | GL | SYSTEM | Timeliness | Batch within SLA > 98 %, ≤ 2 h |
| TST-NFR-002 | Concurrency vs RFC pool ≤ 50 (Canonical §5) | All 12 EPs | TRANSIENT (ERR-EXT-002) | Timeliness | RFC ≤ 50 |
| TST-NFR-003 | API latency under load (D1 NFR) | FinSight API | TRANSIENT (ERR-LOAD-001) | Timeliness | P95 < 5 s |
| TST-NFR-004 | Volume scalability / ceiling (D1 NFR) | GL | — | Timeliness | Linear scaling |
| TST-NFR-005 | Endurance / leak-free 24 h (D1 NFR) | All | SYSTEM (ERR-SYS-002) | Timeliness | Stable throughput |
| TST-FLR-001 | Circuit breaker + no-loss recovery (D4) | GL / SAP RFC | TRANSIENT (ERR-EXT-001/002) | Completeness | No data loss |
| TST-FLR-002 | 429 backoff w/ jitter (D4, Appendix B) | FinSight API | TRANSIENT (ERR-LOAD-001) | Completeness | No data loss |
| TST-FLR-003 | Kafka failover, ISR recovery (D1/D4) | Kafka broker | SYSTEM (ERR-SYS-001) | Completeness | Zero message loss |
| TST-FLR-004 | Malformed → DLQ routing (D4) | Any / DLQ | DATA_QUALITY (ERR-MAP-*) | Validity | Validity > 99 % |
| TST-FLR-005 | Buffering + eventual consistency (D4) | FinSight target | TRANSIENT | Consistency | Eventual consistency |
| TST-SEC-001 | OAuth auto-refresh (D2/D4, Appendix B) | FinSight auth | PERMANENT→recover (TOKEN_EXPIRED) | — | No data loss |
| TST-SEC-002 | Unauthorised access → 401 + audit (D2) | FinSight auth | PERMANENT (INVALID_TOKEN) | — | Zero data exposure |
| TST-SEC-003 | Encryption in transit/at rest (D1 security) | Kafka, DLQ, transport | SYSTEM | — | TLS 1.2+ / AES-256 |
| TST-REC-001 | Variance break detection + alert (D5/D6) | Any (INR 1.50) | DATA_QUALITY (ERR-RECON-002) | Accuracy | Zero unexplained variance |
| TST-REC-002 | Referential-integrity orphan routing (D5) | GL→Cost Centre | DATA_QUALITY (ERR-RECON-003) | Consistency | Consistency 100 % |

**Coverage summary:** all 10 domains, all 4 error classes (`TRANSIENT/PERMANENT/DATA_QUALITY/SYSTEM`),
all 5 `ERR-*` registries, and all 4 reconciliation dimensions
(Completeness/Accuracy/Timeliness/Consistency) are exercised by at least one scenario.

---

## 7. Extension roadmap to 30+ scenarios (Test Tactician)

The 25 mandatory scenarios above are complete. The following additional scenarios would extend
coverage to 30+ and are recommended for a follow-on cycle, each with full automation specs:

1. **TST-FNC-011 — GST breakdown preservation:** assert CGST/SGST/IGST survive any invoice-amount
   transform (Canonical §5 GST constraint). *Priority P1.*
2. **TST-FLR-006 — Schema-evolution / support-pack:** SAP alters CDS `I_JournalEntry` structure;
   verify detection (ERR-EXT-003), alert, and graceful degradation. *Priority P2.*
3. **TST-NFR-006 — Festival surge (5× spike):** sudden 5× volume burst; Kafka partition design and
   autoscaling hold SLA (Case Study 1). *Priority P2.*
4. **TST-SEC-004 — Data-residency assertion:** prove no processing/telemetry egresses `ap-south-1`
   (RBI). *Priority P1.*
5. **TST-REC-003 — Intercompany elimination:** matching partner entries across MC01–MC03. *Priority P2.*
6. **TST-FLR-007 — DLQ reprocessing:** drain and reprocess DLQ after fix; verify idempotent re-load
   (no duplicates via business-key idempotency). *Priority P2.*
7. **TST-NFR-007 — Bandwidth cap:** confirm business-hours throughput stays ≤ 25 % of 450 Mbps.

These bring the plan to **32 scenarios** while keeping the 25 mandatory scenarios fully specified.

---

## 8. Traceability to canonical facts

- SLA & DQ targets — Canonical §8 (freshness ≤ 4 h, completeness/accuracy/consistency/timeliness).
- Company-code → tenant routing — §2 (MC01/02/03 → MERIDIAN-MC0x); verified by FNC-004.
- Constraints (RFC ≤ 50, ODP ≤ 30 min, bandwidth ≤ 25 %, residency) — §5; verified by NFR-002/004/007.
- Error classes & registries — §7 + Appendix B/J; mapped in the traceability matrix.
- Reconciliation zero-break target across 10 domains — §8; verified by FNC-010, REC-001/002.

---

*End of D7 — Integration Test Plan v1.*
