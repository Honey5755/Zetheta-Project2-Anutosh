# MAP_AccountsPayable_v1

**Project:** 493560B — FDE-9B Custom API Integration (SAP S/4HANA → Zetheta FinSight)
**Domain:** Accounts Payable · **Source EP:** SRC-002 · **Dest EP:** DST-002
**Source tables:** BSIK (vendor open items), LFA1 (vendor master — general), LFC1 (vendor master — company code)
**FinSight resource:** `POST /api/v2/payables`
**Extraction:** ODP delta, every 30 min · **Idempotency key:** `payable.payableId`
**Mapping count:** 12

> Vendor master (LFA1/LFC1) must sync before BSIK transactional load (AP→vendor referential integrity). Any invoice-amount transform preserves GST components (CGST/SGST/IGST) — see spec §5.4.

| Mapping ID | Source Field | Target Field | Transformation Rule | Validation Rule | Error Handling | Business Context |
|---|---|---|---|---|---|---|
| MAP-AP-001 | `BSIK.BUKRS` + `BSIK.GJAHR` + `BSIK.BELNR` + `BSIK.LIFNR` | `payable.payableId` | `CONCAT(BUKRS,'-',GJAHR,'-',LTRIM(BELNR,'0'),'-',LTRIM(LIFNR,'0'))` | NOT NULL; regex `^[A-Z0-9]{2,6}-\d{4}-\d{1,10}-\d{1,10}$` | ERR-MAP-001; ERR-VAL-001 | Composite open-item key; idempotency + lineage from SAP document to FinSight payable. |
| MAP-AP-002 | `BSIK.LIFNR` | `payable.vendorId` | `LTRIM(LIFNR,'0')` then validate against vendor master | Must exist in LFA1 | ERR-MAP-001; ERR-VAL-004 (missing vendor → business queue) | Links payable to vendor dimension; enforces master-before-transaction order. |
| MAP-AP-003 | `LFA1.NAME1` | `payable.vendorName` | Enrichment join on `LIFNR`; TRIM | NOT NULL; max length 35 | ERR-MAP-001; ERR-VAL-001 | Human-readable vendor name for AP ageing and payment reports. |
| MAP-AP-004 | `BSIK.DMBTR` + `BSIK.SHKZG` | `payable.openAmount` | `DECIMAL(DMBTR,2)`; sign `= -DMBTR IF SHKZG='S' ELSE +DMBTR` (H = credit/open payable) | NOT NULL; range `-999999999.99`…`999999999.99` | ERR-MAP-001; ERR-VAL-002 | Open liability amount; core of AP balance and cash-flow forecasting. |
| MAP-AP-005 | `LFA1.STCD3` (GSTIN) / `LFA1.STCD1` (PAN) | `payable.gstin`, `payable.panNumber` | Direct map, UPPERCASE; validate GSTIN pattern | GSTIN regex `^\d{2}[A-Z]{5}\d{4}[A-Z]{1}[A-Z\d]{1}Z[A-Z\d]{1}$`; PAN regex `^[A-Z]{5}\d{4}[A-Z]$` | ERR-MAP-001 (null → quarantine); ERR-VAL-001 (invalid GSTIN → business queue, notify AP team) | India GST compliance; invalid GSTINs quarantined without blocking clean records (Case Study 3). |
| MAP-AP-006 | `BSIK.ZFBDT` | `payable.ageingBucket`, `payable.daysOverdue` | `days = DATEDIFF(extractionDate, ZFBDT)`; bucket `0–30 / 31–60 / 61–90 / 90+` | `ZFBDT` NOT NULL and a valid date | ERR-MAP-002; ERR-VAL-002 | Ageing analysis for AP dashboard and DPO metrics (Al-Hassan / CFO). |
| MAP-AP-007 | `BSIK.ZFBDT` + `LFC1.ZTERM` | `payable.dueDate` | `dueDate = ZFBDT + termDays(ZTERM)` then `FORMAT('YYYY-MM-DD')` | Must be ≥ `ZFBDT` | ERR-MAP-002; ERR-VAL-005 | Net due date drives payment scheduling and overdue detection. |
| MAP-AP-008 | `LFC1.AKONT` | `payable.reconciliationAccount` | `LTRIM(AKONT,'0')` then CoA lookup | Must exist in target chart of accounts | ERR-VAL-004 | Ties AP sub-ledger to GL reconciliation account (AP↔GL consistency). |
| MAP-AP-009 | `LFC1.ZTERM` | `payable.paymentTerms` | MAP `ZTERM` → enum {NET30, NET45, NET60, IMMEDIATE, …} + `termDays` | Must map to a known payment-term code | ERR-VAL-003 | Standardises heterogeneous SAP payment-term keys for analytics. |
| MAP-AP-010 | `BSIK.AUGBL` + `BSIK.AUGDT` | `payable.clearingStatus`, `payable.clearingDate` | `IF AUGBL <> '' THEN 'CLEARED' (clearingDate=FORMAT(AUGDT)) ELSE 'OPEN'` | Enum {OPEN, CLEARED} | ERR-VAL-003 | Clearing status determines whether item appears in open-items reports. |
| MAP-AP-011 | `BSIK.UMSKZ` | `payable.specialGlIndicator` | MAP `UMSKZ` → enum {DOWN_PAYMENT, BANK_GUARANTEE, NONE}; default NONE if blank | Must map to known indicator | ERR-VAL-003 | Distinguishes special GL transactions (down payments) from normal payables. |
| MAP-AP-012 | `LFA1.LAND1` | `payable.vendorCountry` | MAP SAP country key → ISO 3166 alpha-2 (`IN`→`IN`) | Must be valid ISO 3166 code | ERR-VAL-003 | Domestic vs import classification; drives IGST-vs-CGST/SGST GST treatment. |

## Notes

- **GST preservation:** where `openAmount` is derived or netted, the invoice's CGST/SGST/IGST tax lines (SAP condition records / `J_1I*`) are carried as `payable.taxComponents[]` unchanged — no re-derivation from the net.
- **Ageing baseline:** `ZFBDT` (baseline date for payment) is preferred over `BUDAT`; if `ZFBDT` is null, fall back to document date with a `baselineImputed=true` flag.
- **Quarantine flow:** GSTIN validation failures route to the business exception queue (not the DLQ) so AP can remediate the vendor master while clean payables continue loading.
