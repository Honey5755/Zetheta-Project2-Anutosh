# TST_Functional_002 — AP Open Items Multi-Currency

**Project:** FDE-9B (493560B) — SAP S/4HANA → Zetheta FinSight · **Author:** Anutosh Mishra
**Test ID:** TST-FNC-002 · **Category:** Functional · **Priority:** P1
**Domain:** Accounts Payable (SRC-002 / DST-002, `/api/v2/payables`)

## Objective
Verify multi-currency conversion to INR at point-in-time rates, correct ageing-bucket calculation,
and vendor enrichment for AP open items.

## Preconditions
- 50 AP open items in INR, USD, and EUR in the SAP sandbox (BSIK/LFA1).
- `TCURR` rates loaded (KURST = M), point-in-time by `GDATU`.
- ODP subscription active for the AP provider.

## Test data
50 open items spread across ageing brackets (0–30 / 31–60 / 61–90 / 90+ days), 3 currencies, vendors in LFA1.

## Steps
1. Extract AP open items via ODP delta.
2. Verify currency conversion of each item to INR using the correct `GDATU` rate.
3. Check ageing buckets are computed correctly from the baseline date.
4. Validate vendor enrichment (name and master attributes from LFA1) on the loaded records.

## Expected result
- All amounts converted to INR at correct point-in-time rates.
- Ageing buckets correct.
- Vendor fields enriched on every record.

## Pass/fail criteria
PASS only if converted amounts match an independent `TCURR` calculation, ageing matches expected buckets, and no vendor field is null.

## Traceability
D3 AP mappings (currency conversion, ageing, enrichment) · Canonical §4 (TCURR) · error codes ERR-MAP-004/005 for rate/currency issues.

## Automation
Parametrised pytest over the 3 currencies; oracle re-computes INR from `TCURR`; assert delta < rounding tolerance and ageing bucket equality.
