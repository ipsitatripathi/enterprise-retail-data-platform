
# Data Quality Framework

**Document ID:** DQ-001
**Project:** Enterprise Retail Data Platform
**Version:** 1.0
**Status:** Active
**Last Updated:** September 2026

---

# 1. Document Purpose

This document defines the Data Quality (DQ) framework for the Enterprise Retail Data Platform.

The framework establishes how incoming retail transaction data is validated before being promoted into the trusted Silver layer.

It also defines how records that fail validation are isolated in the Quarantine layer and how processing completeness is verified through reconciliation.

The objective is to ensure that downstream consumers receive reliable and validated data while invalid records remain available for investigation and remediation.

---

# 2. Data Quality Objectives

The Data Quality framework is designed to:

* Identify invalid records before Silver consumption.
* Prevent poor-quality records from contaminating trusted datasets.
* Detect missing mandatory values.
* Detect invalid business values.
* Detect duplicate business keys.
* Validate numeric business rules.
* Separate valid and invalid records.
* Preserve failed records for investigation.
* Provide record-level reconciliation.
* Produce measurable quality metrics.
* Provide a foundation for automated monitoring.

---

# 3. Data Quality Architecture

The current DQ architecture is:

```text
Bronze
   |
   v
Data Quality Validation
   |
   +-------------------------+
   |                         |
   v                         v
Valid Records           Invalid Records
   |                         |
   v                         v
Silver                  Quarantine
```

The Data Quality layer acts as a controlled gate between Bronze and Silver.

Only records that satisfy the applicable validation rules are promoted to Silver.

---

# 4. Data Quality Scope

The current framework focuses primarily on the retail transaction dataset.

The principal business dataset is:

```text id="q7q9u1"
retail_orders
```

Expected grain:

> One record represents one retail order transaction.

The current dataset contains:

```text id="0m1dtr"
100,000 records
```

---

# 5. Data Quality Dimensions

The framework considers the following standard data quality dimensions.

| Dimension    | Description                                            | Current Status        |
| ------------ | ------------------------------------------------------ | --------------------- |
| Completeness | Required values are present                            | Implemented           |
| Validity     | Values conform to defined rules                        | Implemented           |
| Uniqueness   | Business keys are not duplicated                       | Implemented           |
| Consistency  | Related values follow expected standards               | Partially implemented |
| Accuracy     | Values correctly represent the source/business reality | Source-dependent      |
| Timeliness   | Data arrives within expected time                      | Planned               |
| Integrity    | Relationships and dependencies remain valid            | Planned               |

Not every dimension can be fully validated using the current source dataset.

---

# 6. Data Quality Rule Structure

Each quality rule can be represented conceptually as:

```text id="s2mtib"
Rule ID
Rule Name
Dataset
Column(s)
Validation Condition
Severity
Failure Action
```

Example:

```text id="j0c6q8"
Rule ID       : DQ-001
Rule Name     : Customer ID Not Null
Dataset       : retail_orders
Column        : customer_id
Condition     : customer_id IS NOT NULL
Severity      : Error
Failure Action: Quarantine
```

This structure provides a foundation for expanding the framework into a metadata-driven DQ system later.

---

# 7. Current Data Quality Rules

## DQ-001 — Customer ID Not Null

### Objective

Ensure every trusted Silver transaction has a customer identifier.

### Rule

```text id="2tq7w6"
customer_id IS NOT NULL
```

### Failure Action

Route the record to Quarantine.

### Current Result

```text id="8b7n0p"
Null customer IDs in Silver = 0
```

---

## DQ-002 — Valid Quantity

### Objective

Ensure transaction quantity represents a valid purchase quantity.

### Rule

```text id="v6ev45"
quantity > 0
```

### Failure Action

Route the record to Quarantine.

### Current Result

```text id="zz7t5q"
Invalid quantities in Silver = 0
```

---

## DQ-003 — Unique Order ID

### Objective

Ensure the business key is not duplicated in trusted Silver data.

### Rule

```text id="j7y7fv"
order_id must be unique
```

### Failure Action

Identify duplicate records and prevent invalid duplicates from being treated as trusted Silver records.

### Current Result

```text id="w1q6so"
Duplicate order IDs in Silver = 0
```

---

## DQ-004 — Valid Payment Method

### Objective

Ensure payment method values conform to the accepted business vocabulary.

### Rule

Payment method must belong to the approved set of values.

### Failure Action

Route invalid records to Quarantine.

### Current Result

```text id="l5t4x9"
Invalid payment methods in Silver = 0
```

---

## DQ-005 — Valid Order Status

### Objective

Ensure order status contains an accepted business value.

### Rule

Order status must belong to the approved set of values.

### Failure Action

Route invalid records to Quarantine.

### Current Result

```text id="2j6t6m"
Invalid order statuses in Silver = 0
```

---

## DQ-006 — Valid Order Date

### Objective

Ensure order dates can be represented as valid dates.

### Rule

The source value must successfully convert to the target date type.

### Failure Action

Invalid records should be excluded from Silver and routed to Quarantine.

### Current Status

Date standardization is implemented.

Formalized date-quality metadata and monitoring are planned.

---

# 8. Data Quality Rule Matrix

| Rule ID | Rule                 | Column           | Severity | Failure Action | Status         |
| ------- | -------------------- | ---------------- | -------- | -------------- | -------------- |
| DQ-001  | Customer ID not null | `customer_id`    | Error    | Quarantine     | Implemented    |
| DQ-002  | Quantity > 0         | `quantity`       | Error    | Quarantine     | Implemented    |
| DQ-003  | Order ID unique      | `order_id`       | Error    | Quarantine     | Implemented    |
| DQ-004  | Valid payment method | `payment_method` | Error    | Quarantine     | Implemented    |
| DQ-005  | Valid order status   | `order_status`   | Error    | Quarantine     | Implemented    |
| DQ-006  | Valid order date     | `order_date`     | Error    | Quarantine     | Implemented    |
| DQ-007  | Valid product        | `product`        | Error    | Quarantine     | Planned/Expand |
| DQ-008  | Valid category       | `category`       | Error    | Quarantine     | Planned/Expand |
| DQ-009  | Valid region         | `region`         | Error    | Quarantine     | Planned/Expand |
| DQ-010  | Valid unit price     | `unit_price`     | Error    | Quarantine     | Planned/Expand |

---

# 9. Severity Model

The framework uses a simple severity model.

## Error

A critical quality failure that prevents the record from entering Silver.

Examples:

* Null mandatory identifier
* Invalid quantity
* Invalid business status
* Duplicate business key

### Action

```text id="f5y1jp"
Quarantine
```

---

## Warning

A quality issue that may not prevent processing but should be monitored.

Examples may include:

* Unexpected but permissible business values
* Non-critical formatting differences

### Action

```text id="t0q2fz"
Process + Monitor
```

Warning-level rules are planned for future expansion.

---

# 10. Quarantine Architecture

Records failing an Error-level rule are routed to Quarantine.

```text id="zpw7qk"
Bronze
   |
   v
Validate
   |
   +---- PASS ----> Silver
   |
   +---- FAIL ----> Quarantine
```

Quarantine provides an isolation boundary between untrusted and trusted data.

---

# 11. Quarantine Information

A production-oriented quarantine record should retain:

| Attribute            | Purpose                         |
| -------------------- | ------------------------------- |
| Original record      | Preserve failed data            |
| Rule ID              | Identify failed validation      |
| Failure reason       | Explain failure                 |
| Processing timestamp | Establish when failure occurred |
| Source identifier    | Trace source                    |
| Batch ID             | Trace processing batch          |

The current implementation provides the quarantine dataset.

Additional operational metadata will be introduced as the monitoring framework is implemented.

---

# 12. Multiple Rule Failures

A single record may violate multiple quality rules.

For example:

```text id="f7ujj7"
customer_id = NULL
quantity = -5
order_status = INVALID
```

The record should remain outside Silver.

A mature DQ framework should retain all applicable failure reasons rather than reporting only the first failure encountered.

This capability can be expanded when the DQ framework becomes metadata-driven.

---

# 13. Silver Quality Gate

Silver acts as the trusted quality boundary.

The conceptual condition is:

```text id="4m4j9a"
Silver = Records that pass required validation rules
```

This means downstream consumers do not need to independently perform the same basic validation before using Silver transactional data.

---

# 14. Record-Level Reconciliation

Reconciliation is a critical control.

The expected relationship is:

```text id="i7v0s4"
Input Records
=
Silver Records
+
Quarantine Records
```

Current execution:

```text id="d7mjw8"
100,000
=
99,055
+
945
```

Therefore:

```text id="w7wh9n"
Reconciliation = PASSED
```

This confirms that the processing flow accounted for all input records.

---

# 15. Current Data Quality Results

The latest validation results are:

| Quality Metric                    |  Result |
| --------------------------------- | ------: |
| Total input records               | 100,000 |
| Silver records                    |  99,055 |
| Quarantine records                |     945 |
| Invalid quantities in Silver      |       0 |
| Null customer IDs in Silver       |       0 |
| Invalid payment methods in Silver |       0 |
| Invalid order statuses in Silver  |       0 |
| Duplicate order IDs in Silver     |       0 |

These metrics describe the current implementation run and should not be treated as permanent production statistics.

---

# 16. Quality Score

A future DQ framework may calculate a quality score based on the proportion of records passing defined rules.

For example:

```text id="kgyk29"
Quality Pass Rate =
Valid Records / Total Input Records × 100
```

For the current processing run:

```text id="hgl0yx"
99,055 / 100,000 × 100
```

The resulting percentage can be used as an operational metric.

However, a single quality score should not replace individual rule-level metrics because different quality failures can have different business impacts.

### Current Status

**Planned**

---

# 17. Data Quality Metrics

The target framework should monitor metrics such as:

### Volume Metrics

* Total records received
* Total valid records
* Total invalid records
* Total quarantined records

### Rule Metrics

* Failure count by rule
* Failure rate by rule
* Failure count by column

### Processing Metrics

* Reconciliation status
* Processing duration
* Batch processing status

### Trend Metrics

* Daily quality failure rate
* Failure rate by source
* Failure rate by business domain
* Recurring quality failures

Advanced automated metrics are planned.

---

# 18. Data Quality Monitoring

The target monitoring architecture should detect:

* Sudden increase in invalid records
* Unexpected reduction in input volume
* Unexpected increase in duplicates
* Increase in null values
* Rule failure spikes
* Reconciliation failures
* Pipeline failures

These events can eventually generate alerts for data engineering and operations teams.

### Current Status

**Planned**

---

# 19. Data Quality Failure Handling

The current handling model is:

```text id="4h9y8v"
Quality Check
     |
     +---- PASS ----> Silver
     |
     +---- FAIL ----> Quarantine
```

The pipeline does not intentionally discard failed records.

This supports:

* Investigation
* Root-cause analysis
* Data correction
* Reprocessing
* Auditability

---

# 20. Data Quality Reprocessing

A future remediation workflow may follow:

```text id="g3ly2c"
Quarantine
    |
    v
Investigate Failure
    |
    v
Correct Source / Data
    |
    v
Reprocess
    |
    v
Data Quality Validation
    |
    +---- PASS ----> Silver
    |
    +---- FAIL ----> Quarantine
```

This creates a controlled path for recovering records after data-quality problems are corrected.

### Current Status

Basic quarantine processing is implemented.

Automated remediation/reprocessing is **planned**.

---

# 21. Data Quality Ownership

| Responsibility                    | Owner                            |
| --------------------------------- | -------------------------------- |
| Define technical validation rules | Data Engineering                 |
| Define business validation rules  | Business/Data Owners             |
| Implement DQ rules                | Data Engineering                 |
| Investigate failures              | Data Engineering + Source Owners |
| Approve business values           | Business/Data Owners             |
| Monitor quality trends            | Data Engineering / Operations    |
| Governance and classification     | Data Governance                  |

For this portfolio project, these represent the intended enterprise operating model.

---

# 22. DQ Framework Evolution

The current implementation uses explicit validation logic.

The target architecture can evolve toward a metadata-driven framework.

### Current

```text
Pipeline Code
     |
     v
Validation Rules
```

### Future

```text
DQ Rule Metadata
       |
       v
Generic DQ Engine
       |
       v
Validation Results
       |
       +---- Pass ----> Silver
       |
       +---- Fail ----> Quarantine
```

A metadata-driven design would make it easier to add or modify rules without duplicating validation logic across multiple pipelines.

---

# 23. Data Quality Testing

Current quality testing includes:

* Null checks
* Duplicate checks
* Business-value validation
* Quantity validation
* Record reconciliation
* Silver output validation
* Quarantine count validation

Future automated testing may include:

* Unit tests
* Rule-level automated tests
* Regression tests
* Boundary-value tests
* Negative test cases
* Schema tests
* Pipeline integration tests

---

# 24. Performance Considerations

Data quality checks must be designed carefully for large datasets.

Potential performance considerations include:

* Avoiding unnecessary scans
* Combining compatible validation expressions
* Reducing repeated transformations
* Minimizing shuffles
* Using efficient Spark DataFrame operations
* Applying filters early where appropriate
* Avoiding unnecessary data movement

For very large datasets, DQ rules should be evaluated as part of the overall Spark execution plan rather than treated as independent row-by-row operations.

---

# 25. Governance Considerations

Data quality is part of the overall data governance framework.

Important governance capabilities include:

* Data ownership
* Data classification
* Quality expectations
* Quality metrics
* Data lineage
* Auditability
* Access control
* Retention policies

Unity Catalog provides the governance foundation for the platform.

Advanced governance implementation is planned.

---

# 26. Current Implementation Status

| Capability                      | Status      |
| ------------------------------- | ----------- |
| DQ validation layer             | Implemented |
| Null validation                 | Implemented |
| Duplicate validation            | Implemented |
| Quantity validation             | Implemented |
| Payment method validation       | Implemented |
| Order status validation         | Implemented |
| Date validation/standardization | Implemented |
| Silver quality gate             | Implemented |
| Quarantine routing              | Implemented |
| Record reconciliation           | Implemented |
| Rule-level metadata framework   | Planned     |
| Automated DQ metrics            | Planned     |
| DQ monitoring                   | Planned     |
| Alerting                        | Planned     |
| Automated remediation           | Planned     |
| DQ trend analysis               | Planned     |

---

# 27. Key Data Quality Decisions

## Decision 1 — Treat Silver as a quality boundary

Only validated records are promoted into Silver.

**Reason:**
This provides downstream consumers with a trusted transactional dataset.

## Decision 2 — Quarantine instead of silent deletion

Failed records are preserved separately.

**Reason:**
Data quality failures may be recoverable and should remain traceable.

## Decision 3 — Reconcile every processing batch

Input, Silver, and Quarantine counts are compared.

**Reason:**
This provides a basic control against unexpected record loss.

## Decision 4 — Track rule-level results

Individual validation results are more useful than a single aggregate quality number.

**Reason:**
Different quality failures can require different remediation actions.

---

# 28. Future Enhancements

The Data Quality framework will evolve to include:

* Metadata-driven validation
* Rule versioning
* Automated quality metrics
* DQ dashboards
* Alerting
* Quality trend analysis
* Automated remediation workflows
* Data quality SLAs
* Source-level quality scorecards
* Integration with pipeline monitoring
* Historical DQ reporting

Each capability will be marked as implemented only after actual development and validation.

---

# 29. Change History

| Version | Date           | Change                                       |
| ------- | -------------- | -------------------------------------------- |
| 1.0     | September 2026 | Initial data quality framework documentation |
