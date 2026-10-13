# Testing and Validation Strategy

**Document Version:** 1.1
**Project:** Enterprise Retail Data Platform
**Processing Engine:** Azure Databricks, Apache Spark, and Delta Lake
**Storage:** Azure Data Lake Storage Gen2

## 1. Purpose

This document describes the testing and validation performed for the Enterprise Retail Data Platform across ingestion, transformation, data quality, incremental processing, and Gold aggregation.

The testing approach emphasizes record-level correctness, data quality, reconciliation between layers, duplicate prevention, and repeatable execution.

Testing evidence is distinguished from capabilities that remain planned for production.

## 2. Testing Scope

The project testing scope includes:

* Source ingestion and row-count reconciliation.
* Silver transformation and schema validation.
* Data quality checks and quarantine processing.
* Business-rule validation for Gold aggregates.
* Incremental insert and update processing.
* Watermark and batch audit validation.
* Idempotency testing.
* Gold-to-Silver metric reconciliation.
* Pipeline execution and operational verification.

Production-scale performance, failure recovery, concurrency, and automated alerting require further testing.

## 3. Baseline Full-Load Validation

The original source dataset contained 100,000 records.

| Validation                    | Expected / Observed Result |
| ----------------------------- | -------------------------: |
| Raw input records             |                    100,000 |
| Silver records                |                     99,055 |
| Quarantine records            |                        945 |
| Silver plus quarantine        |                    100,000 |
| Duplicate order IDs in Silver |                          0 |
| Invalid quantities in Silver  |                          0 |
| Null customer IDs in Silver   |                          0 |

The reconciliation confirms:

`Silver records + Quarantine records = Raw input records`

The observed counts reconcile to 100,000 records.

Additional Silver checks included validation of order statuses and required fields. Payment-method validation must follow the project's explicit normalization and accepted-value rules.

## 4. Gold Business-Rule Validation

Gold sales calculations use completed orders for financial metrics.

The implemented calculations are:

* `gross_sales = quantity × unit_price`
* `net_sales = gross_sales − discount_amount + tax_amount + shipping_cost`
* `average_order_value = net_sales ÷ completed_orders`

The daily regional aggregate is grouped by order date and region. Validation includes record counts, duplicate grouping keys, and reconciliation of business metrics against the relevant Silver dataset.

The original full-load Gold results were reconciled against the completed Silver orders during baseline implementation.

## 5. Incremental Processing Test

### 5.1 Test setup

A controlled simulated batch was persisted at:

`abfss://raw@stdeproject2it482.dfs.core.windows.net/incremental_test/batch_001`

The batch contained eight records:

* Three updates to existing orders.
* Five new orders.

The isolated Silver test target was:

`abfss://silver@stdeproject2it482.dfs.core.windows.net/orders_incremental_test`

The isolated Gold test aggregate was:

`abfss://gold@stdeproject2it482.dfs.core.windows.net/regional_sales_incremental_test`

The baseline Silver, original Gold outputs, and quarantine data were kept separate from these test writes.

### 5.2 Source and target validation

| Test                           | Result |
| ------------------------------ | ------ |
| Eligible records selected      | 8      |
| Duplicate source order IDs     | 0      |
| Existing records updated       | 3      |
| New records inserted           | 5      |
| Final Silver test rows         | 99,060 |
| Distinct Silver test order IDs | 99,060 |
| Duplicate target order IDs     | 0      |

The record-level checks confirmed that the three intended existing orders were updated and all five new order IDs were present.

### 5.3 Gold reconciliation

The daily regional Gold test aggregate was rebuilt from the updated Silver test target.

| Metric           |           Silver |             Gold | Result |
| ---------------- | ---------------: | ---------------: | ------ |
| Completed orders |           24,766 |           24,766 | Pass   |
| Completed units  |          111,517 |          111,517 | Pass   |
| Net sales        | 1,560,798,861.00 | 1,560,798,861.00 | Pass   |

The Gold test output contained 153 daily regional groups, with no duplicate `(order_day, region)` keys.

All three reconciled metrics matched.

### 5.4 Idempotency

The same simulated batch was merged a second time.

| Check                 | Before rerun | After rerun |
| --------------------- | -----------: | ----------: |
| Silver test row count |       99,060 |      99,060 |
| Distinct order IDs    |       99,060 |      99,060 |
| Duplicate order IDs   |            0 |           0 |

Completed orders, completed units, and net sales also remained unchanged.

**Result:** Idempotency passed for the tested batch and merge conditions.

This test does not prove correctness for every concurrent execution, late-arriving record, equal-timestamp conflict, or failure-recovery scenario.

## 6. Watermark and Audit Validation

The incremental test control record was initialized with the baseline Silver maximum update timestamp.

| Attribute            | Value                     |
| -------------------- | ------------------------- |
| Initial watermark    | `2026-10-01 23:22:00`     |
| Batch ID             | `retail_orders_batch_001` |
| Eligible rows        | 8                         |
| Batch end watermark  | `2026-10-12 12:00:00`     |
| Final control status | `SUCCEEDED`               |
| Final audit status   | `SUCCEEDED`               |

The audit record was created with status `STARTED`. After the MERGE, Gold reconciliation, and idempotency checks passed, the control record and audit record were updated to `SUCCEEDED`.

The final watermark was verified as `2026-10-12 12:00:00`.

The control and audit tables use separate Delta transactions. Recovery from partial updates between these tables remains a production-hardening requirement.

## 7. Orchestration Validation

The V1 full-load pipeline was executed through the configured Azure Data Factory and Azure Databricks Job integration.

The Bronze, Silver, and Gold tasks completed successfully during the recorded full-load test.

The controlled incremental-processing test was validated separately in Databricks. Automated incremental orchestration through ADF has not yet been established by this test.

## 8. Test Results Summary

| Area                                             | Status                     | Evidence                                     |
| ------------------------------------------------ | -------------------------- | -------------------------------------------- |
| Full-load row reconciliation                     | Passed                     | Raw, Silver, and quarantine counts           |
| Silver data quality                              | Passed for recorded checks | Null, quantity, status, and duplicate checks |
| Gold business calculations                       | Passed                     | Baseline metric reconciliation               |
| Incremental source uniqueness                    | Passed                     | 8 eligible records, no duplicate source IDs  |
| Incremental MERGE                                | Passed                     | 3 updates and 5 inserts                      |
| Target uniqueness                                | Passed                     | 99,060 distinct IDs, no duplicates           |
| Gold reconciliation after incremental processing | Passed                     | Orders, units, and net sales match           |
| Incremental idempotency                          | Passed for test batch      | Rerun did not change target metrics          |
| Watermark and audit completion                   | Passed for test batch      | Both final statuses are `SUCCEEDED`          |
| Automated incremental orchestration              | Not yet validated          | Future work                                  |
| Concurrent-run and recovery scenarios            | Not yet validated          | Future work                                  |
| Production-scale performance                     | Not yet validated          | Future work                                  |

## 9. Outstanding Testing Requirements

Before production deployment, add repeatable tests for:

1. Empty batches and batches containing only already-processed records.
2. Duplicate source keys with conflicting values.
3. Equal timestamps and late-arriving updates.
4. Failure before and after the Silver MERGE.
5. Failure between control-table and audit-table updates.
6. Safe retries after partial failure.
7. Concurrent pipeline executions.
8. Source deletions, if supported by the source contract.
9. Schema evolution and malformed input.
10. Large-volume performance and resource consumption.
11. Automated alerts and operational recovery.
12. End-to-end incremental orchestration.

## 10. Conclusion

The full-load implementation has baseline validation evidence, and the controlled incremental test successfully demonstrated timestamp-based eligibility, Delta MERGE behavior, duplicate prevention, Gold reconciliation, idempotency, audit completion, and watermark advancement.

**The incremental test is validated; production incremental processing remains in progress.**
