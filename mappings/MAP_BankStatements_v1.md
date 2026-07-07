# MAP_BankStatements_v1

**Project:** 493560B — FDE-9B Custom API Integration (SAP S/4HANA → Zetheta FinSight)
**Domain:** Bank Statements · **Source EP:** SRC-010 · **Dest EP:** DST-010
**Source tables:** FEBKO (electronic bank statement header), FEBEP (bank statement line items)
**FinSight resource:** `POST /api/v2/bank-statements`
**Extraction:** IDoc (FINSTA), event-driven · **Idempotency key:** `bankStatement.statementLineId`
**Mapping count:** 5

> Statement normalisation across banks; clearing status derived from the posting/clearing document link.

| Mapping ID | Source Field | Target Field | Transformation Rule | Validation Rule | Error Handling | Business Context |
|---|---|---|---|---|---|---|
| MAP-BNK-001 | `FEBKO.KUKEY` + `FEBKO.AZNUM` + `FEBEP.ESNUM` | `bankStatement.statementLineId` | `CONCAT(KUKEY,'-',AZNUM,'-',LPAD(ESNUM,4,'0'))` | NOT NULL; unique per statement line | ERR-MAP-001; ERR-VAL-001 / ERR-VAL-006 | Composite key of a single statement line; idempotency + cash lineage. |
| MAP-BNK-002 | `FEBKO.KONTS` | `bankStatement.bankAccount` | Normalise to canonical account/IBAN; strip separators, UPPERCASE | NOT NULL; passes IBAN/account checksum where applicable | ERR-MAP-001; ERR-VAL-001 | Normalised house-bank account across heterogeneous bank formats. |
| MAP-BNK-003 | `FEBEP.KWBTR` + `FEBEP.VGTYP` | `bankStatement.lineAmount`, `bankStatement.direction` | `DECIMAL(KWBTR,2)`; `direction = IF VGTYP indicates credit 'CREDIT' ELSE 'DEBIT'`; signed amount accordingly | Range checks; direction enum {DEBIT, CREDIT} | ERR-MAP-001; ERR-VAL-002 / ERR-VAL-003 | Cash movement amount and direction for cash-position and liquidity analytics. |
| MAP-BNK-004 | `FEBEP.VALUT` | `bankStatement.valueDate` | `FORMAT(VALUT,'YYYY-MM-DD')` | Valid date | ERR-MAP-002; ERR-VAL-005 | Value date drives cash-flow timing and interest reconciliation. |
| MAP-BNK-005 | `FEBEP.VBLNR` | `bankStatement.clearingStatus`, `bankStatement.clearingDocument` | `IF VBLNR <> '' THEN 'CLEARED' (clearingDocument=VBLNR) ELSE 'UNMATCHED'` | Enum {CLEARED, UNMATCHED} | ERR-VAL-003 | Auto-clearing status; unmatched lines flow to bank-reconciliation exception handling. |

## Notes

- **Field naming:** SAP field abbreviations follow the FINSTA/EBS model — `KUKEY` (statement key), `AZNUM` (statement number), `KONTS` (bank G/L account), `ESNUM` (line-item number), `KWBTR` (amount), `VALUT` (value date), `VGTYP` (transaction type), `VBLNR` (posting/clearing document).
- **Statement normalisation:** MT940/CAMT.053 differences are absorbed upstream in the IDoc mapping so FinSight receives a single canonical shape regardless of originating bank.
- **Reconciliation link:** `clearingDocument` (`VBLNR`) is the join key back to GL/AP/AR clearing, enabling end-to-end bank-to-book reconciliation (D5).
