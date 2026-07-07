# MAP_AccountsReceivable_v1

**Project:** 493560B — FDE-9B Custom API Integration (SAP S/4HANA → Zetheta FinSight)
**Domain:** Accounts Receivable · **Source EP:** SRC-003 · **Dest EP:** DST-003
**Source tables:** BSID (customer open items), KNA1 (customer master — general), KNC1 (customer master — company code); KNKK (credit management) for credit limit
**FinSight resource:** `POST /api/v2/receivables`
**Extraction:** ODP delta, every 30 min · **Idempotency key:** `receivable.receivableId`
**Mapping count:** 12

> Customer master (KNA1/KNC1) must sync before BSID transactional load (AR→customer referential integrity). GST components preserved on any invoice-amount transform (spec §5.4).

| Mapping ID | Source Field | Target Field | Transformation Rule | Validation Rule | Error Handling | Business Context |
|---|---|---|---|---|---|---|
| MAP-AR-001 | `BSID.BUKRS` + `BSID.GJAHR` + `BSID.BELNR` + `BSID.KUNNR` | `receivable.receivableId` | `CONCAT(BUKRS,'-',GJAHR,'-',LTRIM(BELNR,'0'),'-',LTRIM(KUNNR,'0'))` | NOT NULL; regex `^[A-Z0-9]{2,6}-\d{4}-\d{1,10}-\d{1,10}$` | ERR-MAP-001; ERR-VAL-001 | Composite open-item key; idempotency + audit lineage. |
| MAP-AR-002 | `BSID.KUNNR` | `receivable.customerId` | `LTRIM(KUNNR,'0')` then validate against customer master | Must exist in KNA1 | ERR-MAP-001; ERR-VAL-004 | Links receivable to customer dimension. |
| MAP-AR-003 | `KNA1.NAME1` | `receivable.customerName` | Enrichment join on `KUNNR`; TRIM | NOT NULL; max length 35 | ERR-MAP-001; ERR-VAL-001 | Customer name for AR ageing and collections reports. |
| MAP-AR-004 | `BSID.DMBTR` + `BSID.SHKZG` | `receivable.openAmount` | `DECIMAL(DMBTR,2)`; sign `= +DMBTR IF SHKZG='S' (debit/open) ELSE -DMBTR` | NOT NULL; range `-999999999.99`…`999999999.99` | ERR-MAP-001; ERR-VAL-002 | Open receivable amount; core of AR balance and DSO. |
| MAP-AR-005 | `KNA1.STCD3` (GSTIN) / `KNA1.STCD1` (PAN) | `receivable.gstin`, `receivable.panNumber` | Direct map, UPPERCASE; validate GSTIN pattern | GSTIN regex `^\d{2}[A-Z]{5}\d{4}[A-Z]{1}[A-Z\d]{1}Z[A-Z\d]{1}$` | ERR-MAP-001; ERR-VAL-001 (invalid → business queue) | Customer GST identity for GSTR-1 outward-supply reconciliation. |
| MAP-AR-006 | `BSID.ZFBDT` | `receivable.ageingBucket`, `receivable.daysOverdue` | `days = DATEDIFF(extractionDate, ZFBDT)`; bucket `0–30 / 31–60 / 61–90 / 90+` | `ZFBDT` NOT NULL, valid date | ERR-MAP-002; ERR-VAL-002 | AR ageing for collections prioritisation and DSO metric. |
| MAP-AR-007 | `BSID.ZFBDT` + `KNC1.ZTERM` | `receivable.dueDate` | `dueDate = ZFBDT + termDays(ZTERM)` then `FORMAT('YYYY-MM-DD')` | Must be ≥ `ZFBDT` | ERR-MAP-002; ERR-VAL-005 | Net due date drives overdue detection and dunning trigger. |
| MAP-AR-008 | `KNKK.KLIMK` | `receivable.creditLimit`, `receivable.creditExposurePct` | `DECIMAL(KLIMK,2)`; `creditExposurePct = openBalance / KLIMK * 100` | `KLIMK` ≥ 0; guard divide-by-zero | ERR-MAP-006 (div-by-zero → DLQ); ERR-VAL-002 | Credit-limit integration; flags customers over exposure threshold. |
| MAP-AR-009 | `BSID.MAHNS` | `receivable.dunningLevel` | MAP `MAHNS` (0–9) → enum {NONE, REMINDER, DUNNING_1, DUNNING_2, LEGAL} | Range 0–9 | ERR-VAL-002; ERR-VAL-003 | Dunning level for collections workflow and bad-debt risk. |
| MAP-AR-010 | `BSID.AUGBL` + `BSID.AUGDT` | `receivable.clearingStatus`, `receivable.clearingDate` | `IF AUGBL <> '' THEN 'CLEARED' (clearingDate=FORMAT(AUGDT)) ELSE 'OPEN'` | Enum {OPEN, CLEARED} | ERR-VAL-003 | Clearing status determines open-items report inclusion. |
| MAP-AR-011 | `KNC1.AKONT` | `receivable.reconciliationAccount` | `LTRIM(AKONT,'0')` then CoA lookup | Must exist in target chart of accounts | ERR-VAL-004 | Ties AR sub-ledger to GL reconciliation account (AR↔GL consistency). |
| MAP-AR-012 | `KNA1.LAND1` | `receivable.customerCountry` | MAP SAP country key → ISO 3166 alpha-2 | Must be valid ISO 3166 code | ERR-VAL-003 | Domestic vs export classification; export = zero-rated / IGST GST treatment. |

## Notes

- **Credit exposure:** `creditExposurePct` guards against `KLIMK = 0` (unlimited/unassigned limit) by emitting `null` exposure with a `limitUnassigned=true` flag rather than raising ERR-MAP-006.
- **Dunning source:** `MAHNS` (dunning level) is read from BSID; if BSID is superseded by dunning-run tables in the client's config, the enrichment falls back to `KNB5.MAHNS`.
- **GST preservation:** outward-invoice CGST/SGST/IGST lines carried as `receivable.taxComponents[]` for GSTR-1 tie-out.
