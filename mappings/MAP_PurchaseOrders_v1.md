# MAP_PurchaseOrders_v1

**Project:** 493560B — FDE-9B Custom API Integration (SAP S/4HANA → Zetheta FinSight)
**Domain:** Purchase Orders · **Source EP:** SRC-007 · **Dest EP:** DST-007
**Source tables:** EKKO (PO header), EKPO (PO item), EKET (PO schedule lines)
**FinSight resource:** `POST /api/v2/purchase-orders`
**Extraction:** ODP delta, every 30 min · **Idempotency key:** `purchaseOrder.poLineId`
**Mapping count:** 6

> GR/IR reconciliation via ordered vs goods-received quantity; pricing conditions unitised by `PEINH`.

| Mapping ID | Source Field | Target Field | Transformation Rule | Validation Rule | Error Handling | Business Context |
|---|---|---|---|---|---|---|
| MAP-PO-001 | `EKPO.EBELN` + `EKPO.EBELP` | `purchaseOrder.poLineId` | `CONCAT(LTRIM(EBELN,'0'),'-',LTRIM(EBELP,'0'))` | NOT NULL; regex `^\d{1,10}-\d{1,5}$`; unique | ERR-MAP-001; ERR-VAL-001 / ERR-VAL-006 | Composite PO-line key; idempotency + procurement lineage. |
| MAP-PO-002 | `EKKO.LIFNR` | `purchaseOrder.vendorId` | `LTRIM(LIFNR,'0')` then validate against vendor master | Must exist in LFA1 | ERR-MAP-001; ERR-VAL-004 | Links PO to vendor dimension; enables spend-by-vendor analytics. |
| MAP-PO-003 | `EKPO.MATNR` | `purchaseOrder.materialId` | `LTRIM(MATNR,'0')` then validate against material master | Must exist in material master (blank allowed for text POs) | ERR-VAL-004 | Links PO line to material; feeds procurement-cost analysis. |
| MAP-PO-004 | `EKPO.NETPR` + `EKPO.PEINH` | `purchaseOrder.netPriceUnit`, `purchaseOrder.lineNetValue` | `netPriceUnit = DECIMAL(NETPR / NULLIF(PEINH,0), 2)`; `lineNetValue = netPriceUnit * MENGE` | `PEINH` ≠ 0; range checks | ERR-MAP-006; ERR-VAL-002 | Pricing-condition net price (per base unit) and extended line value. |
| MAP-PO-005 | `EKPO.MENGE` + `EKET.WEMNG` | `purchaseOrder.openGrQty`, `purchaseOrder.grirStatus` | `openGrQty = SUM(MENGE) - SUM(WEMNG)`; `grirStatus = IF openGrQty=0 'FULLY_RECEIVED' ELSE IF WEMNG=0 'NOT_RECEIVED' ELSE 'PARTIAL'` | `WEMNG ≤ MENGE`; qty ≥ 0 | ERR-MAP-006; ERR-VAL-002 | GR/IR reconciliation — detects unbilled receipts / undelivered POs. |
| MAP-PO-006 | `EKET.EINDT` | `purchaseOrder.deliveryDate` | `FORMAT(EINDT,'YYYY-MM-DD')` (earliest open schedule line) | Valid date; ≥ PO creation date `AEDAT` | ERR-MAP-002; ERR-VAL-005 | Committed delivery date for on-time-delivery and supply-risk metrics. |

## Notes

- **Schedule aggregation:** EKET may hold multiple schedule lines per PO item; `WEMNG` (goods received) and `MENGE` (scheduled qty) are summed per (`EBELN`,`EBELP`) before the GR/IR calculation.
- **Order type / channel:** `EKKO.BSART` (order type) and `EKKO.BSTYP` (document category) are carried as `purchaseOrder.orderType` / `documentCategory` attributes for filtering (not counted as separate money mappings).
- **Currency:** PO currency `EKKO.WAERS` follows the same ISO 4217 validation as GL; non-INR POs are convertible via TCURR (spec §5.1) keyed on `AEDAT`.
