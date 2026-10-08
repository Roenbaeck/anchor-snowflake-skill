# Metadata Column

Every table in an Anchor model has a `Metadata_{mnemonic}` column (`int not null`). It appears on anchors, nexuses, knots, attributes and ties. The column is not part of any key; it is carried through perspectives alongside the data it describes.

## Purpose

The Metadata column connects every row to the **batch** that loaded it: which run of the task graph, when, from which source, and whether it passed integrity checks. This makes it possible to:

- **Trace** any row back to the load that created it.
- **Roll back** a bad batch by deleting all rows with its Metadata value.
- **Audit** what changed between two loads by comparing Metadata values.

## Batch tracking infrastructure

Create these objects once, in the same database as the model:

```sql
CREATE SEQUENCE IF NOT EXISTS {db}.{sch}.LOAD_BATCH_SEQ START 1 INCREMENT 1;

-- One-row table: the current batch ID, read by every child task.
CREATE TABLE IF NOT EXISTS {db}.{sch}.LOAD_BATCH_CURRENT (
    BATCH_ID INT NOT NULL
);

-- Log of all batches, written by the root and updated by the finalizer.
CREATE TABLE IF NOT EXISTS {db}.{sch}.LOAD_BATCH_LOG (
    BATCH_ID            INT            NOT NULL,
    STARTED_AT          TIMESTAMP_NTZ  NOT NULL,
    COMPLETED_AT        TIMESTAMP_NTZ,
    STATUS              VARCHAR(20),       -- RUNNING → SUCCEEDED | VIOLATIONS
    INTEGRITY_VIOLATIONS INT,
    GRAPH_RUN_GROUP_ID  VARCHAR(100)       -- from TASK_HISTORY, backfilled
);
```

## How the task graph uses it

### Root task

The root task draws the next batch ID from the sequence and publishes it so every child task can read it:

```sql
-- In LOAD_ROOT:
LET bid INT := (SELECT {db}.{sch}.LOAD_BATCH_SEQ.nextval);
DELETE FROM {db}.{sch}.LOAD_BATCH_CURRENT;
INSERT INTO {db}.{sch}.LOAD_BATCH_CURRENT (BATCH_ID) VALUES (:bid);
INSERT INTO {db}.{sch}.LOAD_BATCH_LOG (BATCH_ID, STARTED_AT, STATUS)
VALUES (:bid, SYSDATE(), 'RUNNING');
```

### Child tasks

Every child task begins by reading the current batch ID into a local variable and uses it as the Metadata value on every INSERT:

```sql
-- At the top of every child task body:
LET md INT := (SELECT BATCH_ID FROM {db}.{sch}.LOAD_BATCH_CURRENT);

-- Then in every INSERT:
INSERT INTO {db}.{sch}."{AN}_{Descriptor}" ("{AN}_ID", "Metadata_{AN}")
SELECT "{AN}_ID", :md FROM {an}_new ORDER BY "{AN}_ID";
```

This is safe because:

- `LOAD_BATCH_CURRENT` has exactly one row, written by the root before any child runs.
- The root task is the predecessor of every first-wave child, so it always completes first.
- Each child reads the table once and holds the value for its entire body.

### Finalizer

The finalizer runs after all children (including on failure). It records the outcome:

```sql
-- In LOAD_FINAL (FINALIZE = LOAD_ROOT):
LET md INT := (SELECT BATCH_ID FROM {db}.{sch}.LOAD_BATCH_CURRENT);
LET violations INT := (SELECT COUNT(*) FROM {db}.{sch}.INTEGRITYVIOLATIONS);
LET status_val VARCHAR := IFF(:violations = 0, 'SUCCEEDED', 'VIOLATIONS');
UPDATE {db}.{sch}.LOAD_BATCH_LOG
SET COMPLETED_AT = SYSDATE(),
    STATUS = :status_val,
    INTEGRITY_VIOLATIONS = :violations
WHERE BATCH_ID = :md;
```

## Linking to Snowflake task history

Every execution of a task graph gets a `graph_run_group_id` (a UUID) in `INFORMATION_SCHEMA.TASK_HISTORY`. This UUID is shared by all tasks in one run. It can be backfilled into `LOAD_BATCH_LOG` after the run:

```sql
UPDATE {db}.{sch}.LOAD_BATCH_LOG bl
SET GRAPH_RUN_GROUP_ID = (
    SELECT graph_run_group_id
    FROM TABLE({db}.INFORMATION_SCHEMA.TASK_HISTORY(RESULT_LIMIT => 50))
    WHERE name = 'LOAD_ROOT' AND scheduled_time >= bl.STARTED_AT
    ORDER BY scheduled_time
    LIMIT 1
)
WHERE bl.GRAPH_RUN_GROUP_ID IS NULL;
```

The finalizer itself cannot reliably read `graph_run_group_id` for its own run because `TASK_HISTORY` has a short latency and the root task's row may not be visible yet.

## Rolling back a batch

Because every row carries the batch ID, removing a bad load is a targeted delete:

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

Delete in reverse dependency order (ties first, knots last) to avoid leaving orphaned foreign key references, even though Snowflake does not enforce them. After deleting, run `IntegrityViolations` to confirm the model is clean.

Mark the batch in the log:

```sql
UPDATE {db}.{sch}.LOAD_BATCH_LOG SET STATUS = 'ROLLED_BACK' WHERE BATCH_ID = :batch_id;
```

## What not to use the Metadata column for

- **It is not a source-system identifier.** Use it to identify which batch loaded the row, not where the data came from. Source provenance belongs in the batch log (add a `SOURCE` column if needed) or in the identifier attribute of the entity.
- **It is not a timestamp.** The batch log records when the batch ran; the Metadata column is an integer key into that log, which is smaller and faster to filter on.
