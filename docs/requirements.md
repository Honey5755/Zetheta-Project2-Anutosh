# Requirements — FDE-9B SAP S/4HANA → Zetheta FinSight Integration

| | |
|---|---|
| **Project code** | `493560B` |
| **Client** | Meridian Manufacturing Ltd. |
| **Document** | Requirements Specification |
| **Version** | v1 |
| **Status** | Issued for review |

---

## 1. Business Objectives

| # | Objective | Measure of success |
|---|---|---|
| BO-1 | Reduce financial-data latency from 24h to under 4h | ≥ 83% freshness improvement; delta domains visible in FinSight ≤ 4h |
| BO-2 | Give the CFO near-real-time visibility of cash, cost and margin across 3 company codes / 7 plants | Dashboards refreshed within SLA per tenant MERIDIAN-MC01..03 |
| BO-3 | Provide auditable, provable data integrity | Zero debit–credit variance per document; zero-break reconciliation across 10 domains |
| BO-4 | Ensure regulatory compliance | 100% India data residency (RBI); GST breakdown preserved on amount transforms |
| BO-5 | Protect SAP production performance | No breach of RFC pool, ODP frequency, or batch-window constraints |
| BO-6 | Build for growth | Sustain 10× current volume and absorb a 5× festival spike without data loss |

---

## 2. Functional Requirements (per domain)

Extraction covers 12 source endpoints (SRC-001…012) mapped to 12 FinSight destination resources (DST-001…012). Company code is the routing key: `MC01→MERIDIAN-MC01`, `MC02→MERIDIAN-MC02`, `MC03→MERIDIAN-MC03`.

| Req ID | Domain | Source EP → Dest EP | Extraction / Freq | FinSight resource | Key functional notes |
|---|---|---|---|---|---|
| FR-GL | General Ledger | SRC-001 → DST-001 | ODP delta / 30 min | `/journal-entries` | ACDOCA/BKPF/BSEG; composite key, currency, V3 fiscal period, debit=credit per doc |
| FR-AP | Accounts Payable | SRC-002 → DST-002 | ODP delta / 30 min | `/payables` | BSIK/LFA1/LFC1; ageing buckets, vendor enrichment, GSTIN validation |
| FR-AR | Accounts Receivable | SRC-003 → DST-003 | ODP delta / 30 min | `/receivables` | BSID/KNA1/KNC1; credit limit, dunning level |
| FR-CC | Cost Centre Accounting | SRC-004 → DST-004 | Batch (master) / 4 h | `/cost-centres` | CSKS/CSKT/COSS; hierarchy flattening L1–L7 |
| FR-PC | Profit Centre Accounting | SRC-005 → DST-005 | Batch (master) / 4 h | `/profit-centres` | CEPC/CEPCT; segment reporting alignment |
| FR-ML | Material Ledger | SRC-006 → DST-006 | ODP delta / 60 min | `/material-ledger` | MBEW/COSP; actual costing, price-difference allocation |
| FR-PO | Purchase Orders | SRC-007 → DST-007 | ODP delta / 30 min | `/purchase-orders` | EKKO/EKPO/EKET; GR/IR, pricing conditions |
| FR-SO | Sales Orders | SRC-008 → DST-008 | ODP delta / 30 min | `/sales-orders` | VBAK/VBAP/VBEP; revenue-recognition stage |
| FR-FA | Fixed Assets | SRC-009 → DST-009 | Batch / daily | `/fixed-assets` | ANLA/ANLZ/ANLP; depreciation method |
| FR-BK | Bank Statements | SRC-010 → DST-010 | IDoc (FINSTA) / event | `/bank-statements` | FEBKO/FEBEP; statement format normalisation |
| FR-BA | Budget/Actual | SRC-011 → DST-011 | Batch / daily | `/budget-actuals` | COSP/COSS; folds into GL/CC analytics |
| FR-IN | Inventory | SRC-012 → DST-012 | ODP delta / 60 min | `/inventory` | MARD/MBEW; folds into Material Ledger analytics |

**Cross-cutting functional requirements**

- FR-X1 **Delta tokens:** advance only on successful extraction (no data loss on retry).
- FR-X2 **Master before transactional:** enforce referential integrity (GL→CC, GL→PC, AP→vendor, AR→customer).
- FR-X3 **Currency:** point-in-time conversion via TCURR (KURST=`M`, by `GDATU`) to ISO 4217.
- FR-X4 **Hierarchies:** flatten cost/profit-centre via SETNODE/SETLEAF to Level1–Level7.
- FR-X5 **GST:** preserve CGST/SGST/IGST on any invoice-amount transform.
- FR-X6 **Idempotency:** FinSight `POST` idempotency key = business key (`documentId`).
- FR-X7 **Reconciliation:** per-document and per-domain completeness/accuracy/consistency/timeliness.
- FR-X8 **Audit lineage:** SAP document number → FinSight record, retained immutably.
- FR-X9 **Mappings:** ≥ 56 mandatory field mappings (target 80+).

---

## 3. Non-Functional Requirements

| Category | Requirement |
|---|---|
| Performance | Delta data freshness ≤ 4h; aggressive per-call timeouts (~5s API, ~30s batch) |
| Scalability | Design for 10× volume; absorb 5× festival spike with no loss; autoscale on consumer lag |
| Availability | Kafka RF=3/ISR=2 (tolerate 1 broker loss); RDS Multi-AZ; ≥2 stateless replicas across AZs |
| Security | TLS/IPsec in transit; encryption at rest; OAuth2+X.509 (SAP), OAuth 2.0 client_credentials (FinSight); Azure AD identities; least-privilege; no public ingress |
| Data residency | 100% processing in AWS `ap-south-1` (RBI) |
| Data quality | Completeness > 99.5%, Accuracy > 99.9%, Consistency 100%, Timeliness > 98%, Validity > 99%, Uniqueness 100% |
| Observability | Metrics (Prometheus/Grafana), logs (ELK, structured JSON w/ correlation_id), alerting (PagerDuty P1–P4) |
| Auditability | Full lineage SAP document number → FinSight record |
| Maintainability | Versioned OpenAPI 3.0 contracts; expand-contract schema migration; blue-green / rolling deploys |

---

## 4. Constraints (canonical §5 — all must be respected)

| # | Constraint | Rule |
|---|---|---|
| C-1 | SAP batch window | Job `RGGBS000` runs 01:00–04:30 IST — no heavy extraction in this window |
| C-2 | RFC pool | ≤ **50 concurrent** RFC connections (exceeding blocks SAP dialog users) |
| C-3 | ODP frequency | ≤ once / **30 min** per provider |
| C-4 | Bandwidth | ≤ **25%** of 450 Mbps MPLS during business hours |
| C-5 | Maintenance windows | SAP: 2nd & 4th Sat 22:00–06:00 IST · FinSight: 1st Sun 02:00–06:00 IST |
| C-6 | Data residency | India only (RBI) — all processing in AWS `ap-south-1` |
| C-7 | Audit | Full lineage: SAP document number → FinSight record |
| C-8 | GST | Preserve CGST/SGST/IGST breakdown on any invoice-amount transform |

---

## 5. Success Criteria

- SC-1: Delta domains demonstrate ≤ 4h freshness in UAT and PROD (BO-1).
- SC-2: Reconciliation shows zero debit–credit variance per document and zero-break across all 10 domains (BO-3).
- SC-3: All data-quality thresholds (§3) met in acceptance testing.
- SC-4: No constraint breach observed (RFC ≤ 50, ODP ≥ 30 min, bandwidth ≤ 25%, batch window respected) (C-1…C-4).
- SC-5: Independent audit confirms complete lineage SAP doc → FinSight record (C-7).
- SC-6: GST breakdown preserved on 100% of sampled invoice-amount transforms (C-8).
- SC-7: A simulated 5× spike processes without data loss and within SLA (BO-6).
- SC-8: All processing verified resident in `ap-south-1` (C-6).

---

## 6. Assumptions

- A-1: SAP Basis provisions the ODP providers / CDS views and a dedicated RFC pool of up to 50 connections.
- A-2: FinSight 4.2 exposes stable `/api/v2/` resources with OAuth 2.0 `client_credentials` and idempotency support.
- A-3: MPLS + SD-WAN link sustains an average 450 Mbps and the integration is entitled to up to 25% in business hours.
- A-4: Azure AD is the identity provider for service principals.
- A-5: Reference data (TCURR, SETNODE/SETLEAF) is accessible for enrichment.
- A-6: AWS `ap-south-1` capacity is available for the full topology and all backups remain in-region.
- A-7: Company-code-to-tenant mapping (MC01→MERIDIAN-MC01, etc.) is fixed for the engagement.
- A-8: Festival-season spikes are bounded at ≈5× normal volume.

---

## 7. Stakeholder Matrix

| Persona | Role | Primary interest | What they evaluate |
|---|---|---|---|
| Ananya Krishnan | CFO | KPIs, ROI, data-freshness promise | Business value, freshness, risk communication (no jargon) |
| Rajesh Venkataraman | VP IT Infrastructure | SLA, security, change management | System stability, security compliance, change control |
| Priya Deshmukh | SAP Basis Admin | RFC limits, tcodes, transport, performance | Performance impact on SAP, RFC pool, transport management |
| Marcus Wei | Zetheta Platform Engineer | API contracts, JSON schemas, throughput | Contract stability, throughput, schema compatibility |
| Dr. Sanjay Kulkarni | Head of Internal Audit | Lineage, reconciliation, audit trail | Data integrity, audit trail, reconciliation accuracy |
| Fatima Al-Hassan | Manufacturing Operations Director | Cost allocation, variance reports | Cost-allocation accuracy, production-variance analysis |

---

*End of Requirements Specification v1.*
