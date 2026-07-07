# TST_Functional_006 — 7-Level Hierarchy Flatten

**Project:** FDE-9B (493560B) — SAP S/4HANA → Zetheta FinSight · **Author:** Anutosh Mishra
**Test ID:** TST-FNC-006 · **Category:** Functional · **Priority:** P2
**Domain:** Cost Centre / Profit Centre Accounting (SRC-004/005, DST-004/005)

## Objective
Verify the hierarchy-flattening algorithm converts a 7-level SAP cost-centre hierarchy
(SETNODE/SETLEAF) into a flat table with Level1–Level7 columns.

## Preconditions
- Cost-centre hierarchy with 7 levels defined in SAP (SETNODE/SETLEAF).

## Test data
A cost-centre standard hierarchy exactly 7 levels deep with leaf cost centres at the bottom.

## Steps
1. Extract the hierarchy.
2. Verify the flattened output table structure.
3. Check that all seven level columns are populated for every leaf.

## Expected result
Flat table with Level1 through Level7 columns, all populated (no gaps for leaves at full depth).

## Pass/fail criteria
PASS if every leaf row has Level1–Level7 populated and the parent-child lineage matches SAP.

## Traceability
D3 hierarchy-flattening algorithm · Canonical §4 (SETNODE/SETLEAF flatten to Level1–Level7) · Consistency = 100 %.

## Automation
Compare flattened rows against a reference tree walk of SETNODE/SETLEAF; assert 7 non-null level columns per leaf.
