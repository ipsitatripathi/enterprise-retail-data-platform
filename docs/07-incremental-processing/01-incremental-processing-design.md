# Incremental Processing Design

**Document Version:** 1.1
**Implementation Status:** Controlled incremental-processing test completed
**Processing Engine:** Azure Databricks with Apache Spark and Delta Lake
**Storage:** Azure Data Lake Storage Gen2 (ADLS Gen2)

## 1. Purpose

This document describes the incremental-processing approach implemented and validated in the Enterprise Retail Data Platform.

The objective is to process new and updated order records without duplicating existing orders, maintain batch execution history, validate downstream Gold aggregates, and advance the processing watermark only after successful validation.

The current implementation is a controlled test using a simulated incremental batch. It is not yet the production incremental pipeline.

## 2. Incremental Processing Architecture

The tested workflow consists of the following stages:

1. Read the last successful watermark from the control table.
2. Read the persisted incremental test batch from the landing path.
3. Select eligible records whose `last_updated_timestamp` is greater than the stored watermark.
4. Validate source record uniqueness and required data quality conditions.
5. Record the batch as `STARTED` in the audit table.
6. Merge the eligible records into an isolated Silver Delta test target.
7. Rebuild the daily regional Gold test aggregate.
8. Reconcile Gold business metrics against Silver.
9. Rerun the same batch to validate idempotency.
10. Mark the batch `SUCCEEDED` and advance the control watermark after the final validation gates pass.

## 3. Change Detection and Watermark Strategy

The implementation uses `last_updated_timestamp` as the candidate change-tracking column.

### Initial watermark

The baseline Silver dataset had a maximum `last_updated_timestamp` of:

`2026-10-01 23:22:00`

This value was used to initialize the incremental test control record.

### Record eligibility

A record is eligible for processing when:

`last_updated_timestamp > last_successful_watermark`

The strict greater-than comparison was used in the controlled test.

### Watermark advancement

The watermark is advanced to the maximum `last_updated_timestamp` processed in the validated batch.

For the successful test batch:

* Previous watermark: `2026-10-01 23:22:00`
* New watermark: `2026-10-12 12:00:00`

The watermark was advanced only after the Silver MERGE, Gold reconciliation, and idempotency checks passed.

### Production considerations

A strict timestamp watermark alone does not guarantee reliable change capture. Production implementation must address:

* Late-arriving records with timestamps older than the current watermark.
* Multiple records sharing the same timestamp.
* Source timestamp reliability and consistency.
* Source-system deletes, if applicable.
* Overlapping or concurrent pipeline executions.
* Recovery when processing succeeds but control or audit updates fail.
* A source-supported change tracking mechanism, CDC feed, or an appropriately designed lookback and deduplication strategy.

The current source is a static CSV snapshot. Its timestamp column has been tested for parsing and filtering, but a reliable ongoing source change feed has not yet been established.

## 4. Incremental MERGE Strategy

The test implementation uses Delta Lake `MERGE` against the isolated Silver test target.

### Match condition

Records are matched using `order_id`.

### Update rule

An existing target record is updated only when the incoming record has a newer `last_updated_timestamp`.

### Insert rule

An incoming record is inserted when its `order_id` does not exist in the target.

### Duplicate protection

The source batch is checked for duplicate order IDs before the MERGE. Target uniqueness is validated after processing.

This approach was validated against a repeated execution of the same simulated batch. Production use also requires an explicit strategy for conflicting source records and duplicate keys across overlapping batches.

## 5. Control Table

**Test path:**

`abfss://silver@stdeproject2it482.dfs.core.windows.net/incremental_control_test`

The control table contains one record for the test pipeline.

| Column                      | Purpose                                       |
| --------------------------- | --------------------------------------------- |
| `pipeline_name`             | Identifies the processing pipeline            |
| `last_successful_watermark` | Tracks the last committed processing boundary |
| `last_batch_id`             | Identifies the last successful batch          |
| `last_batch_status`         | Stores the latest pipeline batch status       |
| `updated_at`                | Records the control update timestamp          |

### Successful test state

| Attribute            | Value                            |
| -------------------- | -------------------------------- |
| Pipeline             | `retail_orders_incremental_test` |
| Last batch ID        | `retail_orders_batch_001`        |
| Batch status         | `SUCCEEDED`                      |
| Successful watermark | `2026-10-12 12:00:00`            |

The control table is currently a test artifact, not a production control framework.

## 6. Batch Audit Table

**Test path:**

`abfss://silver@stdeproject2it482.dfs.core.windows.net/incremental_batch_audit_test`

The audit table records batch execution metadata.

| Column                  | Purpose                                  |
| ----------------------- | ---------------------------------------- |
| `batch_id`              | Unique identifier for the batch          |
| `pipeline_name`         | Pipeline identifier                      |
| `batch_start_watermark` | Watermark before processing              |
| `batch_end_watermark`   | Maximum eligible source update timestamp |
| `eligible_row_count`    | Number of eligible records               |
| `batch_status`          | Execution status                         |
| `started_at`            | Batch start timestamp                    |
| `completed_at`          | Batch completion timestamp               |
| `error_message`         | Error details when applicable            |

The test batch progressed from `STARTED` to `SUCCEEDED`.

Because the control and audit records are stored in separate Delta transactions, their updates are not atomic as a unit. Production implementation must include recovery and reconciliation logic for partial failures and retries.

## 7. Simulated Test Batch

The persisted test batch contained eight records:

* Three updates to existing order IDs: `1000004`, `1000007`, and `1000009`.
* Five new order IDs: `1100000` through `1100004`.

The test batch was stored at:

`abfss://raw@stdeproject2it482.dfs.core.windows.net/incremental_test/batch_001`

The isolated Silver test target was stored at:

`abfss://silver@stdeproject2it482.dfs.core.windows.net/orders_incremental_test`

The original Silver dataset and original Gold outputs were not overwritten by this test.

## 8. Gold Recalculation and Business Rules

The test recalculated the daily regional Gold aggregate from the updated Silver test dataset.

The aggregate is grouped by:

* `order_day`
* `region`

Only orders with `order_status = 'COMPLETED'` contribute to the completed-order, completed-unit, and financial metrics.

The calculations are:

* `gross_sales = quantity × unit_price`
* `net_sales = gross_sales − discount_amount + tax_amount + shipping_cost`
* `average_order_value = net_sales ÷ completed_orders`

The test aggregate was written to:

`abfss://gold@stdeproject2it482.dfs.core.windows.net/regional_sales_incremental_test`

This test currently rebuilds the complete test aggregate. Incremental recomputation of only affected dates and regions remains future work.

## 9. Validation and Test Results

The following checks passed during the controlled test.

| Validation                             | Result      |
| -------------------------------------- | ----------- |
| Eligible records selected              | 8           |
| Duplicate source order IDs             | 0           |
| Existing records updated               | 3           |
| New records inserted                   | 5           |
| Final Silver test row count            | 99,060      |
| Distinct Silver test order IDs         | 99,060      |
| Duplicate target order IDs             | 0           |
| Gold daily regional aggregate rows     | 153         |
| Duplicate `(order_day, region)` groups | 0           |
| Idempotency test                       | Passed      |
| Batch audit status                     | `SUCCEEDED` |
| Control status                         | `SUCCEEDED` |

### Gold reconciliation

| Metric           |           Silver |             Gold |
| ---------------- | ---------------: | ---------------: |
| Completed orders |           24,766 |           24,766 |
| Completed units  |          111,517 |          111,517 |
| Net sales        | 1,560,798,861.00 | 1,560,798,861.00 |

All three business metrics matched between the updated Silver test dataset and the persisted Gold test aggregate.

### Idempotency validation

The same batch was processed a second time.

The final Silver test target remained at 99,060 rows and 99,060 distinct order IDs, with zero duplicate order IDs. Completed orders, completed units, and net sales remained unchanged.

These results demonstrate idempotency for the tested batch and merge conditions; they do not establish correctness for every possible late-arriving, conflicting, or concurrent batch.

## 10. Security and Data Isolation

The incremental test uses separate landing, Silver, Gold, control, and audit paths to isolate its outputs from the baseline full-load pipeline.

The implementation uses the configured Databricks and ADLS access mechanisms. Storage account keys and ad hoc Spark authentication configuration are not part of this design.

Production deployment must apply appropriate access controls, secret management, audit retention, and least-privilege permissions to the control and audit data.

## 11. Known Limitations

The following capabilities are not yet implemented or fully validated:

* Automated incremental orchestration through Azure Data Factory.
* Reliable source CDC or change-feed integration.
* Handling of late-arriving and equal-timestamp records.
* Production-grade concurrent-run protection.
* Complete failure recovery and atomic batch-state management.
* Automated retry and alerting.
* Incremental Gold recomputation by affected date and region.
* Production-scale performance testing.
* Automated deployment and environment-specific configuration.
* Production monitoring and operational runbooks.

## 12. Next Steps

1. Define the source change-capture contract.
2. Harden watermark selection and late-arriving record handling.
3. Add reliable batch recovery, concurrency control, and failure-state management.
4. Recompute only affected Gold aggregates where appropriate.
5. Integrate the incremental workflow into orchestration.
6. Add automated monitoring, alerting, and operational recovery procedures.
7. Add repeatable tests for retries, failures, duplicate source keys, and watermark boundaries.

## 13. Current Status

The controlled incremental-processing test is complete. The tested MERGE, audit lifecycle, watermark advancement, Gold reconciliation, and idempotency checks passed.

**Production incremental processing remains in progress.**
