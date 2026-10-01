# Ingestion Design

**Document ID:** ING-001
**Project:** Enterprise Retail Data Platform
**Version:** 1.0
**Status:** Active
**Last Updated:** September 2026

---

## 1. Document Purpose

This document defines the ingestion architecture and implementation approach for the Enterprise Retail Data Platform.

It describes how retail source data enters the platform, how it is stored in the Raw layer, how it is processed into Bronze, and how the ingestion process integrates with downstream data quality and transformation stages.

The document also distinguishes between capabilities that are currently implemented and capabilities planned for future production evolution.

---

# 2. Ingestion Objectives

The ingestion architecture is designed to:

* Capture source retail data reliably.
* Preserve incoming source data.
* Provide a consistent input to downstream processing.
* Apply controlled schema handling.
* Support distributed processing.
* Prevent invalid data from silently entering trusted layers.
* Maintain traceability between ingestion and downstream processing.
* Provide a foundation for future incremental ingestion.
* Scale to larger enterprise datasets.

---

# 3. Source Data

The current project processes retail transactional data.

The primary logical dataset is:

```text
retail_orders
```

The dataset represents retail order transactions.

The expected grain is:

> One record represents one retail order transaction.

The current development/validation dataset contains:

```text
100,000 records
```

---

# 4. Source Data Characteristics

The source dataset contains transactional attributes such as:

```text
order_id
customer_id
order_date
product
category
region
quantity
unit_price
```

Additional business attributes used by the enterprise data-quality contract include fields such as:

```text
payment_method
order_status
```

The ingestion design allows source attributes to be standardized before trusted downstream consumption.

---

# 5. Ingestion Architecture

The current logical ingestion flow is:

```text
Source Data
     |
     v
Raw Storage
     |
     v
Bronze
     |
     v
Data Quality Validation
     |
     +-------------------+
     |                   |
     v                   v
Silver              Quarantine
```

The ingestion process is intentionally separated from business-oriented analytical transformations.

This allows the platform to preserve source data before applying downstream transformations.

---

# 6. Raw Ingestion

The Raw layer acts as the initial landing area for source data.

The project uses a managed Unity Catalog volume:

```text
retail_raw
```

The Raw layer is intended to preserve source-level data with minimal transformation.

### Raw Layer Responsibilities

* Receive source data.
* Preserve source structure.
* Provide a stable starting point for processing.
* Support reprocessing.
* Provide traceability to the original input.

### Current Status

**Implemented**

---

# 7. Bronze Ingestion

Bronze ingestion reads data from the Raw layer and creates a structured representation suitable for downstream processing.

The Bronze layer is intentionally close to the source structure.

### Bronze Responsibilities

* Read raw records.
* Apply the expected schema.
* Standardize basic data types.
* Preserve relevant source attributes.
* Make data available for quality validation.

Bronze processing does not perform extensive business aggregation.

### Current Status

**Implemented**

---

# 8. Databricks Processing

Azure Databricks is used as the distributed processing engine.

The ingestion and transformation logic is implemented using Apache Spark/PySpark.

The project notebook used for the current ingestion workflow is:

```text
01_Bronze_Ingestion
```

The notebook is maintained in the Databricks project workspace under:

```text
Project_2_Enterprise_Retail
```

The current implementation uses Databricks serverless compute.

---

# 9. Processing Model

The current implementation follows a batch-processing model.

Conceptually:

```text
Read Source
    |
    v
Load Raw/Bronze Data
    |
    v
Apply Schema
    |
    v
Run Transformations
    |
    v
Run Data Quality Rules
    |
    +------------+
    |            |
    v            v
 Silver      Quarantine
```

The architecture is designed so that the processing model can later evolve into incremental processing.

---

# 10. Schema Handling

Schema handling is an important part of ingestion.

The platform uses an explicitly defined schema rather than relying entirely on inferred data types for downstream processing.

Example logical schema:

| Column        | Data Type |
| ------------- | --------- |
| `order_id`    | Integer   |
| `customer_id` | String    |
| `order_date`  | Date      |
| `product`     | String    |
| `category`    | String    |
| `region`      | String    |
| `quantity`    | Integer   |
| `unit_price`  | Numeric   |

Explicit schema handling provides greater control over:

* Data types
* Nullability
* Validation
* Transformation behavior
* Downstream compatibility

---

# 11. Date Standardization

Source systems may provide dates in formats that are not immediately compatible with the target schema.

The ingestion/transformation process standardizes the order date into a proper date data type.

For example, source values such as:

```text
9/1/2026
```

are converted into a standard date representation.

This prevents downstream analytical operations from treating dates as arbitrary strings.

---

# 12. Data Type Standardization

Source data may arrive with inconsistent or generic data types.

The ingestion process establishes standardized types for important fields.

Examples:

```text
order_id     -> integer
quantity     -> integer
order_date   -> date
customer_id  -> string
unit_price   -> numeric
```

This improves consistency between Bronze and downstream layers.

---

# 13. File and Storage Strategy

The platform separates logical data layers from physical storage.

The current architecture uses Unity Catalog-managed storage capabilities.

The Raw layer is represented through the managed volume:

```text
retail_raw
```

The physical storage implementation can evolve without changing the logical responsibilities of the data layers.

---

# 14. Ingestion Data Quality Boundary

Data ingestion and data quality validation are treated as separate responsibilities.

The ingestion process is responsible for making source data available in a controlled structure.

The data quality layer then determines whether records are suitable for trusted Silver processing.

```text
Bronze
   |
   v
Quality Validation
   |
   +---- Valid ----> Silver
   |
   +---- Invalid --> Quarantine
```

This separation prevents ingestion logic from becoming tightly coupled with every business validation rule.

---

# 15. Error Handling

The ingestion architecture distinguishes between different categories of failures.

### 15.1 File-Level Failure

Examples:

* File unavailable
* Invalid file format
* Corrupted input
* Permission failure

A file-level failure should prevent that input from being treated as successfully processed.

### 15.2 Schema-Level Failure

Examples:

* Unexpected data type
* Missing required column
* Incompatible schema

These failures should be detected before trusted downstream processing.

### 15.3 Record-Level Failure

Examples:

* Invalid quantity
* Null customer ID
* Invalid payment method
* Invalid order status
* Duplicate order ID

Record-level failures are handled through the data quality and quarantine process.

---

# 16. Invalid Record Handling

Invalid records are not silently discarded.

The current implementation separates failed records into the Quarantine dataset.

Current processing results:

```text
Total Input Records : 100,000
Valid Silver        : 99,055
Quarantine          : 945
```

Reconciliation:

```text
99,055 + 945 = 100,000
```

This provides complete accounting of the processed records.

---

# 17. Ingestion Reconciliation

Reconciliation is used to verify that records are not unexpectedly lost during processing.

The fundamental reconciliation rule is:

```text
Input Records
=
Valid Records
+
Invalid Records
```

For the current implementation:

```text
100,000
=
99,055
+
945
```

The reconciliation passed successfully.

---

# 18. Idempotency Considerations

The ingestion architecture should eventually support idempotent processing.

Idempotency means that reprocessing the same source input should not unintentionally create duplicate downstream records.

Potential mechanisms include:

* Source file identifiers
* Batch identifiers
* Processing timestamps
* Business-key validation
* Delta Lake transactional writes
* Merge-based processing
* Processed-file tracking

### Current Status

Basic duplicate detection is implemented.

A complete production-grade ingestion idempotency framework is **planned**.

---

# 19. Incremental Ingestion

The current implementation processes the available dataset as a batch.

The target architecture will support incremental ingestion.

Potential approaches include:

### File-Based Incremental Processing

Track newly arrived files and process only files that have not previously been processed.

### Timestamp-Based Processing

Use source-system timestamps to identify newly created or modified records.

### Change Data Capture

Consume inserts, updates, and deletes from a source system when CDC is available.

### Delta-Based Processing

Use Delta Lake transaction capabilities to identify changes between processing cycles.

The final approach will depend on the characteristics of the source system.

### Current Status

**Planned**

---

# 20. Duplicate Handling

Duplicate detection is performed using the business key:

```text
order_id
```

The current Silver dataset contains:

```text
Duplicate order IDs = 0
```

Duplicate handling will become more important when incremental ingestion is implemented because the same business record may appear in multiple ingestion batches.

---

# 21. Scalability

The ingestion architecture uses Spark-based distributed processing so that processing can scale beyond the current 100,000-record dataset.

Future scalability considerations include:

* Parallel file processing
* Partition-aware processing
* Incremental ingestion
* Efficient serialization
* Appropriate file sizing
* Avoiding unnecessary data movement
* Delta Lake optimization
* Adaptive Query Execution
* Cluster/compute optimization

---

# 22. Performance Considerations

Ingestion performance depends on:

* Source file size
* Number of files
* File format
* Number of partitions
* Data volume
* Schema complexity
* Transformation complexity
* Network/storage throughput
* Spark execution characteristics

Performance optimization will be addressed separately after the core pipeline is implemented.

### Current Status

Basic distributed processing is implemented.

Detailed performance optimization is **planned**.

---

# 23. Security Considerations

Access to ingestion data should be controlled through the platform's governance layer.

The target model includes:

* Unity Catalog permissions
* Controlled access to Raw data
* Role-based access
* Separation of development and production access
* Auditability
* Least-privilege access

The Raw layer should not automatically be exposed to all downstream consumers.

### Current Status

Unity Catalog is available.

Detailed production security roles and access policies are **planned**.

---

# 24. Operational Metadata

A production ingestion framework should maintain metadata such as:

```text
source_file
batch_id
ingestion_timestamp
processing_timestamp
record_count
success_count
failure_count
pipeline_status
error_details
```

This metadata can support:

* Monitoring
* Troubleshooting
* Auditing
* Reprocessing
* SLA measurement
* Operational reporting

### Current Status

Operational metadata framework is **planned**.

---

# 25. Current Implementation Summary

| Capability                        | Status      |
| --------------------------------- | ----------- |
| Raw storage                       | Implemented |
| `retail_raw` managed volume       | Implemented |
| Bronze ingestion                  | Implemented |
| Explicit schema handling          | Implemented |
| Data type standardization         | Implemented |
| Date standardization              | Implemented |
| Data quality integration          | Implemented |
| Quarantine processing             | Implemented |
| Record reconciliation             | Implemented |
| Duplicate detection               | Implemented |
| Batch processing                  | Implemented |
| Production idempotency framework  | Planned     |
| Incremental ingestion             | Planned     |
| Operational metadata              | Planned     |
| Automated monitoring              | Planned     |
| Advanced performance optimization | Planned     |

---

# 26. Target Ingestion Architecture

The future production-oriented ingestion architecture is:

```text
Source Systems
      |
      v
Ingestion Layer
      |
      v
Raw Storage
      |
      v
Bronze
      |
      v
Data Quality
      |
      +--------------------+
      |                    |
      v                    v
    Silver             Quarantine
      |
      v
     Gold
      |
      v
Analytics / BI
```

Future capabilities will be introduced incrementally rather than treating the entire target architecture as already implemented.

---

# 27. Key Ingestion Design Decisions

## Decision 1 — Separate Raw and Bronze

Raw preserves source-level data while Bronze provides a structured processing layer.

**Reason:**
This creates a recoverable source boundary and prevents downstream transformations from modifying the original input representation.

## Decision 2 — Use Explicit Schemas

Structured processing uses controlled data types.

**Reason:**
Explicit schemas reduce unexpected type inference behavior and improve downstream consistency.

## Decision 3 — Separate Record-Level Quality Failures

Invalid records are routed to Quarantine rather than causing valid records to be rejected.

**Reason:**
This maximizes usable data while preserving failed records for investigation.

## Decision 4 — Use Distributed Processing

Spark running on Azure Databricks is used for processing.

**Reason:**
The platform is intended to scale beyond small datasets and support enterprise-sized workloads.

---

# 28. Future Enhancements

The ingestion architecture will evolve to include:

* Incremental file processing
* CDC-based ingestion where applicable
* Idempotent processing
* Automated ingestion metadata
* Pipeline orchestration
* Retry mechanisms
* Monitoring and alerting
* SLA tracking
* Automated schema validation
* CI/CD integration

These capabilities will be documented as implemented only after they are developed and validated.

---

# 29. Change History

| Version | Date           | Change                                 |
| ------- | -------------- | -------------------------------------- |
| 1.0     | September 2026 | Initial ingestion design documentation |

