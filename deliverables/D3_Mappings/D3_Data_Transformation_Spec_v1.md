# D3 — Data Transformation & Mapping Specification (v1)

**Project:** 493560B — FDE-9B Custom API Integration (SAP S/4HANA → Zetheta FinSight)
**Client:** Meridian Manufacturing Ltd. · **Source:** SAP S/4HANA 2023 FPS02 (PRD, Client 100) · **Target:** Zetheta FinSight 4.2 (AWS `ap-south-1`)
**Deliverable:** D3 (Level 4 — "Data Alchemy") · **Submission name:** `493560B_AnutoshMishra_DataTransformationSpec`
**Status:** Complete · **Total field mappings:** **85** (≥ 80 → qualifies for the *Data Alchemist* badge)

---

## 1. Introduction

This document specifies the field-level data transformation and mapping layer that sits inside the **Transformation Engine** (content-based router) of the integration pipeline:

```
SAP S/4HANA → ODP Extractor → Kafka (topic per domain + dlq) → Transformation Engine → FinSight Loader → Zetheta FinSight
```

Every SAP source field consumed by the pipeline is mapped to exactly one FinSight target field through a declarative rule set. Each mapping records not only the structural transformation but also the validation rule, the error-handling behaviour (by registry code), and the business rationale — so the mapping catalogue doubles as living documentation for auditors (Dr. Kulkarni) and platform engineers (Marcus Wei).

The mappings span all **10 data domains**. The two extraction-only endpoints — Budget/Actual (SRC-011) and Inventory (SRC-012) — do not carry an independent target resource; their fields fold into the General Ledger / Cost Centre and Material Ledger analytics respectively, and are referenced within those domain mapping files.

### 1.1 Design principles

- **Universal Journal first.** ACDOCA is the single source of truth for financial postings; BKPF/BSEG are used only for header/segment enrichment.
- **Business key = idempotency key.** Every domain emits a composite `*Id` (e.g. `documentId`) used both as the FinSight `POST` idempotency key and as the audit-lineage anchor back to the SAP document number.
- **Master before transactional.** Cost centres, profit centres, vendors and customers must be synced before dependent transactional loads (referential integrity: GL→CC, GL→PC, AP→vendor, AR→customer).
- **Preserve, don't re-derive.** GST components (CGST/SGST/IGST) and transaction-currency amounts are carried through unchanged; INR figures are added alongside, never in place of, the source values.
- **India residency.** All transformation executes in AWS `ap-south-1`; no field ever leaves Indian borders (RBI localisation).

### 1.2 Companion artefacts

| Artefact | Location | Purpose |
|---|---|---|
| Per-domain mapping files (10) | `mappings/MAP_{Domain}_v1.md` | Full field-by-field tables per domain |
| Machine-readable catalogue | `mappings/MAP_ALL_consolidated.csv` | All 85 mappings, openable as `.xlsx` |
| This specification | `deliverables/D3_Mappings/D3_Data_Transformation_Spec_v1.md` | Template, summaries, algorithms, DQ note |

---

## 2. Mapping template definition

Every mapping — in every domain file and in the consolidated CSV — is expressed with the following seven attributes. This is the canonical D3 template (extended from the Appendix C worked example by adding explicit *Error Handling* and *Business Context* columns).

| Attribute | Definition |
|---|---|
| **Mapping ID** | Stable identifier `MAP-{DOMAIN}-{NNN}` (e.g. `MAP-GL-001`). Immutable across versions. |
| **Source Field** | SAP origin as `Table.FIELD`; composite sources shown as `FIELD_A + FIELD_B`. |
| **Target Field** | FinSight JSON path as `resource.attribute` (camelCase). |
| **Transformation Rule** | Deterministic expression: format, cast, concat, lookup, arithmetic, or enum map. |
| **Validation Rule** | Predicate the transformed value must satisfy (null/format/range/enum/referential/cross-field). |
| **Error Handling** | One or more registry codes (`ERR-MAP-*` transformation, `ERR-VAL-*` validation) with the default action defined in §6. |
| **Business Context** | Why the field matters — the stakeholder/report it serves. |

CSV column order: `MappingID, Domain, SourceTable, SourceField, TargetField, TransformationRule, ValidationRule, ErrorCode, BusinessContext`.

---

## 3. Per-domain summary

| # | Domain | SRC / DST EP | Primary SAP tables | FinSight resource | Mappings | Signature transformations |
|---|---|---|---|---|---|---|
| 1 | General Ledger | SRC-001 / DST-001 | ACDOCA, BKPF, BSEG | `/journal-entries` | **18** | Composite `documentId`, dual-currency amounts, fiscal-period map, reversal & D/C flags, tenant routing |
| 2 | Accounts Payable | SRC-002 / DST-002 | BSIK, LFA1, LFC1 | `/payables` | **12** | Ageing buckets, vendor enrichment, GSTIN validation, clearing status |
| 3 | Accounts Receivable | SRC-003 / DST-003 | BSID, KNA1, KNC1, KNKK | `/receivables` | **12** | Credit-limit exposure, dunning level, ageing, GSTIN |
| 4 | Cost Centre | SRC-004 / DST-004 | CSKS, CSKT, COSS | `/cost-centres` | **9** | 7-level hierarchy flattening, time-slice validity, category enum |
| 5 | Profit Centre | SRC-005 / DST-005 | CEPC, CEPCT | `/profit-centres` | **6** | Segment reporting, hierarchy flattening |
| 6 | Material Ledger | SRC-006 / DST-006 | MBEW, COSP | `/material-ledger` | **7** | Actual costing, unit prices, price-difference allocation |
| 7 | Purchase Orders | SRC-007 / DST-007 | EKKO, EKPO, EKET | `/purchase-orders` | **6** | GR/IR reconciliation, pricing conditions |
| 8 | Sales Orders | SRC-008 / DST-008 | VBAK, VBAP, VBEP | `/sales-orders` | **5** | Revenue-recognition stage |
| 9 | Fixed Assets | SRC-009 / DST-009 | ANLA, ANLZ, ANLP, ANLB | `/fixed-assets` | **5** | Depreciation method, periodic depreciation |
| 10 | Bank Statements | SRC-010 / DST-010 | FEBKO, FEBEP | `/bank-statements` | **5** | Statement normalisation, clearing status |
| | **TOTAL** | | | | **85** | |

> **Total mapping count = 85**, which is **29 beyond the 56-mapping mandatory floor** and **5 beyond the 80-mapping threshold** for the *Data Alchemist* achievement badge. Domain-by-domain counts each meet or exceed the canonical minimum (GL 12→18, AP 8→12, AR 8→12, CC 6→9, PC 4→6, ML 5→7, PO 4→6, SO 3→5, FA 3→5, Bank 3→5).

Full field-by-field detail lives in the ten `MAP_{Domain}_v1.md` files; the consolidated `MAP_ALL_consolidated.csv` carries all 85 rows for spreadsheet review.

---

## 4. Cross-domain reference data

| Reference | Source | Use |
|---|---|---|
| Exchange rates | `TCURR` (`KURST='M'`), point-in-time by `GDATU` | Currency conversion (§5.1) |
| Company-code → tenant | `MC01`→`MERIDIAN-MC01`, `MC02`→`MERIDIAN-MC02`, `MC03`→`MERIDIAN-MC03` | Multi-tenant routing + Kafka partitioning |
| Standard hierarchies | `SETNODE` / `SETLEAF` | Cost-/profit-centre flattening (§5.3) |
| Chart of accounts | Analytics CoA master | GL account mapping (consistency = 100%) |
| ISO 4217 / ISO 3166 | Reference tables | Currency & country validation |

---

## 5. Advanced transformation patterns

### 5.1 Currency conversion — point-in-time TCURR with triangulation

Non-INR transaction amounts are converted to the INR group currency using the exchange rate **effective on the posting date**, with triangulation through INR and a stale-rate guard.

```
function convertToINR(amount, fromCurr, postingDateYYYYMMDD):
    if fromCurr == 'INR':
        return round2(amount)

    gdatu = invertDate(postingDateYYYYMMDD)   # SAP stores GDATU as (99999999 - YYYYMMDD)
    rate  = lookupTCURR(kurst='M', fcurr=fromCurr, tcurr='INR', gdatu>=gdatu, orderBy gdatu ASC, top 1)

    stale = false
    if rate is NULL:
        # No direct pair on/after date -> nearest earlier rate, flag as stale (ERR-MAP-005)
        rate = lookupTCURR(kurst='M', fcurr=fromCurr, tcurr='INR', nearest earlier gdatu)
        stale = true
        if rate is NULL:
            # Triangulate via a base currency (USD) when no direct INR pair exists
            r1 = lookupTCURR('M', fromCurr, 'USD', date)     # from -> USD
            r2 = lookupTCURR('M', 'USD', 'INR', date)        # USD  -> INR
            if r1 is NULL or r2 is NULL: raise ERR-MAP-005   # -> nearest-date fallback, stale flag
            effRate = applyFactors(r1) * applyFactors(r2)
            return { value: round2(amount * effRate), staleRate: true, viaTriangulation: 'USD' }

    effRate = applyFactors(rate)              # UKURS scaled by FFACT/TFACT ratio units
    return { value: round2(amount * effRate), staleRate: stale }

function applyFactors(rate):
    # TCURR quotes UKURS per (from-factor FFACT : to-factor TFACT) units
    return rate.UKURS * (rate.TFACT / rate.FFACT)

function round2(x): return HALF_UP(x, 2)      # 2-decimal rounding, banker's HALF_UP
```

- **`GDATU` inversion:** SAP stores `GDATU` as the 9s-complement of the date (`99999999 - YYYYMMDD`), so the *latest* rate on/before a date is the *smallest* `GDATU` ≥ the inverted value — the lookup orders ascending and takes the first row.
- **`FFACT`/`TFACT`:** ratio factors normalise quotations such as "per 100 JPY"; the effective rate is `UKURS × (TFACT / FFACT)`.
- **Stale-rate flag:** when no rate exists on/after the posting date, the nearest earlier rate is used and `staleRate=true` is emitted with `ERR-MAP-005` logged (does not block the record).
- **Rounding:** all monetary results are rounded HALF_UP to 2 decimals; INR is the group currency, so `amountLC` (`HSL`) is never re-converted — only non-INR `transactionCurrency` amounts are.

### 5.2 Fiscal-to-calendar period mapping (V3, April–March)

Meridian uses fiscal-year variant **V3** (April–March). SAP posting period `MONAT` maps to a calendar month, with special periods 013–016 collapsing into period 012 (March) and flagged.

```
function mapFiscalPeriod(monat, gjahr):
    p = int(monat)
    if 1 <= p <= 12:
        # 001 -> April ... 009 -> December, 010 -> January(+1yr) ... 012 -> March(+1yr)
        calMonth = ((p - 1 + 3) mod 12) + 1
        calYear  = gjahr if p <= 9 else gjahr + 1
        return { fiscalPeriod: p, calendarMonth: calMonth, calendarYear: calYear, specialPeriod: false }
    else if 13 <= p <= 16:
        return { fiscalPeriod: 12, calendarMonth: 3, calendarYear: gjahr + 1,
                 specialPeriod: true, adjustmentPeriod: p }        # year-end adjustments
    else:
        raise ERR-VAL-002        # period outside 001-016 -> DLQ
```

| SAP period | Calendar month | | SAP period | Calendar month |
|---|---|---|---|---|
| 001 | April | | 008 | November |
| 002 | May | | 009 | December |
| 003 | June | | 010 | January (+1 yr) |
| 004 | July | | 011 | February (+1 yr) |
| 005 | August | | 012 | March (+1 yr) |
| 006 | September | | 013–016 | March — `specialPeriod=true` |
| 007 | October | | | (year-end adjustments) |

### 5.3 Hierarchy flattening — SETNODE/SETLEAF recursive walk → Level1..Level7

Cost-centre (`SETCLASS='0101'`) and profit-centre (`SETCLASS='0106'`) standard hierarchies are stored as recursive node/leaf sets. They are flattened to seven fixed columns for star-schema dimensional modelling.

```
function flattenHierarchy(setClass, rootSetName):
    result = {}                       # leafId -> [ancestorName ...]
    walk(setClass, rootSetName, path=[])
    return result

    function walk(setClass, setName, path):
        node = SETHEADER(setClass, setName)          # human-readable node title
        newPath = path + [node.title]
        for child in SETNODE(setClass, subsetOf=setName) order by LINEID:
            walk(setClass, child.SUBSETNAME, newPath)     # recurse into sub-nodes
        for leaf in SETLEAF(setClass, setName) order by LINEID:
            for id in expandRange(leaf.VALFROM, leaf.VALTO):   # leaf may be a value range
                result[id] = newPath

function toLevels(ancestorPath):     # ancestorPath = [L1, L2, ... Ln]
    levels = [null] * 7
    if len(ancestorPath) <= 7:
        for i in 0..len(ancestorPath)-1: levels[i] = ancestorPath[i]      # null-fill trailing
    else:
        for i in 0..5: levels[i] = ancestorPath[i]
        levels[6] = join(ancestorPath[6:], ' / ')                          # collapse depth >7 into Level7
    return { Level1:levels[0], ..., Level7:levels[6] }
```

- **Cycle guard:** a visited-set per root aborts on any revisited `SETNAME` and raises `ERR-MAP-003` (unresolved node → business exception queue).
- **Uniqueness:** a leaf resolving to more than one root path is a data-quality break (`ERR-VAL-004`); the first path wins and the ambiguity is logged.
- **Incremental refresh:** the 4-hour master batch memoises per `SETNAME` and re-flattens only changed subtrees.

### 5.4 GST preservation

Any transformation that touches an invoice amount must keep the tax breakdown intact for GSTR filing (RBI/GST compliance).

```
function preserveGST(sourceLine, targetRecord):
    # NEVER re-derive tax from a net figure; carry the SAP components verbatim
    targetRecord.taxComponents = [
        { type:'CGST', rate: sourceLine.cgstRate, amount: round2(sourceLine.cgstAmt) },
        { type:'SGST', rate: sourceLine.sgstRate, amount: round2(sourceLine.sgstAmt) },
        { type:'IGST', rate: sourceLine.igstRate, amount: round2(sourceLine.igstAmt) }
    ]
    # Invariant: base + CGST + SGST + IGST == gross (within 0.01 tolerance)
    assert abs((base + cgst + sgst + igst) - gross) <= 0.01     # else ERR-VAL-005
    # Domestic intra-state -> CGST+SGST populated, IGST=0
    # Inter-state / import  -> IGST populated, CGST=SGST=0 (driven by vendor/customer country, LAND1)
    return targetRecord
```

- Components are attached to AP (`payable.taxComponents[]`) and AR (`receivable.taxComponents[]`) records; the net/gross amount transforms in MAP-AP-004 / MAP-AR-004 never overwrite them.
- The intra-state vs inter-state split is inferred from vendor/customer country (`LFA1.LAND1` / `KNA1.LAND1`, MAP-AP-012 / MAP-AR-012).

---

## 6. Error-handling registries used by the mappings

Two registries are referenced in the Error-Handling column. `ERR-MAP-*` is the transformation registry from the master error-code registry (Appendix J); `ERR-VAL-*` is the validation extension defined here for D3, consistent with that registry's format. Full retry/circuit-breaker semantics are specified in D4.

### 6.1 Transformation errors — `ERR-MAP-*`

| Code | Class | Trigger | Default action |
|---|---|---|---|
| ERR-MAP-001 | DATA_QUALITY | Source NULL but target NOT NULL | Apply default if defined, else route to DLQ |
| ERR-MAP-002 | DATA_QUALITY | Date format invalid (not `YYYYMMDD` / not a valid date) | Attempt format correction; if it fails, DLQ |
| ERR-MAP-003 | DATA_QUALITY | Referenced cost/profit centre (or hierarchy node) not found in master | Route to business exception queue, notify functional team |
| ERR-MAP-004 | DATA_QUALITY | Currency code not found in ISO 4217 lookup | Check common typos (`RS`→`INR`), else DLQ |
| ERR-MAP-005 | DATA_QUALITY | Exchange rate not found for currency pair + date | Use nearest available date rate with `staleRate=true` flag |
| ERR-MAP-006 | PERMANENT | Transformation runtime error (division by zero, overflow) | Log full context, route to DLQ, alert engineering |

### 6.2 Validation errors — `ERR-VAL-*` (D3 extension)

| Code | Class | Trigger | Default action |
|---|---|---|---|
| ERR-VAL-001 | DATA_QUALITY | Value fails a format/regex/length constraint (e.g. `documentId` pattern, GSTIN) | Route to DLQ (or business queue for GSTIN quarantine) |
| ERR-VAL-002 | DATA_QUALITY | Numeric value out of allowed range | Route to DLQ, log min/max breach |
| ERR-VAL-003 | DATA_QUALITY | Value not in allowed enum/domain (currency, category, status) | Route to DLQ; unmapped enum → notify functional team |
| ERR-VAL-004 | DATA_QUALITY | Referential integrity — referenced master entity missing (GL account, CC, PC, vendor, customer) | Check master-data sync; may require master resync first |
| ERR-VAL-005 | DATA_QUALITY | Cross-field / business-rule failure (date ordering, GST balance, `validFrom≤validTo`) | Log rule name, route to business exception queue |
| ERR-VAL-006 | DATA_QUALITY | Uniqueness violation — duplicate business key in target | Compare payloads: skip if identical (idempotent), alert if different |

---

## 7. Data quality note — the six DMBOK dimensions

The mapping layer is the primary enforcement point for the six DAMA-DMBOK data-quality dimensions. Each mapping's validation rule contributes to one or more dimensions; the pipeline measures and reports against the following targets (aligned with the SLA in the canonical facts).

| Dimension | Definition | How the mappings enforce it | Target |
|---|---|---|---|
| **Completeness** | All required values present | `NOT NULL` validations (e.g. MAP-GL-001/002/005, MAP-AP-003); defaults via ERR-MAP-001 | **> 99.5%** |
| **Accuracy** | Data correctly represents the real-world entity | Deterministic casts/formats; source↔target amount checks; currency conversion (§5.1) | **> 99.9%** |
| **Consistency** | Values consistent across datasets | Referential validations GL→CC/PC, AP→vendor, AR→customer (ERR-VAL-004); sub-ledger↔GL recon accounts | **100%** |
| **Timeliness** | Available within the agreed SLA | 30-min ODP delta cadence; master batch every 4 h; freshness ≤ 4 h end-to-end | **> 98%** |
| **Validity** | Conforms to defined rules/constraints | Regex/format (ERR-VAL-001), range (ERR-VAL-002), enum (ERR-VAL-003), cross-field (ERR-VAL-005) | **> 99%** |
| **Uniqueness** | No duplicate records | Composite business keys as idempotency keys; duplicate detection (ERR-VAL-006) | **100%** |

**Coverage:** the D3 catalogue provides **25+ distinct data-quality rules** (every mapping carries at least one validation predicate; several carry two or three). These feed directly into the D5 reconciliation and D6 monitoring specifications, where per-dimension pass rates are tracked as dashboard panels with alert thresholds.

---

## 8. Change control

| Version | Date | Change |
|---|---|---|
| v1 | 2026-07-07 | Initial complete D3: 85 field mappings across 10 domains; four advanced transformation patterns; `ERR-VAL-*` validation registry; DMBOK data-quality note. |

*All monetary and date semantics conform to SAP S/4HANA 2023 FPS02 field definitions (Appendix A) and Zetheta FinSight 4.2 `/api/v2/` resource schemas. Processing is confined to AWS `ap-south-1` per RBI data-localisation requirements.*
