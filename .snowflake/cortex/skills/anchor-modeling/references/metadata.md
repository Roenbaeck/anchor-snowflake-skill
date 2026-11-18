# Metadata Column

When metadata is enabled, `Metadata_…` columns can link recorded rows to the batch that inserted them. The patterns below assume the model has appropriate Metadata columns.

## Purpose

The Metadata column connects every row to the **batch** that loaded it: which run of the task graph, when, from which source, and whether it passed integrity checks. This makes it possible to:

- **Trace** any row back to the load that created it.
- **Identify** the rows a bad batch inserted, for recovery after a dependency check.
- **Audit** what changed between two loads by comparing Metadata values.

## Batch tracking infrastructure

Create these objects once, in the same database as the model:

```sql
CREATE SEQUENCE IF NOT EXISTS {db}.{sch}.LOAD_BATCH_SEQ START 1 INCREMENT 1;

-- One batch per graph run group, including retries.
CREATE TABLE IF NOT EXISTS {db}.{sch}.LOAD_BATCH_LOG (
    BATCH_ID             INT            NOT NULL,
    GRAPH_RUN_GROUP_ID   VARCHAR        NOT NULL,
    STARTED_AT           TIMESTAMP_NTZ  NOT NULL,
    EXECUTION_STATUS     VARCHAR        NOT NULL,  -- RUNNING, SUCCEEDED, FAILED, etc.
    INTEGRITY_STATUS     VARCHAR        NOT NULL,  -- PENDING, PASSED, VIOLATIONS, CHECK_FAILED
    PUBLICATION_STATUS   VARCHAR        NOT NULL,  -- UNPUBLISHED, PUBLISHED, ROLLED_BACK
    INTEGRITY_VIOLATIONS INT
);
```

## How the task graph uses it

### Root task

Use `OVERLAP_POLICY = NO_OVERLAP` on the root and serialize other writers to the same target. The root gets the [runtime graph run group ID](https://docs.snowflake.com/en/sql-reference/functions/system_task_runtime_info) and assigns a batch ID once. Keep the same batch ID on a retry of the same graph run group.

```sql
-- In LOAD_ROOT:
LET run_group VARCHAR := (
    SELECT SYSTEM$TASK_RUNTIME_INFO('CURRENT_TASK_GRAPH_RUN_GROUP_ID')
);
INSERT INTO {db}.{sch}.LOAD_BATCH_LOG
    (BATCH_ID, GRAPH_RUN_GROUP_ID, STARTED_AT, EXECUTION_STATUS, INTEGRITY_STATUS, PUBLICATION_STATUS)
SELECT {db}.{sch}.LOAD_BATCH_SEQ.nextval, :run_group, SYSDATE(), 'RUNNING', 'PENDING', 'UNPUBLISHED'
WHERE NOT EXISTS (
    SELECT 1 FROM {db}.{sch}.LOAD_BATCH_LOG WHERE GRAPH_RUN_GROUP_ID = :run_group
);
```

### Child tasks

Every child task resolves its own run group to a batch ID. Fail if there is not exactly one mapping; do not take a fallback ID. Use the same frozen input on retries and verify incomplete steps before continuing.

```sql
-- At the top of every child task body:
LET run_group VARCHAR := (
    SELECT SYSTEM$TASK_RUNTIME_INFO('CURRENT_TASK_GRAPH_RUN_GROUP_ID')
);
LET md INT := (
    SELECT BATCH_ID FROM {db}.{sch}.LOAD_BATCH_LOG WHERE GRAPH_RUN_GROUP_ID = :run_group
);
-- Fail the task if md is NULL before using it.

-- Then in every INSERT:
INSERT INTO {db}.{sch}."{AN}_{Descriptor}" ("{AN}_ID", "Metadata_{AN}")
SELECT "{AN}_ID", :md FROM {an}_new ORDER BY "{AN}_ID";
```

### Finalizer and publication

1. **Integrity:** the finalizer can record the result of `IntegrityViolations` (or equivalent hand-written checks) and all custom rule checks for the candidate version. It runs after other tasks finish or fail, not if the root is skipped. Record `PASSED` only after every check finishes without violations; a failed check is `CHECK_FAILED`. Revalidate the final candidate after a retry.
2. **Execution:** outside the graph, wait for the definitive result (`EXECUTE TASK … AWAIT` for manual runs, or [`COMPLETE_TASK_GRAPHS`](https://docs.snowflake.com/en/sql-reference/functions/complete_task_graphs) by graph run group ID for scheduled runs). Check that all expected tasks ran successfully, including required children that might have been suspended or skipped. Do not treat missing or delayed history as success or use a prior attempt's status.
3. **Publication:** keep the candidate out of consumer-facing perspectives until execution and integrity both pass. For example, load and validate an isolated candidate in a work schema and publish only the validated result through a controlled cutover. Otherwise consumers can see a partial load or wrong results under `RELY`. Mark `PUBLISHED` only after the cutover succeeds. A set of tasks does not share one transaction.

## Rolling back a batch

Metadata identifies inserted rows; it does not prove that removing them is safe. Stop competing writes and check retained batches for dependencies first. A later batch might reference an identity created in this batch or rely on its key match or restatement decisions. If there are dependencies, plan which downstream work to replay or recover rather than deleting this batch alone. Deleting inserts does not undo changes to existing rows or already published outputs.

After approval and a successful dependency check, use a targeted delete in a DML transaction. For example:

```sql
-- Find the batch
SELECT * FROM {db}.{sch}.LOAD_BATCH_LOG ORDER BY BATCH_ID DESC;

-- Delete it from every table it touched (order: ties → nexus attrs → nexuses → anchor attrs → anchors → knots)
-- Example for one nexus and its attributes:
DELETE FROM {db}.{sch}."{NX}_KOD_{Descriptor}_Kod" WHERE "Metadata_{NX}_KOD" = :batch_id;
DELETE FROM {db}.{sch}."{NX}_{ATR}_{Descriptor}_{AttrDesc}" WHERE "Metadata_{NX}_{ATR}" = :batch_id;
DELETE FROM {db}.{sch}."{NX}_{Descriptor}" WHERE "Metadata_{NX}" = :batch_id;
-- ... repeat for each table the batch wrote to.
```

Follow the actual model dependencies in reverse, not just the order in the example. Run integrity and rule checks before committing, and roll back the transaction on failure. Keep the recovery isolated from readers until validated.

Mark the batch in the log:

```sql
UPDATE {db}.{sch}.LOAD_BATCH_LOG
SET PUBLICATION_STATUS = 'ROLLED_BACK' WHERE BATCH_ID = :batch_id;
```

## What not to use the Metadata column for

- **It is not a source-system identifier.** Use it to identify which batch loaded the row, not where the data came from. Source provenance belongs in the batch log (add a `SOURCE` column if needed) or in the identifier attribute of the entity.
- **It is not a timestamp.** The batch log records when the batch ran; the Metadata column is an integer key into that log, which is smaller and faster to filter on.
