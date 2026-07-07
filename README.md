# FDE-9B — Custom API Integration: SAP S/4HANA → Zetheta FinSight

**Project code:** 493560B · **Client (simulated):** Meridian Manufacturing Ltd. · **Role:** Forward Deployed Engineer

An end-to-end **design and specification** for a production-grade integration that streams
financial data from Meridian Manufacturing's SAP S/4HANA 2023 ERP into the Zetheta FinSight
analytics platform — reducing financial data latency from **24 hours to under 4 hours** while
respecting Indian data-residency, RFC-pool, and bandwidth constraints.

> **Confidential.** Private repository. All work is the property of Zetheta Algorithms Private
> Limited. Not for circulation.

## The problem

Meridian runs SAP S/4HANA in nightly batch mode; the CFO has no real-time visibility into
financial performance across 7 plants and 3 legal entities (company codes MC01–MC03). This
project designs the integration bridge: extraction (ODP delta / IDoc / batch), a Kafka-based
transformation pipeline, resilient error handling, reconciliation that proves every rupee is
accounted for, and full observability.

## Repository structure

```
docs/                     Canonical facts, requirements, glossary, risk register
deliverables/
  D1_Architecture/        Integration architecture (C4 L1–L3, network, deployment)
  D2_API_Spec/            OpenAPI 3.0 for 12 SAP source + 12 FinSight destination endpoints
  D3_Mappings/            80+ field mappings across 10 data domains
  D4_Error_Handling/      Error taxonomy, retry, circuit breaker, DLQ, error-code registry
  D5_Reconciliation/      Reconciliation framework + 25+ data quality rules
  D6_Monitoring/          12-panel dashboard spec, logging standard, 15+ alert rules
  D7_Testing/             25 test scenarios + traceability matrix
  D8_Deployment/          Deployment runbook + rollback procedure
  D9_Stakeholder/         CFO / IT / Platform-Engineering communication documents
api/                      OpenAPI YAML (sap/, finsight/) + Postman collection
diagrams/                 Mermaid (.mmd) + Draw.io (.drawio) sources and PNG exports
mappings/                 Per-domain mapping tables (MAP_*.md)
monitoring/               Grafana dashboard JSON, Prometheus alert rules
tests/                    Test scenario files (TST_*.md)
presentation/             Oral-defence deck outline
```

## Deliverables index

| # | Deliverable | Location |
|---|---|---|
| D1 | Integration Architecture | [deliverables/D1_Architecture](deliverables/D1_Architecture/) |
| D2 | API Specification (OpenAPI 3.0) | [deliverables/D2_API_Spec](deliverables/D2_API_Spec/) |
| D3 | Data Transformation & Mapping | [deliverables/D3_Mappings](deliverables/D3_Mappings/) |
| D4 | Error Handling & Retry Framework | [deliverables/D4_Error_Handling](deliverables/D4_Error_Handling/) |
| D5 | Reconciliation & Data Quality | [deliverables/D5_Reconciliation](deliverables/D5_Reconciliation/) |
| D6 | Monitoring & Alerting | [deliverables/D6_Monitoring](deliverables/D6_Monitoring/) |
| D7 | Integration Test Plan | [deliverables/D7_Testing](deliverables/D7_Testing/) |
| D8 | Deployment Runbook | [deliverables/D8_Deployment](deliverables/D8_Deployment/) |
| D9 | Stakeholder Communication | [deliverables/D9_Stakeholder](deliverables/D9_Stakeholder/) |

## Technology stack (design target)

Apache Kafka · SAP ODP/CDS + OData Gateway · OAuth 2.0 · OpenAPI 3.0 · Prometheus + Grafana ·
ELK · PagerDuty · AWS `ap-south-1` (India data residency). See
[D1 technology stack justification](deliverables/D1_Architecture/) for alternatives considered.

## Conventions

All artifacts follow the Zetheta naming standard documented in
[docs/00_CANONICAL_FACTS.md](docs/00_CANONICAL_FACTS.md) §9. That file is the single source of
truth for domains, endpoint IDs, company codes, and tenant routing used across every deliverable.
