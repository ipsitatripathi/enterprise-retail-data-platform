
# ADR-002: Data Quality and Quarantine Strategy

## 1. Status

**Accepted**

## 2. Date

2026-09-30

## 3. Decision

The platform will validate data before it enters the trusted Silver layer.

Records that satisfy the defined data quality rules will be written to **Silver**, while records that fail one or more critical validation rules will be routed to a dedicated **Quarantine** dataset.

This establishes a controlled quality boundary between Bronze and Silver.

---

## 4. Context

Retail source data can contain incomplete, invalid, duplicated, or inconsistent records.

Examples include:

* Missing customer identifiers
* Invalid quantities
* Duplicate order IDs
* Invalid payment methods
* Invalid order statuses
* Invalid dates
* Invalid or malformed business attributes

Allowing these records directly into the trusted Silver layer could result in unreliable downstream analytics and business reporting.

The platform therefore requires an explicit mechanism to:

1. Detect invalid records.
2. Prevent invalid records from contaminating Silver.
3. Preserve rejected records for investigation.
4. Reconcile accepted and rejected records against the original input.
5. Support future remediation and reprocessing.

---

## 5. Considered Alternatives

### Alternative 1 — Allow all records into Silver

Rejected.

This would simplify ingestion but would allow poor-quality data into the trusted layer and shift data-quality problems to downstream consumers.

### Alternative 2 — Fail the entire pipeline when any invalid record exists

Rejected for the current use case.

A single bad record should not necessarily prevent thousands of valid records from being processed.

### Alternative 3 — Silently discard invalid records

Rejected.

Discarding records would create data loss and make investigation and reconciliation difficult.

### Alternative 4 — Route invalid records to Quarantine

**Selected.**

Valid records continue through the pipeline while invalid records are preserved separately for investigation and future remediation.

---

## 6. Quality Boundary

The current logical flow is:

```text
Raw
  ↓
Bronze
  ↓
Data Quality Validation
  ├── Valid Records ──→ Silver
  │
  └── Invalid Records → Quarantine
```

Silver therefore represents the trusted transactional layer rather than an unvalidated copy of Bronze.

---

## 7. Current Data Quality Rules

The current implementation validates rules including:

| Rule   | Validation                           |
| ------ | ------------------------------------ |
| DQ-001 | `customer_id` must not be null       |
| DQ-002 | `quantity` must be greater than zero |
| DQ-003 | `order_id` must be unique            |
| DQ-004 | `payment_method` must be valid       |
| DQ-005 | `order_status` must be valid         |
| DQ-006 | `order_date` must be valid           |

Additional validation rules for attributes such as product, category, region, and unit price are part of the framework and can be expanded as the platform evolves.

---

## 8. Quarantine Strategy

A record is routed to Quarantine when it fails one or more applicable critical validation rules.

The Quarantine layer is intended to preserve:

* Original invalid record
* Validation failure information
* Applicable rule identifier(s)
* Processing context where available
* Future remediation status

The objective is to make rejected data traceable rather than silently removing it.

---

## 9. Reconciliation

The platform performs record-count reconciliation between the input population and the resulting Silver and Quarantine populations.

Current validation result:

```text
Total Input Records : 100,000
Silver Records      : 99,055
Quarantine Records  :    945
--------------------------------
Reconciled Total     : 100,000
```

Therefore:

```text
Silver + Quarantine = Total Input
99,055 + 945        = 100,000
```

The reconciliation check passed.

Additional Silver validation results:

```text
Invalid Quantities       : 0
Null Customer IDs        : 0
Invalid Payment Methods  : 0
Invalid Order Statuses   : 0
Duplicate Order IDs      : 0
```

---

## 10. Benefits of the Decision

This approach provides:

* Protection of the trusted Silver layer
* Preservation of invalid records
* Improved data traceability
* Easier investigation of source-data problems
* Record-level reconciliation
* Support for future remediation and reprocessing
* Better downstream data reliability
* Separation between data ingestion and data quality enforcement

---

## 11. Consequences

### Positive Consequences

* Silver consumers receive validated data.
* Invalid records remain available for investigation.
* Data-quality failures become measurable.
* The pipeline can process valid records without necessarily failing because of individual bad records.
* Reconciliation provides evidence that records were not silently lost.

### Trade-offs

* Additional storage is required for Quarantine.
* Data-quality logic adds processing overhead.
* Quarantined records require operational monitoring.
* A future remediation and reprocessing process will be required for production use.

---

## 12. Current Implementation Status

| Capability                | Status      |
| ------------------------- | ----------- |
| Bronze ingestion          | Implemented |
| Data quality validation   | Implemented |
| Silver routing            | Implemented |
| Quarantine routing        | Implemented |
| Record reconciliation     | Implemented |
| Basic validation evidence | Implemented |
| Automated DQ monitoring   | Planned     |
| DQ alerting               | Planned     |
| Metadata-driven DQ rules  | Planned     |
| Automated remediation     | Planned     |
| Automated reprocessing    | Planned     |
| Production DQ dashboards  | Planned     |

---

## 13. Future Evolution

The current implementation can evolve toward a more enterprise-grade framework with:

* Metadata-driven validation rules
* Configurable rule severity
* Automated DQ metrics
* Quality scoring
* DQ trend monitoring
* Automated alerts
* Quarantine aging monitoring
* Remediation workflows
* Reprocessing of corrected records
* Data-quality dashboards
* Integration with operational monitoring

---

## 14. Decision Summary

The platform adopts a **validate-and-quarantine** approach.

The key principle is:

> **Only validated records enter Silver; invalid records are preserved separately for investigation and remediation.**

This provides a controlled quality boundary while avoiding unnecessary loss of source records.
