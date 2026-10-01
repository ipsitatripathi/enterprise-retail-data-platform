
# Data Dictionary

**Document ID:** DD-001
**Project:** Enterprise Retail Data Platform
**Version:** 1.0
**Status:** Active
**Last Updated:** September 2026

---

## 1. Document Purpose

This document defines the data dictionary for the Enterprise Retail Data Platform.

It describes the structure, meaning, data type, business purpose, and data quality expectations of the retail transactional dataset processed by the platform.

The data dictionary provides a common reference for:

* Data engineers
* Analytics engineers
* Data analysts
* BI developers
* Business stakeholders
* QA engineers
* Platform and governance teams

The document also establishes the expected data contract between ingestion, transformation, data quality, and downstream analytical layers.

---

# 2. Dataset Overview

The primary dataset represents retail order transactions.

### Dataset Name

```text
retail_orders
```

### Business Domain

```text
Retail / E-commerce
```

### Dataset Grain

> One record represents one retail order transaction.

### Primary Business Key

```text
order_id
```

The `order_id` is expected to uniquely identify a transaction.

Duplicate `order_id` values are treated as a data quality issue.

---

# 3. Source-to-Target Data Flow

The dataset follows the following logical path:

```text
Source Data
     |
     v
Raw
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
     |
     v
Gold
(Planned)
```

The same logical business dataset may have different representations at different layers.

---

# 4. Column-Level Data Dictionary

## 4.1 `order_id`

| Attribute                    | Definition                           |
| ---------------------------- | ------------------------------------ |
| Column Name                  | `order_id`                           |
| Business Meaning             | Unique identifier for a retail order |
| Data Type                    | Integer                              |
| Nullable                     | No                                   |
| Business Key                 | Yes                                  |
| Used for Duplicate Detection | Yes                                  |
| Silver Status                | Validated                            |
| Gold Status                  | Planned                              |

### Data Quality Rules

* Must be present.
* Must contain a valid integer value.
* Duplicate values must be identified.
* A valid Silver record must have a unique order ID.

---

## 4.2 `customer_id`

| Attribute        | Definition                                                     |
| ---------------- | -------------------------------------------------------------- |
| Column Name      | `customer_id`                                                  |
| Business Meaning | Identifier representing the customer associated with the order |
| Data Type        | String                                                         |
| Nullable         | No                                                             |
| Business Key     | No                                                             |
| Silver Status    | Validated                                                      |
| Gold Status      | Planned                                                        |

### Data Quality Rules

* Must not be null for valid Silver records.
* Should follow the expected source-system identifier format.

---

## 4.3 `order_date`

| Attribute        | Definition                         |
| ---------------- | ---------------------------------- |
| Column Name      | `order_date`                       |
| Business Meaning | Date on which the order was placed |
| Data Type        | Date                               |
| Nullable         | No                                 |
| Business Key     | No                                 |
| Silver Status    | Standardized                       |
| Gold Status      | Planned                            |

### Data Quality Rules

* Must be convertible to a valid date.
* Invalid date values should not enter the trusted Silver dataset.
* Date format should be standardized during transformation.

---

## 4.4 `product`

| Attribute        | Definition                               |
| ---------------- | ---------------------------------------- |
| Column Name      | `product`                                |
| Business Meaning | Product associated with the retail order |
| Data Type        | String                                   |
| Nullable         | No                                       |
| Business Key     | No                                       |
| Silver Status    | Validated                                |
| Gold Status      | Planned                                  |

### Data Quality Rules

* Must contain a valid product value.
* Blank or unusable values should be investigated.
* Product naming should remain consistent across records.

---

## 4.5 `category`

| Attribute        | Definition                                     |
| ---------------- | ---------------------------------------------- |
| Column Name      | `category`                                     |
| Business Meaning | Business category to which the product belongs |
| Data Type        | String                                         |
| Nullable         | No                                             |
| Business Key     | No                                             |
| Silver Status    | Validated                                      |
| Gold Status      | Planned                                        |

### Data Quality Rules

* Must contain a valid category.
* Category values should conform to the expected business domain.

---

## 4.6 `region`

| Attribute        | Definition                                           |
| ---------------- | ---------------------------------------------------- |
| Column Name      | `region`                                             |
| Business Meaning | Geographic/business region associated with the order |
| Data Type        | String                                               |
| Nullable         | No                                                   |
| Business Key     | No                                                   |
| Silver Status    | Validated                                            |
| Gold Status      | Planned                                              |

### Data Quality Rules

* Must contain a valid region.
* Region values should follow the approved business vocabulary.

---

## 4.7 `quantity`

| Attribute        | Definition                             |
| ---------------- | -------------------------------------- |
| Column Name      | `quantity`                             |
| Business Meaning | Number of units purchased in the order |
| Data Type        | Integer                                |
| Nullable         | No                                     |
| Business Key     | No                                     |
| Silver Status    | Validated                              |
| Gold Status      | Planned                                |

### Data Quality Rules

* Must contain a valid numeric value.
* Quantity must be greater than zero.
* Invalid quantities must not enter the Silver dataset.

---

## 4.8 `unit_price`

| Attribute        | Definition                                 |
| ---------------- | ------------------------------------------ |
| Column Name      | `unit_price`                               |
| Business Meaning | Price of one unit of the purchased product |
| Data Type        | Numeric                                    |
| Nullable         | No                                         |
| Business Key     | No                                         |
| Silver Status    | Validated                                  |
| Gold Status      | Planned                                    |

### Data Quality Rules

* Must contain a valid numeric value.
* Should not contain invalid negative values unless explicitly supported by a business rule.
* Must be suitable for revenue calculations.

---

# 5. Derived Columns

The platform may create derived attributes during transformation.

A key derived metric is:

```text
revenue = quantity × unit_price
```

### Revenue

| Attribute        | Definition                              |
| ---------------- | --------------------------------------- |
| Column Name      | `revenue`                               |
| Business Meaning | Total monetary value of the transaction |
| Derivation       | `quantity * unit_price`                 |
| Data Type        | Numeric                                 |
| Source Columns   | `quantity`, `unit_price`                |
| Silver Status    | Derived                                 |
| Gold Status      | Planned                                 |

The revenue field should be calculated consistently rather than independently populated by downstream consumers.

---

# 6. Data Type Standards

The platform follows the following general type standards:

| Business Attribute  | Standard Type |
| ------------------- | ------------- |
| Order identifier    | Integer       |
| Customer identifier | String        |
| Transaction date    | Date          |
| Product             | String        |
| Category            | String        |
| Region              | String        |
| Quantity            | Integer       |
| Unit price          | Numeric       |
| Revenue             | Numeric       |

Source systems may provide these values in different physical formats. Transformation logic is responsible for converting them into the standardized platform representation.

---

# 7. Nullability Rules

The following fields are required for trusted Silver records:

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

A null value in a mandatory field should cause the record to fail the relevant validation rule.

The current Silver validation confirmed:

```text
Null customer IDs in Silver = 0
```

---

# 8. Key and Uniqueness Rules

## Primary Business Key

```text
order_id
```

The order ID is expected to uniquely identify a transaction.

### Duplicate Rule

```text
COUNT(order_id) > 1
```

indicates a duplicate business key.

The current Silver dataset contains:

```text
Duplicate order IDs = 0
```

---

# 9. Data Quality Rules Summary

| Rule                 | Column(s)                 | Expected Result     |
| -------------------- | ------------------------- | ------------------- |
| Order ID required    | `order_id`                | No null             |
| Order ID uniqueness  | `order_id`                | No duplicates       |
| Customer ID required | `customer_id`             | No null             |
| Valid date           | `order_date`              | Valid date          |
| Valid product        | `product`                 | Valid value         |
| Valid category       | `category`                | Valid value         |
| Valid region         | `region`                  | Valid value         |
| Positive quantity    | `quantity`                | Greater than zero   |
| Valid payment method | Payment-related attribute | Approved values     |
| Valid order status   | Order status attribute    | Approved values     |
| Valid unit price     | `unit_price`              | Valid numeric value |

> Note: Payment method and order status are part of the broader enterprise data-quality contract and are validated in the current pipeline where those source attributes are present.

---

# 10. Silver Data Contract

The Silver layer represents trusted retail transaction data.

A record should enter Silver only after passing the applicable validation rules.

The conceptual contract is:

```text
Silver Record
    |
    +-- Valid order_id
    +-- Valid customer_id
    +-- Valid order_date
    +-- Valid product
    +-- Valid category
    +-- Valid region
    +-- Valid quantity
    +-- Valid unit_price
    +-- No duplicate business key
```

Records failing applicable validation rules are routed to Quarantine.

---

# 11. Quarantine Data Contract

Quarantine contains records that fail one or more quality checks.

The target quarantine design should retain enough information to support investigation.

Recommended attributes include:

```text
Original Record
Validation Rule
Failure Reason
Processing Timestamp
Source Identifier
Pipeline/Batch Identifier
```

The exact metadata structure may evolve as the monitoring and operational framework is implemented.

---

# 12. Data Volume

The current enterprise dataset contains:

```text
Total Input Records : 100,000
Silver Records      : 99,055
Quarantine Records  : 945
```

Reconciliation:

```text
99,055 + 945 = 100,000
```

This demonstrates complete accounting of the processed records between the trusted and quarantined outcomes.

---

# 13. Data Quality Results

Current Silver validation results:

| Validation                        |  Result |
| --------------------------------- | ------: |
| Total Input Records               | 100,000 |
| Silver Records                    |  99,055 |
| Quarantine Records                |     945 |
| Invalid Quantities in Silver      |       0 |
| Null Customer IDs in Silver       |       0 |
| Invalid Payment Methods in Silver |       0 |
| Invalid Order Statuses in Silver  |       0 |
| Duplicate Order IDs in Silver     |       0 |

These values represent the current implementation state and may change when the source dataset or processing logic changes.

---

# 14. Data Classification

The dataset should be classified according to organizational data-governance policies.

At the current project stage, classification is defined conceptually as:

| Data Element | Classification Consideration     |
| ------------ | -------------------------------- |
| Order ID     | Business identifier              |
| Customer ID  | Potentially sensitive identifier |
| Product      | Business data                    |
| Category     | Business data                    |
| Region       | Business data                    |
| Quantity     | Business data                    |
| Unit Price   | Business data                    |
| Revenue      | Business/financial data          |

Final classification, masking, encryption, and access requirements should be aligned with organizational governance policies.

---

# 15. Schema Evolution

The platform should account for changes in source schemas.

Potential changes include:

* New columns
* Removed columns
* Data type changes
* Renamed columns
* New business values
* Structural changes

Schema changes should be evaluated before being propagated into trusted downstream datasets.

### Current Status

Basic schema handling is implemented.

Automated schema evolution management is **planned**.

---

# 16. Naming Conventions

The following naming principles are used:

* Lowercase column names
* `snake_case` naming
* Descriptive names
* Avoid unnecessary abbreviations
* Consistent naming across layers
* Business-friendly names for analytical datasets

Examples:

```text
order_id
customer_id
order_date
unit_price
revenue
```

---

# 17. Gold Layer Data Products

The Gold layer is planned to provide business-oriented datasets.

Potential future data products include:

### Sales Performance

Possible metrics:

* Total revenue
* Order count
* Units sold
* Average order value

### Product Performance

Possible dimensions:

* Product
* Category
* Region
* Date

### Customer Analytics

Possible metrics:

* Customer order count
* Customer revenue
* Average customer order value

### Regional Analytics

Possible dimensions:

* Region
* Date
* Product category

These datasets will be formally defined when the Gold layer is implemented.

---

# 18. Data Dictionary Ownership

| Area                      | Responsibility                               |
| ------------------------- | -------------------------------------------- |
| Source schema             | Data engineering                             |
| Transformation schema     | Data engineering                             |
| Data quality rules        | Data engineering + business stakeholders     |
| Business definitions      | Business/data owners                         |
| Gold metrics              | Business/data owners + analytics engineering |
| Governance classification | Data governance                              |
| Access control            | Platform/security team                       |

For this portfolio implementation, these responsibilities represent the intended enterprise operating model.

---

# 19. Current Implementation Status

| Capability                   | Status      |
| ---------------------------- | ----------- |
| Source schema definition     | Implemented |
| Bronze schema                | Implemented |
| Silver schema                | Implemented |
| Data type standardization    | Implemented |
| Duplicate validation         | Implemented |
| Null validation              | Implemented |
| Quantity validation          | Implemented |
| Payment method validation    | Implemented |
| Order status validation      | Implemented |
| Quarantine handling          | Implemented |
| Revenue derivation           | Implemented |
| Gold data products           | Planned     |
| Automated schema evolution   | Planned     |
| Advanced data classification | Planned     |

---

# 20. Change History

| Version | Date           | Change                          |
| ------- | -------------- | ------------------------------- |
| 1.0     | September 2026 | Initial data dictionary created |
