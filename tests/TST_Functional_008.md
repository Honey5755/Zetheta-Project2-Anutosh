# TST_Functional_008 — Bank Statement Load

**Project:** FDE-9B (493560B) — SAP S/4HANA → Zetheta FinSight · **Author:** Anutosh Mishra
**Test ID:** TST-FNC-008 · **Category:** Functional · **Priority:** P2
**Domain:** Bank Statements (SRC-010 / DST-010, `/api/v2/bank-statements`)

## Objective
Verify bank-statement ingestion via FINSTA IDoc, normalisation, and correct clearing-status mapping
for matched vs unmatched items.

## Preconditions
- Bank statement with 50 items available (mix of matched and unmatched), FEBKO/FEBEP.

## Test data
50-line bank statement: a subset auto-matched to open items, the remainder unmatched.

## Steps
1. Load the statement via FINSTA IDoc.
2. Transform and load to the `bank-statements` endpoint.
3. Verify the clearing status on each loaded item.

## Expected result
Matched items show **CLEARED**; unmatched items show **OPEN**.

## Pass/fail criteria
PASS if every matched item is CLEARED and every unmatched item is OPEN, with 50 items total loaded.

## Traceability
D3 bank-statement mapping (statement format normalisation, clearing status) · Accuracy > 99.9 %.

## Automation
Assert per-item clearing status equals the expected match outcome; assert loaded count = 50.
