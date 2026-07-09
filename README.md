# ADR-example

# ADR-012: Adopt the Expand and Contract Pattern for Database Deployments

**Status:** Accepted
**Date:** 2026-07-09

## Context

Historically, database deployments have been one of the more challenging aspects of application releases. Changes that required both schema modifications and updates to existing data have often been performed manually to minimise deployment risk. While this approach has generally resulted in successful individual deployments, it has also introduced a number of long-term issues.

Manual deployment activities have resulted in databases across environments drifting into different states, making it difficult to confidently reproduce deployments or verify that each environment has been updated consistently. Data transformations required by evolving business rules have not been versioned alongside application code, and deployment success has often relied on manual execution and validation of SQL scripts.

In addition, combining long-running data updates with schema changes increases deployment complexity and makes failures more difficult to diagnose and recover from.

As the application continues to evolve, database deployments should become repeatable, version-controlled and automated. Schema evolution and data transformation should be treated as separate concerns while remaining part of a single deployment process.

To achieve this, we will adopt the Expand and Contract pattern, using EF Core migrations for schema evolution and versioned data deployments executed during application startup.

## Decision

We will adopt the **Expand and Contract** pattern for all database deployments.

The deployment process will consist of three phases.

### 1. Expand

Additive database schema changes will be implemented using EF Core migrations. These changes may include:

* New tables
* New columns
* New indexes
* Nullable fields
* Additional constraints that do not break compatibility

The expanded schema must remain compatible with both the current and newly deployed versions of the application.

### 2. Data Processing

Where existing data must be transformed to support new business rules, the transformation will occur separately from the schema migration.

Data deployments will be executed by an ASP.NET Core Worker Service that runs once during application startup as part of the deployment process. The worker service will execute any pending data deployments before completing and will not continue running afterwards.

This separates potentially long-running business data updates from schema evolution while allowing both to remain version-controlled alongside the application.

### 3. Contract

Once all application code has been updated to use the new schema and all required data deployments have completed successfully, obsolete schema elements may be removed through subsequent EF Core migrations.

Examples include:

* Dropping deprecated columns
* Removing legacy tables
* Removing obsolete indexes or constraints

Destructive schema changes should only occur after the application no longer depends on the previous schema.

## Consequences

### Benefits

* Provides a repeatable and version-controlled deployment process.
* Reduces reliance on manual deployment activities.
* Minimises environment drift between deployments.
* Separates schema evolution from business data transformation.
* Supports lower-risk deployments by introducing schema changes incrementally.
* Improves traceability of both schema and data changes.
* Simplifies future maintenance by keeping EF Core migrations focused on schema evolution.

### Trade-offs

* Database objects may temporarily exist beyond their immediate use until the Contract phase is completed.
* Additional deployment infrastructure is required to execute and track data deployments.
* Data deployment failures must be monitored and remediated before deployment can complete.
* During rollout, the application may need to support both old and new data representations until all deployments have completed.

## Implementation Notes

### Schema Changes

* All schema changes will be implemented using EF Core migrations.
* Expand migrations should be additive and backwards compatible.
* Contract migrations should only remove schema elements that are no longer referenced by the application.

### Data Deployments

Business data transformations will be implemented as versioned data deployment classes.

An ASP.NET Core Worker Service will execute once during application startup and:

* Discover pending data deployments.
* Execute them sequentially.
* Record successful execution.
* Exit once all pending deployments have completed.

Each data deployment will have a unique identifier and descriptive name.

A dedicated database table (for example, `DataDeploymentHistory`) will be used to record:

* Deployment identifier
* Deployment description
* Execution timestamp
* Execution duration
* Execution status
* Failure details where applicable

Before executing a deployment, the worker service will consult this table to determine whether the deployment has already completed successfully.

Data deployments must be idempotent so they can be safely re-executed if deployment is interrupted or retried.

### Failure Handling

The worker service will produce structured logging for all deployment activity, including:

* Deployment start and completion
* Number of records processed
* Execution duration
* Warnings
* Errors and exceptions

If a data deployment fails:

* The deployment will not be marked as successful.
* Application startup will fail, preventing the application from running against an unsupported database state.
* Detailed logs will be available to assist diagnosis.

Where automated recovery is not appropriate, corrective SQL scripts may be developed to remediate existing data before the deployment is executed again.

### Operational Assumptions

The application is currently deployed as a single application instance.

As only one instance of the application performs deployments, no distributed coordination mechanisms (such as distributed locks or leader election) are required.

Should the deployment architecture change to support multiple concurrent application instances, this approach should be reviewed to ensure data deployments continue to execute safely.
