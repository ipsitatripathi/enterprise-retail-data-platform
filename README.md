# Enterprise Retail Data Platform

An enterprise-style retail data engineering project built using Azure Data Factory, Azure Databricks, Azure Data Lake Storage Gen2, and Delta Lake. The platform implements a Medallion Architecture with data quality controls, quarantine processing, business-oriented Gold datasets, and orchestrated full-load processing.

## Project Overview

Retail organizations need reliable data pipelines that consolidate transactional data, validate data quality, standardize business attributes, and produce trusted datasets for analytics.

This project demonstrates the implementation of a layered data platform that processes retail order data from ingestion through business-level aggregation.

The current implementation includes Bronze ingestion, Silver transformation, data-quality validation, quarantine handling, Gold aggregations, and Azure Data Factory orchestration using Databricks serverless compute.

## Architecture

```text
Retail Source Data
       |
       v
Raw Layer - ADLS Gen2
       |
       v
Bronze Layer - ADLS Gen2
       |
       v
Silver Layer - ADLS Gen2
       |
       +--------------------+
       |                    |
       v                    v
 Validated Records      Quarantine Layer
       |
       v
Gold Layer - ADLS Gen2
       |
       +-------------------------+
       |                         |
       v                         v
Regional Orders          Daily Regional Sales
```

Azure Data Factory orchestrates the Databricks job that executes the Bronze, Silver, and Gold notebook tasks in sequence.

## Technology Stack

| Technology                   | Purpose                                     |
| ---------------------------- | ------------------------------------------- |
| Azure Data Factory           | Pipeline orchestration                      |
| Azure Databricks             | Distributed data processing                 |
| Databricks Serverless        | Notebook task execution                     |
| Azure Data Lake Storage Gen2 | Layered data storage                        |
| Delta Lake                   | Structured data storage and reliable writes |
| PySpark                      | Data transformation and aggregation         |
| GitHub                       | Source control and project documentation    |

## Medallion Architecture

### Raw Layer

The Raw layer preserves landed source data for subsequent processing and traceability.

### Bronze Layer

The Bronze layer ingests source records into the data platform.

### Silver Layer

The Silver layer standardizes and validates retail order data. Invalid records are routed to the Quarantine layer.

### Quarantine Layer

The Quarantine layer stores records rejected by the implemented validation rules for investigation and potential remediation.

### Gold Layer

The Gold layer provides business-oriented datasets designed for downstream reporting and analysis.

#### Gold Data Product 1: Regional Orders

Provides regional order-status metrics and completed-order sales measures.

Current metrics include:

* Total orders.
* Completed orders.
* Pending orders.
* Cancelled orders.
* Returned orders.
* Total units.
* Gross sales.
* Total discount.
* Total tax.
* Total shipping cost.
* Net sales.
* Average order value.

Order-status metrics cover all statuses. Sales measures are calculated from completed orders.

#### Gold Data Product 2: Daily Regional Sales

Provides daily sales metrics by region at the `order_day + region` grain.

Current metrics include:

* Total completed orders.
* Total units.
* Gross sales.
* Total discount.
* Total tax.
* Total shipping cost.
* Net sales.
* Average order value.

## Data Quality and Reconciliation

The implementation includes validation rules for customer identifiers, quantities, payment methods, order statuses, duplicate order IDs, and reconciliation between processing layers.

Current validation results:

| Metric                            |  Result |
| --------------------------------- | ------: |
| Input records                     | 100,000 |
| Silver records                    |  99,055 |
| Quarantine records                |     945 |
| Invalid quantities in Silver      |       0 |
| Null customer IDs in Silver       |       0 |
| Invalid payment methods in Silver |       0 |
| Invalid order statuses in Silver  |       0 |
| Duplicate order IDs in Silver     |       0 |

Record reconciliation:

`100,000 = 99,055 + 945`

The current Gold implementation has also been validated against Silver for completed-order counts, units, and financial totals.

## Orchestration

The full-load orchestration is implemented using Azure Data Factory and an Azure Databricks job.

**ADF pipeline:** `PL_Retail_Full_Load_V1`

**Databricks job:** `JOB_Retail_Full_Load_V1`

The job executes three dependent notebook tasks:

1. `Bronze_Ingestion`
2. `Silver_Transformation`
3. `Gold_Aggregation`

The pipeline and all three Databricks tasks have completed successfully during validation.

The persisted Gold datasets have been read back successfully from ADLS Gen2.

| Gold dataset         | Validated row count |
| -------------------- | ------------------: |
| Regional Orders      |                   5 |
| Daily Regional Sales |                 150 |

## Repository Structure

```text
enterprise-retail-data-platform/
|
├── README.md
|
└── docs/
    ├── 01-business-requirements/
    ├── 02-architecture/
    ├── 03-data-design/
    ├── 04-ingestion/
    ├── 05-transformation/
    ├── 06-data-quality/
    ├── 07-incremental-processing/
    ├── 08-security-governance/
    ├── 09-monitoring-operations/
    ├── 10-performance/
    ├── 11-testing/
    ├── 12-deployment/
    ├── 13-disaster-recovery/
    └── 14-decisions/
```

The repository contains architecture documentation, business requirements, data design, implementation strategy, data-quality controls, operational guidance, deployment planning, and architecture decision records.

Finalized notebook source files and additional deployment artifacts will be published as they are reviewed and prepared for source control.

## Current Implementation Status

### Implemented

* Raw and Bronze ingestion.
* Silver transformation and standardization.
* Data-quality validation.
* Quarantine processing.
* Record-count reconciliation.
* Regional Orders Gold dataset.
* Daily Regional Sales Gold dataset.
* Gold Delta persistence to ADLS Gen2.
* Gold read-back validation.
* Silver-to-Gold reconciliation.
* Azure Data Factory full-load orchestration.
* Databricks serverless job execution.
* Sequential notebook task dependencies.
* GitHub architecture and implementation documentation.

### Planned Enhancements

* Incremental ingestion and transformation.
* Watermark and control-table management.
* Idempotent incremental processing.
* Automated CI/CD deployment.
* Automated monitoring and failure notifications.
* Advanced performance optimization.
* Metadata-driven data-quality rules.
* Centralized operational logging and alerting.
* Production-grade recovery and disaster-recovery automation.
* Additional analytics-ready Gold data products.

Planned capabilities will be marked as implemented only after development and validation.

## Documentation

Project documentation is maintained under the `docs/` directory.

Key documents include:

* Business requirements.
* Solution and data architecture.
* Data dictionary.
* Ingestion and transformation design.
* Data-quality framework.
* Incremental-processing design.
* Security and governance.
* Monitoring and operations.
* Performance and optimization.
* Testing and validation.
* Deployment and CI/CD.
* Disaster recovery and business continuity.
* Architecture decision records.

## Project Goals

The project is intended to demonstrate practical data engineering capabilities, including distributed transformations, layered data architecture, data-quality management, financial reconciliation, cloud storage integration, orchestration, and operational validation.

The implementation will continue to evolve toward a more production-oriented platform through incremental processing, automated testing, deployment automation, monitoring, and recovery improvements.

---

**Project status:** Full-load Bronze-to-Gold processing and ADF orchestration implemented and validated in the development environment. Advanced production capabilities remain in progress.
