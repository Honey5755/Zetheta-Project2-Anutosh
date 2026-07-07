# MAP_CostCentre_v1

**Project:** 493560B — FDE-9B Custom API Integration (SAP S/4HANA → Zetheta FinSight)
**Domain:** Cost Centre Accounting · **Source EP:** SRC-004 · **Dest EP:** DST-004
**Source tables:** CSKS (cost-centre master), CSKT (cost-centre texts), COSS (CO totals — internal postings); SETNODE/SETLEAF for standard-hierarchy flattening
**FinSight resource:** `POST /api/v2/cost-centres`
**Extraction:** Batch master, every 4 h · **Idempotency key:** `costCentre.costCentreId`
**Mapping count:** 9

> Cost-centre master must sync before any GL/CO transactional load (GL→CC referential integrity). The 7-level standard hierarchy is flattened to `Level1..Level7` columns via the SETNODE/SETLEAF recursive walk (spec §5.3).

| Mapping ID | Source Field | Target Field | Transformation Rule | Validation Rule | Error Handling | Business Context |
|---|---|---|---|---|---|---|
| MAP-CC-001 | `CSKS.KOSTL` | `costCentre.costCentreId` | `LTRIM(KOSTL,'0')` | NOT NULL; regex `^[A-Z0-9]{1,10}$`; unique within controlling area | ERR-MAP-001; ERR-VAL-001 / ERR-VAL-006 (dup key → alert) | Primary key of the cost-centre dimension; target for all GL→CC references. |
| MAP-CC-002 | `CSKT.KTEXT` | `costCentre.costCentreName` | Enrichment join on (`KOKRS`,`KOSTL`,`DATBI`) with `SPRAS='E'`; TRIM | NOT NULL; max length 20 | ERR-MAP-001; ERR-VAL-001 | Short name for cost reports (Al-Hassan variance analysis). |
| MAP-CC-003 | `CSKT.LTEXT` | `costCentre.description` | Enrichment join with `SPRAS='E'`; TRIM | Optional; max length 40 | ERR-MAP-001 (null → default '') | Long descriptive text for dashboard tooltips. |
| MAP-CC-004 | `CSKS.KOKRS` | `costCentre.controllingArea` | Direct map | NOT NULL; must exist in controlling-area master | ERR-VAL-003; ERR-VAL-004 | Controlling area scopes the cost-centre namespace. |
| MAP-CC-005 | `CSKS.BUKRS` | `costCentre.tenantId` | MAP company code → tenant: `MC01`→`MERIDIAN-MC01`, `MC02`→`MERIDIAN-MC02`, `MC03`→`MERIDIAN-MC03` | Must resolve to a canonical tenant | ERR-MAP-001; ERR-VAL-003 | Routes master record to correct FinSight tenant. |
| MAP-CC-006 | `CSKS.KOSAR` | `costCentre.category` | MAP `KOSAR` → enum {PRODUCTION, SERVICE, ADMIN, SALES, DEVELOPMENT, MANAGEMENT} | Must map to known category | ERR-VAL-003 | Cost-centre category drives cost-allocation and overhead pooling. |
| MAP-CC-007 | `CSKS.VERAK` | `costCentre.personResponsible` | Direct map; TRIM | Optional; max length 20 | ERR-MAP-001 (null → default '') | Accountability owner for variance follow-up. |
| MAP-CC-008 | `CSKS.DATAB` + `CSKS.DATBI` | `costCentre.validFrom`, `costCentre.validTo` | `FORMAT(DATAB,'YYYY-MM-DD')`, `FORMAT(DATBI,'YYYY-MM-DD')` | `validFrom ≤ validTo`; `DATBI` default `9999-12-31` | ERR-MAP-002; ERR-VAL-005 | Time-dependent validity; ensures point-in-time-correct dimension joins. |
| MAP-CC-009 | `CSKS.KHINR` + `SETNODE.*` + `SETLEAF.*` | `costCentre.hierarchyLevel1` … `hierarchyLevel7` | Recursive SETNODE/SETLEAF walk from standard-hierarchy root to leaf `KOSTL`; emit ancestor node names into `Level1..Level7`; pad deeper-than-7 by collapsing into Level7, shallower by null-filling trailing levels | Leaf must resolve to exactly one root path; no cycles | ERR-MAP-003 (unresolved node → business queue); ERR-VAL-004 | Flattens the 7-level hierarchy for dimensional (star-schema) modelling and roll-up reporting. |

## Notes

- **Hierarchy set class:** cost-centre standard hierarchy uses `SETNODE.SETCLASS='0101'`. The walk is memoised per `SETNAME` so a 4-hour batch re-flattens only changed subtrees.
- **Time slices:** CSKS/CSKT are `DATBI`-keyed; the loader selects the record valid on the extraction date to avoid emitting overlapping intervals.
- **COSS usage:** internal-posting totals (`COSS.WKG001–WKG016`) are not loaded here (they fold into GL/Budget-Actual analytics via SRC-011); COSS is referenced only to confirm a cost centre carries activity for completeness checks.
