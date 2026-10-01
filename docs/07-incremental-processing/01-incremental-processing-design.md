
# Incremental Processing Design

## 1. Purpose

This document defines the incremental processing strategy for the Enterprise Retail Data Platform.

The objective is to enable the platform to process only new or changed source records rather than repeatedly processing the complete dataset.

Incremental processing is a planned enhancement to the current batch architecture and will be introduced after the foundational Raw, Bronze, Data Quality, Silver, and Quarantine layers are established.

---

## 2. Current Status

| Capability                       | Status      |
| -------------------------------- | ----------- |
| Full/batch ingestion             | Implemented |
| Raw ingestion                    | Implemented |
| Bronze processing                | Implemented |
| Data Quality validation          | Implemented |
| Silver processing                | Implemented |
| Quarantine processing            | Implemented |
| Record reconciliation            | Implemented |
| Incremental ingestion            | Planned     |
| Incremental Silver processing    | Planned     |
| Watermark management             | Planned     |
| Change detection                 | Planned     |
| Incremental operational metadata | Planned     |

The current implementation processes the available dataset as a batch.

Incremental processing will be implemented as a subsequent enhancement.

---

## 3. Business Requirement

As source data volume grows, repeatedly processing the complete historical dataset becomes increasingly expensive and inefficient.

The platform should therefore support:

* Processing newly arrived records
* Processing changed records where applicable
* Avoiding unnecessary reprocessing of unchanged data
* Maintaining historical data correctly
* Supporting restart and recovery scenarios
* Preventing duplicate processing
* Tracking the processing boundary
* Providing operational visibility into each incremental run

---

## 4. Incremental Processing Architecture

The target processing flow is:

```text
Source System
     |
     v
Raw Layer
     |
     v
Bronze Layer
     |
     v
Incremental Detection
     |
     +--------------------+
     |                    |
 New / Changed         Unchanged
     |                    |
     v                    |
Data Quality              |
     |                    |
     +----------+---------+
                |
        Valid / Invalid
           |       |
           |       +------> Quarantine
           |
           v
        Silver
           |
           v
          Gold
```

Incremental detection will determine which records require processing during each execution.

---

## 5. Incremental Processing Strategy

The platform will support a watermark-based incremental processing approach.

A watermark represents the latest successfully processed source position.

Depending on the source system, the watermark may be based on:

* Source modification timestamp
* Creation timestamp
* Increasing numeric identifier
* Change Data Capture sequence
* Source-specific version or offset

The preferred mechanism will depend on the capabilities of the source system.

---

## 6. Watermark Concept

The target design will maintain processing metadata containing information such as:

| Metadata           | Description                                |
| ------------------ | ------------------------------------------ |
| Pipeline name      | Name of the processing pipeline            |
| Source name        | Source system or dataset                   |
| Entity name        | Source entity/table/file                   |
| Watermark column   | Column used for incremental detection      |
| Previous watermark | Last successfully processed value          |
| Current watermark  | Maximum value processed in the current run |
| Run ID             | Unique execution identifier                |
| Start time         | Processing start timestamp                 |
| End time           | Processing completion timestamp            |
| Status             | Success or failure                         |
| Records read       | Number of source records read              |
| Records processed  | Number of records processed                |
| Records rejected   | Number of quarantined records              |

The metadata design will be implemented as part of the future operational framework.

---

## 7. Watermark Processing Logic

The target processing pattern is:

```text
Read previous successful watermark
              |
              v
Determine incremental boundary
              |
              v
Read new/changed records
              |
              v
Process Bronze
              |
              v
Run Data Quality
              |
       +------+------+
       |             |
     Valid         Invalid
       |             |
       v             v
    Silver       Quarantine
       |
       v
Update watermark
       |
       v
Mark run successful
```

The watermark should only be advanced after the required processing stages have completed successfully.

This prevents a failed execution from incorrectly moving the processing boundary forward.

---

## 8. Handling New Records

New records will be identified using the configured incremental column.

For example:

```text
source_timestamp > previous_watermark
```

Only records satisfying the incremental condition will enter the processing flow.

The exact implementation will depend on the source system and its supported change-detection mechanism.

---

## 9. Handling Changed Records

For sources that support updates, the platform must identify records that have changed after the previous successful processing point.

Potential approaches include:

* Last modified timestamp
* Change Data Capture
* Change Tracking
* Source version number
* Hash-based change detection

Changed records will be processed according to the target Silver and Gold data model.

For update-capable datasets, the implementation must ensure that existing records are updated rather than creating unintended duplicates.

---

## 10. Idempotency

Incremental processing must be idempotent.

Running the same processing interval more than once should not create duplicate business records.

The platform will use the business key:

```text
order_id
```

as the primary record identifier for the `retail_orders` dataset.

Future implementation will use appropriate merge/upsert logic where updates are supported.

---

## 11. Failure Handling

If an incremental processing run fails:

1. The current run will be marked as failed.
2. The previous successful watermark will remain unchanged.
3. The failed interval will remain eligible for reprocessing.
4. After correction, the same interval can be processed again.
5. The watermark will advance only after successful completion.

This prevents data loss caused by prematurely advancing the processing boundary.

---

## 12. Late-Arriving Data

The target architecture must account for records that arrive later than their expected processing window.

Potential mechanisms include:

* Processing windows with overlap
* Event timestamps versus ingestion timestamps
* Reprocessing a configurable lookback period
* Change Data Capture
* MERGE-based reconciliation

The final approach will depend on the characteristics of the production source systems.

---

## 13. Duplicate Prevention

Incremental processing must prevent duplicate records caused by:

* Pipeline retries
* Reprocessing
* Overlapping incremental windows
* Duplicate source records
* Late-arriving records

The platform will use business-key validation and appropriate merge/upsert logic.

The existing Data Quality framework will continue to validate uniqueness before records enter the trusted Silver layer.

---

## 14. Raw Layer and Incremental Processing

The Raw layer will continue to preserve source data independently of downstream processing.

The target architecture is:

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
Incremental Processing
  |
  +----> Silver
  |
  +----> Quarantine
```

Keeping Raw data separate from incremental processing provides a recovery and replay capability.

If downstream processing fails, the source data does not need to be reacquired from the original system when the required Raw data is already available.

---

## 15. Bronze Incremental Processing

The Bronze layer will maintain the source representation required for downstream processing.

Future implementation may include:

* Ingestion timestamp
* Source file name
* Source system identifier
* Pipeline run ID
* Record ingestion metadata
* Source modification timestamp

These metadata fields will support lineage, troubleshooting, and incremental processing.

---

## 16. Silver Incremental Processing

The Silver layer will process only the records identified by the incremental processing boundary.

The processing flow will remain:

```text
Bronze
  |
  v
Standardization
  |
  v
Data Quality
  |
  +----> Quarantine
  |
  v
Silver
```

Silver processing will continue to enforce:

* Standard data types
* Valid dates
* Valid quantities
* Valid payment methods
* Valid order statuses
* Non-null customer identifiers
* Unique order identifiers
* Derived revenue calculation

---

## 17. Gold Incremental Processing

Gold datasets will be designed to support incremental updates where practical.

Potential Gold data products include:

* Sales performance
* Customer analytics
* Product performance
* Regional performance
* Revenue analysis
* Order metrics

The final incremental strategy will depend on the aggregation grain and business requirements of each Gold dataset.

For example, daily or regional aggregations may require recalculation when late-arriving or corrected Silver records affect previously calculated results.

---

## 18. Incremental Processing and Data Quality

Incremental processing will not bypass Data Quality validation.

Every newly received or changed record must pass through the established quality framework.

```text
New / Changed Records
        |
        v
Data Quality
    |       |
 Valid    Invalid
    |       |
    v       v
 Silver   Quarantine
```

This maintains the same quality gate regardless of whether records are processed for the first time or incrementally.

---

## 19. Reconciliation

Each incremental run will maintain reconciliation metrics.

Target metrics include:

```text
Records Read
      =
Valid Records
+
Quarantined Records
```

Additional reconciliation metrics may include:

* Records inserted
* Records updated
* Records rejected
* Records skipped
* Duplicate records detected
* Records reprocessed

These metrics will support operational monitoring and troubleshooting.

---

## 20. Performance Considerations

Incremental processing is intended to reduce unnecessary computation as data volume increases.

Expected benefits include:

* Reduced data scanned
* Reduced Spark processing
* Reduced shuffle volume
* Reduced execution time
* Lower compute consumption
* Faster downstream availability

The final implementation will be benchmarked against full-refresh processing.

Performance optimization techniques such as partitioning, predicate pushdown, optimized file layout, and appropriate Spark configuration may be introduced where justified by measured workload characteristics.

---

## 21. Security and Governance

Incremental processing will follow the same security and governance controls as the rest of the platform.

Controls will include:

* Unity Catalog governance
* Controlled access to source and target datasets
* Appropriate service identities
* Auditability of processing runs
* Data lineage
* Controlled access to operational metadata

Security implementation will be expanded as the platform moves toward production deployment.

---

## 22. Testing Strategy

The incremental processing implementation will require testing for:

### Functional Testing

* New records are processed
* Changed records are processed
* Unchanged records are not unnecessarily processed
* Watermark advances correctly
* Failed runs do not advance the watermark
* Retries do not create duplicates

### Data Quality Testing

* Invalid incremental records are quarantined
* Valid records reach Silver
* Existing DQ rules remain effective

### Recovery Testing

* Pipeline failure
* Pipeline restart
* Reprocessing of failed intervals
* Late-arriving records
* Duplicate source records

### Performance Testing

Compare:

* Full refresh processing
* Incremental processing

Metrics will include:

* Records processed
* Execution duration
* Compute consumption
* Data scanned
* Shuffle volume where applicable

---

## 23. Future Operational Metadata

A future control/metadata layer will maintain pipeline execution information.

Conceptually:

```text
Pipeline Run Metadata
        |
        +-- Pipeline
        +-- Source
        +-- Entity
        +-- Run ID
        +-- Watermark
        +-- Start Time
        +-- End Time
        +-- Status
        +-- Records Read
        +-- Records Written
        +-- Records Rejected
```

This metadata will later support monitoring, alerting, restartability, and operational reporting.

---

## 24. Current Implementation vs Target State

| Capability                        | Current State   | Target State                  |
| --------------------------------- | --------------- | ----------------------------- |
| Batch processing                  | Implemented     | Supported                     |
| Raw preservation                  | Implemented     | Supported                     |
| Bronze processing                 | Implemented     | Incremental                   |
| Data Quality                      | Implemented     | Incremental validation        |
| Silver                            | Implemented     | Incremental/upsert processing |
| Quarantine                        | Implemented     | Incremental quarantine        |
| Watermarking                      | Not implemented | Planned                       |
| Change detection                  | Not implemented | Planned                       |
| Idempotent incremental processing | Not implemented | Planned                       |
| Late-arriving data handling       | Not implemented | Planned                       |
| Operational metadata              | Not implemented | Planned                       |
| Incremental Gold processing       | Not implemented | Planned                       |
| Monitoring                        | Planned         | Planned                       |
| Automated alerting                | Planned         | Planned                       |

---

## 25. Design Decisions

### Decision 1 — Preserve Raw Data

Raw data will remain available independently of downstream incremental processing.

**Reason:** Supports replay, recovery, auditability, and troubleshooting.

### Decision 2 — Use Watermark-Based Processing

A watermark will be used where the source system provides a reliable incremental attribute.

**Reason:** Avoids repeatedly processing unchanged historical data.

### Decision 3 — Advance Watermark Only After Successful Processing

The watermark will only be updated after successful completion of the required processing stages.

**Reason:** Prevents data loss when a processing run fails.

### Decision 4 — Maintain Idempotency

Incremental processing must be safe to retry.

**Reason:** Production pipelines commonly experience retries and partial failures.

### Decision 5 — Keep Data Quality in the Incremental Path

Incremental records must pass through the same quality gates as batch records.

**Reason:** Incremental processing must not weaken the trusted-data contract.

---

## 26. Future Enhancements

Planned enhancements include:

* Implement watermark control tables
* Implement incremental ingestion
* Implement MERGE/upsert processing
* Add late-arriving data handling
* Add incremental Gold processing
* Add pipeline run metadata
* Add automated monitoring
* Add failure alerting
* Add restart/recovery automation
* Benchmark incremental versus full processing
* Add metadata-driven incremental configuration

---

## 27. Status

**Design Status:** Planned

**Implementation Status:** Not yet implemented

The current platform uses batch processing. Incremental processing will be implemented after the foundational data-quality and Silver processing capabilities are stabilized.

---

## 28. Change History

| Date       | Change                                        |
| ---------- | --------------------------------------------- |
| 2026-10-01 | Initial incremental processing design created |
