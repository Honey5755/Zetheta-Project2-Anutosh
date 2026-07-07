# TST_Functional_003 — Master Data Delta Sync

**Project:** FDE-9B (493560B) — SAP S/4HANA → Zetheta FinSight · **Author:** Anutosh Mishra
**Test ID:** TST-FNC-003 · **Category:** Functional · **Priority:** P2
**Domain:** Cost Centre Accounting (SRC-004 / DST-004, `/api/v2/cost-centres`)

## Objective
Confirm that a newly created master-data record (cost centre) is picked up by delta extraction and
appears in FinSight within one extraction cycle — proving master syncs before transactional data.

## Preconditions
- Existing cost-centre master present in both SAP and FinSight.
- Cost-centre batch extraction (4 h master cadence) configured; CSKS/CSKT available.

## Test data
One new cost centre created in SAP after the baseline sync.

## Steps
1. Create a new cost centre in SAP (tcode KS01 / master maintenance).
2. Wait one extraction cycle.
3. Query the FinSight target system for the new cost-centre code.

## Expected result
New cost centre appears in the target within one extraction cycle, with hierarchy attributes populated.

## Pass/fail criteria
PASS if the new cost centre is present and complete in FinSight after exactly one cycle; FAIL if missing or partial.

## Traceability
D1 §4 (master before transactional; referential integrity GL→CC) · D3 cost-centre mappings · Canonical §8 (freshness ≤ 4 h).

## Automation
Create-via-API → sleep(cycle) → poll FinSight GET; assert record present and hierarchy levels non-null.
