---
name: sql-migration-safety-review
description: Review SQL schema and data migrations before deployment, focusing on correctness, lock duration, rollback safety, and mixed-version compatibility.
---

# SQL Migration Safety Review

Review a migration as a change to both the database and every application
version that can overlap during rollout. Base findings on the supplied SQL and
the target database's documented behavior. Do not run the migration against a
production database.

## Establish the deployment context

Read the migration files, nearby application code, and repository deployment
notes before judging safety. Identify:

- database engine and version;
- table sizes, write rates, replication topology, and maintenance windows;
- whether the migration runs in a transaction and how the migrator records it;
- old and new application versions that may overlap;
- backup, rollback, and restore capabilities;
- existing data that must satisfy new constraints or keys.

If a missing fact could reverse a finding, state the assumption and the exact
check needed. Do not present a vendor-specific guarantee as universal.

## Review the migration

Walk through the migration in execution order and check each category:

1. **Correctness and idempotence**
   - Confirm that types, defaults, nullability, foreign keys, and indexes match
     application behavior.
   - Check whether rerunning after a partial failure is safe. Require guards or
     resumable batches where the migrator can retry without a transaction.
   - Check deterministic ordering for deduplication and backfills.

2. **Lock and rewrite risk**
   - Identify statements that scan or rewrite the full table, acquire an
     exclusive lock, block writes, or wait behind long transactions.
   - Separate metadata-only changes from operations that require table
     rebuilds or index validation.
   - Avoid claiming that a statement is online solely because of its syntax;
     verify the behavior for the stated engine and version.

3. **Mixed-version compatibility**
   - Use expand-and-contract ordering: add compatible structures first,
     deploy code that can tolerate both shapes, backfill, then remove the old
     shape in a later release.
   - Reject rename/drop/not-null tightening steps that can break the currently
     running version unless the deployment explicitly stops old writers.

4. **Data backfills**
   - Batch large updates by a stable key; commit between batches when that is
     compatible with migration semantics.
   - Make progress observable and restarts safe.
   - Bound locks and replication lag. Avoid one unbounded update of all rows.
   - Check concurrent writes during the backfill and define which value wins.

5. **Constraints and indexes**
   - Verify existing data before validating uniqueness, foreign keys, checks,
     or not-null constraints.
   - Prefer building large indexes through the target database's documented
     online or concurrent mechanism, then validating constraints separately.
   - Check whether creating the constraint also builds an index and whether a
     duplicate index remains.

6. **Rollback**
   - Provide a forward fix when a down migration would lose data.
   - Specify code, schema, and data rollback order.
   - Test rollback on a production-like copy, not only an empty database.
   - Note that a down migration is not a substitute for a backup.

## Report

Return a concise review with:

- overall verdict: block, deploy with controls, or no material issue found;
- findings ordered by severity, with file/line references;
- concrete evidence from the SQL or documented target behavior;
- a safe revised statement or deployment sequence;
- pre-deployment data checks and post-deployment verification queries;
- rollback limitations and unresolved assumptions.

Use these severity levels:

- **Blocker**: likely data loss, irreversible incompatibility, or an operation
  that can take the table unavailable without a bounded plan.
- **High**: can break running application versions, exceed the maintenance
  window, or fail after partial execution.
- **Medium**: correctness or operational risk under plausible data sizes or
  traffic.
- **Low**: maintainability, efficiency, or missing observability without a
  material near-term failure mode.

Do not mix observations with commands that modify production data. Read-only
queries are acceptable when the target environment and access are authorized.

## Example request

```text
Review migrations 0042 through 0045 for a 300 GB MySQL 8.0 table that receives
continuous writes. The old and new services will overlap for 30 minutes.
Return blockers, a safe ordering, validation queries, and rollback limits.
```

<!-- OB-S:MjWHCJ3eVtoCokpZbKdE5w -->
