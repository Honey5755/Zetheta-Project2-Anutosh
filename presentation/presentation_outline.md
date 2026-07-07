# Oral Defence — Presentation Outline

| | |
|---|---|
| **Project** | 493560B — FDE-9B — Custom API Integration (SAP S/4HANA → Zetheta FinSight) |
| **Client** | Meridian Manufacturing Ltd. |
| **Presenter** | Anutosh Mishra, Forward Deployed Engineer |
| **Version** | v1.0 · **Date** 2026-07-07 |
| **Format** | 14 slides · ~20–25 min + Q&A |

> Consistency anchors used throughout: company codes `MC01/MC02/MC03` → tenants
> `MERIDIAN-MC01/02/03`; 10 data domains / 12 endpoint pairs; freshness **24 h → ≤ 4 h
> (83% improvement)**; constraints (batch blackout, ≤ 50 RFC, ODP ≤ 30 min, ≤ 25% of
> 450 Mbps, India-only residency); canonical components (ODP Extractor → Kafka →
> Transformation Engine → FinSight Loader → FinSight; plus Scheduler, Reconciliation Service,
> Monitoring Stack, Audit/Wire-Tap, DLQ).

---

## Slide 1 — Title

- **Real-time financial visibility for Meridian Manufacturing**
- SAP S/4HANA → Zetheta FinSight 4.2 integration
- Presenter: Anutosh Mishra, Forward Deployed Engineer · Project 493560B
- Date and audience (CFO, VP IT, SAP Basis, Platform Eng, Audit, Ops)

**Speaker note:** Set the frame in one line — "We took Meridian's finance reporting from a day
old to under four hours, safely and audit-ready." Name the six stakeholders in the room so each
knows they'll be addressed.

---

## Slide 2 — Agenda

- The problem and the goal
- Architecture and key design decisions
- Data transformation, resilience, reconciliation
- Monitoring, deployment and rollback
- Stakeholder communication, risks, results
- Q&A

**Speaker note:** Signpost that the deck moves from business framing to engineering depth and
back to outcomes, so every stakeholder sees their concern covered. Keep it to 20 seconds.

---

## Slide 3 — The problem

- SAP holds every transaction, but analytics arrive on a **24-hour** batch delay
- ~2.1M financial transactions/month across **7 plants, 3 company codes**
- Finance team stitches numbers by hand; decisions made on yesterday's data
- CFO mandate: near-real-time, trustworthy, audit-ready financial view
- Must respect real SAP and regulatory constraints — this can't destabilise production

**Speaker note:** Ground it in the business pain (stale data, manual effort) before any tech.
The constraint line signals to Priya and Rajesh that we respect their guardrails from the start.

---

## Slide 4 — Architecture overview

- Canonical flow: **SAP S/4HANA → ODP Extractor → Kafka → Transformation Engine → FinSight Loader → FinSight**
- Cross-cutting: Scheduler, Reconciliation Service, Monitoring Stack, Audit/Wire-Tap, DLQ
- Kafka: 3 brokers, RF=3, ISR=2, topics per domain + `dlq`, **partitioned by company code**
- Event-driven for deltas (30 min) + batch for master data — a tiered freshness model
- All processing in AWS `ap-south-1` (India residency)

**Speaker note:** Walk left to right once. Emphasise decoupling via Kafka: SAP and FinSight
run at different speeds and Kafka absorbs bursts (including festival-season 5x) without pushing
back on SAP.

---

## Slide 5 — Key design decisions

- **Kafka message broker** — decouple producer/consumer speeds, buffer, replay safety
- **Content-based router** in the Transformation Engine — one path per domain
- **Idempotency on `documentId`** — at-least-once delivery without duplicates
- **Tiered extraction** — ODP delta for hot data, batch for master (cost/perf balance)
- **Bulkhead per domain** — a GL failure never blocks AP
- Decisions traded ideal patterns against real constraints (RFC pool, ODP cadence, bandwidth)

**Speaker note:** This is the "why", not the "what". For each decision, name the alternative
rejected (e.g. direct point-to-point rejected → tight coupling, no replay). This is where oral
defence points are won.

---

## Slide 6 — Data transformation approach

- 10 domains, **56+ field mappings** (target 80+): structural, semantic, enrichment, validation
- Semantic routing: **company code → tenant** (`MC01→MERIDIAN-MC01`, etc.)
- Currency: point-in-time `TCURR` (KURST=M), ISO 4217 target
- Fiscal handling: variant **V3** (Apr–Mar), special periods **013–016**
- Hierarchy flattening (SETNODE/SETLEAF → Level1–7); **GST breakdown (CGST/SGST/IGST) preserved**

**Speaker note:** Pick one worked example live — a GL journal entry through composite-key
generation, currency conversion, fiscal-period mapping. The GST point is the compliance hook
for Audit and Ops.

---

## Slide 7 — Error handling & resilience

- Error classes: `TRANSIENT`, `PERMANENT`, `DATA_QUALITY`, `SYSTEM`
- Retry with jitter: `wait = min(cap, random(base, base * 2^attempt))`
- Circuit breaker (`CLOSED / OPEN / HALF-OPEN`) at every service boundary
- **DLQ** for unprocessable messages — 14-day retention, manual reprocessing
- Data-quality failures **quarantined**, not shown — clean records keep flowing

**Speaker note:** Tell the vendor-GSTIN story from the case studies: 15% of records fail
validation — we quarantine and notify AP, we don't block the pipeline. Shows graceful
degradation, not brittle all-or-nothing.

---

## Slide 8 — Reconciliation & data quality

- **Zero debit–credit variance per document; zero-break target across all 10 domains**
- Four dimensions: completeness > 99.5%, accuracy > 99.9%, consistency 100%, timeliness > 98%
- Full lineage: **SAP document number → FinSight record** (Audit/Wire-Tap)
- Reconciliation runs every batch; breaks quarantined and alerted, not silently accepted
- Referential integrity enforced: master before transactional (GL→CC, GL→PC, AP→vendor, AR→customer)

**Speaker note:** This slide is aimed squarely at Dr. Kulkarni. The message: every rupee is
mathematically accounted for and traceable. Reconciliation is not optional in financial systems.

---

## Slide 9 — Monitoring approach

- Three pillars: metrics (Prometheus/Grafana), logs (ELK, JSON + `correlation_id`), traces
- **12-panel Grafana dashboard**; 15+ alert rules, severities P1–P4, PagerDuty paging
- Feeds client's existing **Nagios** via passive checks (freshness, DLQ, recon, RFC usage, bandwidth)
- Key SLOs watched live: end-to-end lag ≤ 4 h, DLQ depth, ISR, RFC ≤ 50, bandwidth ≤ 25%
- Schema-drift detection so SAP support packs don't silently break mappings

**Speaker note:** Emphasise we plug into what IT already runs (Nagios), not a parallel tool
they must learn. Single pane of glass for Rajesh's team.

---

## Slide 10 — Deployment & rollback

- **Blue-green** service tier + **per-domain canary** via feature flags
- Copy-pasteable runbook (kubectl / kafka-topics / curl health checks / spectral lint)
- Deploy in SAP maintenance window (2nd/4th Sat 22:00–06:00), clear of `RGGBS000` blackout
- Progressive domain enablement — GL first, promote only on reconciliation **PASS**
- **Verified rollback < 15 minutes**, no data loss (idempotency + Kafka RF=3), decision matrix

**Speaker note:** Stress the reversibility. The atomic blue↔green switch plus a rollback
decision matrix (trigger → action → authority → time) is how we de-risk a financial go-live.
Reference D8.

---

## Slide 11 — Stakeholder communication

- One architecture, three tailored documents (the "Bridge Builder" principle)
- **CFO (Ananya):** plain-language value, 83% freshness gain, ROI, clear Ask — no jargon
- **Client IT (Rajesh & Priya):** SAP changes, network/firewall, perf impact, security, support model
- **Platform Eng (Marcus):** API contract, OAuth flow, throughput, limitations, requests
- Audit (Dr. Kulkarni) and Ops (Fatima) served by lineage and cost-variance views

**Speaker note:** The lesson from Palantir/Airbus: the same design needs different
documentation per audience. Show the three D9 covers side by side. Communication is a
graded deliverable, not an afterthought.

---

## Slide 12 — Risks

- **SAP performance** — mitigated: ≤ 50 RFC cap, ODP ≤ 30 min, batch blackout, live monitoring
- **Data quality / wrong number** — mitigated: reconciliation + quarantine, > 99.9% accuracy
- **Compliance / residency** — mitigated: India-only processing, full audit lineage, GST preserved
- **Schema evolution** (SAP support packs) — mitigated: drift detection + graceful degradation
- Each risk has an owner, a control, and a tested response (risk register + rollback)

**Speaker note:** Frame risks as owned and controlled, not open. Tie the top three back to the
CFO's business-language risks so the room hears one consistent story.

---

## Slide 13 — Results & KPIs

- **Data freshness: 24 h → ≤ 4 h = 83% improvement** (headline)
- ~**120–160 finance-team hours/month** returned; manual reconciliation near-eliminated
- Quality: completeness > 99.5%, accuracy > 99.9%, consistency 100%, uniqueness 100%
- Coverage: all 10 domains, 12 endpoints, 3 company codes, 7 plants
- Zero-break reconciliation; go-live reversible in < 15 min

**Speaker note:** Close the loop to Slide 3's problem. Lead with the 83% number, then quality
and coverage. This is the slide the CFO remembers — keep it crisp and quantified.

---

## Slide 14 — Q&A / Closing

- Summary: day-old → under-4-hour, reconciled, audit-ready finance across all plants
- Design respected every SAP and regulatory constraint
- Ready for go-live in the next maintenance window
- Thank you — questions
- Contact: Anutosh Mishra, Forward Deployed Engineer

**Speaker note:** Anticipate the likely questions and have back-pocket answers ready:
(1) *What if SAP slows down?* → RFC cap + blackout + monitoring;
(2) *What if a number is wrong?* → reconciliation quarantine, nothing wrong is shown;
(3) *Can we roll back?* → yes, < 15 min, no data loss;
(4) *How do you handle the 5x festival spike?* → Kafka buffering, cadence unchanged;
(5) *Where does data live?* → AWS Mumbai, India-only. Keep answers to 30 seconds each.

---

## Appendix — Backup slides (hold in reserve)

- B1: Kafka partitioning & consumer-group detail
- B2: OAuth 2.0 sequence + token lifecycle (from D9c)
- B3: Rollback decision matrix (from D8)
- B4: Full 12-panel dashboard layout (from D6)
- B5: GL end-to-end worked mapping example (from D3)

**Speaker note:** Only surface a backup slide if a stakeholder drills in; keep the main flow tight.

*End of presentation outline v1.0.*
