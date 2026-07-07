# D8 — Deployment Runbook

| | |
|---|---|
| **Document** | `D8_Deployment_Runbook_v1.md` |
| **Project** | 493560B — FDE-9B — Custom API Integration (SAP S/4HANA → Zetheta FinSight) |
| **Client** | Meridian Manufacturing Ltd. |
| **Author** | Anutosh Mishra |
| **Version** | v1.0 |
| **Date** | 2026-07-07 |
| **Status** | Approved for production go-live |
| **Audience** | Release manager, on-call SRE, SAP Basis, FinSight platform on-call, CAB |

> **Scope.** This runbook governs the production deployment of the integration platform that
> streams financial data from SAP S/4HANA (PRD, Client 100) into Zetheta FinSight 4.2
> (AWS `ap-south-1`, Mumbai). It covers the 10 mapping domains and 12 source/destination
> endpoint pairs (`SRC-001…012` → `DST-001…012`), routed by SAP company code
> (`MC01`, `MC02`, `MC03`) to FinSight tenants `MERIDIAN-MC01/02/03`.
>
> **Golden constraint.** Any deployment or rollback action MUST keep data freshness at
> **≤ 4 hours** and must respect every business constraint in §5 of the Canonical Facts
> (SAP batch window, ≤ 50 concurrent RFC connections, ODP ≤ once / 30 min per provider,
> ≤ 25% of 450 Mbps in business hours, India-only processing).

---

## 1. Deployment summary

| Item | Value |
|---|---|
| Release ID | `REL-493560B-2026.07` |
| Target environment | Production namespace `finsight-integ-prod` (EKS, `ap-south-1`) |
| Deployment strategy | Blue-green at the service tier + per-domain canary via feature flags |
| Planned window | SAP maintenance window — **2nd or 4th Saturday, 22:00–06:00 IST** (see §9) |
| Expected total duration | ~3 h 30 m (deploy + verify), leaving buffer inside the window |
| Rollback budget | **< 15 minutes** from decision to previous-known-good |
| Change record | CAB-ticket `CHG-493560B-0001` (must be Approved before Step 0) |
| Data residency | All compute, storage and processing confined to AWS `ap-south-1` |

**Component inventory being deployed** (canonical names): `ODP Extractor`, `Kafka`
(3 brokers, RF=3, ISR=2, topics per domain + `dlq`), `Transformation Engine`
(content-based router), `FinSight Loader`, `Scheduler`, `Reconciliation Service`,
`Monitoring Stack` (Prometheus + Grafana + ELK + PagerDuty), `Audit/Wire-Tap`.

---

## 2. Pre-deployment checklist

All items must be **PASS** and initialled by the named owner before Step 0 of §5.
Do not begin if any item is RED.

| # | Category | Check | Verification method | Owner | Status |
|---|---|---|---|---|---|
| 1 | Environment readiness | EKS cluster `finsight-integ-prod` healthy; all nodes `Ready`; capacity headroom ≥ 30% | `kubectl get nodes`; `kubectl top nodes` | SRE on-call | ☐ |
| 2 | Environment readiness | Green (idle) service stack provisioned and reachable, blue (live) stack serving current traffic | `kubectl get deploy -n finsight-integ-prod` | Release mgr | ☐ |
| 3 | Config | Release image digests pinned and immutable; config maps rendered for prod (no dev/UAT values) | `kubectl get cm integ-config -o yaml`; diff vs golden | Release mgr | ☐ |
| 4 | Config | Company-code → tenant routing table correct (`MC01→MERIDIAN-MC01`, `MC02→…MC02`, `MC03→…MC03`) | Inspect `routing.yaml` | Integration lead | ☐ |
| 5 | Backups | FinSight Loader offset store + Reconciliation Service state snapshotted; Kafka topic offsets recorded | Snapshot job `snap-preprod-2026-07` complete | SRE on-call | ☐ |
| 6 | Backups | SAP-side: ODP delta tokens exported; last-known-good token per provider recorded for replay safety | Basis export note | Priya Deshmukh (Basis) | ☐ |
| 7 | CAB approval | `CHG-493560B-0001` in state **Approved**; risk assessment & this runbook attached | CAB portal | Rajesh Venkataraman (VP IT) | ☐ |
| 8 | Stakeholder notification | Go-live notice sent T-48h and T-2h to CFO office, IT, Audit, Ops; status page set to "Maintenance" | Email + status page | Release mgr | ☐ |
| 9 | SAP transports | Transport requests imported to PRD in correct order (see §2.1); import logs error-free (RC ≤ 4) | STMS import history | Priya Deshmukh (Basis) | ☐ |
| 10 | ODP subscriptions | ODP subscriptions activated for all delta providers; delta init run once in test-mode; no full-load storm | ODQMON / RODPS_REPL_TEST | Priya Deshmukh (Basis) | ☐ |
| 11 | Kafka topics | All domain topics + `dlq` created with RF=3, ISR=2, partitioned by company code (3 partitions min) | `kafka-topics --describe` (see §5 Step 3) | SRE on-call | ☐ |
| 12 | TLS certificates | X.509 certs for SAP Gateway/OData mTLS valid > 30 days; FinSight edge cert valid; chain complete | `openssl s_client` check | Security eng | ☐ |
| 13 | Secrets in vault | OAuth client id/secret (FinSight), SAP service-account creds, Kafka SASL creds present in Vault path `secret/prod/integ/*`; no secrets in manifests | `vault kv list secret/prod/integ` | Security eng | ☐ |
| 14 | DNS | Records for `integ-api.meridian.finsight.internal` and `loader.finsight-integ-prod` resolve to green LB target group | `dig +short` | SRE on-call | ☐ |
| 15 | Rate-limit config | FinSight Loader rate limits set to respect ≤ 25% of 450 Mbps in business hours and endpoint quotas; RFC pool cap set to ≤ 50 concurrent | Inspect `ratelimit.yaml`; SAP RZ12 quota | Integration lead | ☐ |
| 16 | Health-check endpoints | `/healthz` (liveness) and `/readyz` (readiness) implemented and returning 200 on green stack | `curl` (see §5 Step 5) | SRE on-call | ☐ |
| 17 | Idempotency | FinSight `POST` idempotency keyed on business `documentId` verified against a replayed sample batch | Loader integration test | Integration lead | ☐ |
| 18 | API contract lint | OpenAPI specs pass Spectral with zero errors (no drift since D2 sign-off) | `spectral lint` (see §5 Step 6) | Marcus Wei (Platform Eng) | ☐ |
| 19 | Reconciliation baseline | Reconciliation Service pre-loaded with source control totals per domain for the first post-deploy run | Recon seed job | Integration lead | ☐ |
| 20 | On-call & escalation | Primary + secondary on-call confirmed for the full window; escalation tree (§10) reachable | Roster ack | Release mgr | ☐ |

### 2.1 SAP transport import order (placeholder numbers — confirm with Basis before import)

| Order | Transport | Contents |
|---|---|---|
| 1 | `PRDK900101` | Custom CDS consumption views `Z_I_GLINTEG`, `Z_I_APINTEG`, `Z_I_ARINTEG` |
| 2 | `PRDK900102` | ODP provider registration + extraction structures |
| 3 | `PRDK900103` | RFC destination definitions + authorisation role `Z_INTEG_RFC` |
| 4 | `PRDK900104` | IDoc partner profile (`FINSTA` inbound/outbound) + port config |

---

## 3. Deployment strategy

### 3.1 Blue-green at the service tier

The `Transformation Engine`, `FinSight Loader` and API front-end are deployed **blue-green**.
Two identical stacks run side by side; the currently live stack is **blue**, the newly deployed
stack is **green**. Traffic cutover is a single, reversible load-balancer/label switch.

**Rationale for this integration:**

- **Instant, low-risk rollback.** Because SAP financial data is audited end-to-end (SAP
  document number → FinSight record), we cannot tolerate a half-migrated state. Blue-green
  gives an atomic switch and an atomic switch-back — the core requirement for the
  **< 15-minute rollback** budget.
- **Zero-downtime read path.** FinSight dashboards for the CFO must stay live; blue keeps
  serving until green is proven.
- **Warm validation.** Green consumes from Kafka in a *shadow consumer group* before
  cutover, so we validate transforms against real messages without writing to FinSight.

### 3.2 Per-domain canary via feature flags

Kafka is **not** blue-green (it is stateful, RF=3). Instead, the *flow* through green is gated
by **per-domain feature flags**. After cutover we enable domains progressively, watching
error rate and reconciliation before widening.

| Wave | Feature flag | Domains enabled | Why this order |
|---|---|---|---|
| Canary 1 | `flag.domain.gl=on` | General Ledger (`SRC/DST-001`) | Highest value, ODP delta, exercises currency + fiscal-period + composite-key logic |
| Canary 2 | `flag.domain.ap=on`, `flag.domain.ar=on` | AP (`-002`), AR (`-003`) | Depends on GL master sync; validates vendor/customer referential integrity |
| Wave 3 | `flag.domain.cc=on`, `flag.domain.pc=on`, `flag.domain.ml=on` | Cost Centre, Profit Centre, Material Ledger (`-004/-005/-006`) | Batch/master + hierarchy flattening |
| Wave 4 | `flag.domain.po=on`, `flag.domain.so=on`, `flag.domain.fa=on`, `flag.domain.bank=on` | PO, SO, Fixed Assets, Bank Statements (`-007/-008/-009/-010`) | Lower volume; bank via IDoc (event) |
| Wave 5 | `flag.domain.budget=on`, `flag.domain.inventory=on` | Budget/Actual (`-011`), Inventory (`-012`) | Fold into GL/CC and ML analytics respectively |

- Master-data domains (Cost Centre, Profit Centre) and vendor/customer masters must be
  flowing **before** their dependent transactional domains — referential integrity rule
  GL→CC, GL→PC, AP→vendor, AR→customer.
- Each wave is promoted only after the domain's first reconciliation returns **PASS**
  (zero debit–credit variance per document) and DLQ for that domain stays empty for 15 min.
- Traffic split per canary domain starts at ~5% of partitions consumed, then 100% within the
  same domain once healthy (canary applies to *which domains*, not a fraction of a domain's
  documents, to preserve per-document reconciliation).

---

## 4. Roles for this deployment

| Role | Person | Responsibility during window |
|---|---|---|
| Release manager | Anutosh Mishra | Runs the runbook, owns go/no-go, holds rollback authority |
| SRE on-call (primary) | On-call rotation P1 | Executes `kubectl`/`kafka` commands, watches dashboards |
| SAP Basis | Priya Deshmukh | Transports, ODP, RFC pool, SAP-side health |
| Platform Eng on-call | Marcus Wei (or delegate) | FinSight API/token/rate-limit issues |
| VP IT Infra / CAB | Rajesh Venkataraman | Change authority, business-impact escalation |
| Audit observer (async) | Dr. Sanjay Kulkarni | Confirms lineage/audit trail intact post-go-live |

---

## 5. Deployment steps

> Conventions: `NS=finsight-integ-prod`. Set `export NS=finsight-integ-prod` in every shell.
> All commands are copy-pasteable; replace placeholder hosts/topics only where noted.
> Durations are estimates inside the maintenance window.

### Step 0 — Freeze & confirm gate (5 min)

Confirm CAB Approved, checklist §2 all PASS, status page in maintenance.

```bash
export NS=finsight-integ-prod
kubectl config current-context   # MUST be the ap-south-1 prod context
kubectl get ns "$NS"
```

### Step 1 — Snapshot current-known-good for rollback (10 min)

Capture the live (blue) revision and consumer offsets so rollback is deterministic.

```bash
# Record current live deployment revisions (save this output!)
kubectl -n "$NS" rollout history deploy/transformation-engine   | tee /var/runbook/te_history.txt
kubectl -n "$NS" rollout history deploy/finsight-loader         | tee /var/runbook/loader_history.txt

# Record current image digests
kubectl -n "$NS" get deploy -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.template.spec.containers[0].image}{"\n"}{end}' \
  | tee /var/runbook/blue_images.txt

# Snapshot Kafka consumer-group offsets for the loader
kafka-consumer-groups --bootstrap-server kafka-0.kafka:9092 \
  --describe --group finsight-loader-blue | tee /var/runbook/blue_offsets.txt
```

### Step 2 — Deploy green service stack (idle, no traffic) (20 min)

```bash
# Apply the green manifests (image digests pinned in the release bundle)
kubectl -n "$NS" apply -f release/2026.07/green/

# Wait for green rollout to complete
kubectl -n "$NS" rollout status deploy/transformation-engine-green --timeout=600s
kubectl -n "$NS" rollout status deploy/finsight-loader-green       --timeout=600s

# Confirm green pods are Running and Ready but receiving NO production traffic yet
kubectl -n "$NS" get pods -l stack=green -o wide
```

### Step 3 — Ensure Kafka topics exist (idempotent) (10 min)

Create per-domain topics + `dlq` if absent (partitioned by company code → 3 partitions).

```bash
BOOT=kafka-0.kafka:9092
for d in gl ap ar cost-centres profit-centres material-ledger \
         purchase-orders sales-orders fixed-assets bank-statements \
         budget-actuals inventory; do
  kafka-topics --bootstrap-server "$BOOT" --create --if-not-exists \
    --topic "finsight.$d" --partitions 3 --replication-factor 3 \
    --config min.insync.replicas=2
done

# Dead-letter queue
kafka-topics --bootstrap-server "$BOOT" --create --if-not-exists \
  --topic finsight.dlq --partitions 3 --replication-factor 3 \
  --config min.insync.replicas=2 --config retention.ms=1209600000  # 14 days

# Verify
kafka-topics --bootstrap-server "$BOOT" --describe --topic finsight.gl
```

### Step 4 — Green shadow validation (green reads, does NOT write to FinSight) (20 min)

Green runs the `FinSight Loader` in **dry-run** so transforms are validated against live Kafka
messages without POSTing to FinSight.

```bash
kubectl -n "$NS" set env deploy/finsight-loader-green LOADER_MODE=dry-run
kubectl -n "$NS" logs deploy/transformation-engine-green --tail=100 -f   # watch for transform errors
# Confirm zero transform errors and DLQ not growing during shadow
kafka-run-class kafka.tools.GetOffsetShell --broker-list kafka-0.kafka:9092 --topic finsight.dlq
```

### Step 5 — Health checks on green before cutover (10 min)

```bash
GREEN=$(kubectl -n "$NS" get svc finsight-loader-green -o jsonpath='{.spec.clusterIP}')
curl -fsS "http://$GREEN:8080/healthz" && echo "  <- liveness OK"
curl -fsS "http://$GREEN:8080/readyz"  && echo "  <- readiness OK"
# Component self-report
curl -fsS "http://$GREEN:8080/status" | jq '.components'
```

### Step 6 — Contract lint gate (5 min)

Confirm no API contract drift since D2 sign-off.

```bash
spectral lint deliverables/D2_API_Spec/API_FinSight_*.yaml --ruleset .spectral.yaml
spectral lint deliverables/D2_API_Spec/API_SAP_*.yaml       --ruleset .spectral.yaml
# Expected: "No results with a severity of 'error' found!"
```

### Step 7 — Cutover blue → green (atomic) (5 min)

Switch live traffic and the loader consumer group. All domain feature flags start **off**.

```bash
# Flip loader out of dry-run
kubectl -n "$NS" set env deploy/finsight-loader-green LOADER_MODE=live

# Ensure ALL domain flags start disabled for a controlled canary
kubectl -n "$NS" patch cm feature-flags --type merge -p \
  '{"data":{"flag.domain.gl":"off","flag.domain.ap":"off","flag.domain.ar":"off","flag.domain.cc":"off","flag.domain.pc":"off","flag.domain.ml":"off","flag.domain.po":"off","flag.domain.so":"off","flag.domain.fa":"off","flag.domain.bank":"off","flag.domain.budget":"off","flag.domain.inventory":"off"}}'

# Atomic traffic switch: move the Service selector from blue to green
kubectl -n "$NS" patch svc finsight-integ-edge -p '{"spec":{"selector":{"stack":"green"}}}'

# Confirm the edge now points to green
kubectl -n "$NS" describe svc finsight-integ-edge | grep -i selector
```

### Step 8 — Canary 1: enable General Ledger (20 min hold)

```bash
kubectl -n "$NS" patch cm feature-flags --type merge -p '{"data":{"flag.domain.gl":"on"}}'
# Hold 15-20 min. Watch error rate, DLQ, and trigger the first GL reconciliation.
kubectl -n "$NS" logs deploy/finsight-loader-green --tail=50 -f
kafka-run-class kafka.tools.GetOffsetShell --broker-list kafka-0.kafka:9092 --topic finsight.dlq
```

Promote to Canary 2 **only if** GL reconciliation = PASS and `finsight.dlq` did not grow.

### Step 9 — Progressive waves 2–5 (per §3.2) (60 min)

Enable each wave's flags, hold ~10–15 min, verify per §6 before the next wave.

```bash
# Wave 2
kubectl -n "$NS" patch cm feature-flags --type merge -p '{"data":{"flag.domain.ap":"on","flag.domain.ar":"on"}}'
# Wave 3
kubectl -n "$NS" patch cm feature-flags --type merge -p '{"data":{"flag.domain.cc":"on","flag.domain.pc":"on","flag.domain.ml":"on"}}'
# Wave 4
kubectl -n "$NS" patch cm feature-flags --type merge -p '{"data":{"flag.domain.po":"on","flag.domain.so":"on","flag.domain.fa":"on","flag.domain.bank":"on"}}'
# Wave 5
kubectl -n "$NS" patch cm feature-flags --type merge -p '{"data":{"flag.domain.budget":"on","flag.domain.inventory":"on"}}'
```

### Step 10 — Scale down blue (keep warm for rollback) (5 min)

Do **not** delete blue. Scale to zero traffic but keep manifests for a fast switch-back.

```bash
kubectl -n "$NS" scale deploy/transformation-engine-blue --replicas=1   # keep 1 warm
kubectl -n "$NS" scale deploy/finsight-loader-blue        --replicas=1
```

### Step 11 — Exit maintenance & notify (5 min)

Run the full §6 verification, then set status page to "Operational" and send the go-live
confirmation to stakeholders.

---

## 6. Post-deployment verification

Run all checks. Every one must PASS before declaring go-live. Blue is retained until all PASS.

| # | Check | How | Expected |
|---|---|---|---|
| 1 | Service health | `curl http://$GREEN:8080/healthz` and `/readyz` | Both 200 |
| 2 | All pods healthy | `kubectl -n $NS get pods -l stack=green` | All `Running`/`Ready`, 0 restarts |
| 3 | Kafka cluster healthy | `kafka-topics --bootstrap-server kafka-0.kafka:9092 --describe` | All topics: RF=3, ISR=2, no under-replicated partitions |
| 4 | Data flow — GL (`MC01/02/03`) | Confirm records landing in FinSight `/journal-entries` for all 3 tenants | Non-zero, all tenants |
| 5 | Data flow — every domain | Loader success counter per `DST-001…012` increasing | All 12 endpoints > 0 |
| 6 | First reconciliation PASS | Trigger recon per domain (see below) | Zero debit–credit variance per document; zero breaks |
| 7 | Freshness SLA | Grafana "end-to-end lag" panel | ≤ 4 h (target well under) |
| 8 | Alerting fires (test) | Inject a synthetic failure, confirm PagerDuty page | Alert received, then auto-clears |
| 9 | Dashboards live | Open Grafana integration board | All 12 panels rendering live data |
| 10 | DLQ empty | `finsight.dlq` offset stable | No net growth over 15 min |
| 11 | Idempotency | Re-POST a known `documentId` | FinSight returns idempotent 200/409, no duplicate |
| 12 | Audit lineage | Pick 1 SAP doc no., trace to FinSight record via Audit/Wire-Tap | Full lineage resolves (Dr. Kulkarni check) |
| 13 | Constraint — RFC pool | SAP SM59/RZ12 concurrent RFCs | ≤ 50 concurrent |
| 14 | Constraint — bandwidth | Network panel, business hours | ≤ 25% of 450 Mbps |

**Health, flow and reconciliation trigger commands:**

```bash
# 1 & 2 — health
GREEN=$(kubectl -n "$NS" get svc finsight-loader-green -o jsonpath='{.spec.clusterIP}')
curl -fsS "http://$GREEN:8080/healthz"; curl -fsS "http://$GREEN:8080/readyz"
kubectl -n "$NS" get pods -l stack=green

# 3 — Kafka health / under-replicated partitions
kafka-topics --bootstrap-server kafka-0.kafka:9092 --describe --under-replicated-partitions

# 4/5 — data flow smoke per tenant (read-back)
for T in MERIDIAN-MC01 MERIDIAN-MC02 MERIDIAN-MC03; do
  curl -fsS -H "Authorization: Bearer $FINSIGHT_TOKEN" \
    "https://api.finsight.internal/api/v2/journal-entries?tenant=$T&limit=1" | jq '.data | length'
done

# 6 — kick off reconciliation for a domain and read result
curl -fsS -X POST "http://$GREEN:8080/recon/run?domain=general-ledger" | jq '.status,.breaks'
# Expected: "PASS", 0

# 8 — alerting test (synthetic)
curl -fsS -X POST "http://$GREEN:8080/test/inject-failure?type=TRANSIENT&domain=ap"
# Confirm PagerDuty page then:
curl -fsS -X POST "http://$GREEN:8080/test/clear-failure?domain=ap"

# 10 — DLQ depth
kafka-run-class kafka.tools.GetOffsetShell --broker-list kafka-0.kafka:9092 --topic finsight.dlq
```

---

## 7. Rollback procedure

> **Objective:** return to the previous-known-good (blue) within **< 15 minutes** of a rollback
> decision, with no data loss and no duplicate FinSight records (idempotency on `documentId`
> guarantees replay safety).

### 7.1 Decision matrix

| # | Trigger condition | Action | Decision authority | Max decision time |
|---|---|---|---|---|
| R1 | Green fails health (`/healthz` or `/readyz` non-200) after Step 7 | Immediate full rollback (§7.2) | SRE on-call | 2 min |
| R2 | DLQ grows > 100 msgs / 5 min on any domain | Disable that domain's flag; if unresolved in 10 min → full rollback | SRE on-call → Release mgr | 5 min |
| R3 | Reconciliation FAIL (any debit–credit variance / break) on canary domain | Disable that domain's flag, hold; escalate to Integration lead | Integration lead | 10 min |
| R4 | Freshness SLA breached (> 4 h projected) and not recoverable | Full rollback (§7.2) | Release mgr | 10 min |
| R5 | SAP impact: RFC pool > 50 or dialog users blocked | Halt extraction, disable all flags; Basis throttles ODP; assess rollback | Priya Deshmukh + Release mgr | 5 min |
| R6 | FinSight platform outage / sustained 5xx from loader | Pause loader (flags off), keep Kafka buffering; rollback if > 30 min | Marcus Wei + Release mgr | 15 min |
| R7 | Security/cert/secret failure (mTLS handshake, token invalid) | Full rollback (§7.2); rotate credentials before retry | Security eng + Release mgr | 10 min |
| R8 | Data-residency / compliance violation detected | Immediate full rollback + notify Audit | Release mgr (mandatory) | Immediate |
| R9 | CAB revokes change mid-window | Full rollback (§7.2) | Rajesh Venkataraman (CAB) | Immediate |

**Partial vs full:** R2/R3 first try a *targeted* rollback (disable the offending domain flag)
because it is faster and lower-blast-radius. Escalate to full blue restore if the issue is
platform-wide or not contained within the decision time.

### 7.2 Full rollback commands (blue restore)

```bash
export NS=finsight-integ-prod

# 1) Stop green writing immediately — turn every domain flag off
kubectl -n "$NS" patch cm feature-flags --type merge -p \
  '{"data":{"flag.domain.gl":"off","flag.domain.ap":"off","flag.domain.ar":"off","flag.domain.cc":"off","flag.domain.pc":"off","flag.domain.ml":"off","flag.domain.po":"off","flag.domain.so":"off","flag.domain.fa":"off","flag.domain.bank":"off","flag.domain.budget":"off","flag.domain.inventory":"off"}}'

# 2) Restore blue to full capacity
kubectl -n "$NS" scale deploy/transformation-engine-blue --replicas=3
kubectl -n "$NS" scale deploy/finsight-loader-blue        --replicas=3
kubectl -n "$NS" rollout status deploy/finsight-loader-blue --timeout=300s

# 3) Atomic traffic switch green -> blue
kubectl -n "$NS" patch svc finsight-integ-edge -p '{"spec":{"selector":{"stack":"blue"}}}'

# 4) Point loader back to the blue consumer group (offsets from Step 1 snapshot)
kubectl -n "$NS" set env deploy/finsight-loader-blue LOADER_GROUP=finsight-loader-blue

# 5) Scale green to zero (retain for post-mortem, do not delete)
kubectl -n "$NS" scale deploy/transformation-engine-green --replicas=0
kubectl -n "$NS" scale deploy/finsight-loader-green        --replicas=0
```

### 7.3 Targeted (single-domain) rollback

```bash
# Example: back out AP only
kubectl -n "$NS" patch cm feature-flags --type merge -p '{"data":{"flag.domain.ap":"off"}}'
# Kafka retains AP messages (RF=3); replayed safely on next enable thanks to idempotency.
```

### 7.4 Post-rollback verification (must PASS to close the window)

```bash
BLUE=$(kubectl -n "$NS" get svc finsight-integ-edge -o jsonpath='{.spec.clusterIP}')
curl -fsS "http://$BLUE:8080/healthz" && curl -fsS "http://$BLUE:8080/readyz"
kubectl -n "$NS" get pods -l stack=blue
kafka-topics --bootstrap-server kafka-0.kafka:9092 --describe --under-replicated-partitions
curl -fsS -X POST "http://$BLUE:8080/recon/run?domain=general-ledger" | jq '.status'   # expect "PASS"
```

Confirm: blue serving, DLQ stable, one reconciliation PASS, no duplicate FinSight records.
File the rollback in `CHG-493560B-0001` and schedule post-mortem within 48 h.

---

## 8. SAP-side & Kafka safety notes

- **No heavy extraction 01:00–04:30 IST** (job `RGGBS000`). If the window slips past 01:00,
  pause the `ODP Extractor` and resume after 04:30.
- ODP delta must not be triggered more than **once / 30 min per provider**; the Scheduler
  enforces this — do not manually force deltas during deployment.
- On rollback, Kafka retains all in-flight messages (RF=3, ISR=2); the DLQ has 14-day
  retention for later reprocessing.

---

## 9. Maintenance windows (from Canonical Facts §5)

| System | Window (IST) | Use for this integration |
|---|---|---|
| SAP | Every **2nd & 4th Saturday, 22:00–06:00** | Primary deployment / rollback window (transports, ODP, RFC changes) |
| FinSight | **1st Sunday of month, 02:00–06:00** | Loader/API-side changes; do not schedule SAP transports here |
| SAP nightly batch | `RGGBS000` **01:00–04:30 daily** | **Blackout** — no heavy extraction; avoid overlapping cutover |

**Selected go-live window for `REL-493560B-2026.07`:** 2nd Saturday, 22:00–06:00 IST,
executing cutover before 01:00 to stay clear of the `RGGBS000` blackout.

---

## 10. Escalation contacts & on-call

### 10.1 Escalation ladder

| Tier | Trigger | Contact | Channel | Response target |
|---|---|---|---|---|
| L1 | Alert / health degradation | SRE on-call (primary) | PagerDuty `integ-primary` | 5 min ack |
| L2 | Domain error / recon fail / DLQ growth | Integration lead — Anutosh Mishra | PagerDuty `integ-secondary` + call | 10 min |
| L2-SAP | SAP perf / RFC / transport / ODP | **Priya Deshmukh** (SAP Basis Admin) | Phone + SAP Solution Manager | 10 min |
| L2-Platform | FinSight API / token / rate limit / schema | **Marcus Wei** (Zetheta Platform Eng) | PagerDuty `finsight-platform` | 15 min |
| L3 | Change authority / business impact / SLA breach | **Rajesh Venkataraman** (VP IT Infra, CAB chair) | Phone bridge | 15 min |
| Exec | Go/no-go on business risk, comms to board | **Ananya Krishnan** (CFO) — informed, not paged | Email + scheduled call | Next business hour |
| Audit | Lineage / reconciliation / compliance sign-off | **Dr. Sanjay Kulkarni** (Head Internal Audit) | Email (async) | Next business day |
| Ops | Cost allocation / variance report impact | **Fatima Al-Hassan** (Mfg Ops Director) | Email (async) | Next business day |

### 10.2 On-call rotation (deployment weekend)

| Slot | Primary | Secondary |
|---|---|---|
| 22:00–02:00 IST | SRE on-call A | Anutosh Mishra (Release mgr) |
| 02:00–06:00 IST | SRE on-call B | Priya Deshmukh (SAP-side) |
| Platform Eng standby (full window) | Marcus Wei | Platform on-call delegate |

**War-room bridge:** persistent video bridge + `#rel-493560b-golive` channel open for the
entire window. All decisions (esp. rollback) are logged in the channel with a timestamp.

---

## 11. Sign-off

| Gate | Owner | Signature | Date |
|---|---|---|---|
| Pre-deployment checklist complete | Release manager | | |
| CAB approval | Rajesh Venkataraman | | |
| SAP-side readiness | Priya Deshmukh | | |
| Platform (FinSight) readiness | Marcus Wei | | |
| Go-live verification complete | Release manager | | |
| Audit lineage confirmed | Dr. Sanjay Kulkarni | | |

*End of D8 — Deployment Runbook v1.0.*
