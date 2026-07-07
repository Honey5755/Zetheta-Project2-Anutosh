# MAP_SalesOrders_v1

**Project:** 493560B — FDE-9B Custom API Integration (SAP S/4HANA → Zetheta FinSight)
**Domain:** Sales Orders · **Source EP:** SRC-008 · **Dest EP:** DST-008
**Source tables:** VBAK (SO header), VBAP (SO item), VBEP (SO schedule lines)
**FinSight resource:** `POST /api/v2/sales-orders`
**Extraction:** ODP delta, every 30 min · **Idempotency key:** `salesOrder.soLineId`
**Mapping count:** 5

> Revenue-recognition stage derived from schedule-line confirmation vs order quantity.

| Mapping ID | Source Field | Target Field | Transformation Rule | Validation Rule | Error Handling | Business Context |
|---|---|---|---|---|---|---|
| MAP-SO-001 | `VBAP.VBELN` + `VBAP.POSNR` | `salesOrder.soLineId` | `CONCAT(LTRIM(VBELN,'0'),'-',LTRIM(POSNR,'0'))` | NOT NULL; regex `^\d{1,10}-\d{1,6}$`; unique | ERR-MAP-001; ERR-VAL-001 / ERR-VAL-006 | Composite SO-line key; idempotency + order-to-cash lineage. |
| MAP-SO-002 | `VBAK.KUNNR` | `salesOrder.customerId` | `LTRIM(KUNNR,'0')` then validate against customer master | Must exist in KNA1 | ERR-MAP-001; ERR-VAL-004 | Links SO to customer dimension; feeds revenue-by-customer analytics. |
| MAP-SO-003 | `VBAP.NETWR` + `VBAP.WAERK` | `salesOrder.netValue`, `salesOrder.currency` | `DECIMAL(NETWR,2)`; `WAERK` validated ISO 4217 | Range checks; `WAERK` valid ISO 4217 | ERR-MAP-001; ERR-MAP-004; ERR-VAL-002 / ERR-VAL-003 | Order net value (transaction currency) for order-book and pipeline reporting. |
| MAP-SO-004 | `VBEP.KWMENG`/`VBAP.KWMENG` + `VBEP.BMENG` | `salesOrder.revenueRecognitionStage` | `IF BMENG=0 'ORDERED' ELSE IF BMENG<KWMENG 'PARTIALLY_CONFIRMED' ELSE IF fully delivered 'DELIVERED' ELSE 'CONFIRMED'` | Enum {ORDERED, CONFIRMED, PARTIALLY_CONFIRMED, PARTIALLY_DELIVERED, DELIVERED} | ERR-MAP-006; ERR-VAL-003 | Revenue-recognition stage for order-to-cash progression and forecast. |
| MAP-SO-005 | `VBEP.EDATU` | `salesOrder.requestedDeliveryDate` | `FORMAT(EDATU,'YYYY-MM-DD')` (earliest schedule line) | Valid date; ≥ `VBAK.ERDAT` | ERR-MAP-002; ERR-VAL-005 | Requested delivery date for on-time-in-full (OTIF) service metrics. |

## Notes

- **Confirmed quantity:** `VBEP.BMENG` (confirmed qty) is summed per (`VBELN`,`POSNR`) and compared to order quantity `KWMENG` to derive the recognition stage; delivery status is confirmed against `LMENG` (remaining delivery qty) when present.
- **Order attributes:** `VBAK.AUART` (order type), `VKORG` (sales org), `VTWEG` (distribution channel), `SPART` (division) are carried as descriptive attributes for slicing.
- **Currency:** foreign-currency sales orders convert to INR via TCURR (spec §5.1) keyed on `VBAK.ERDAT` for consolidated pipeline value.
