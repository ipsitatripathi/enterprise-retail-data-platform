
# Data Architecture

**Document ID:** DAD-001
**Project:** Enterprise Retail Data Platform
**Version:** 1.0
**Status:** Active
**Last Updated:** September 2026

---

## 1. Document Purpose

This document defines the data architecture for the Enterprise Retail Data Platform.

It describes how retail data is ingested, stored, validated, transformed, quarantined, and prepared for downstream analytics.

The document focuses specifically on the **data structures, data layers, data flow, data quality boundaries, storage organization, and data lifecycle** within the platform.

The architecture follows a **Medallion-style data architecture** implemented using Azure Databricks, Unity Catalog, and Azure Data Lake Storage Gen2.

---

# 2. Data Architecture Overview

The platform processes retail transactional data through multiple logical data layers.

```text
Source Systems
     |
     v
Raw Data
     |
     v
Bronze Layer
     |
     v
Data Quality Validation
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
Analytics / Reporting
```

The primary objectives of this architecture are:

* Preserve raw source data.
* Create a reliable Bronze representation of incoming data.
* Identify invalid records before analytical consumption.
* Separate valid and invalid records.
* Produce a trusted Silver dataset.
* Provide a foundation for business-oriented Gold datasets.
* Support scalable processing as data volume increases.
* Maintain clear data ownership and lineage between layers.

---

# 3. Data Architecture Principles

The following principles govern the data architecture:

### 3.1 Layer Separation

Each data layer has a specific responsibility.

Raw, Bronze, Silver, Quarantine, and Gold data should not be treated as interchangeable datasets.

### 3.2 Preserve Source Data

Raw source data should be retained independently from transformed datasets wherever the storage and retention policies permit.

This allows:

* Reprocessing
* Auditing
* Troubleshooting
* Historical comparison
* Recovery from transformation issues

### 3.3 Quality Before Consumption

Data quality validation is performed before data is promoted into the trusted Silver layer.

### 3.4 Invalid Data Isolation

Records that fail defined quality rules should not silently enter the trusted analytical dataset.

They are instead routed to a dedicated quarantine path.

### 3.5 Traceability

The architecture should allow data to be traced from the source through Bronze, validation, Silver, and eventually Gold.

### 3.6 Schema Awareness

Schemas should be explicitly defined and controlled rather than relying entirely on automatic schema inference for production processing.

### 3.7 Scalability

The architecture uses distributed processing through Azure Databricks so that the same design can scale beyond the current development dataset.

---

# 4. Data Layers

## 4.1 Raw Layer

The Raw layer represents the initial landing area for source data.

### Responsibility

* Receive source files.
* Preserve source-level information.
* Provide an immutable or minimally modified representation of incoming data.
* Act as the starting point for downstream processing.

### Characteristics

* Source-oriented
* Minimal transformation
* Historical retention
* Suitable for reprocessing

The project uses a managed Unity Catalog volume named:

```text
retail_raw
```

This volume is used as the raw storage area for the project.

### Current Status

**Implemented**

---

# 5. Bronze Layer

The Bronze layer provides the first structured representation of the incoming retail data.

### Responsibilities

* Read data from the Raw layer.
* Apply the required schema.
* Standardize basic structural representation.
* Preserve source-level records required for downstream processing.
* Provide a consistent input for data quality processing.

Bronze processing does not attempt to create business-level analytical datasets.

### Characteristics

* Structured source data
* Minimal business transformation
* Distributed processing using Spark
* Foundation for data quality validation

### Current Status

**Implemented**

---

# 6. Data Quality Layer

The Data Quality layer evaluates Bronze records against defined business and technical validation rules.

The objective is to ensure that only valid records enter the trusted Silver dataset.

Examples of validation rules implemented in the current pipeline include:

* Quantity must be valid.
* Customer ID must not be null.
* Payment method must contain an accepted value.
* Order status must contain an accepted value.
* Duplicate order IDs must be identified.

The validation process produces two logical outcomes:

```text
Valid Records
      |
      v
   Silver

Invalid Records
      |
      v
 Quarantine
```

### Current Status

**Implemented**

---

# 7. Silver Layer

The Silver layer contains cleaned and validated transactional data.

It is considered the trusted operational/analytical foundation of the platform.

### Responsibilities

* Store valid records.
* Apply required data cleansing.
* Enforce defined data quality rules.
* Standardize data types and representations.
* Remove or prevent invalid records from entering the trusted dataset.
* Provide reliable data for downstream business transformations.

### Current Validation Results

The current implementation processed:

```text
Total Input Records     : 100,000
Valid Silver Records    : 99,055
Quarantine Records      : 945
```

Reconciliation:

```text
99,055 + 945 = 100,000
```

This confirms that every processed input record is accounted for between the Silver and Quarantine outcomes.

Additional validation results:

```text
Invalid quantities in Silver : 0
Null customer IDs in Silver  : 0
Invalid payment methods      : 0
Invalid order statuses       : 0
Duplicate order IDs          : 0
```

### Current Status

**Implemented**

---

# 8. Quarantine Layer

The Quarantine layer contains records that fail one or more defined data quality rules.

These records are separated from the trusted Silver dataset rather than being silently discarded.

### Responsibilities

* Preserve invalid records.
* Prevent invalid records from entering Silver.
* Support investigation of data quality failures.
* Provide a basis for remediation and reprocessing.
* Support operational troubleshooting.

The quarantine dataset should ideally contain sufficient information to determine:

* Original record
* Validation failure
* Validation rule
* Processing timestamp
* Source information

### Current Status

**Implemented**

---

# 9. Gold Layer

The Gold layer is the planned business-facing analytical layer.

It will contain datasets designed around specific business use cases rather than source-system structures.

Potential Gold datasets may include:

* Sales performance
* Customer analytics
* Product performance
* Regional performance
* Revenue analysis
* Order metrics

Gold datasets should be optimized for consumption by reporting, analytics, and downstream business applications.

### Current Status

**Planned**

Gold implementation will be documented separately once development begins.

---

# 10. Source-to-Target Data Flow

The current data flow is:

```text
Retail Source Data
        |
        v
   Raw Storage
   retail_raw
        |
        v
      Bronze
        |
        v
 Data Quality Rules
        |
        +----------------------+
        |                      |
        v                      v
      Silver              Quarantine
        |
        v
      Gold
   (Planned)
```

The architecture deliberately separates:

1. Source preservation
2. Structural processing
3. Data quality validation
4. Trusted data
5. Invalid data
6. Business-level analytical data

---

# 11. Data Grain

The primary transactional dataset represents retail order-level records.

The expected grain is:

> **One record represents one retail order transaction.**

The order identifier is therefore treated as an important business key for duplicate detection.

The current data quality validation includes duplicate detection on the order ID.

---

# 12. Data Quality and Reconciliation

Data quality is implemented as a gate between Bronze and Silver.

The general processing model is:

```text
Bronze Records
      |
      v
Apply Validation Rules
      |
      +----------------+
      |                |
      v                v
 Valid             Invalid
      |                |
      v                v
   Silver         Quarantine
```

A reconciliation check is performed to verify that:

```text
Silver Records + Quarantine Records
=
Input Records
```

For the current implementation:

```text
99,055 + 945 = 100,000
```

This provides record-level accounting across the processing boundary.

---

# 13. Schema Management

The platform uses explicit schema definitions during structured processing.

Schema management is intended to prevent unexpected source changes from silently propagating into downstream datasets.

Important schema considerations include:

* Data type consistency
* Required fields
* Nullable vs non-nullable fields
* Column naming conventions
* Schema evolution
* Backward compatibility
* Source-to-target mappings

### Current Status

Basic schema enforcement is **implemented**.

Advanced automated schema evolution and schema registry capabilities are **planned**.

---

# 14. Storage Organization

The platform uses Azure Data Lake Storage Gen2 and Unity Catalog-managed storage capabilities.

The logical organization is:

```text
Catalog
   |
   +-- Schema
        |
        +-- retail_raw
        |
        +-- Bronze
        |
        +-- Silver
        |
        +-- Quarantine
        |
        +-- Gold
```

The exact physical storage layout may evolve as the implementation progresses.

Logical data-layer boundaries remain independent of the underlying physical storage implementation.

---

# 15. Partitioning Strategy

Partitioning is an important consideration for large-scale datasets.

Partitioning should be based on query patterns, data volume, and cardinality rather than automatically partitioning every column.

Potential partitioning candidates for large transactional datasets include:

* Order date
* Transaction date
* Business processing date

High-cardinality fields such as customer ID or order ID should generally not be used as traditional storage partition columns without a specific performance justification.

### Current Status

Partitioning strategy will be finalized during performance optimization and large-volume testing.

**Status: Planned**

---

# 16. Incremental Data Architecture

The current implementation processes the available dataset as a batch.

The target architecture will support incremental processing so that only newly arrived or changed data needs to be processed.

Potential incremental mechanisms include:

* Processing timestamps
* Watermarks
* File arrival tracking
* Change Data Capture
* Delta Lake transaction history
* Source-system change indicators

The selected approach will depend on the characteristics of the source system.

### Current Status

**Planned**

---

# 17. Data Lineage

The intended lineage path is:

```text
Source
  |
  v
Raw
  |
  v
Bronze
  |
  v
Data Quality
  |
  +---------> Quarantine
  |
  v
Silver
  |
  v
Gold
```

Lineage should allow technical users and data engineers to understand:

* Where a dataset originated.
* Which transformation produced it.
* Which validation rules were applied.
* Where invalid records were routed.
* Which downstream datasets depend on it.

Unity Catalog provides the governance foundation for implementing centralized data discovery and lineage capabilities.

### Current Status

Basic architectural lineage is defined.

Advanced operational lineage and monitoring are **planned**.

---

# 18. Data Lifecycle

The intended lifecycle of a retail record is:

```text
Ingestion
   |
   v
Raw Retention
   |
   v
Bronze Processing
   |
   v
Data Quality Validation
   |
   +---- Invalid ----> Quarantine
   |
   v
Silver
   |
   v
Gold
   |
   v
Analytics / Reporting
```

Retention policies will eventually be defined according to:

* Business requirements
* Regulatory requirements
* Storage cost
* Recovery requirements
* Historical reporting requirements

### Current Status

Lifecycle architecture defined.

Formal retention policies are **planned**.

---

# 19. Data Security and Governance

Data access should be controlled through Azure Databricks and Unity Catalog capabilities.

The target governance model includes:

* Catalog-level access control
* Schema-level access control
* Table/volume permissions
* Role-based access
* Data ownership
* Data discovery
* Data lineage
* Auditing
* Controlled access to sensitive datasets

Sensitive data should not be unnecessarily exposed across layers or to users who do not require access.

### Current Status

Unity Catalog is available and the managed raw volume has been created.

Detailed production-grade security policies and access roles are **planned**.

---

# 20. Data Quality Failure Handling

A failed record should not cause the entire dataset to become unreliable.

The architecture therefore separates record-level failures from successful records.

Example:

```text
100,000 Input Records
        |
        v
Validation
        |
        +---- 99,055 Valid ----> Silver
        |
        +---- 945 Invalid ----> Quarantine
```

This allows valid data processing to continue while preserving invalid records for investigation.

---

# 21. Scalability Considerations

The architecture is designed to scale from development datasets to significantly larger enterprise workloads.

Scalability considerations include:

* Distributed Spark processing
* Delta-based storage
* Partition-aware processing
* Incremental ingestion
* Efficient joins
* Shuffle optimization
* Adaptive Query Execution
* Appropriate file sizes
* Avoiding unnecessary data movement
* Parallel processing

Performance optimization will be addressed in greater detail during the performance engineering phase.

---

# 22. Current Data Architecture Status

| Capability                  | Status      |
| --------------------------- | ----------- |
| Raw ingestion               | Implemented |
| Managed `retail_raw` volume | Implemented |
| Bronze processing           | Implemented |
| Data quality validation     | Implemented |
| Quarantine processing       | Implemented |
| Silver transformation       | Implemented |
| Record reconciliation       | Implemented |
| Gold layer                  | Planned     |
| Incremental processing      | Planned     |
| Advanced lineage            | Planned     |
| Production security model   | Planned     |
| Retention policies          | Planned     |
| Performance optimization    | Planned     |
| Monitoring                  | Planned     |

---

# 23. Key Data Architecture Decisions

### Decision 1 — Medallion-style data layers

The platform separates raw, Bronze, Silver, Quarantine, and Gold responsibilities.

**Reason:**
This provides clear data lifecycle boundaries and improves maintainability, quality control, and downstream consumption.

### Decision 2 — Separate invalid records from trusted data

Invalid records are routed to Quarantine rather than silently dropped.

**Reason:**
This preserves potentially recoverable data and provides traceability for data quality failures.

### Decision 3 — Silver as the trusted transactional layer

Only records passing defined validation rules are promoted to Silver.

**Reason:**
Downstream consumers should not need to independently repeat basic data quality validation.

### Decision 4 — Unity Catalog for governance

Unity Catalog is used as the governance foundation.

**Reason:**
It provides centralized data organization, access control, discovery, and a foundation for lineage and auditing.

---

# 24. Future Data Architecture Evolution

The architecture will evolve through the following phases:

```text
Current
   |
   +-- Raw
   +-- Bronze
   +-- Data Quality
   +-- Silver
   +-- Quarantine
   |
   v
Next
   |
   +-- Gold
   +-- Business Data Products
   |
   v
Advanced
   |
   +-- Incremental Processing
   +-- Monitoring
   +-- Automated Testing
   +-- CI/CD
   +-- Advanced Governance
   +-- Disaster Recovery
   +-- Performance Optimization
```

Each capability will be documented and marked as implemented only after it has been actually developed and validated.

---

# 25. Document Change History

| Version | Date           | Change                                  |
| ------- | -------------- | --------------------------------------- |
| 1.0     | September 2026 | Initial data architecture documentation |
