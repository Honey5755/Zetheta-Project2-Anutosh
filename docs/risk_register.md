# Risk Register — FDE-9B SAP S/4HANA → Zetheta FinSight Integration

| | |
|---|---|
| **Project code** | `493560B` |
| **Client** | Meridian Manufacturing Ltd. |
| **Document** | Risk Register |
| **Version** | v1 |

## Scoring key

- Probability / Impact: **H** = High (3), **M** = Medium (2), **L** = Low (1).
- **Risk score = Probability × Impact** (1–9). Score ≥ 6 = high priority (bold), 3–4 = medium, 1–2 = low.

| ID | Category | Description | Prob. | Impact | Score | Mitigation | Owner |
|---|---|---|---|---|---|---|---|
| R-01 | SAP performance | **RFC pool exhaustion** — integration exceeds 50 concurrent RFC connections, blocking SAP dialog users | M | H | **6** | Hard cap connection pool at < 50; circuit breaker on RFC boundary; back-pressure from Kafka; alert at 80% utilisation; co-locate Extractor on-prem to shorten connection hold time | SAP Basis Admin (Priya Deshmukh) |
| R-02 | Schema / data | **ODP / CDS schema change** — a support pack alters a CDS view (e.g. I_JournalEntry), breaking mappings | M | H | **6** | Version API contracts; schema-drift detection alert; graceful degradation (route affected records to DLQ, continue clean records); mapping change process | Zetheta Platform Eng (Marcus Wei) |
| R-03 | Network | **Bandwidth saturation** — integration exceeds 25% of the 450 Mbps MPLS in business hours | M | M | 4 | Rate-shaping + gzip compression; claim-check offload; schedule bulk/master loads off-peak; monitor WAN utilisation with alert at 20% | VP IT Infrastructure (Rajesh Venkataraman) |
| R-04 | Compliance | **Data residency breach** — a component, log sink or backup provisioned outside India (RBI violation) | L | H | 3 | Pin every service, S3 bucket, RDS, and ELK to `ap-south-1`; org-level SCP denying other regions; automated residency audit; no cross-region replication | VP IT Infrastructure (Rajesh Venkataraman) |
| R-05 | Data integrity | **Reconciliation break** — source/target counts or amounts diverge, or debit≠credit per document | M | H | **6** | Automated per-document + per-domain recon; targeted re-load of missing docs from lineage; PagerDuty on break; zero-break gate before sign-off | Head of Internal Audit (Dr. Kulkarni) |
| R-06 | Resilience | **DLQ overflow** — sustained failures fill the DLQ faster than it is drained | L | M | 2 | Extended-retention DLQ (topic + S3 overflow); dlq_depth metric with alert; on-call runbook for bulk reprocess; root-cause triage SLA | Zetheta Platform Eng (Marcus Wei) |
| R-07 | Security | **Token / auth failure** — OAuth token expiry or revocation halts FinSight loads | M | M | 4 | Automatic refresh-token flow on TOKEN_EXPIRED; re-auth on INVALID_TOKEN with security alert; monitor auth-failure rate; secrets in managed store with rotation | Zetheta Platform Eng (Marcus Wei) |
| R-08 | Availability | **Kafka broker loss** — a broker fails in `ap-south-1` | M | M | 4 | RF=3 / ISR=2 tolerates one broker loss with no data loss; one broker per AZ; automated broker replacement; under-replicated-partition alert | VP IT Infrastructure (Rajesh Venkataraman) |
| R-09 | Scheduling | **Batch-window overrun** — extraction runs into the 01:00–04:30 IST RGGBS000 window | M | M | 4 | Scheduler encodes batch + maintenance windows; job time-boxing; early-warning alert if a batch job trends late; defer non-critical loads | SAP Basis Admin (Priya Deshmukh) |
| R-10 | Compliance | **GST breakdown loss** — CGST/SGST/IGST components dropped during amount transformation | L | H | 3 | Dedicated GST Preserver component; validation rule asserting components sum to invoice amount; route violations to DLQ; sample audit of transforms | Head of Internal Audit (Dr. Kulkarni) |
| R-11 | Data integrity | **Master-data lag** — transactional records load before their master data, breaking referential integrity | M | M | 4 | Enforce master-before-transactional sequencing in Scheduler; referential-integrity validation (GL→CC/PC, AP→vendor, AR→customer); quarantine orphans to DLQ | Zetheta Platform Eng (Marcus Wei) |
| R-12 | Scalability | **Festival volume spike** — a 5× seasonal surge overwhelms transform/load capacity | M | H | **6** | Partition for 10× baseline (5× is headroom); autoscale on consumer lag; Kafka log absorbs surge while consumers catch up; bulkhead per domain; load-test the spike scenario | VP IT Infrastructure (Rajesh Venkataraman) |

## Summary

- **High priority (score ≥ 6):** R-01 (RFC pool), R-02 (schema change), R-05 (reconciliation break), R-12 (festival spike).
- **Medium (3–4):** R-03, R-04, R-07, R-08, R-09, R-10, R-11.
- **Low (1–2):** R-06 (DLQ overflow).

Detailed error handling for R-02, R-05, R-06, R-07, R-11 is specified in D4 (Error Handling & Retry Framework) and D5 (Reconciliation Logic).

*End of Risk Register v1.*
