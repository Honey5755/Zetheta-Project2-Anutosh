# MAP_GeneralLedger_v1

**Project:** 493560B — FDE-9B Custom API Integration (SAP S/4HANA → Zetheta FinSight)
**Domain:** General Ledger · **Source EP:** SRC-001 · **Dest EP:** DST-001
**Source tables:** ACDOCA (Universal Journal line items), BKPF (document header), BSEG (document segment)
**FinSight resource:** `POST /api/v2/journal-entries`
**Extraction:** ODP delta (CDS `ZZ_CDS_ACDOCA_DELTA`), every 30 min · **Idempotency key:** `journalEntry.documentId`
**Mapping count:** 18

> Each row follows the Deliverable-3 template: Mapping ID · Source Field (`Table.FIELD`) · Target Field · Transformation Rule · Validation Rule · Error Handling · Business Context. Error codes are drawn from the `ERR-MAP-*` (transformation) and `ERR-VAL-*` (validation) registries defined in `D3_Data_Transformation_Spec_v1.md` §6.

| Mapping ID | Source Field | Target Field | Transformation Rule | Validation Rule | Error Handling | Business Context |
|---|---|---|---|---|---|---|
| MAP-GL-001 | `ACDOCA.RBUKRS` + `ACDOCA.GJAHR` + `ACDOCA.BELNR` | `journalEntry.documentId` | `CONCAT(RBUKRS,'-',GJAHR,'-',LTRIM(BELNR,'0'))` | NOT NULL; regex `^[A-Z0-9]{2,6}-\d{4}-\d{1,10}$` | ERR-MAP-001 (null source → DLQ); ERR-VAL-001 (regex fail → DLQ) | Composite business key linking SAP document to FinSight record; drives idempotency and full audit lineage (Kulkarni). |
| MAP-GL-002 | `ACDOCA.BUDAT` | `journalEntry.postingDate` | `FORMAT(BUDAT,'YYYY-MM-DD')` from SAP internal `YYYYMMDD` | NOT NULL; must fall within fiscal year ±1 month | ERR-MAP-002 (date parse → correct/DLQ); ERR-VAL-005 (out-of-window → business queue) | Posting date determines the fiscal period bucket for analytics; ISO 8601 required by FinSight. |
| MAP-GL-003 | `ACDOCA.BLDAT` | `journalEntry.documentDate` | `FORMAT(BLDAT,'YYYY-MM-DD')` | NOT NULL; must be ≤ `postingDate + 30 days` | ERR-MAP-002; ERR-VAL-005 (ordering rule → business queue) | Original document date for invoice-to-posting lag analysis. |
| MAP-GL-004 | `ACDOCA.RACCT` | `journalEntry.glAccount` | `LTRIM(RACCT,'0')` then lookup analytics CoA mapping | Must exist in target chart-of-accounts master | ERR-MAP-001; ERR-VAL-004 (referential integrity → sync master, business queue) | Maps SAP CoA (Schedule III) to FinSight analytical CoA; consistency dimension = 100%. |
| MAP-GL-005 | `ACDOCA.HSL` | `journalEntry.amountLC` | `DECIMAL(HSL,2)` in group currency (INR) | NOT NULL; range `-999999999.99` … `999999999.99` | ERR-MAP-001; ERR-VAL-002 (range → DLQ) | Local/group-currency amount; feeds debit-credit reconciliation (SUM must net to zero per document). |
| MAP-GL-006 | `ACDOCA.WSL` | `journalEntry.amountTC` | `DECIMAL(WSL,2)` in transaction currency | NOT NULL; same range as `amountLC` | ERR-MAP-001; ERR-VAL-002 | Transaction-currency amount preserved for multi-currency reporting and FX revaluation. |
| MAP-GL-007 | `ACDOCA.RHCUR` | `journalEntry.localCurrency` | Direct map; validate ISO 4217 | Must be valid 3-letter ISO 4217 code | ERR-MAP-004 (unknown currency → typo check/DLQ); ERR-VAL-003 (enum → DLQ) | Group currency (INR) code; guards against `RS`→`INR` typo class (see AP GSTIN/currency case study). |
| MAP-GL-008 | `ACDOCA.RWCUR` | `journalEntry.transactionCurrency` | Direct map; validate ISO 4217 | Must be valid 3-letter ISO 4217 code | ERR-MAP-004; ERR-VAL-003 | Transaction currency; input to point-in-time TCURR conversion (spec §5.1). |
| MAP-GL-009 | `ACDOCA.KOSTL` | `journalEntry.costCentre` | `LTRIM(KOSTL,'0')` then validate against CC master | If present, must exist in target cost-centre dimension | ERR-MAP-003 (CC not found → business queue); ERR-VAL-004 | GL→CC referential link; master data must sync before transactional load. |
| MAP-GL-010 | `ACDOCA.PRCTR` | `journalEntry.profitCentre` | `LTRIM(PRCTR,'0')` then validate against PC master | If present, must exist in target profit-centre dimension | ERR-MAP-003; ERR-VAL-004 | GL→PC referential link; drives segment profitability reporting (Al-Hassan). |
| MAP-GL-011 | `ACDOCA.MONAT` | `journalEntry.fiscalPeriod` | MAP: SAP period 001→April … 012→March (V3); 013–016 → 012 with `specialPeriod=true` | Range 001–016; flag special periods | ERR-MAP-002; ERR-VAL-002 (out of 001–016 → DLQ) | Fiscal-to-calendar period mapping for India V3 variant; special periods carry year-end adjustments. |
| MAP-GL-012 | `ACDOCA.BLART` + `BKPF.STBLG` | `journalEntry.documentType`, `journalEntry.isReversal` | MAP `BLART` → analytics type enum; if `STBLG` populated set `isReversal=true` | Must map to a valid analytics document type | ERR-MAP-001; ERR-VAL-003 (unmapped type → DLQ) | Document type drives downstream processing/validation rules; reversal flag prevents double counting. |
| MAP-GL-013 | `ACDOCA.RBUKRS` | `journalEntry.tenantId` | MAP company code → FinSight tenant: `MC01`→`MERIDIAN-MC01`, `MC02`→`MERIDIAN-MC02`, `MC03`→`MERIDIAN-MC03` | Must resolve to one of 3 canonical tenants | ERR-MAP-001; ERR-VAL-003 (unknown company code → DLQ) | Canonical multi-tenant routing; also selects Kafka partition (partitioned by company code). |
| MAP-GL-014 | `ACDOCA.GJAHR` | `journalEntry.fiscalYear` | Direct map; cast to integer (YYYY) | Range 2000–2099; must equal year component of `documentId` | ERR-VAL-002 (range → DLQ) | Fiscal year for period-over-period analytics; cross-checks composite key integrity. |
| MAP-GL-015 | `ACDOCA.BUZEI` | `journalEntry.lineNumber` | `CAST(BUZEI AS INT)` | NOT NULL; > 0; unique within `documentId` | ERR-MAP-001; ERR-VAL-002 | Line-item sequence; guarantees uniqueness of each journal line in target. |
| MAP-GL-016 | `ACDOCA.HSL` (sign) / `BSEG.SHKZG` | `journalEntry.debitCreditIndicator` | `IF SHKZG='S' OR HSL>=0 THEN 'DEBIT' ELSE 'CREDIT'` | Enum {DEBIT, CREDIT} | ERR-MAP-006 (rule error → DLQ); ERR-VAL-003 | Explicit D/C flag underpins the zero-variance debit=credit reconciliation per document. |
| MAP-GL-017 | `BKPF.USNAM` | `journalEntry.postedBy` | Direct map; UPPERCASE trim | Max length 12; NOT NULL for posted docs | ERR-MAP-001; ERR-VAL-001 | User who posted the document; supports segregation-of-duties audit trail. |
| MAP-GL-018 | `BKPF.TCODE` | `journalEntry.sourceTransactionCode` | Direct map; default `UNKNOWN` if null | Max length 20 | ERR-MAP-001 (null → default) | Originating SAP transaction (e.g. FB01, MIRO); classifies posting channel for lineage. |

## Notes

- **Header/segment enrichment:** BKPF is joined on (`BUKRS`,`BELNR`,`GJAHR`); BSEG on (`BUKRS`,`BELNR`,`GJAHR`,`BUZEI`). ACDOCA is the primary line-item driver (Universal Journal single source of truth).
- **Currency conversion:** where downstream analytics require an INR-normalised figure for a non-INR `transactionCurrency`, apply the point-in-time `TCURR` conversion described in the spec (§5.1) keyed on `BUDAT` (`GDATU`).
- **Reconciliation contract:** `SUM(amountLC WHERE debitCreditIndicator='DEBIT') = SUM(amountLC WHERE debitCreditIndicator='CREDIT')` for every `documentId` (zero-break target, D5).
