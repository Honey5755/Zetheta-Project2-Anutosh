# MAP_ProfitCentre_v1

**Project:** 493560B — FDE-9B Custom API Integration (SAP S/4HANA → Zetheta FinSight)
**Domain:** Profit Centre Accounting · **Source EP:** SRC-005 · **Dest EP:** DST-005
**Source tables:** CEPC (profit-centre master), CEPCT (profit-centre texts); SETNODE/SETLEAF for hierarchy flattening
**FinSight resource:** `POST /api/v2/profit-centres`
**Extraction:** Batch master, every 4 h · **Idempotency key:** `profitCentre.profitCentreId`
**Mapping count:** 6

> Profit-centre master must sync before GL/CO transactional load (GL→PC referential integrity). The hierarchy is flattened to `Level1..Level7` via the SETNODE/SETLEAF walk (spec §5.3). Segment drives segment reporting.

| Mapping ID | Source Field | Target Field | Transformation Rule | Validation Rule | Error Handling | Business Context |
|---|---|---|---|---|---|---|
| MAP-PC-001 | `CEPC.PRCTR` | `profitCentre.profitCentreId` | `LTRIM(PRCTR,'0')` | NOT NULL; regex `^[A-Z0-9]{1,10}$`; unique within controlling area | ERR-MAP-001; ERR-VAL-001 / ERR-VAL-006 | Primary key of the profit-centre dimension; target for GL→PC references. |
| MAP-PC-002 | `CEPCT.KTEXT` | `profitCentre.profitCentreName` | Enrichment join on (`PRCTR`,`DATBI`) with `SPRAS='E'`; TRIM | NOT NULL; max length 20 | ERR-MAP-001; ERR-VAL-001 | Name for profitability and segment reports. |
| MAP-PC-003 | `CEPC.SEGMENT` | `profitCentre.segment` | Direct map; validate against segment master | Must exist in segment master; NOT NULL for reporting-relevant PCs | ERR-MAP-003; ERR-VAL-004 | Segment underpins IFRS 8 / Ind AS 108 segment reporting to the CFO. |
| MAP-PC-004 | `CEPC.KOKRS` | `profitCentre.controllingArea` | Direct map | NOT NULL; must exist in controlling-area master | ERR-VAL-003; ERR-VAL-004 | Controlling area scopes the profit-centre namespace. |
| MAP-PC-005 | `CEPC.KHINR` + `SETNODE.*` + `SETLEAF.*` | `profitCentre.hierarchyLevel1` … `hierarchyLevel7` | Recursive SETNODE/SETLEAF walk from standard-hierarchy root to leaf `PRCTR`; emit ancestor names into `Level1..Level7`; collapse >7 into Level7, null-fill shallower | Leaf resolves to one root path; no cycles | ERR-MAP-003; ERR-VAL-004 | Flattens the hierarchy for segment roll-ups and drill-down dashboards. |
| MAP-PC-006 | `CEPC.DATAB` + `CEPC.DATBI` | `profitCentre.validFrom`, `profitCentre.validTo` | `FORMAT(DATAB,'YYYY-MM-DD')`, `FORMAT(DATBI,'YYYY-MM-DD')` | `validFrom ≤ validTo`; `DATBI` default `9999-12-31` | ERR-MAP-002; ERR-VAL-005 | Time-dependent validity for point-in-time-correct dimension joins. |

## Notes

- **Hierarchy set class:** profit-centre standard hierarchy uses `SETNODE.SETCLASS='0106'`; the same recursive flattening engine as cost centres is reused with a different set class.
- **Segment fallback:** if `CEPC.SEGMENT` is blank, the loader derives segment from the Level2 hierarchy node with a `segmentDerived=true` flag rather than rejecting the record.
- **Tenant routing:** `CEPC.BUKRS` maps to the FinSight tenant using the same `MCnn`→`MERIDIAN-MCnn` rule as the GL and Cost Centre domains.
