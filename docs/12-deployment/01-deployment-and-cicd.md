# Deployment and CI/CD Strategy

**Document ID:** DEP-001
**Project:** Enterprise Retail Data Platform
**Version:** 1.1
**Status:** Initial full-load orchestration implemented; CI/CD automation planned
**Last Updated:** October 2026

---

## 1. Purpose

This document defines the deployment approach for the Enterprise Retail Data Platform built using Azure Data Factory, Azure Databricks, and Azure Data Lake Storage Gen2 (ADLS Gen2).

It describes the current deployment architecture, notebook execution strategy, pipeline orchestration, validation procedures, configuration management, release controls, and planned CI/CD improvements.

The document distinguishes implemented capabilities from future enhancements to maintain an accurate record of project maturity.

## 2. Deployment Objectives

The deployment strategy aims to:

* Deploy and execute data processing workloads reliably.
* Orchestrate Bronze, Silver, and Gold processing in the correct sequence.
* Use serverless compute where supported by the Databricks workspace.
* Preserve data quality and reconciliation controls during execution.
* Validate persisted Delta datasets after pipeline completion.
* Support repeatable full-load executions.
* Separate environment-specific configuration from processing logic where practical.
* Introduce automated deployment and release management in future iterations.

## 3. Technology Stack

| Component          | Technology                                         | Purpose                                                              |
| ------------------ | -------------------------------------------------- | -------------------------------------------------------------------- |
| Data orchestration | Azure Data Factory                                 | Coordinates execution of the Databricks job                          |
| Data processing    | Azure Databricks                                   | Executes Bronze, Silver, and Gold transformations                    |
| Compute            | Databricks Serverless                              | Executes notebook tasks without traditional clusters                 |
| Data lake          | Azure Data Lake Storage Gen2                       | Stores raw, Bronze, Silver, Quarantine, and Gold datasets            |
| Storage format     | Delta Lake                                         | Supports structured datasets and reliable writes                     |
| Source control     | GitHub                                             | Stores project documentation and source-controlled project artifacts |
| Authentication     | Databricks access token for the ADF linked service | Authenticates ADF to the Databricks workspace                        |

## 4. Deployment Architecture

The current deployment flow is:

```text
Azure Data Factory
        |
        v
PL_Retail_Full_Load_V1
        |
        v
Databricks Job
        |
        v
JOB_Retail_Full_Load_V1
        |
        +-----------------------------+
        |                             |
        v                             |
Bronze Ingestion                     |
        |                             |
        v                             |
Silver Transformation                |
        |                             |
        v                             |
Gold Aggregation                     |
        |                             |
        +-----------------------------+
                      |
                      v
             ADLS Gen2 Gold Layer
                      |
          +-----------+-----------+
          |                       |
          v                       v
   Regional Orders        Daily Regional Sales
```

The Bronze, Silver, and Gold tasks execute sequentially using task dependencies in the Databricks job.

Azure Data Factory triggers the Databricks job. Databricks executes the notebook tasks, and the resulting datasets are persisted in ADLS Gen2.

## 5. Azure Data Factory Configuration

### 5.1 Pipeline

**Pipeline name:** `PL_Retail_Full_Load_V1`

The pipeline is responsible for initiating the full-load Databricks job.

### 5.2 Linked Service

**Linked service name:** `LS_AzureDatabricks_Retail`

The linked service connects Azure Data Factory to the existing Azure Databricks workspace.

Current configuration characteristics:

* Authentication type: Access Token.
* Compute execution: Serverless-compatible Databricks job execution.
* Connection validation: Successful.
* Notebook execution: Validated through the Databricks job.

The access token is stored in the linked service configuration and must not be committed to GitHub, embedded in notebooks, or included in documentation.

For a more production-oriented implementation, secret storage and rotation should be managed through an appropriate secrets-management solution.

### 5.3 Pipeline Activity

The current pipeline uses a Databricks Job activity to trigger the configured Databricks job.

This approach was selected after the Databricks Notebook activity configuration required a non-serverless cluster, which was incompatible with the serverless-only workspace.

The Job activity invokes the existing Databricks job, where the three notebook tasks and their execution dependencies are defined.

## 6. Databricks Job Configuration

**Job name:** `JOB_Retail_Full_Load_V1`

The job contains three dependent notebook tasks.

| Execution order | Task                    | Responsibility                                                 |
| --------------: | ----------------------- | -------------------------------------------------------------- |
|               1 | `Bronze_Ingestion`      | Ingests source data into the Bronze layer                      |
|               2 | `Silver_Transformation` | Applies standardization, validation, and quarantine processing |
|               3 | `Gold_Aggregation`      | Produces Regional Orders and Daily Regional Sales              |

The Silver task depends on successful completion of Bronze. The Gold task depends on successful completion of Silver.

All three tasks have been executed successfully as part of the validated full-load orchestration.

## 7. Compute Strategy

The current Databricks workspace supports serverless compute.

The job uses serverless execution rather than a traditional cluster configuration.

The serverless approach avoids requiring the project to provision a conventional cluster with an explicit node type and worker count.

The compute configuration must remain compatible with the capabilities and policies of the target Databricks workspace.

Traditional job-cluster deployment is not part of the current implementation.

## 8. Data Storage Configuration

The platform uses ADLS Gen2 containers for the different data layers.

| Layer      | Container    | Dataset path                |
| ---------- | ------------ | --------------------------- |
| Raw        | `raw`        | Source landing data         |
| Bronze     | `bronze`     | Ingested source records     |
| Silver     | `silver`     | `orders`                    |
| Quarantine | `quarantine` | Invalid or rejected records |
| Gold       | `gold`       | `regional_orders`           |
| Gold       | `gold`       | `regional_sales`            |

The current Gold datasets are persisted as Delta data and can be read back successfully from their ADLS paths.

The implementation uses Unity Catalog external-location-backed access to the storage paths.

Storage credentials, access tokens, account keys, and other secrets must not be committed to source control.

## 9. Full-Load Execution Procedure

The current deployment supports a full-load execution through Azure Data Factory.

The operational sequence is:

1. Confirm that the ADF linked service is configured and its connection test succeeds.
2. Open `PL_Retail_Full_Load_V1`.
3. Trigger the pipeline using Debug or the applicable execution option.
4. Confirm that the Databricks Job activity succeeds.
5. Open the Databricks job run details.
6. Verify successful completion of Bronze, Silver, and Gold in the expected order.
7. Review notebook outputs and available data-quality validation results.
8. Read the persisted Gold datasets from ADLS Gen2.
9. Verify the expected row counts and check for duplicate business-grain records where applicable.
10. Record any failures and investigate them before declaring the run successful.

A successful orchestration run does not eliminate the need for data-level validation.

## 10. Deployment Validation

### 10.1 Orchestration Validation

The initial ADF full-load orchestration has been executed successfully.

Validated outcomes:

* The ADF linked service connection succeeds.
* ADF successfully triggers the Databricks job.
* Bronze ingestion completes successfully.
* Silver transformation completes successfully.
* Gold aggregation completes successfully.
* The task dependencies execute in the required order.

### 10.2 Gold Dataset Validation

The persisted Gold datasets have been read back successfully from ADLS Gen2.

| Dataset              | Expected row count | Validation status |
| -------------------- | -----------------: | ----------------- |
| Regional Orders      |                  5 | Passed            |
| Daily Regional Sales |                150 | Passed            |

Regional Orders contains one record for each of the five regions.

Daily Regional Sales contains 150 records at the intended `order_day + region` grain, representing 30 days across five regions.

These results validate the current test dataset and configuration. They do not establish production-scale performance or availability guarantees.

### 10.3 Data Quality and Reconciliation

The Silver transformation and Gold processing have also been validated using record reconciliation, duplicate checks, null checks, unit reconciliation, and financial metric reconciliation.

The current source dataset contains 100,000 input records, with 99,055 records in Silver and 945 records in Quarantine.

The reconciliation is:

`99,055 + 945 = 100,000`

Gold order counts, units, and financial metrics have been reconciled against the applicable Silver records.

## 11. GitHub and Source Control

GitHub is used to maintain the project's documentation and source-controlled artifacts.

The repository contains architecture, business requirements, data design, ingestion, transformation, data-quality, governance, monitoring, performance, testing, deployment, disaster-recovery, and architecture-decision documentation.

The current implementation has been developed and tested in the Azure and Databricks environments.

Finalized notebooks and supporting deployment artifacts should be added to the repository in a structured manner after they have been reviewed and sanitized.

The repository must not contain:

* Access tokens or passwords.
* Storage account keys.
* Client secrets.
* Unredacted connection strings.
* Other confidential credentials or environment-specific secrets.

## 12. Environment Configuration

The current implementation has been validated in the configured development environment.

A future multi-environment deployment should separate environment-specific settings, including:

* Azure subscriptions and resource identifiers.
* ADLS storage account and container paths.
* Databricks workspace and job identifiers.
* ADF linked services.
* Secret references.
* Environment-specific access permissions.
* Pipeline parameters and execution settings.

Development, test, and production environments should not share credentials unnecessarily.

Environment promotion should use controlled configuration changes and validation gates.

## 13. Release and Change Management

Future releases should follow a controlled process:

1. Make the change in a development environment.
2. Review notebook code and configuration changes.
3. Run relevant unit, data-quality, and reconciliation checks.
4. Execute the full-load pipeline in the development environment.
5. Validate persisted outputs.
6. Review documentation and implementation status.
7. Commit the approved changes to GitHub.
8. Deploy through the applicable release mechanism.
9. Validate the target environment after deployment.

For production-oriented use, changes affecting business metrics, schema, data quality, access permissions, or storage paths should receive explicit review.

## 14. CI/CD Strategy

Automated CI/CD is a planned enhancement and is not yet implemented.

Potential future capabilities include:

* Automated validation of source-controlled notebooks and configuration.
* Automated testing during pull requests.
* Deployment of Databricks notebooks and job definitions.
* Deployment of Azure Data Factory pipeline artifacts.
* Environment-specific configuration injection.
* Automated release approval gates.
* Post-deployment smoke tests.
* Deployment audit records and rollback procedures.

The eventual implementation may use GitHub Actions, Azure DevOps, or another supported deployment mechanism.

The final choice should account for Azure resource deployment, Databricks job deployment, identity management, secret handling, and environment promotion.

## 15. Failure Handling and Recovery

The current full-load pipeline can be inspected through Azure Data Factory monitoring and Databricks job run details.

When a task fails:

1. Identify the first failed activity or task.
2. Inspect its error message and execution logs.
3. Determine whether the failure is related to authentication, compute, permissions, source data, transformation logic, or storage access.
4. Correct the underlying issue.
5. Rerun the appropriate workload.
6. Validate the resulting datasets before considering the recovery complete.

Automated notifications, retry policies beyond the current configuration, failure-routing workflows, and formal recovery automation remain future enhancements unless explicitly implemented and tested.

## 16. Security Considerations

Deployment and orchestration must follow least-privilege principles.

Security requirements include:

* Restricting access to the Databricks workspace and ADF resources.
* Restricting ADLS access through appropriate Azure permissions and Unity Catalog controls.
* Keeping credentials outside source control.
* Rotating credentials according to the applicable security policy.
* Reviewing permissions when access requirements change.
* Avoiding unnecessary broad access to storage and processing resources.
* Maintaining auditable configuration and deployment changes.

A comprehensive production identity strategy and automated secret rotation are planned enhancements.

## 17. Current Implementation Status

| Capability                               | Status                                |
| ---------------------------------------- | ------------------------------------- |
| ADF-to-Databricks linked service         | Implemented and connection-tested     |
| Serverless-compatible Databricks job     | Implemented                           |
| Bronze notebook task                     | Implemented and executed successfully |
| Silver notebook task                     | Implemented and executed successfully |
| Gold notebook task                       | Implemented and executed successfully |
| Sequential task dependencies             | Implemented and tested                |
| ADF full-load pipeline                   | Implemented and tested                |
| Regional Orders Gold output              | Persisted and read-back validated     |
| Daily Regional Sales Gold output         | Persisted and read-back validated     |
| Data-quality and reconciliation checks   | Implemented for the current pipeline  |
| GitHub documentation                     | Implemented                           |
| Automated CI/CD                          | Planned                               |
| Automated deployment across environments | Planned                               |
| Automated alerting and notifications     | Planned                               |
| Incremental processing orchestration     | Planned                               |
| Automated rollback                       | Planned                               |
| Production release approval workflow     | Planned                               |

## 18. Future Enhancements

The following capabilities are planned for future iterations:

* Incremental ingestion and transformation.
* Watermark management and restart-safe processing.
* Automated CI/CD for ADF and Databricks artifacts.
* Environment-specific deployment configuration.
* Automated regression and integration testing.
* Monitoring dashboards and automated alerting.
* Enhanced failure recovery and retry handling.
* Formal deployment approvals and rollback procedures.
* Production-grade secret management and rotation.
* Automated post-deployment validation.

Each capability will be marked as implemented only after its implementation and validation are complete.

## 19. Change History

| Version | Date           | Change                                                                                                                                                      |
| ------- | -------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1.0     | September 2026 | Initial deployment and CI/CD strategy documentation                                                                                                         |
| 1.1     | October 2026   | Documented successful ADF full-load orchestration, Databricks serverless job execution, sequential task dependencies, and Gold dataset read-back validation |

