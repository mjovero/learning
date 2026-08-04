
## Known issues and recommended improvements

### Priority 1 — address before relying on unattended production recovery

#### 1. Gold refreshes are not atomic

Gold operations use `TRUNCATE TABLE` followed by `INSERT`. The procedural `BEGIN ... END` block is not a transaction. An insert failure after truncation can leave the production reporting table empty.

**Recommendation:** build a replacement table and atomically swap/replace it, or use a supported BigQuery transactional pattern with rollback protection. Add target snapshots before high-risk releases.

#### 2. Legacy payroll incremental action can duplicate data

`ecpr_payroll_legacy` is incremental but has no unique key or incremental-only filter. Every rerun can append the full configured source date range again.

**Recommendation:** convert it to a controlled table rebuild, add a true MERGE key and incremental predicate, or disable it after the one-time load.

#### 3. Silver and Gold target bootstrap is incomplete

Silver create actions are not dependencies of upserts, and Gold targets have no create actions. A new environment can compile successfully but fail at runtime because target tables do not exist.

**Recommendation:** create a complete bootstrap graph, add dependencies where safe, and provision every Gold target schema as code.

#### 4. Audit count assertions can produce false passes

The count helper:

- references `bq_etl_audit` without project/dataset qualification;
- does not check `load_status`;
- does not fail when no row exists;
- can evaluate to unknown when values are null;
- labels itself source-target count checking without comparing source and target.

Failure audit rows containing zeros can satisfy the arithmetic check.

**Recommendation:** fully qualify the audit table, require a successful latest run, explicitly detect missing/failure rows, and use `COALESCE`/clear semantics.

#### 5. Assertion failure does not roll back data changes

Assertions depend on upserts and Gold loads, so invalid data may already be committed when an assertion fails.

**Recommendation:** move critical validation into pre-MERGE/pre-replace scripts, quarantine invalid rows, or stage and publish only after all validations pass.

### Priority 2 — reliability and correctness

#### 6. Persistent Silver staging tables are concurrency-sensitive

Each upsert uses `CREATE OR REPLACE TABLE <entity>_upsert`. Concurrent runs can replace another run's stage table.

**Recommendation:** prevent overlapping executions or use temporary/run-ID-specific staging tables.

#### 7. Watermark logic can miss sufficiently late data

Only rows at or after `last_success - 30 minutes` are staged. A source record arriving more than 30 minutes late may never be considered in normal runs.

**Recommendation:** base the overlap on measured source lateness, add periodic wider reconciliation/backfill, and track source event time separately from ingestion time.

#### 8. Delete events are not implemented

`event_type` is stored, but MERGE scripts perform only insert/update. Source deletions will remain in Silver unless handled elsewhere.

**Recommendation:** define delete semantics per entity: physical delete, soft-delete flag, tombstone, or retained historical record.

#### 9. Silver uniqueness assertions do not consistently match MERGE keys

Most Silver MERGEs use `sys_id`, but uniqueness assertions use `sys_id + publish_time + sys_updated_on`. Duplicate `sys_id` rows with different timestamps could pass the assertion. Payroll uses another assertion key rather than the actual `sys_id + employee_row_key` merge key.

**Recommendation:** assert the exact target grain/merge key.

#### 10. Business metadata keys are duplicated

`letf_column_definitions.js` contains repeated flat keys such as `state`, `name`, `description`, `email`, `type`, `craft`, and others. JavaScript retains only the last definition, so table documentation can be inaccurate.

**Recommendation:** namespace definitions by table and add a test that rejects duplicate property names.

#### 11. Combined payroll uses `UNION ALL` without cross-source precedence

If legacy and NRT ranges overlap, both records are retained.

**Recommendation:** add source markers and an explicit deduplication/precedence policy.

### Priority 3 — maintainability and observability

#### 12. Audit and validator code is duplicated or unused

`audit.js` and `assertion_validators.js` are unused, while many SQLX operations duplicate long audit blocks.

**Recommendation:** standardize on shared generators/macros or remove unused files.

#### 13. Assertion output lacks diagnostic details

Null and uniqueness helpers generally return only a validation label, not the offending keys/columns.

**Recommendation:** return actionable key values and failure reasons while avoiding exposure of sensitive data.

#### 14. Cloud Build compiles but does not deploy or execute

The build can succeed without proving BigQuery objects execute correctly.

**Recommendation:** add controlled integration execution in a test project and make the production execution mechanism explicit in documentation.

#### 15. Hard-coded deployment values

Region and Cloud Build service account are embedded in YAML. The environment helper supports only three one-character suffixes.

**Recommendation:** parameterize these values and validate substitutions.

#### 16. Sensitive data governance is not visible in the repository

Payroll and contact models include SSN, tax identifiers, addresses, and payment-related fields, but the repository does not visibly apply BigQuery policy tags or column-level access policies.

**Recommendation:** document the external controls or manage policy tags/access as code. Confirm Gold models expose only fields required for their purpose.

#### 17. No automated tests or lint configuration

The repository has no test runner, formatting rules, or SQL lint configuration.

**Recommendation:** add unit tests for includes, compile checks for all environments, SQL formatting/linting, and BigQuery integration tests.

---