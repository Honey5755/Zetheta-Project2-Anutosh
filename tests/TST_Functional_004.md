# TST_Functional_004 — Multi-Company-Code Batch

**Project:** FDE-9B (493560B) — SAP S/4HANA → Zetheta FinSight · **Author:** Anutosh Mishra
**Test ID:** TST-FNC-004 · **Category:** Functional · **Priority:** P1
**Domain:** All (company-code → tenant routing); primary GL (SRC-001 / DST-001)

## Objective
Verify that documents from the three company codes route to the correct FinSight tenant with no
cross-tenant contamination. Kafka partitions by company code, so this also exercises partition routing.

## Preconditions
- Entries from 3 company codes staged: MC01, MC02, MC03.
- Tenants provisioned: MERIDIAN-MC01, MERIDIAN-MC02, MERIDIAN-MC03.

## Test data
Mixed batch containing documents for all three legal entities (e.g. 40/30/30 split).

## Steps
1. Extract the mixed batch.
2. Verify tenant routing: MC01→MERIDIAN-MC01, MC02→MERIDIAN-MC02, MC03→MERIDIAN-MC03.
3. Query each tenant and check for no cross-contamination (no document appears under the wrong tenant).

## Expected result
Each company code routes to the correct analytics tenant; zero cross-tenant leakage.

## Pass/fail criteria
PASS only if each tenant contains exactly its own documents and no others.

## Traceability
Canonical §2 (company code → plant → tenant routing) · D1 Kafka partition-by-company-code · Consistency = 100 %.

## Automation
Assert per-tenant document sets equal expected sets; assert symmetric difference is empty across all tenants.
