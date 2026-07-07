# TST_Functional_007 — P2P Flow Tracing

**Project:** FDE-9B (493560B) — SAP S/4HANA → Zetheta FinSight · **Author:** Anutosh Mishra
**Test ID:** TST-FNC-007 · **Category:** Functional · **Priority:** P2
**Domain:** Purchase Orders + Material Ledger + Accounts Payable (SRC-007/006/002)

## Objective
Verify that a full Procure-to-Pay chain (PO → GR → Invoice → Payment) is extracted and linked, with
statuses correctly mapped across the four stages.

## Preconditions
- Complete P2P chain present in SAP: PO (EKKO/EKPO), Goods Receipt, Invoice (BSIK), Payment.

## Test data
One PO with a matching goods receipt, supplier invoice, and outgoing payment.

## Steps
1. Extract all 4 stages (PO, GR, Invoice, Payment).
2. Trace the document flow linkage in the target.
3. Verify status mapping at each stage.

## Expected result
All four stages are linked into one traceable chain; statuses are correctly mapped.

## Pass/fail criteria
PASS if the four documents resolve to a single linked chain and each stage status matches the SAP source.

## Traceability
D3 P2P mappings (GR/IR reconciliation, status mapping) · D1 lineage · Consistency = 100 %.

## Automation
Assert the four document IDs share a common chain key and the status enum at each hop equals expected.
