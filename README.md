# Enterprise Retail Data Platform

An enterprise-style data engineering project built using Azure Data Lake Storage Gen2, Azure Databricks, Apache Spark, Delta Lake, and Azure Data Factory.

The project demonstrates a layered data platform for ingesting retail order data, applying data quality controls, transforming business data, and producing analytics-ready Gold datasets.

## Project Objectives

* Implement a Bronze–Silver–Gold data architecture.
* Preserve source data and isolate invalid records.
* Apply data quality checks and reconciliation controls.
* Build daily regional sales and order aggregates.
* Orchestrate the full-load pipeline using Azure Data Factory and Azure Databricks Jobs.
* Demonstrate incremental processing with timestamp-based change selection, Delta MERGE, watermark tracking, audit records, idempotency, and downstream reconciliation.
* Maintain architecture, data design, testing, operations, security, and deployment documentation.

## Technology Stack

| Technology                   | Purpose                                    |
| ---------------------------- | ------------------------------------------ |
| Azure Data Lake Storage Gen2 | Data lake storage                          |
| Azure Databricks             | Distributed data processing                |
| Apache Spark / PySpark       | Data transformation and validation         |
| Delta Lake                   | Transactional storage and MERGE operations |
| Azure Data Factory           | Full-load pipeline orchestration           |
| Unity Catalog                | Data access and governance integration     |
| GitHub                       | Source control and project documentation   |

## Architecture Overview

```text
Retail Order Source
        |
        v
   Raw Data Layer
        |
        v
 Bronze - Source Ingestion
        |
        v
 Silver - Cleansing and Validation
        |
        +------> Quarantine - Invalid Records
        |
        v
 Gold - Business Aggregations
        |
        v
 Analytics-Ready Datasets
```

The full-load pipeline is orchestrated through Azure Data Factory and an Azure Databricks Job. Incremental processing has been validated separately using isolated test paths and is not yet integrated into the production-style orchestration flow.

## Data Quality and Reconciliation

The baseline dataset contains 100,000 source records.

| Metric                        | Observed Value |
| ----------------------------- | -------------: |
| Raw records                   |        100,000 |
| Silver records                |         99,055 |
| Quarantined records           |            945 |
| Silver plus quarantine        |        100,000 |
| Duplicate order IDs in Silver |              0 |

The baseline reconciliation is:

`Silver records + Quarantine records = Raw input records`

The implementation includes checks for required fields, quantities, order statuses, duplicates, and business-rule consistency.

## Gold Data Products

The project includes daily regional sales aggregation and regional order-status reporting.

Financial calculations follow these rules:

* `gross_sales = quantity × unit_price`
* `net_sales = gross_sales − discount_amount + tax_amount + shipping_cost`
* Only `COMPLETED` orders contribute to completed-order financial metrics.

Gold datasets are validated against Silver to verify record counts, units, financial totals, and aggregation-key uniqueness.

## Incremental Processing — Controlled Test

A simulated batch of eight records was processed against isolated test targets.

| Test Metric                     | Result |
| ------------------------------- | -----: |
| Eligible records                |      8 |
| Existing records updated        |      3 |
| New records inserted            |      5 |
| Final Silver test rows          | 99,060 |
| Distinct order IDs              | 99,060 |
| Duplicate target order IDs      |      0 |
| Gold daily regional groups      |    153 |
| Duplicate daily regional groups |      0 |

### Reconciliation Results

| Metric           |           Silver |             Gold |
| ---------------- | ---------------: | ---------------: |
| Completed orders |           24,766 |           24,766 |
| Completed units  |          111,517 |          111,517 |
| Net sales        | 1,560,798,861.00 | 1,560,798,861.00 |

The same batch was processed again to test idempotency. The Silver test row count and distinct ID count remained unchanged, and the reconciled Gold metrics remained consistent.

The batch audit and control records reached `SUCCEEDED`. The test watermark advanced from `2026-10-01 23:22:00` to `2026-10-12 12:00:00`.

**Important:** This is a controlled incremental-processing test using a simulated batch. It does not yet establish a production-ready source change feed, automated incremental orchestration, concurrent-run safety, or complete failure recovery.

## Implementation Status

### Completed and validated

* Bronze ingestion and Silver transformation.
* Data quality validation and quarantine processing.
* Baseline reconciliation of raw, Silver, and quarantine records.
* Gold daily regional sales aggregation.
* Gold business-metric reconciliation.
* Azure Data Factory V1 full-load orchestration through an Azure Databricks Job.
* Controlled incremental batch selection and Delta MERGE.
* Incremental test control table and batch audit record.
* Incremental idempotency and Gold reconciliation checks.
* Architecture, design, governance, testing, and operations documentation.

### Planned enhancements

* Production-grade incremental processing and source CDC integration.
* Late-arriving and equal-timestamp record handling.
* Reliable retry, recovery, and concurrent-run protection.
* Incremental recomputation of affected Gold aggregates.
* Automated incremental orchestration.
* Automated monitoring, alerting, and failure notifications.
* CI/CD and environment-specific deployment configuration.
* Production-scale performance testing.
* Disaster recovery validation and operational runbooks.

## Documentation

Detailed project documentation is organized under `docs/`:

* `01-business-requirements/` — business requirements and scope.
* `02-architecture/` — solution and data architecture.
* `03-data-design/` — data dictionary.
* `04-ingestion/` — ingestion design.
* `05-transformation/` — transformation design.
* `06-data-quality/` — data quality framework.
* `07-incremental-processing/` — incremental processing design and current limitations.
* `08-security-governance/` — security and governance.
* `09-monitoring-operations/` — monitoring and operational procedures.
* `10-performance/` — performance and optimization.
* `11-testing/` — testing strategy and validation evidence.
* `12-deployment/` — deployment and CI/CD.
* `13-disaster-recovery/` — disaster recovery and business continuity.
* `14-decisions/` — architecture decision records.

## Project Status

The full-load Bronze–Silver–Gold pipeline and its V1 orchestration have been implemented and validated. Incremental processing has passed a controlled test covering MERGE behavior, idempotency, audit tracking, watermark advancement, and Gold reconciliation.

The project is continuing toward production-grade incremental processing, automated operations, and deployment hardening.
