# Canonical Facts — Single Source of Truth

> **Purpose:** Every deliverable (D1–D9) MUST reference these exact names, IDs, codes, and
> conventions. Cross-reference consistency is directly scored. Do not invent alternative
> names for domains, endpoints, tenants, or company codes anywhere in the repository.

---

## 1. Engagement identity

| Item | Value |
|---|---|
| Project code | `493560B` |
| Project | FDE-9B — Custom API Integration (SAP S/4HANA → Zetheta FinSight) |
| Candidate | Anutosh Mishra *(rename in file headers if different)* |
| Repository | `FDE-9B-AnutoshMishra-BATCH2026` (PRIVATE) |
| Collaborator | `@ZethetaIntern` (Admin) — repo transferred Day 15–30 |
| Submission file convention | `493560B_AnutoshMishra_{DeliverableName}` |

## 2. Client — Meridian Manufacturing Ltd.

- Mid-sized Indian manufacturer, **INR 2,400 crore** revenue, 7 plants.
- SAP S/4HANA **2023 FPS02**, ~**2.1M financial transactions/month**.
- Target analytics platform: **Zetheta FinSight 4.2** (AWS Mumbai `ap-south-1`).
- Data warehouse: Snowflake Enterprise (~15 TB). Identity: Azure AD. Monitoring: Nagios + Grafana.
- Network: MPLS + SD-WAN, avg 450 Mbps (Pune DC ↔ AWS Mumbai).

### Company codes → plants → FinSight tenant (CANONICAL routing)

| SAP Company Code | Plants | Location | FinSight Tenant |
|---|---|---|---|
| `MC01` | PL01 Pune, PL02 Nashik, PL03 Aurangabad | Maharashtra | `MERIDIAN-MC01` |
| `MC02` | PL04 Ahmedabad, PL05 Rajkot | Gujarat | `MERIDIAN-MC02` |
| `MC03` | PL06 Chennai, PL07 Coimbatore | Tamil Nadu | `MERIDIAN-MC03` |

- SAP Production system: **PRD, Client 100**.
- Fiscal year variant **V3** (Apr–Mar). Group currency **INR**. Special periods **013–016**.

## 3. The 10 data domains + 12 endpoints (CANONICAL IDs)

Mapping domains = 10. Source/destination endpoints = 12 each (Budget/Actual and Inventory
are extracted as endpoints; their mappings fold into GL/Material Ledger analytics).

| Domain | Source EP | Dest EP | Primary SAP tables | Extraction | Freq | # Mappings |
|---|---|---|---|---|---|---|
| General Ledger | SRC-001 | DST-001 | ACDOCA, BKPF, BSEG | ODP delta (CDS) | 30 min | 12 |
| Accounts Payable | SRC-002 | DST-002 | BSIK, LFA1, LFC1 | ODP delta | 30 min | 8 |
| Accounts Receivable | SRC-003 | DST-003 | BSID, KNA1, KNC1 | ODP delta | 30 min | 8 |
| Cost Centre Accounting | SRC-004 | DST-004 | CSKS, CSKT, COSS | Batch (master) | 4 h | 6 |
| Profit Centre Accounting | SRC-005 | DST-005 | CEPC, CEPCT | Batch (master) | 4 h | 4 |
| Material Ledger | SRC-006 | DST-006 | MBEW, COSP | ODP delta | 60 min | 5 |
| Purchase Orders | SRC-007 | DST-007 | EKKO, EKPO, EKET | ODP delta | 30 min | 4 |
| Sales Orders | SRC-008 | DST-008 | VBAK, VBAP, VBEP | ODP delta | 30 min | 3 |
| Fixed Assets | SRC-009 | DST-009 | ANLA, ANLZ, ANLP | Batch | Daily | 3 |
| Bank Statements | SRC-010 | DST-010 | FEBKO, FEBEP | IDoc (FINSTA) | Event | 3 |
| *(Budget/Actual)* | SRC-011 | DST-011 | COSP, COSS | Batch | Daily | folds → GL/CC |
| *(Inventory)* | SRC-012 | DST-012 | MARD, MBEW | ODP delta | 60 min | folds → ML |

**Total mandatory mappings ≥ 56 (target 80+ for the Data Alchemist badge).**

### FinSight destination resources (nouns, versioned `/api/v2/`)

`/journal-entries`, `/payables`, `/receivables`, `/cost-centres`, `/profit-centres`,
`/material-ledger`, `/purchase-orders`, `/sales-orders`, `/fixed-assets`,
`/bank-statements`, `/budget-actuals`, `/inventory`.

## 4. Cross-domain reference data

- **Exchange rates:** SAP `TCURR` (KURST=`M`), point-in-time by `GDATU`. ISO 4217 target.
- **Hierarchies:** `SETNODE` / `SETLEAF` — flatten cost-centre & profit-centre to Level1–Level7.
- **Master data** must sync before transactional (referential integrity: GL→CC, GL→PC, AP→vendor, AR→customer).

## 5. Business constraints (design must respect ALL)

| Constraint | Rule |
|---|---|
| SAP batch window | Job `RGGBS000` runs 01:00–04:30 IST — no heavy extraction |
| RFC pool | ≤ **50 concurrent** RFC connections |
| ODP frequency | ≤ once / **30 min** per provider |
| Bandwidth | ≤ **25%** of 450 Mbps MPLS in business hours |
| Maintenance | SAP: 2nd & 4th Sat 22:00–06:00 IST · FinSight: 1st Sun 02:00–06:00 IST |
| Data residency | **India only** (RBI) — all processing in AWS `ap-south-1` |
| Audit | Full lineage: SAP document number → FinSight record |
| GST | Preserve CGST/SGST/IGST breakdown on any invoice-amount transform |

## 6. Target architecture (CANONICAL component names)

`SAP S/4HANA` → **`ODP Extractor`** → **`Kafka`** (topics per domain + `dlq`) →
**`Transformation Engine`** (content-based router) → **`FinSight Loader`** → `Zetheta FinSight`.
Cross-cutting: **`Scheduler`**, **`Reconciliation Service`**, **`Monitoring Stack`**
(Prometheus + Grafana + ELK + PagerDuty), **`Audit/Wire-Tap`**, **`DLQ`**.

- Message broker: **Apache Kafka** (3 brokers, RF=3, ISR=2). Partitions by company code.
- Idempotency key on FinSight `POST` = business key (e.g. `documentId`).
- Auth: SAP via **OAuth2 + X.509** on SAP Gateway/OData; FinSight via **OAuth 2.0** (`client_credentials`, refresh token).

## 7. Error model (CANONICAL — see D4)

- Classes: `TRANSIENT`, `PERMANENT`, `DATA_QUALITY`, `SYSTEM`.
- Backoff: `wait = min(cap, random(base, base * 2^attempt))`.
- Circuit breaker states: `CLOSED / OPEN / HALF-OPEN`.
- Error-code registries: `ERR-EXT-*`, `ERR-MAP-*`, `ERR-LOAD-*`, `ERR-RECON-*`, `ERR-SYS-*`
  (Appendix J) + FinSight HTTP codes (Appendix B).

## 8. SLA & data quality targets

| Metric | Target |
|---|---|
| Data freshness | ≤ 4 h (from 24 h) — **83% improvement** headline for CFO |
| Completeness | > 99.5% · Accuracy > 99.9% · Consistency 100% |
| Timeliness (batch within SLA) | > 98% · Validity > 99% · Uniqueness 100% |
| Recon | Zero debit–credit variance per document; zero-break target across 10 domains |

## 9. Naming conventions (Zetheta standard — enforce everywhere)

| Artifact | Pattern | Example |
|---|---|---|
| Documents | `D{n}_{Name}_{version}.md` | `D1_Integration_Architecture_v1.md` |
| Diagrams | `DGM_{Type}_{Name}.(drawio\|png\|mmd)` | `DGM_C4_SystemContext.mmd` |
| API specs | `API_{System}_{Domain}.yaml` | `API_SAP_GeneralLedger.yaml` |
| Mappings | `MAP_{Domain}_{version}.(md\|xlsx)` | `MAP_GeneralLedger_v1.md` |
| Test plans | `TST_{Category}_{ID}.md` | `TST_Functional_001.md` |

## 10. Stakeholder personas (tune D9 to these)

| Persona | Role | Wants |
|---|---|---|
| Ananya Krishnan | CFO | KPIs, ROI, no jargon, data-freshness promise |
| Rajesh Venkataraman | VP IT Infra | SLA, security, change management |
| Priya Deshmukh | SAP Basis Admin | RFC limits, tcodes, transport, perf impact |
| Marcus Wei | Zetheta Platform Eng | API contracts, JSON schemas, throughput |
| Dr. Sanjay Kulkarni | Head Internal Audit | lineage, reconciliation, audit trail |
| Fatima Al-Hassan | Mfg Ops Director | cost allocation, variance reports |
