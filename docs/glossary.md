# Glossary of Terms

Reference glossary for the FDE-9B integration. Terms are used consistently across all
deliverables (D1–D9).

| Term | Definition |
|---|---|
| **ACDOCA** | SAP Universal Journal table — single source of truth for all financial line items in S/4HANA, replacing legacy tables BSEG, BSIS, BSIK, BSID. |
| **BAPI** | Business Application Programming Interface — SAP's standard programmatic interface for executing business transactions remotely. |
| **C4 Model** | Software-architecture modelling framework with four levels: Context, Container, Component, and Code. |
| **CDS View** | Core Data Services — SAP's virtual data-modelling framework on HANA that defines analytical views over database tables. |
| **Circuit Breaker** | Resilience pattern that halts calls to a failing downstream service after a failure threshold to prevent cascade failures. Three states: CLOSED / OPEN / HALF-OPEN. |
| **Company Code** | Legal entity within SAP representing an independent accounting unit. Meridian: MC01 (Maharashtra), MC02 (Gujarat), MC03 (Tamil Nadu). |
| **Correlation ID** | Unique identifier propagated across SAP → Kafka → Transformation → FinSight to trace a single record end-to-end. |
| **Dead Letter Queue (DLQ)** | Dedicated store for records that could not be processed after exhausting retries, enabling manual review and reprocessing. |
| **Delta Extraction** | Extracting only records changed since the last successful extraction, using ODP delta tokens. |
| **DMBOK** | DAMA Data Management Body of Knowledge — defines the six data-quality dimensions used in D5. |
| **ERP** | Enterprise Resource Planning — integrated suite managing finance, procurement, manufacturing, sales, and HR. |
| **Exponential Backoff** | Retry strategy where wait time grows exponentially per attempt: `wait = min(cap, random(base, base·2^attempt))`. |
| **FDE** | Forward Deployed Engineer — engineer embedded at a client site to customise, integrate, and deploy enterprise platforms. |
| **FinSight** | Zetheta FinSight 4.2 — the destination financial-analytics platform (AWS `ap-south-1`). |
| **Fiscal Year Variant** | Client-specific definition of fiscal periods. Meridian uses variant **V3** (Apr–Mar) with special periods 013–016. |
| **GST** | Goods and Services Tax — India's indirect-tax regime; CGST/SGST/IGST breakdown must be preserved through transforms. |
| **Idempotency** | Property ensuring repeating an operation yields the same result as performing it once — enforced via `Idempotency-Key` on FinSight POST. |
| **IDoc** | Intermediate Document — SAP's message format for asynchronous exchange (used here for bank statements, FINSTA). |
| **Jitter** | Random variation added to retry timing to prevent the thundering-herd problem. |
| **ODP** | Operational Data Provisioning — SAP framework for delta-capable extraction with subscription management. |
| **OpenAPI** | Industry-standard specification format (formerly Swagger) for describing REST APIs. |
| **RFC** | Remote Function Call — SAP's synchronous inter-system protocol. Integration is capped at 50 concurrent connections. |
| **SLA** | Service Level Agreement — here, financial data freshness ≤ 4 hours (from 24 hours). |
| **TCURR** | SAP exchange-rate table — point-in-time rates keyed by `GDATU`, used for currency conversion. |
| **TDS** | Tax Deducted at Source — Indian tax-collection mechanism deducted before payment. |
| **Tenant** | FinSight logical partition per company code: MC01→MERIDIAN-MC01, MC02→MERIDIAN-MC02, MC03→MERIDIAN-MC03. |
