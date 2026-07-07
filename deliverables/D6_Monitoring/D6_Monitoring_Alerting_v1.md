# D6 — Monitoring & Alerting Specification (v1)

**Project:** FDE-9B — Custom API Integration (SAP S/4HANA → Zetheta FinSight)
**Project code:** 493560B · **Client (simulated):** Meridian Manufacturing Ltd.
**Author:** Anutosh Mishra · **Role:** Forward Deployed Engineer
**Deliverable:** D6 — Monitoring & Alerting Dashboard Specification
**Status:** Complete · **Version:** 1.0

> Companion artifacts: `monitoring/grafana_dashboard.json` (importable Grafana model for the
> 12 panels) and `monitoring/prometheus_alerts.yml` (the alerting rules encoded for Prometheus /
> Alertmanager). This document is the authoritative narrative; the two files are the executable form.

---

## 1. Purpose and scope

This specification defines how the Meridian → FinSight integration is observed in production. It
covers the three observability pillars (metrics, logs, traces), the monitoring stack, the twelve
dashboard panels (`MON-001`..`MON-012`), the structured-logging standard, the alerting catalogue
(P1–P4) with automated actions and runbook links, and the distributed-tracing approach that
carries a single `correlation_id` from SAP extraction through to FinSight load.

The design goal is operational: an on-call engineer must be able to answer three questions within
seconds of an alert — *what broke, where in the pipeline, and which domain / company code is
affected* — and must be able to act (retry, drain DLQ, pause a tenant) from a documented runbook.

The integration path being observed is the canonical one:

```
SAP S/4HANA → ODP Extractor → Kafka (per-domain topics + dlq) →
Transformation Engine (content-based router) → FinSight Loader → Zetheta FinSight
```

Cross-cutting services observed alongside it: Scheduler, Reconciliation Service, Audit/Wire-Tap,
and the DLQ.

---

## 2. The three pillars of observability

The monitoring design follows the standard three-pillar model. Each pillar answers a different
class of question and feeds a different part of the operational workflow.

| Pillar | What it captures | Primary tool | Retention | Answers |
|---|---|---|---|---|
| **Metrics** | Numerical time-series (throughput, latency, error rate, lag, resource use) | **Prometheus** (scrape + TSDB) visualised in **Grafana** | 15 days raw, 13 months downsampled (Thanos) | *Is the pipeline healthy right now? Are we within SLA?* |
| **Logs** | Discrete structured events with full context | **ELK** (Elasticsearch + Logstash + Kibana) | 30 days hot, 1 year warm/cold (audit) | *What exactly happened to this record / batch / document?* |
| **Traces** | End-to-end request flow across the five hops | **OpenTelemetry** SDK → OTLP collector → Jaeger backend | 7 days | *Where in SAP→Kafka→Transform→FinSight was the time / the failure?* |

Alerting sits on top of all three and is routed through **PagerDuty**. Metrics thresholds fire the
majority of alerts; log-pattern and trace-error signals fire a smaller, targeted set.

### 2.1 Chosen stack

| Layer | Product | Why it is the choice here |
|---|---|---|
| Metrics collection & storage | **Prometheus** | Pull-based scraping suits containerised services in AWS `ap-south-1`; PromQL expresses every panel query in Appendix F; already familiar to Meridian IT (they run Grafana today). |
| Dashboards | **Grafana** | Meridian's existing IT monitoring standard (Nagios + Grafana) — reuses operator muscle memory and SSO. All 12 panels ship as `grafana_dashboard.json`. |
| Logs | **ELK Stack** | Structured JSON ingestion via Logstash, full-text + field search in Kibana, and long-retention indices satisfy Dr. Kulkarni's audit-lineage requirement (SAP document number → FinSight record). |
| Alerting / on-call | **PagerDuty** | Severity-based escalation policies (P1→immediate page, P4→ticket), on-call schedules, and incident timelines. Alertmanager routes Prometheus alerts into PagerDuty services. |
| Tracing | **OpenTelemetry** | Vendor-neutral instrumentation; a single `correlation_id`/`trace_id` propagates across SAP extraction, Kafka headers, the Transformation Engine, and the FinSight Loader. Exported to Jaeger. |

All monitoring components run **inside AWS `ap-south-1` (Mumbai)** to respect the RBI
data-residency constraint — no telemetry containing financial payload leaves India. Logs and traces
carry business keys and metadata only; full financial payloads are referenced by document number,
not copied into the observability tier.

---

## 3. Dashboard specification — 12 panels (MON-001 .. MON-012)

The production dashboard is titled **"Meridian → FinSight Integration — Operations"**. It is
organised top-to-bottom as: health overview → throughput/latency → errors/DLQ → reconciliation →
system health (SAP / FinSight / nodes / Kafka) → resilience (circuit breakers). Every query below is
the exact metric query from Appendix F, expanded with the visualisation, refresh cadence, and alert
binding used in production. Refresh intervals are chosen against the SLA: fast-moving resilience
signals refresh at 5–15s; reconciliation and freshness (which change on batch boundaries) refresh at
1–5m.

| Panel ID | Title | Purpose | Data source | Metric / PromQL query | Visualisation | Refresh | Alert threshold | Severity |
|---|---|---|---|---|---|---|---|---|
| **MON-001** | Pipeline Health Overview | Single at-a-glance verdict across all 10 domains — the first thing on-call reads | Prometheus (recording rule `pipeline:health:state`) | Status aggregation: **GREEN** if all domains synced within SLA, **YELLOW** if any >2 h stale, **RED** if any >4 h stale or last run failed | Stat / state-timeline (colour-coded) | 30s | **RED** → page | **P1** (PagerDuty) |
| **MON-002** | Throughput (Records/Sec) | Detect stalls / collapse in processing rate per domain | Prometheus | `rate(records_processed_total[5m])` by `(domain, operation)` | Time-series (multi-line by domain) | 15s | Warning if **< 50 % of baseline** for the domain | **P3** Warning |
| **MON-003** | Latency Distribution | Track end-to-end and API latency; catch tail-latency regressions | Prometheus | `histogram_quantile(0.95, rate(api_duration_seconds_bucket[5m]))` | Time-series (P50/P95/P99 overlay) | 15s | **P95 > 5s** | **P3** Warning |
| **MON-004** | Error Rate (%) | Proportion of failed operations, split by error class | Prometheus | `rate(errors_total[5m]) / rate(requests_total[5m]) * 100` by `(error_class)` | Time-series + threshold band | 15s | **> 5 %** | **P1** Critical |
| **MON-005** | DLQ Depth | Backlog of unprocessable messages in the `dlq` topic; leading indicator of systemic data-quality or load failure | Prometheus (Kafka exporter) | `kafka_consumer_group_lag{topic="dlq"}` + 24 h trend line | Time-series + current-value stat | 30s | **> 500** messages | **P1** Critical |
| **MON-006** | Reconciliation Status | Last reconciliation verdict per domain with variance amount | Prometheus (Reconciliation Service metrics) | Last recon result per domain: `RECONCILED / VARIANCE / BREAK` with variance amount (`recon_variance_amount` gauge) | Table (per-domain rows, coloured status) | 5m | Any **BREAK** | **P2** High |
| **MON-007** | SAP System Health | Guard the RFC pool (≤ 50 concurrent) and SAP responsiveness; protect dialog users | Prometheus (SAP exporter) | `sap_rfc_pool_utilisation`, `sap_response_time_ms`, `sap_work_process_available` | Gauge cluster + time-series | 30s | **RFC pool > 90 %** | **P1** Critical |
| **MON-008** | FinSight API Health | Watch destination API latency, remaining rate-limit headroom, and token TTL | Prometheus (Loader metrics) | `finsight_api_response_ms`, `finsight_rate_limit_remaining`, `finsight_token_expiry_seconds` | Gauge cluster + time-series | 30s | **Rate-limit headroom < 10 %** | **P2** High |
| **MON-009** | Data Freshness | Age of the most recent successful load per domain against the ≤ 4 h SLA | Prometheus | `time() - max(last_successful_load_timestamp) by (domain)` | Bar gauge (per domain, SLA line) | 60s | Any domain **exceeds SLA** (>4 h) | **P3** |
| **MON-010** | Resource Utilisation | Node-level CPU / memory / disk / network for the integration cluster | Prometheus (node_exporter) | `node_cpu_utilisation`, `node_memory_utilisation`, `node_disk_utilisation`, `node_network_bytes` | Time-series (per node) | 15s | Any resource **> 90 %** | **P2** High |
| **MON-011** | Kafka Consumer Lag | Per-topic / per-partition consumer lag — detects Transform or Loader falling behind producers | Prometheus (Kafka exporter) | `kafka_consumer_group_lag` by `(topic, partition)` | Heatmap / time-series by topic | 15s | **> 50 000** msgs | **P2** High |
| **MON-012** | Circuit Breaker Status | State of every inter-service breaker (SAP RFC, FinSight API, per domain) | Prometheus | `circuit_breaker_state{service}` — `0=Closed, 1=Open, 2=Half-Open` | State-timeline / status grid | 5s | Any breaker **Open** | **P2** |

**Baseline definition (MON-002 / MON-003).** "Baseline" is a 7-day rolling median of the domain's
throughput at the same 30-minute slot of the week, recorded as `records:baseline:domain`. Comparing
against a same-slot baseline avoids false alarms during the SAP batch window (01:00–04:30 IST) and
maintenance windows, when throughput is legitimately low.

**Template variables.** The dashboard exposes `$domain` (GL, AP, AR, …), `$company_code` (MC01,
MC02, MC03), and `$environment` (prod / staging) as Grafana variables so any panel can be sliced to
a single legal entity or domain during an incident.

---

## 4. Structured logging specification

All services emit **structured JSON logs**, one JSON object per line, to stdout; Logstash ships them
to Elasticsearch. A log line without the mandatory fields is rejected at ingestion and counted in
`log_schema_violations_total` (itself alertable). The standard exists to make the audit-lineage
requirement mechanical: given a SAP document number or a `correlation_id`, Kibana returns the full
journey of a record across all five hops.

### 4.1 Mandatory fields (12)

| # | Field | Type | Description | Example |
|---|---|---|---|---|
| 1 | `correlation_id` | string (UUID) | End-to-end trace key; identical across SAP→Kafka→Transform→FinSight for one logical record/batch | `c7f3a9e2-1b44-4c9a-8e21-0d2f5b6a1c90` |
| 2 | `batch_id` | string | Extraction batch / run identifier assigned by the Scheduler | `BATCH-GL-20260707-0930` |
| 3 | `domain` | string (enum) | Canonical data domain | `GeneralLedger` |
| 4 | `company_code` | string (enum) | SAP company code / legal entity | `MC01` |
| 5 | `timestamp` | string (ISO 8601, UTC) | Event time | `2026-07-07T04:00:12.348Z` |
| 6 | `level` | string (enum) | `DEBUG / INFO / WARN / ERROR / FATAL` | `ERROR` |
| 7 | `service` | string (enum) | Emitting component | `TransformationEngine` |
| 8 | `event` | string | Machine-readable event name | `transform.currency.converted` |
| 9 | `error_code` | string / null | Registry code when applicable (`ERR-EXT-*`, `ERR-MAP-*`, `ERR-LOAD-*`, `ERR-RECON-*`, `ERR-SYS-*`) or null on success | `ERR-MAP-004` |
| 10 | `retry_count` | integer | Attempt number for this operation (0 = first try) | `2` |
| 11 | `record_count` | integer | Records affected by this event | `100` |
| 12 | `duration_ms` | integer | Operation duration in milliseconds | `842` |

### 4.2 Recommended contextual fields (optional but encouraged)

`trace_id`, `span_id` (OpenTelemetry linkage), `sap_document_number`, `finsight_record_id`,
`kafka_topic`, `kafka_partition`, `kafka_offset`, `tenant` (`MERIDIAN-MC0x`), `idempotency_key`,
`host`, `pod`, `error_class` (`TRANSIENT / PERMANENT / DATA_QUALITY / SYSTEM`).

### 4.3 Example log line

```json
{"correlation_id":"c7f3a9e2-1b44-4c9a-8e21-0d2f5b6a1c90","batch_id":"BATCH-GL-20260707-0930","domain":"GeneralLedger","company_code":"MC01","timestamp":"2026-07-07T04:00:12.348Z","level":"ERROR","service":"TransformationEngine","event":"transform.currency.rate_missing","error_code":"ERR-MAP-005","error_class":"DATA_QUALITY","retry_count":1,"record_count":1,"duration_ms":37,"sap_document_number":"4900001234","kafka_topic":"gl","kafka_partition":2,"kafka_offset":91823,"tenant":"MERIDIAN-MC01","idempotency_key":"MC01-4900001234-2026","trace_id":"7b1e...","span_id":"a4c9..."}
```

A successful load line for contrast:

```json
{"correlation_id":"c7f3a9e2-1b44-4c9a-8e21-0d2f5b6a1c90","batch_id":"BATCH-GL-20260707-0930","domain":"GeneralLedger","company_code":"MC01","timestamp":"2026-07-07T04:00:13.190Z","level":"INFO","service":"FinSightLoader","event":"load.journal_entry.created","error_code":null,"retry_count":0,"record_count":100,"duration_ms":842,"finsight_record_id":"JE-MC01-0098231","tenant":"MERIDIAN-MC01"}
```

### 4.4 Log-derived metrics and Kibana saved searches

Logstash increments Prometheus counters from log fields so logs and metrics agree:
`errors_total{error_class,error_code,domain}`, `records_processed_total{domain,operation}`, and
`log_schema_violations_total`. Standing Kibana searches ship with the deliverable: *Errors by
document number*, *DLQ arrivals by error_code*, *Retry storms (retry_count ≥ 3)*, and *Full lineage
by correlation_id* (the audit view).

---

## 5. Alerting rules catalogue (18 rules)

Severities map to PagerDuty escalation policy and response SLA:

| Severity | Meaning | Response SLA | Channel / escalation |
|---|---|---|---|
| **P1** | Pipeline down or data loss risk | Immediate page, ack < 5 min | PagerDuty (primary on-call) + `#ops-critical` Slack |
| **P2** | Degraded; SLA at risk | Page, ack < 15 min | PagerDuty (primary) + `#ops-integration` |
| **P3** | Warning; investigate same shift | < 4 h | `#ops-integration` Slack + email |
| **P4** | Informational / capacity | Next business day | Jira ticket (auto-created) |

The 18 rules below are derived from the panel thresholds (§3) and the error registries (Appendix B
FinSight codes, Appendix J `ERR-*` registry). Runbook links point to `D8_Deployment/runbook.md#<anchor>`.

| # | Alert name | Condition / expression | Severity | Notification channel | Automated action | Runbook |
|---|---|---|---|---|---|---|
| 1 | `PipelineHealthRed` | `pipeline:health:state == 2` (RED) for 1m | P1 | PagerDuty + `#ops-critical` | Page primary; open incident bridge | `#pipeline-red` |
| 2 | `ErrorRateCritical` | `sum(rate(errors_total[5m])) / sum(rate(requests_total[5m])) * 100 > 5` for 5m | P1 | PagerDuty + `#ops-critical` | Snapshot error_class breakdown to incident | `#error-rate` |
| 3 | `DLQDepthCritical` | `kafka_consumer_group_lag{topic="dlq"} > 500` for 5m | P1 | PagerDuty + `#ops-critical` | Trigger DLQ triage job; attach top error_codes | `#dlq-drain` |
| 4 | `SAPRFCPoolCritical` | `sap_rfc_pool_utilisation > 90` for 2m (ties to `ERR-EXT-002` pool exhaustion) | P1 | PagerDuty + `#ops-critical` | Throttle Scheduler concurrency; halt non-critical extractions | `#rfc-pool` |
| 5 | `KafkaBrokerDown` | `kafka_brokers_online < 3` for 1m (`ERR-SYS-001`) | P1 | PagerDuty + `#ops-critical` | Circuit breaker opens on producers; verify ISR recovery | `#kafka-broker` |
| 6 | `ReconciliationBreak` | `max(recon_variance_amount) by (domain) > 0` OR `recon_status == BREAK` (`ERR-RECON-001/002`) | P2 | PagerDuty + `#ops-integration` | Generate break report; notify data steward | `#recon-break` |
| 7 | `ReferentialIntegrityBreak` | `increase(errors_total{error_code="ERR-RECON-003"}[15m]) > 0` | P2 | `#ops-integration` | Trigger master-data resync (GL→CC / GL→PC / AP→vendor / AR→customer) | `#master-data-resync` |
| 8 | `FinSightRateLimitLow` | `finsight_rate_limit_remaining / finsight_rate_limit_total * 100 < 10` for 3m (`ERR-LOAD-001`, HTTP 429) | P2 | PagerDuty + `#ops-integration` | Loader honours `Retry-After`; back off with jitter | `#finsight-429` |
| 9 | `CircuitBreakerOpen` | `circuit_breaker_state == 1` for 1m | P2 | PagerDuty + `#ops-integration` | Confirm downstream health; schedule half-open probe | `#circuit-breaker` |
| 10 | `KafkaConsumerLagHigh` | `kafka_consumer_group_lag > 50000` for 5m (excluding `dlq`) | P2 | `#ops-integration` | Scale Transform/Loader consumers; check partitions | `#consumer-lag` |
| 11 | `ResourceUtilisationHigh` | `max(node_cpu_utilisation, node_memory_utilisation, node_disk_utilisation) > 90` for 5m | P2 | `#ops-integration` | Autoscale node group; page if disk (`ERR-SYS-002`) | `#resource-high` |
| 12 | `DiskSpaceLow` | `node_disk_free_percent < 10` for 2m (`ERR-SYS-002`) | P2 | PagerDuty + `#ops-integration` | Archive old logs/indices; request volume expansion | `#disk-low` |
| 13 | `FinSightAuthFailure` | `increase(errors_total{error_code=~"ERR-LOAD-005"}[10m]) > 3` (HTTP 401 after refresh) | P2 | PagerDuty + `#security` | Refresh token once; if persists, alert security team | `#auth-failure` |
| 14 | `ThroughputBelowBaseline` | `rate(records_processed_total[5m]) < 0.5 * records:baseline:domain` for 10m | P3 | `#ops-integration` | Annotate dashboard; check SAP batch/maintenance window | `#throughput-drop` |
| 15 | `LatencyP95High` | `histogram_quantile(0.95, rate(api_duration_seconds_bucket[5m])) > 5` for 10m | P3 | `#ops-integration` | Capture slow-span trace exemplars from Jaeger | `#latency-p95` |
| 16 | `DataFreshnessSLABreach` | `(time() - max(last_successful_load_timestamp) by (domain)) > 14400` (4 h) | P3 | `#ops-integration` + email | Trigger catch-up extraction for the stale domain | `#freshness-sla` |
| 17 | `ODPSubscriptionInvalidated` | `increase(errors_total{error_code="ERR-EXT-003"}[15m]) > 0` (provider structure changed) | P3 | `#ops-integration` | Halt extraction for domain; graceful degradation; open mapping-update task | `#odp-schema-change` |
| 18 | `DataQualityDLQTrend` | `increase(errors_total{error_class="DATA_QUALITY"}[1h]) > 100` (`ERR-MAP-*`) | P4 | Jira ticket + `#ops-integration` | Auto-file DQ ticket with top `error_code` + document numbers | `#dq-trend` |

**Notes on automated actions.** Actions that touch the pipeline (throttling, autoscaling, master-data
resync, catch-up extraction) are executed by the Scheduler / Reconciliation Service via signed
webhooks from Alertmanager, and every action itself emits a structured log line (`event:
action.<name>.executed`) so the audit trail records both the alert and the remediation. Circuit
breakers are internal to each service and merely *reported* here; the alert does not open them.

**Alert hygiene.** Inhibition rules suppress downstream noise: when `KafkaBrokerDown` (rule 5) is
firing, `KafkaConsumerLagHigh` (rule 10) and `CircuitBreakerOpen` (rule 9) are inhibited for the same
topics. Maintenance windows (SAP 2nd/4th Sat 22:00–06:00 IST; FinSight 1st Sun 02:00–06:00 IST) are
loaded as Alertmanager silences so scheduled downtime does not page.

---

## 6. Distributed tracing approach

### 6.1 Correlation model

A single **`correlation_id`** is minted at the moment of extraction and propagated unchanged across
every hop, so any record can be reconstructed end-to-end. In OpenTelemetry terms the
`correlation_id` is carried as a baggage item alongside the W3C `trace_id`; the two are logged
together, giving both a human-readable business key and a machine trace key.

```
SAP (ODP Extractor)                Kafka                 Transformation Engine        FinSight Loader
─────────────────────   ─────────────────────────   ─────────────────────────   ────────────────────────
mint correlation_id  →  header: correlation_id    →  read header, continue     →  header on POST +
+ root span             + traceparent (W3C)          span, child spans for        Idempotency-Key
(batch_id, domain,      on each Kafka message        map/validate/enrich          = business key
 company_code)          (per-domain topic)                                        (e.g. documentId)
```

### 6.2 Propagation mechanics per hop

| Hop | Span name | Carrier of `correlation_id` / `trace_id` | Key span attributes |
|---|---|---|---|
| **SAP extraction** | `extract.odp.delta` | Root span; IDs minted here | `domain`, `company_code`, `batch_id`, `record_count`, ODP delta token |
| **Kafka publish/consume** | `kafka.produce` / `kafka.consume` | **Kafka message headers**: `correlation_id`, `traceparent` (W3C `tracecontext`) | `kafka_topic`, `kafka_partition`, `kafka_offset` |
| **Transformation** | `transform.route` → child spans `transform.currency`, `transform.hierarchy`, `transform.validate` | Read from Kafka header, continue trace | `error_class`, `error_code`, `mapping_id` |
| **FinSight load** | `load.<resource>` | Propagated as HTTP header + `Idempotency-Key` (business key) on the `POST /api/v2/...` | `finsight_record_id`, HTTP status, `retry_count` |

Because the `Idempotency-Key` on the FinSight `POST` is the business key (`documentId`), a retried
load carries the **same** `correlation_id` and idempotency key — the trace shows the retry, and
FinSight de-duplicates rather than double-posting.

### 6.3 What tracing buys operationally

- **Bottleneck attribution (MON-003):** span durations tell whether latency is in SAP extraction,
  Kafka dwell, transformation, or FinSight — not just "the pipeline is slow".
- **Failure localisation:** a `DLQDepthCritical` or `ErrorRateCritical` alert links straight to the
  offending traces; the failing span carries the `error_code`, mapping the incident to Appendix B/J.
- **Audit lineage (Dr. Kulkarni):** the trace + the `correlation_id` in ELK together reconstruct the
  full path from **SAP document number → FinSight record**, satisfying the audit-trail requirement.
- **Exemplars:** Prometheus histogram buckets (`api_duration_seconds_bucket`) carry trace exemplars,
  so clicking a spike in MON-003 jumps to the exact slow trace in Jaeger.

### 6.4 Sampling

Tail-based sampling: **100 % of error traces and traces exceeding the P95 latency budget are kept**;
successful fast traces are sampled at 10 % to control storage. All sampling happens in-region
(`ap-south-1`); traces carry business keys and timings only, never full financial payloads.

---

## 7. Deployment and configuration notes

- **Prometheus** scrapes each service `/metrics` endpoint every 15s; recording rules precompute
  `pipeline:health:state`, `records:baseline:domain`, and per-domain error ratios to keep panels and
  alerts cheap. Long retention via Thanos sidecar to S3 (`ap-south-1`, versioned, encrypted).
- **Grafana** imports `monitoring/grafana_dashboard.json`; folder "Meridian Integration"; Azure AD
  SSO; per-entity dashboards filtered by the `$company_code` variable for Fatima Al-Hassan's ops view.
- **Alertmanager** routes by `severity` label to PagerDuty services (P1/P2 → page, P3 → Slack, P4 →
  Jira) and applies the inhibition and maintenance-silence rules in §5.
- **ELK** ingests via Logstash with a strict JSON schema (the 12 mandatory fields); ILM policy: 30d
  hot → 1y warm → delete, except audit-tagged indices retained per Meridian's records policy.
- **OpenTelemetry Collector** receives OTLP from all services and exports to Jaeger + Prometheus
  (exemplars). All observability data stays in `ap-south-1`.

---

## 8. Traceability to canonical facts

- SLA thresholds (freshness ≤ 4 h, error targets) — Canonical Facts §8; drives MON-001, MON-009,
  rules 1 and 16.
- RFC pool ≤ 50 concurrent — §5; drives MON-007 and rule 4 (`ERR-EXT-002`).
- Kafka 3 brokers RF=3 ISR=2, per-company-code partitions — §6; drives MON-005/011, rules 3/5/10.
- Error classes `TRANSIENT / PERMANENT / DATA_QUALITY / SYSTEM` and `ERR-*` registries — §7 +
  Appendix J; drive MON-004 and rules 2, 6, 7, 11–13, 17, 18.
- Data residency (India only) — §5; every monitoring component in `ap-south-1`.
- Audit lineage (SAP document number → FinSight record) — §5; delivered by §4 logging + §6 tracing.

---

*End of D6 — Monitoring & Alerting Specification v1.*
