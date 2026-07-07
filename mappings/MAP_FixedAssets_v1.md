# MAP_FixedAssets_v1

**Project:** 493560B — FDE-9B Custom API Integration (SAP S/4HANA → Zetheta FinSight)
**Domain:** Fixed Assets · **Source EP:** SRC-009 · **Dest EP:** DST-009
**Source tables:** ANLA (asset master — general), ANLZ (asset time-dependent data), ANLP (asset periodic values); ANLB (depreciation terms) for depreciation key
**FinSight resource:** `POST /api/v2/fixed-assets`
**Extraction:** Batch, daily · **Idempotency key:** `fixedAsset.assetId`
**Mapping count:** 5

> Depreciation method mapped from the SAP depreciation key; periodic depreciation from ANLP.

| Mapping ID | Source Field | Target Field | Transformation Rule | Validation Rule | Error Handling | Business Context |
|---|---|---|---|---|---|---|
| MAP-FA-001 | `ANLA.BUKRS` + `ANLA.ANLN1` + `ANLA.ANLN2` | `fixedAsset.assetId` | `CONCAT(BUKRS,'-',LTRIM(ANLN1,'0'),'-',ANLN2)` | NOT NULL; regex `^[A-Z0-9]{2,6}-\d{1,12}-\d{1,4}$`; unique | ERR-MAP-001; ERR-VAL-001 / ERR-VAL-006 | Composite asset key (main number + sub-number); asset-register lineage. |
| MAP-FA-002 | `ANLA.ANLKL` | `fixedAsset.assetClass` | MAP `ANLKL` → enum {LAND, BUILDING, PLANT_MACHINERY, FURNITURE, VEHICLE, IT_EQUIPMENT, INTANGIBLE} | Must map to a known asset class | ERR-VAL-003 | Asset class governs Schedule II useful-life and depreciation grouping. |
| MAP-FA-003 | `ANLA.TXT50` | `fixedAsset.description` | Direct map; TRIM | NOT NULL; max length 50 | ERR-MAP-001; ERR-VAL-001 | Asset description for the fixed-asset register and audit. |
| MAP-FA-004 | `ANLZ.KOSTL` + `ANLZ.PRCTR` | `fixedAsset.costCentre`, `fixedAsset.profitCentre` | `LTRIM(KOSTL,'0')`, `LTRIM(PRCTR,'0')` then validate against masters | If present, must exist in CC/PC dimensions | ERR-MAP-003; ERR-VAL-004 | Cost/profit-centre assignment drives depreciation posting allocation. |
| MAP-FA-005 | `ANLB.AFASL` + `ANLP.NAFAZ` + `ANLA.AKTIV` | `fixedAsset.depreciationMethod`, `fixedAsset.periodicDepreciation`, `fixedAsset.capitalizationDate` | MAP `AFASL` (depreciation key) → method enum {SLM, WDV, MANUAL, NO_DEP}; `periodicDepreciation = DECIMAL(NAFAZ,2)`; `capitalizationDate = FORMAT(AKTIV,'YYYY-MM-DD')` | Method must map; `NAFAZ` within amount range; `AKTIV` valid or null (AuC) | ERR-MAP-001; ERR-VAL-002 / ERR-VAL-003 | Depreciation method + periodic charge for asset-value roll-forward and depreciation reporting. |

## Notes

- **Depreciation area:** ANLP periodic values are area-dependent (`AFABE`); the primary book-depreciation area (`AFABE='01'`) is loaded by default, with tax area(s) available as additional records when required.
- **Time-dependent assignment:** ANLZ is `BDATU`/`ADATU` interval-keyed; the record valid on the extraction date is selected for cost/profit-centre assignment.
- **Under-construction assets:** assets with null `ANLA.AKTIV` (not yet capitalised, AuC) are loaded with `capitalizationDate=null` and `depreciationMethod=NO_DEP` rather than rejected.
