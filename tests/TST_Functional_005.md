# TST_Functional_005 — Fiscal Period Mapping

**Project:** FDE-9B (493560B) — SAP S/4HANA → Zetheta FinSight · **Author:** Anutosh Mishra
**Test ID:** TST-FNC-005 · **Category:** Functional · **Priority:** P2
**Domain:** General Ledger (SRC-001 / DST-001)

## Objective
Verify fiscal-period-to-calendar mapping under FY variant V3 (Apr–Mar), including special periods
013–016 collapsing to period 012 with a flag.

## Preconditions
- GL entries posted across SAP periods 001–016.
- FY variant V3 configured; special periods 013–016 present.

## Test data
At least one entry in every period 001 through 016.

## Steps
1. Extract all periods.
2. Verify calendar mapping of periods 001–012 (001 = April … 012 = March).
3. Check the special-period flag on entries in periods 013–016.

## Expected result
Periods 001–012 map to calendar months correctly; periods 013–016 map to period 012 and carry a special-period flag.

## Pass/fail criteria
PASS if every period maps to the expected calendar month and all special-period entries are flagged and folded to 012.

## Traceability
D3 fiscal-period mapping logic · Canonical §2 (FY variant V3, special periods 013–016) · Validity > 99 %.

## Automation
Table-driven assertion of period→month; assert `special_period == true` and `mapped_period == 012` for 013–016.
