# TST_Functional_001 — Happy Path GL Extraction

**Project:** FDE-9B (493560B) — SAP S/4HANA → Zetheta FinSight · **Author:** Anutosh Mishra
**Test ID:** TST-FNC-001 · **Category:** Functional · **Priority:** P1
**Domain:** General Ledger (SRC-001 / DST-001, `/api/v2/journal-entries`)

## Objective
Prove the baseline end-to-end path — ODP delta extraction → Kafka → Transformation Engine →
FinSight Loader → FinSight — moves GL data completely and accurately with a passing reconciliation.

## Preconditions
- 100 balanced GL entries posted in the SAP sandbox (ACDOCA/BKPF/BSEG).
- ODP subscription active for the GL provider; delta token initialised.
- FinSight tenant `MERIDIAN-MC01` reachable; OAuth token valid.
- D6 monitoring live; run tagged `correlation_id` prefix `TST-FNC-001-<run>`.

## Test data
100 GL journal entries, group currency INR, current open period, company code MC01.

## Steps
1. Trigger ODP delta extraction for the GL domain via the Scheduler.
2. Verify the Kafka message count on the `gl` topic equals 100.
3. Verify FinSight `POST /api/v2/journal-entries` returns 201 for all 100 (idempotency key = document number).
4. Run reconciliation for the GL domain (source vs target count + checksum).

## Expected result
- 100 records present in the FinSight target.
- Source-vs-target checksums match (debit/credit totals equal per document).
- Reconciliation verdict = **PASS**, zero variance.

## Pass/fail criteria
PASS only if all 100 loaded, checksums match, and reconciliation reports zero breaks.

## Traceability
D1 pipeline · D3 GL mappings · D5 reconciliation · Canonical §8 (completeness > 99.5 %, accuracy > 99.9 %).

## Automation
pytest E2E harness: seed via SAP sandbox API, assert Kafka offset delta = 100, poll FinSight GET, invoke recon service; assert `recon_status == RECONCILED`.
