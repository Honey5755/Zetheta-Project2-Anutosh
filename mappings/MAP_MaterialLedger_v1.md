# MAP_MaterialLedger_v1

**Project:** 493560B — FDE-9B Custom API Integration (SAP S/4HANA → Zetheta FinSight)
**Domain:** Material Ledger · **Source EP:** SRC-006 · **Dest EP:** DST-006
**Source tables:** MBEW (material valuation), COSP (CO totals — external postings, price-difference cost elements)
**FinSight resource:** `POST /api/v2/material-ledger`
**Extraction:** ODP delta, every 60 min · **Idempotency key:** `materialCost.materialCostId`
**Mapping count:** 7

> Actual costing: standard vs moving-average price, unitised by price unit (`PEINH`), plus price-difference allocation from COSP. Divide-by-zero on `PEINH` is guarded (ERR-MAP-006).

| Mapping ID | Source Field | Target Field | Transformation Rule | Validation Rule | Error Handling | Business Context |
|---|---|---|---|---|---|---|
| MAP-ML-001 | `MBEW.MATNR` + `MBEW.BWKEY` | `materialCost.materialCostId` | `CONCAT(LTRIM(MATNR,'0'),'-',BWKEY)` | NOT NULL; unique per (material, valuation area) | ERR-MAP-001; ERR-VAL-001 / ERR-VAL-006 | Composite key of the material-valuation record. |
| MAP-ML-002 | `MBEW.BWKEY` | `materialCost.valuationArea` | Direct map; enrich with plant→company code | NOT NULL; must exist in plant master | ERR-VAL-004 | Valuation area (plant) scopes the material cost; links to MC0x tenant. |
| MAP-ML-003 | `MBEW.VPRSV` | `materialCost.priceControl` | MAP `VPRSV`: `S`→`STANDARD`, `V`→`MOVING_AVERAGE` | Must be `S` or `V` | ERR-VAL-003 | Price-control indicator governs which price is authoritative for valuation. |
| MAP-ML-004 | `MBEW.STPRS` + `MBEW.PEINH` | `materialCost.standardPriceUnit` | `DECIMAL(STPRS / NULLIF(PEINH,0), 2)` — per-base-unit standard price | `PEINH` ≠ 0; result within amount range | ERR-MAP-006 (div-by-zero → DLQ); ERR-VAL-002 | Unit standard cost for planned-cost variance and inventory valuation. |
| MAP-ML-005 | `MBEW.VERPR` + `MBEW.PEINH` | `materialCost.movingAvgPriceUnit` | `DECIMAL(VERPR / NULLIF(PEINH,0), 2)` — per-base-unit moving-average price | `PEINH` ≠ 0; result within amount range | ERR-MAP-006; ERR-VAL-002 | Actual moving-average cost for actual-costing analytics. |
| MAP-ML-006 | `MBEW.SALK3` + `MBEW.LBKUM` | `materialCost.totalStockValue`, `materialCost.totalStockQty` | `DECIMAL(SALK3,2)`, `DECIMAL(LBKUM,3)` | Range checks; `SALK3` sign-consistent with stock | ERR-MAP-001; ERR-VAL-002 | Total inventory value/quantity for balance-sheet inventory line. |
| MAP-ML-007 | `MBEW.SALK3` + `MBEW.STPRS` + `MBEW.LBKUM` + `MBEW.PEINH` + `COSP.WKG001..016` | `materialCost.priceDifference`, `materialCost.actualCost` | `priceDifference = SALK3 - (LBKUM * STPRS / NULLIF(PEINH,0))`; `actualCost = SALK3 + SUM(COSP price-diff cost-element cols)` | `PEINH` ≠ 0; components reconcile to SALK3 within tolerance | ERR-MAP-006; ERR-VAL-002 | Price-difference allocation reconciles standard to actual cost (actual costing run ML). |

## Notes

- **COSP selection:** actual price differences read from `COSP` where `WRTTP='04'` (actual) and `KSTAR` in the client's price-difference cost-element group; period columns `WKG001..WKG016` map to fiscal periods 001–016 (V3 April–March + special periods).
- **Precision:** quantities carry 3 decimals (`LBKUM`), amounts 2 decimals; rounding is half-up at 2dp on all money fields.
- **Inventory fold-in:** `MARD` storage-location stock (SRC-012) folds into this domain's `totalStockQty` at plant granularity when location-level detail is requested.
