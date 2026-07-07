# Technical Handoff — Client IT

| | |
|---|---|
| **Document** | `D9b_IT_Technical_Handoff.md` |
| **Project** | 493560B — SAP S/4HANA → Zetheta FinSight integration |
| **Client** | Meridian Manufacturing Ltd. |
| **Prepared for** | Rajesh Venkataraman (VP IT Infrastructure), Priya Deshmukh (SAP Basis Administrator) |
| **Prepared by** | Anutosh Mishra, Forward Deployed Engineer |
| **Version** | v1.0 · **Date** 2026-07-07 |
| **Classification** | Confidential — Meridian IT & Zetheta only |

> **Purpose.** This is the operational handover for the integration that streams financial data
> from SAP S/4HANA (**PRD, Client 100**) into Zetheta FinSight 4.2 (AWS `ap-south-1`).
> It documents every change we require in your environment, the performance envelope we
> operate within, the security posture, how our monitoring feeds Nagios, and the support model.
> All transport numbers below are **placeholders** to be confirmed by Basis before import.
>
> **Non-negotiable constraints we honour** (Canonical Facts §5): batch blackout during
> `RGGBS000` (01:00–04:30 IST), **≤ 50 concurrent RFC connections**, ODP delta
> **≤ once / 30 min per provider**, **≤ 25%** of the 450 Mbps MPLS in business hours,
> **India-only** data residency, and full SAP-document-to-FinSight lineage for audit.

**Contents**
1. Changes to the SAP environment
2. Network & firewall requirements
3. Performance impact assessment
4. Security requirements
5. Monitoring integration (Nagios)
6. Support handover (L1/L2/L3)
7. Maintenance schedule

---

## Section 1 — Changes to the SAP environment

All changes are delivered via the transport path **DEV → QAS → PRD**. Nothing is created
directly in PRD. Transport numbers are placeholders pending Basis confirmation.

### 1.1 RFC destinations

| RFC destination | Type | Purpose | Target |
|---|---|---|---|
| `ZFINSIGHT_ODP` | T (TCP/IP, registered) | ODP delta extraction channel used by the ODP Extractor | Extractor service (mTLS via SAP Cloud Connector / reverse proxy) |
| `ZFINSIGHT_META` | G (HTTP to ext. server) | Metadata / CDS discovery calls | Integration control plane |
| `ZFINSIGHT_IDOC` | 3 (ABAP) → port | Inbound/outbound IDoc processing for bank statements | IDoc port `ZFINSTA` |

- Connections use the integration **service account** `SVC_FINSIGHT` (Section 4), not a
  dialog user.
- RFC pool usage is capped so the integration never exceeds **50 concurrent** connections
  (enforced client-side and via RZ12 group quota — see Section 3).

### 1.2 ODP subscriptions

| ODP provider (context `ABAP_CDS`) | Domain(s) | Delta frequency | Source tables |
|---|---|---|---|
| `Z_ODP_GL` | General Ledger (`SRC-001`) | 30 min | ACDOCA, BKPF, BSEG |
| `Z_ODP_AP` | Accounts Payable (`SRC-002`) | 30 min | BSIK, LFA1, LFC1 |
| `Z_ODP_AR` | Accounts Receivable (`SRC-003`) | 30 min | BSID, KNA1, KNC1 |
| `Z_ODP_ML` | Material Ledger (`SRC-006`) | 60 min | MBEW, COSP |
| `Z_ODP_PO` | Purchase Orders (`SRC-007`) | 30 min | EKKO, EKPO, EKET |
| `Z_ODP_SO` | Sales Orders (`SRC-008`) | 30 min | VBAK, VBAP, VBEP |
| `Z_ODP_INV` | Inventory (`SRC-012`) | 60 min | MARD, MBEW |

- Delta tokens are managed via ODQMON; on failure the token is **not** advanced (no data loss
  on retry).
- Cost Centre / Profit Centre / Fixed Assets are **batch/master** (not ODP): 4 h / 4 h / daily.
- Delta initialisation will be run once in test mode (`RODPS_REPL_TEST`) to avoid a full-load
  storm at go-live.

### 1.3 IDoc partner profiles

| Object | Value |
|---|---|
| Message type | `FINSTA` (bank statement) |
| Partner type | LS (logical system) |
| Logical system | `ZFINSIGHTLS` |
| Inbound/outbound port | `ZFINSTA` (tRFC) |
| Process code | inbound `FINS` / outbound as configured |
| Domain | Bank Statements (`SRC-010`, event-driven) |

### 1.4 Custom CDS consumption views

| CDS view | Exposes | Domain | Transport (placeholder) |
|---|---|---|---|
| `Z_I_GLINTEG` | GL line items (ACDOCA-based, delta-enabled) | GL | `PRDK900101` |
| `Z_I_APINTEG` | Vendor open items + master | AP | `PRDK900101` |
| `Z_I_ARINTEG` | Customer open items + master | AR | `PRDK900101` |
| `Z_I_MLINTEG` | Material ledger / actual costing | ML | `PRDK900102` |
| `Z_I_COHIER` | Cost-/profit-centre hierarchy (SETNODE/SETLEAF flattening feed) | CC/PC | `PRDK900102` |

**Transport import order to PRD** (placeholders):

| Order | Transport | Contents |
|---|---|---|
| 1 | `PRDK900101` | CDS views `Z_I_GLINTEG`, `Z_I_APINTEG`, `Z_I_ARINTEG` |
| 2 | `PRDK900102` | CDS views `Z_I_MLINTEG`, `Z_I_COHIER` + ODP provider registration |
| 3 | `PRDK900103` | RFC destinations + authorisation role `Z_INTEG_RFC` |
| 4 | `PRDK900104` | IDoc partner profile + `ZFINSTA` port config |

> **Action for Basis (Priya):** confirm/replace transport numbers, schedule QAS regression
> before PRD import, and verify no object collisions with existing Z-namespace.

---

## Section 2 — Network & firewall requirements

Connectivity is **Pune data centre (SAP)** ↔ **AWS Mumbai `ap-south-1` (FinSight)** over the
existing MPLS + SD-WAN, augmented by an IPsec VPN tunnel for the control/data plane.

### 2.1 Firewall rules

| # | Source | Destination | Port / Protocol | Purpose |
|---|---|---|---|---|
| FW1 | SAP app servers (Pune) | Extractor VIP (AWS) | TCP **443** (HTTPS/mTLS) | ODP/OData extraction over TLS |
| FW2 | Extractor (AWS) | SAP Gateway / OData (Pune) | TCP **44300** (HTTPS OData) | CDS consumption via SAP Gateway |
| FW3 | SAP (Pune) | Extractor registered server | TCP **33xx / 48xx** (RFC/gateway, secure) | Registered RFC (`ZFINSIGHT_ODP`) |
| FW4 | IDoc port | Integration IDoc endpoint | TCP **443** | `FINSTA` IDoc exchange |
| FW5 | FinSight Loader (AWS) | FinSight API edge (AWS) | TCP **443** | Load into `/api/v2/*` |
| FW6 | Integration nodes | OAuth token endpoint | TCP **443** | OAuth 2.0 `client_credentials` |
| FW7 | Monitoring agent (AWS) | Nagios/NSCA (Pune) | TCP **5667** (NSCA) / **443** | Passive check results to Nagios |
| FW8 | Prometheus (AWS) | Exporters | TCP **9100/9308** | Metrics scrape (internal VPC only) |

- Ports are least-privilege; no wildcard rules. SAP RFC uses **Secure Network Communication
  (SNC)** where the gateway supports it.

### 2.2 IP ranges

| Zone | CIDR (placeholder — confirm with network team) |
|---|---|
| SAP app/gateway subnet (Pune DC) | `10.20.10.0/24` |
| Integration VPC — private subnets (AWS `ap-south-1`) | `10.60.0.0/20` |
| Extractor / NAT egress to SAP | fixed EIPs `10.60.4.11`, `10.60.4.12` (allow-list on SAP firewall) |
| Nagios / NSCA collector (Pune) | `10.20.30.15` |

### 2.3 VPN tunnel specification

| Parameter | Value |
|---|---|
| Type | Site-to-site IPsec (IKEv2) |
| Endpoints | Pune DC firewall ↔ AWS VPN Gateway (`ap-south-1`) |
| Phase 1 | AES-256, SHA-256, DH group 14, lifetime 28800 s |
| Phase 2 | AES-256-GCM, PFS group 14, lifetime 3600 s |
| Redundancy | Dual tunnels (active/standby) with BGP failover |
| Interesting traffic | SAP subnet `10.20.10.0/24` ↔ integration `10.60.0.0/20` |

All traffic stays within India; no route egresses AWS `ap-south-1`.

---

## Section 3 — Performance impact assessment

Design goal: **negligible impact on SAP production dialog users** and full compliance with the
constraint envelope.

### 3.1 SAP production impact

| Resource | Expected impact | Control |
|---|---|---|
| **RFC connections** | ≤ **50 concurrent** at peak (typically 20–30) | RZ12 server-group quota `ZFINSIGHT` hard-caps at 50; client-side connection pool limit; alarm at 45 |
| **CPU** | < 3–5% additional on app servers during delta windows | Extraction confined to CDS/ODP (HANA push-down), small package sizes |
| **Memory** | Marginal; bounded ODP package size (e.g. 20k rows/package) | Configurable package size; parallelism capped |
| **Disk I/O** | Read-mostly on delta tables; no writes to source | ODP delta tokens only; no staging tables in PRD |
| **HANA** | Analytical CDS reads scheduled outside batch blackout | Scheduler enforces windows |

### 3.2 Batch-window & schedule considerations

- **Blackout 01:00–04:30 IST:** the Scheduler suspends all heavy extraction while
  `RGGBS000` runs. Only lightweight event handling (IDoc receipt) continues.
- **ODP cadence:** delta pulls occur at most **once every 30 minutes per provider**; providers
  are staggered to smooth load rather than firing simultaneously.
- **Bandwidth:** the FinSight Loader is rate-limited so the integration consumes **≤ 25% of
  the 450 Mbps** MPLS link during business hours (heavier catch-up allowed off-hours).
- **Master before transactional:** master-data providers (CC/PC, vendor, customer) are
  sequenced ahead of dependent transactional deltas to preserve referential integrity.

### 3.3 Capacity headroom

At ~2.1M financial transactions/month, steady-state delta volume per 30-min window is modest
and comfortably within the above envelope. Festival-season 5x spikes are absorbed by Kafka
buffering (RF=3) without increasing SAP pressure — extraction cadence is unchanged; only
downstream processing scales.

---

## Section 4 — Security requirements

### 4.1 SAP authorisation objects

The service account `SVC_FINSIGHT` is granted a **least-privilege, read-only** role
`Z_INTEG_RFC` containing:

| Auth object | Fields (values) | Why |
|---|---|---|
| `S_RFC` | `RFC_TYPE=FUGR`; `RFC_NAME` = ODP/RFC function groups (e.g. `RODPS`, `/1BCDWB/*` CDS); `ACTVT=16` | Permit only the specific RFC function groups used for extraction |
| `S_TABU_DIS` | `DICBERCLS` = auth groups of allowed finance tables; `ACTVT=03` (display) | Read-only access limited to relevant table auth groups |
| `S_TABU_NAM` | `TABLE` = ACDOCA/BKPF/BSEG/BSIK/LFA1/… ; `ACTVT=03` | Table-name-level display restriction where used |
| `S_ODP_SUB` (or ODP equiv.) | subscription objects `Z_ODP_*`; `ACTVT=03/16` | ODP subscription/read only |
| `S_DATASET` | as required for IDoc file ports (if any) | Constrained IDoc handling |
| `S_IDOCDEFT` / `B_ALE_RECV` | message type `FINSTA` | IDoc receive for bank statements |

- **No** change/update/delete authorisations. **No** dialog logon (`S_USER_GRP` excludes
  dialog). Account type = **System/Communication (Type C)**.

### 4.2 Service account specification

| Attribute | Value |
|---|---|
| User ID | `SVC_FINSIGHT` (SAP), user type **System (C)** |
| Password/creds | Managed in Vault (`secret/prod/integ/sap`), rotated per policy; no interactive logon |
| Role | `Z_INTEG_RFC` (read-only, as §4.1) |
| Lockout | Excluded from mass password reset; monitored for failed-auth spikes |
| FinSight side | OAuth 2.0 `client_credentials` client `finsight-integ-loader`, scoped per resource |

### 4.3 Certificate requirements

| Certificate | Use | Requirement |
|---|---|---|
| SAP Gateway/OData X.509 (server) | mTLS on OData/RFC channel | Valid chain, ≥ 2048-bit RSA / P-256, renewed > 30 days before expiry |
| Integration client X.509 | mTLS client auth to SAP | Issued by client CA / SNC PSE; stored in Vault |
| FinSight edge TLS | HTTPS to `/api/v2/*` | TLS 1.2+; cert pinned in Loader config |
| Secrets storage | All keys/secrets | HashiCorp Vault, India region; never in manifests or Git |

- Auth summary (Canonical Facts §6): SAP via **OAuth2 + X.509** on the Gateway/OData;
  FinSight via **OAuth 2.0** (`client_credentials`, refresh token).

---

## Section 5 — Monitoring integration (Nagios)

The integration ships its own `Monitoring Stack` (Prometheus + Grafana + ELK + PagerDuty)
**and** feeds your existing **Nagios + Grafana** estate so IT has a single pane of glass.

### 5.1 How it feeds Nagios

- A monitoring bridge publishes **passive service check results** to your Nagios via **NSCA**
  (TCP 5667) — or the `check_http`/`check_prometheus` active-check pattern if you prefer.
- Each integration health signal maps to a Nagios service on a virtual host
  `finsight-integration`.

| Nagios service | Source metric | WARN | CRIT |
|---|---|---|---|
| `INTEG_Extractor_Health` | Extractor `/healthz` | non-200 once | non-200 3× |
| `INTEG_Kafka_ISR` | under-replicated partitions | ≥ 1 | ISR < 2 |
| `INTEG_Freshness` | end-to-end lag | > 3 h | > 4 h (SLA breach) |
| `INTEG_DLQ_Depth` | `finsight.dlq` depth | > 0 sustained | > 100 / 5 min |
| `INTEG_Recon_Status` | last reconciliation | any warning break | FAIL (variance) |
| `INTEG_RFC_Usage` | concurrent RFCs | > 40 | > 48 (near 50 cap) |
| `INTEG_Bandwidth` | link utilisation (bus. hrs) | > 20% | > 25% |

- Grafana dashboards (12 panels) are shared read-only with IT; ELK holds structured JSON
  logs with `correlation_id` for tracing any record end-to-end; PagerDuty handles primary
  paging with Nagios as the corroborating client-side view.

---

## Section 6 — Support handover (L1/L2/L3)

### 6.1 Support model

| Tier | Owner | Scope | Examples | Response |
|---|---|---|---|---|
| **L1** | Client IT service desk (24×7) | Triage, known-error runbook, restart-safe actions | Alert ack, dashboard down, "is data flowing?" | 15 min ack |
| **L2** | Joint: Meridian IT + Zetheta integration on-call | Domain failures, DLQ handling, reprocessing, SAP/RFC issues | Recon break, ODP token stuck, rate-limit tuning | 30 min |
| **L3** | Zetheta engineering (Anutosh Mishra) + SAP Basis (Priya) + Platform Eng (Marcus Wei) | Code/config change, schema drift, transport, platform change | CDS schema change, API contract change | Next business day / P1 immediate |

### 6.2 Escalation

- **SAP-side** (RFC, ODP, transport, perf): **Priya Deshmukh** → L3 Basis.
- **FinSight/API-side** (tokens, rate limits, schema): **Marcus Wei** (Platform Eng).
- **Change authority / SLA / business impact**: **Rajesh Venkataraman** (VP IT, CAB chair).
- **Audit/lineage queries**: **Dr. Sanjay Kulkarni** (async).
- P1 (integration down / SLA breach) pages primary on-call via PagerDuty and mirrors to
  Nagios; the rollback decision matrix lives in the D8 runbook (< 15-min rollback).

### 6.3 Knowledge transfer schedule

| Session | Audience | Content | Timing |
|---|---|---|---|
| KT-1 | L1 service desk | Alerts, runbook, restart-safe actions, "when to escalate" | Go-live − 5 days |
| KT-2 | L2 (IT + Basis) | DLQ reprocessing, ODP tokens, recon breaks, rate-limit tuning | Go-live − 3 days |
| KT-3 | L3 / Basis | Transports, CDS/ODP internals, schema-drift handling | Go-live − 2 days |
| KT-4 | Shadowing | Live monitoring during first week | Go-live week |
| Handover doc pack | All | This document + D6 monitoring spec + D8 runbook | Go-live − 1 day |

---

## Section 7 — Maintenance schedule

| System | Window (IST) | Use |
|---|---|---|
| **SAP** | 2nd & 4th Saturday, 22:00–06:00 | Transports, ODP/RFC changes, integration deploys/rollbacks |
| **FinSight** | 1st Sunday of month, 02:00–06:00 | Loader/API-side changes (do not run SAP transports here) |
| **SAP nightly batch** | `RGGBS000` 01:00–04:30 daily | **Extraction blackout** — automatically enforced |

### 7.1 Patch & upgrade coordination

- **SAP support packs / notes:** notify the integration team before applying — CDS view or
  ODP structure changes can affect extraction. Regression the `Z_I_*` views in QAS first.
  The monitoring stack detects schema drift and degrades gracefully (alerts, does not silently
  corrupt).
- **FinSight platform releases:** coordinate through Marcus Wei; API is versioned (`/api/v2/`)
  so minor changes are backward-compatible.
- **Integration releases:** always via CAB-approved change in the SAP Saturday window,
  blue-green with a tested < 15-minute rollback (see D8).
- **Certificate renewals:** tracked with 30-day advance alerts (Section 4.3).

---

## Appendix — Ownership & sign-off

| Area | Owner |
|---|---|
| SAP environment changes | Priya Deshmukh (Basis) |
| Network, firewall, VPN | Meridian network team + Rajesh Venkataraman |
| Security / authorisations / certs | Meridian security + Zetheta security eng |
| Monitoring (Nagios feed) | Meridian IT ops + Zetheta SRE |
| Integration platform (L3) | Anutosh Mishra (Zetheta) |

| Sign-off | Name | Date |
|---|---|---|
| VP IT Infrastructure | Rajesh Venkataraman | |
| SAP Basis Administrator | Priya Deshmukh | |
| Integration lead (Zetheta) | Anutosh Mishra | |

*End of D9b — Technical Handoff v1.0.*
