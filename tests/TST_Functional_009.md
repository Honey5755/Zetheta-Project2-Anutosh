# TST_Functional_009 — Budget vs Actual Variance

**Project:** FDE-9B (493560B) — SAP S/4HANA → Zetheta FinSight · **Author:** Anutosh Mishra
**Test ID:** TST-FNC-009 · **Category:** Functional · **Priority:** P3
**Domain:** Budget/Actual (SRC-011 / DST-011, `/api/v2/budget-actuals`) + Cost Centre

## Objective
Verify that budget and actual postings are extracted and that computed variances match the SAP CO
reference report S_ALR_87013611.

## Preconditions
- Budget and actuals loaded for 5 cost centres in SAP CO (COSP/COSS).

## Test data
Budget and actual values for 5 cost centres over the current fiscal period.

## Steps
1. Extract budget data.
2. Extract actual postings.
3. Calculate variances (actual − budget) per cost centre.

## Expected result
Computed variances match the SAP CO report **S_ALR_87013611** for all 5 cost centres.

## Pass/fail criteria
PASS if every cost-centre variance equals the S_ALR_87013611 figure within rounding tolerance.

## Traceability
D3 budget/actual mapping (folds to GL/CC analytics) · Accuracy > 99.9 %.

## Automation
Oracle imports S_ALR_87013611 export; assert per-cost-centre variance equality within tolerance.
