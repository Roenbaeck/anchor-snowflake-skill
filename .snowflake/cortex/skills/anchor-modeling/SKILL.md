---
name: anchor-modeling
description: "Full-lifecycle Anchor Modeling on Snowflake. Use for **ALL** requests involving Anchor Modeling: reverse-engineering an existing database into an Anchor model, generating Anchor model DDL, loading data into an Anchor model, querying Anchor model databases, extending models with new constructs, building semantic views on an Anchor model, or explaining Anchor model patterns. Triggers: anchor model, anchor modeling, 6NF, sixth normal form, anchor table, historized attribute, knot table, tie table, nexus table, perspective view, temporal model, bitemporal, uni-temporal, reverse engineer to anchor, create anchor model, semantic view on anchor model."
---

# Anchor Modeling

Anchor Modeling is a database design technique that decomposes a domain into sixth-normal-form tables: identity tables (anchors), single-property tables (attributes), lookup tables (knots), relationship tables (ties), and event tables (nexuses). This skill targets Snowflake SQL exclusively.

## Setup

Reference files. Load each one when a step says so, not all up front:

- `references/constructs.md`: Anchor Modeling theory and construct definitions.
- `references/ddl-patterns.md`: Snowflake DDL templates, perspective templates, integrity checks.
- `references/naming-conventions.md`: naming rules and identifier-case behaviour.
- `references/generator.md`: the Anchor generator, a set of Snowflake functions that makes the whole DDL script from the model XML (uni, bi and crt). Load it when generating DDL.
- `references/model-xml.md`: the model XML that the generator reads, and how to write it. Load it with `generator.md`.
- `references/semantic-views.md`: semantic views on an Anchor model, including as of a point in time. Load it when the user wants one.

**Environment:**

- Run SQL with the Snowflake SQL execution tool the host provides (for example `snowflake_sql_execute`). If no such tool is available, hand the SQL to the user to run.
- If an `ANCHOR_EXAMPLE.PUBLIC` database exists in the account, it holds a working theatre-domain model you can use as a live reference. Check with `SHOW SCHEMAS IN DATABASE ANCHOR_EXAMPLE` before relying on it. If it is missing, use the examples in the reference files.

**Scope:** the hand-written templates in `references/ddl-patterns.md` cover **uni-temporal** models. The Anchor generator (`references/generator.md`) makes DDL for **uni, bitemporal (bi) and concurrent-reliance-temporal (crt)** models; it is installed once per account. For a bitemporal or concurrent-reliance-temporal request, use the generator if it is installed. If it is not, explain the concepts, say that the generator is needed for them (`generator.md` says how to install it), and ask before improvising DDL.

## Intent Detection

| Intent | Triggers | Go to |
|--------|----------|-------|
| REVERSE-ENGINEER | "reverse engineer", "convert to anchor", "anchor model from", "create anchor model from database", "from staged files" | [Step R1](#r1-discover-source) |
| GENERATE | "generate DDL", "create anchor model", "build anchor model", "new anchor model" | [Step G1](#g1-gather-domain) |
| QUERY | "query anchor", "how to query", "perspective", "latest view", "point in time" | [Step Q1](#q1-identify-model) |
| EXTEND | "add attribute", "add anchor", "add tie", "add knot", "extend model", "new property" | [Step E1](#e1-identify-target) |
| LOAD | "load data", "populate", "insert data", "load from source", "fill anchor model" | [Step L1](#l1-plan-the-task-graph) |
| SEMANTIC VIEW | "semantic view", "semantic model", "Cortex Analyst", "metrics and dimensions on the anchor model" | [Step S1](#s1-choose-what-to-expose) |
| EXPLAIN | "explain anchor", "what is a knot", "how does", "anchor modeling concept" | [Step X1](#x1-explain) |
| INSTALL | "install the generator", "set up the anchor generator", "update the generator", or a GENERATE request when the generator is wanted but not installed | [Step I1](#i1-check-and-choose-a-schema) |

---

## Install Workflow

Installs, or updates, the Anchor generator (`references/generator.md`) in the user's account. You do this yourself with the SQL tool (or the shell, if the host has one); the user is not asked to run SQL, only to approve the steps. Load `references/generator.md`; it holds the exact statements.

### I1: Check and Choose a Schema

1. Check whether it is installed: `SHOW USER FUNCTIONS LIKE 'ANCHOR_GENERATE' IN ACCOUNT;`. If it is, say where (database and schema) and stop, unless the user asked to update it; to update, go on to I2 with the same schema.
2. Ask where to put it: a database and a schema. Suggest a schema called `ANCHOR_TOOLS` in a database the user already works in. The generator and the Git repository object it is installed from live in that schema.

**⚠️ STOP**: Confirm the database, the schema, and that the user wants the generator's objects (three functions, a table and, for way A, a Git repository object) created there.

### I2: Install

Choose the way, in this order (details in `references/generator.md`, Install):

1. **From a Git repository in Snowflake** (A): needs an API integration for GitHub. Check `SHOW API INTEGRATIONS;`. Use the one that allows `https://github.com/Roenbaeck`, else ask the user for its name, or whether they want to set one up (README, Installation, step 1).
2. **With the Snowflake CLI** (B), if the host has a shell and a connected `snow` or SnowSQL.
3. **By the user in Snowsight** (C), if neither is possible: give the steps, do not paste the script.

Never read `anchor_generator.sql` into the conversation to send it back as SQL: it is about 450 KB.

Run the statements in one session. If one fails, show the user the statement and the message and stop; do not work around a missing privilege by guessing.

### I3: Verify

Run the three checks under *After installing* in `references/generator.md`. Report the qualified name of `ANCHOR_GENERATE` to the user, and use that name in later steps.

---

## Reverse-Engineer Workflow

### R1: Discover Source

**Goal:** Understand the source database or staged files.

**Actions:**

1. Ask the user for the source database/schema or stage path.
2. Introspect the source:
   - For a database: `SHOW TABLES IN <db>.<schema>`, then `SHOW COLUMNS IN TABLE <table>` for each table. Examine foreign keys, column names, data types, and comments.
   - For staged files: `LIST @<stage>`, then `SELECT * FROM @<stage>/<file> LIMIT 10` to sample data and infer structure.
3. Build a catalog of entities, their columns, relationships (from FKs or naming patterns), and candidate lookup/reference tables.

**⚠️ STOP**: Present the discovered entities and relationships to the user for confirmation before modeling.

### R2: Propose Anchor Model

**Goal:** Map the source structure to Anchor Modeling constructs.

Load `references/constructs.md`.

**Rules for mapping:**

- Each entity with a surrogate or natural key → **Anchor**. Use a 2-letter mnemonic derived from the entity name.
- **Every anchor MUST have at least one identifier attribute**: a static attribute storing the source natural key (e.g., employee number, project code). Without it, there is no way to map source data to surrogate IDs during loading. Use a KOD (code) or similar mnemonic. Nexuses need one too, holding the source key of the event.
- Each column on an entity → **Attribute** (3-letter mnemonic). Decide:
  - Static vs historized: ask the user which properties change over time.
  - If the column references a small lookup/enum table → **Knot** + knotted attribute.
- Each foreign key or join table → **Tie**. Determine cardinality from key constraints.
- Join tables with their own properties or additional FKs → **Nexus** candidate.
- Small reference/enum tables (< ~50 rows, stable values) → **Knot** (3-letter mnemonic).
- **Knot vs anchor for code+description pairs:** If a source dimension has both a code and a readable description, and you need both in the model, make it an anchor with a code attribute and a historized description attribute. Only use a knot if a single value column suffices.

Present the proposed model as a table:

```
| Construct | Mnemonic | Descriptor       | Notes                    |
|-----------|----------|------------------|--------------------------|
| Anchor    | AC       | Actor            | From source actors table |
| Attribute | KOD      | Code (on AC)     | Static, identifier       |
| Attribute | NAM      | Name (on AC)     | Historized               |
| Knot      | GEN      | Gender           | M/F/Other                |
| Tie       | —        | AC_part_PR_in    | Many-to-many             |
```

**⚠️ STOP**: Get user approval on the proposed model before generating DDL.

### R3: Generate DDL

Proceed to [Step G3](#g3-generate-ddl) with the approved model.

---

## Generate Workflow

### G1: Gather Domain

**Goal:** Understand what the user wants to model.

**Actions:**

1. Ask the user to describe their domain: what are the main things (entities), what properties do they have, how do they relate?
2. Identify which properties change over time (historized) vs are set once (static).
3. Identify small, stable value sets that should be knots.
4. Identify the natural key of each entity and event (becomes the identifier attribute).

**⚠️ STOP**: Confirm understanding of the domain before designing.

### G2: Design Model

**Goal:** Assign mnemonics and structure.

Load `references/constructs.md`.

**Actions:**

1. Assign 2-letter mnemonics for anchors and nexuses.
2. Assign 3-letter mnemonics for attributes and knots.
3. Design ties with role names that read as a sentence.
4. Decide historization for each attribute and tie.
5. Present the model summary table (as in R2).

**⚠️ STOP**: Get approval on the model design.

### G3: Generate DDL

**Goal:** Produce Snowflake SQL DDL.

**Choose how to generate.** Check whether the Anchor generator is installed:

```sql
SHOW USER FUNCTIONS LIKE 'ANCHOR_GENERATE' IN ACCOUNT;
```

- **Installed: use it.** Load `references/generator.md` and `references/model-xml.md`. Write the model XML from the approved design (or take the one the user has from the Anchor Modeler), call `ANCHOR_GENERATE`, search the result for the text `undefined` (it means incomplete `<metadata>`; do not run it), and present the script. This is the only way to get bitemporal and concurrent-reliance-temporal models, equivalence and checksums, and it gives the same DDL as the Anchor Modeler.
- **Not installed:** offer to install it yourself ([Step I1](#i1-check-and-choose-a-schema); it takes a few statements and no manual work for the user). If the user declines, or the model is simple and uni-temporal, generate by hand as below.

**By hand.** Load `references/ddl-patterns.md` and `references/naming-conventions.md`, then generate in this order:

1. **Database and schema** (if new)
2. **Knots**: lookup tables with identity + value + Metadata column
3. **Anchors**: identity-only tables with sequences
4. **Nexuses**: identity + role FK columns, with sequences
5. **Attributes**: one table per property, four flavors:
   - Static: `{anchor}_{attr}_{AnchorDesc}_{AttrDesc}` with FK + value + Metadata
   - Historized: adds `ChangedAt` (timestamp_ntz, UTC) to the PK
   - Knotted static: FK to knot instead of value column
   - Knotted historized: FK to knot + `ChangedAt`
6. **Ties**: relationship tables with role columns
7. **Latest perspectives** (`l` prefix): views joining anchor/nexus to all attributes
8. **Point-in-time perspectives** (`p` prefix): table functions taking a UTC timestamp
9. **Now perspectives** (`n` prefix): views calling the point-in-time function with `sysdate()`
10. **Difference perspectives** (`d` prefix): table functions for change intervals

Rules:

- Declare every PK, UNIQUE and FK constraint with `RELY`. Snowflake does not enforce them, but `RELY` lets the optimizer eliminate unused joins in perspectives.
- Write every name from the model in double quotes (see Identifier Quoting under Snowflake-Specific Pitfalls); the schema is not quoted.
- Add `COMMENT` on every table and view using descriptions.
- Add `CLUSTER BY` to every table, by the identity of its owner (see `ddl-patterns.md`, section 0), as the generator does.
- Use `CREATE OR REPLACE ... COPY GRANTS` for perspectives so grants survive re-creation.

**⚠️ STOP**: Present the generated DDL for review before execution.

### G4: Execute DDL

Execute the approved DDL one statement at a time in dependency order. Verify each object was created successfully.

A script from the generator is already in dependency order (a function exists before another function calls it). A statement ends at the first semicolon that is outside a `$$ ... $$` body. If a statement fails, stop and show the user the statement and the message; do not edit generated SQL to get past it, since the fix belongs in the templates.

---

## Query Workflow

### Q1: Identify Model

**Goal:** Find and understand the Anchor model database.

Load `references/naming-conventions.md`.

**Actions:**

1. Ask which database/schema contains the Anchor model.
2. Classify objects by **kind and column structure**, not by name alone. A database made without quotes has upper case names, so `lAC_Actor` appears as `LAC_ACTOR` and cannot be told apart from a `LAC` mnemonic by its name (a database from the generator keeps the case: `"lAC_Actor"`).
   - `SHOW VIEWS IN SCHEMA`: latest (`L…`) and now (`N…`) perspectives. Confirm by stripping the first character and matching an existing table name.
   - `SHOW USER FUNCTIONS IN SCHEMA`: point-in-time (`P…`) and difference (`D…`) perspectives.
   - Tables: read `INFORMATION_SCHEMA.COLUMNS` and classify by columns:
     - Only `{X}_ID` + `METADATA_{X}` → **anchor**
     - `{X}_ID` + one value column + `METADATA_{X}`, with a unique constraint on the value → **knot**
     - `{X}_ID` + role columns `{T}_ID_{role}` + `METADATA_{X}` → **nexus**
     - `{AN}_{ATR}_{AN}_ID` owner column → **attribute**. Add `…_CHANGEDAT` → historized. `{AN}_{ATR}_{KNT}_ID` instead of a value column → knotted.
     - Only role columns `{T}_ID_{role}` (plus optional `…_CHANGEDAT`) + metadata → **tie**
   - Table and view `COMMENT`s describe what each object represents. Use them.

### Q2: Write Query

**Goal:** Help the user query the model.

**Rules:**

- **For current-state queries**: Use latest perspectives (`lXX_...`). These join everything together, and with `RELY` constraints Snowflake skips joins to attributes that are not selected.
- **For point-in-time queries**: Use point-in-time perspective functions (`pXX_...(timestamp)`). Pass timestamps in UTC.
- **For change tracking**: Use difference perspectives (`dXX_...(start, end)`).
- **For simple lookups**: Query individual tables directly.
- Always explain what the perspective joins under the hood.

---

## Semantic View Workflow

Load `references/semantic-views.md`, which holds the tested pattern and says what is not tested.

### S1: Choose What to Expose

**Goal:** Know which anchors and attributes, and as of when.

1. Ask which database and schema hold the model, and which anchors (or nexuses) and which of their attributes to expose. Keep it to one subject area.
2. Ask as of when: **now** (the current state), or a fixed point in time. A semantic view has one point in time; there is no way to pass one when querying.
3. For each anchor, read the column names of its point-in-time function (`SELECT * FROM TABLE({db}.{sch}."p{AN}_{Descriptor}"(SYSDATE())) LIMIT 0`) instead of building them from the naming rules.

### S2: Design

For each anchor: one logical table (a `SELECT` over `TABLE("p…"(…))`, or the `n`/`l` view for the current state), the identity as `PRIMARY KEY`, an unquoted lower case alias for every column, a dimension per attribute, and at least `COUNT` of the identity as a metric. Use the descriptions of the model as comments. Ties are relationships: say that this part is not tested (see the reference) before writing it.

**⚠️ STOP**: Present the semantic view for review before creating it.

### S3: Create and Check

Run the `CREATE OR REPLACE SEMANTIC VIEW`, then check it with a query: `SELECT * FROM SEMANTIC_VIEW({view} DIMENSIONS {table}.{dimension} METRICS {table}.{metric});`. If Snowflake rejects the logical table, show the user the message and fall back to the `n` view as the source; do not guess.

---

## Extend Workflow

### E1: Identify Target

**Goal:** Find the existing model and understand what to add.

**Actions:**

1. Identify the target database/schema.
2. Ask what the user wants to add: new anchor, attribute, knot, tie, or nexus.
3. Introspect existing objects (classify as in [Q1](#q1-identify-model)) to understand current mnemonics and avoid conflicts.

### E2: Design Extension

**Goal:** Design the new construct(s) following existing conventions.

Load `references/ddl-patterns.md` and `references/naming-conventions.md`.

**Rules:**

- New attributes on existing anchors: pick a 3-letter mnemonic not yet used on that anchor.
- New anchors and nexuses: pick a 2-letter mnemonic not yet used in the model, and include an identifier attribute.
- New ties: compose from existing anchor/knot mnemonics + role names.
- Generate DDL for the new objects AND update affected perspectives.
- `CREATE TABLE IF NOT EXISTS` skips silently when a table with that name exists, even with a different structure. If a planned name already exists, compare it with `DESCRIBE TABLE` and report differences instead of assuming the DDL applied.

**⚠️ STOP**: Present the extension DDL for approval.

### E3: Apply Extension

Execute the DDL. Recreate affected perspectives with `CREATE OR REPLACE ... COPY GRANTS` so existing grants are kept. Extend the load task graph (L1) with loads for the new tables.

---

## Explain Workflow

### X1: Explain

**Goal:** Teach the user about Anchor Modeling concepts.

Load `references/constructs.md` and explain the requested concept. Use `ANCHOR_EXAMPLE.PUBLIC` for concrete examples if it exists, otherwise the examples in the reference files. Key points to cover:

- **Why Anchor Modeling**: non-destructive schema evolution, no ALTER TABLE, history built in, narrow tables favor columnar engines like Snowflake.
- **When to use what**: anchor vs knot, tie vs nexus, static vs historized.
- **Perspectives**: latest for dashboards, point-in-time for audits, difference for CDC.

---

## Data Loading Workflow

Load `references/ddl-patterns.md` and `references/naming-conventions.md`.

**Always implement loading as a Snowflake task graph (DAG).** This makes the load repeatable, captures the dependency order, and enables parallel execution where the graph allows it.

**Loads are incremental and idempotent.** Anchor Modeling keeps history by appending rows. The recurring load never truncates. It inserts only what is new or changed, so running it twice with the same source data changes nothing. Snowflake does not enforce PK/UNIQUE/FK constraints, so every insert must filter out rows that already exist.

### L1: Plan the Task Graph

Design a DAG with these dependency layers:

```
ROOT_TASK (no schedule unless the user wants one; run via EXECUTE TASK)
    ├─→ LOAD_KNOTS ─────────────┐   (all knots)
    └─→ LOAD_ANCHORS ─────┐     │   (new anchor IDs + identifier attributes)
                          │     │
                          ├─────┼─→ LOAD_ANCHOR_ATTRIBUTES  (after KNOTS and ANCHORS)
                          ├─────┼─→ LOAD_TIES               (after KNOTS and ANCHORS)
                          └─────┼─→ LOAD_{nexus1}           (after KNOTS and ANCHORS)
                                └─→ LOAD_{nexus2}
```

**Parallelism rules:**
- LOAD_KNOTS and LOAD_ANCHORS run **in parallel** (no cross-dependencies).
- Everything else depends on **both** LOAD_KNOTS and LOAD_ANCHORS (fan-in), because knotted attributes, knotted ties and nexus roles need knot IDs as well as anchor IDs.
- LOAD_ANCHOR_ATTRIBUTES, LOAD_TIES and the nexus loads run **in parallel** with each other.

**Root task** does no loading work itself (`SELECT 1` or a logging insert). It only anchors the graph.

**Initial load / full rebuild** is not part of the graph. If the user wants to start over, provide a separate truncate script: ties → nexus attributes → nexuses → anchor attributes → anchors → knots. Reset the sequences only if the tables are empty. Warn explicitly that it destroys all history, and run it only after the user confirms.

### L2: Load Patterns

Replace `{db}.{sch}` with the fully-qualified schema and `{md}` with the Metadata value for the batch (agree with the user what it identifies, e.g. source system or batch number). Wrap each task body in `BEGIN ... END;`.

Every table is clustered by the identity of its owner (`ddl-patterns.md`, section 0), so each `INSERT ... SELECT` below ends with an `ORDER BY` on that identity: the rows are written in key order and the micro-partitions start out clustered. Keep the `ORDER BY` in the statements you write for ties, nexus attributes and knotted attributes too.

**Knots**: insert values not yet present. Knots have no sequence, so continue from the current max ID (only this task writes knots, so this is safe):

```sql
INSERT INTO {db}.{sch}."{KNT}_{Descriptor}" ("{KNT}_ID", "{KNT}_{Descriptor}", "Metadata_{KNT}")
SELECT
    COALESCE((SELECT max("{KNT}_ID") FROM {db}.{sch}."{KNT}_{Descriptor}"), 0)
        + ROW_NUMBER() OVER (ORDER BY v.val),
    v.val,
    {md}
FROM (SELECT DISTINCT src.{value_col} AS val FROM {source} src WHERE src.{value_col} IS NOT NULL) v
WHERE NOT EXISTS (
    SELECT 1 FROM {db}.{sch}."{KNT}_{Descriptor}" k WHERE k."{KNT}_{Descriptor}" = v.val
);
```

**Anchors + identifier attribute**: draw new IDs from the sequence for natural keys not seen before, then insert the anchor and its identifier from the same staging table:

```sql
CREATE OR REPLACE TEMPORARY TABLE {an}_new AS
SELECT {db}.{sch}."{AN}_{Descriptor}_ID_SEQ".nextval AS "{AN}_ID", k.natural_key
FROM (SELECT DISTINCT src.{key_col} AS natural_key FROM {source} src WHERE src.{key_col} IS NOT NULL) k
WHERE NOT EXISTS (
    SELECT 1 FROM {db}.{sch}."{AN}_KOD_{Descriptor}_Code" i
    WHERE i."{AN}_KOD_{Descriptor}_Code" = k.natural_key
);

INSERT INTO {db}.{sch}."{AN}_{Descriptor}" ("{AN}_ID", "Metadata_{AN}")
SELECT "{AN}_ID", {md} FROM {an}_new ORDER BY "{AN}_ID";

INSERT INTO {db}.{sch}."{AN}_KOD_{Descriptor}_Code" ("{AN}_KOD_{AN}_ID", "{AN}_KOD_{Descriptor}_Code", "Metadata_{AN}_KOD")
SELECT "{AN}_ID", natural_key, {md} FROM {an}_new ORDER BY "{AN}_ID";

DROP TABLE {an}_new;
```

**Static attributes**: insert only for owners that have no row yet. Map source rows to IDs through the identifier attribute.

```sql
INSERT INTO {db}.{sch}."{AN}_{ATR}_{AnchorDesc}_{AttrDesc}" ("{AN}_{ATR}_{AN}_ID", "{AN}_{ATR}_{AnchorDesc}_{AttrDesc}", "Metadata_{AN}_{ATR}")
SELECT i."{AN}_KOD_{AN}_ID", any_value(src.{value_col}), {md}
FROM {source} src
JOIN {db}.{sch}."{AN}_KOD_{AnchorDesc}_Code" i ON i."{AN}_KOD_{AnchorDesc}_Code" = src.{key_col}
WHERE src.{value_col} IS NOT NULL
  AND NOT EXISTS (
    SELECT 1 FROM {db}.{sch}."{AN}_{ATR}_{AnchorDesc}_{AttrDesc}" a
    WHERE a."{AN}_{ATR}_{AN}_ID" = i."{AN}_KOD_{AN}_ID"
  )
GROUP BY i."{AN}_KOD_{AN}_ID"
ORDER BY i."{AN}_KOD_{AN}_ID";
```

**Historized attributes (restatement check)**: insert a row only when the value differs from the latest stored value and is newer. `ChangedAt` comes from the source's change timestamp if it has one, otherwise the load time `sysdate()` (UTC).

```sql
INSERT INTO {db}.{sch}."{AN}_{ATR}_{AnchorDesc}_{AttrDesc}" ("{AN}_{ATR}_{AN}_ID", "{AN}_{ATR}_{AnchorDesc}_{AttrDesc}", "{AN}_{ATR}_ChangedAt", "Metadata_{AN}_{ATR}")
SELECT s.id, s.val, s.changed_at, {md}
FROM (
    SELECT i."{AN}_KOD_{AN}_ID" AS id, src.{value_col} AS val, {changed_at_expr} AS changed_at
    FROM {source} src
    JOIN {db}.{sch}."{AN}_KOD_{AnchorDesc}_Code" i ON i."{AN}_KOD_{AnchorDesc}_Code" = src.{key_col}
    WHERE src.{value_col} IS NOT NULL
    QUALIFY ROW_NUMBER() OVER (PARTITION BY i."{AN}_KOD_{AN}_ID" ORDER BY {changed_at_expr} DESC) = 1
) s
LEFT JOIN (
    SELECT *
    FROM {db}.{sch}."{AN}_{ATR}_{AnchorDesc}_{AttrDesc}"
    QUALIFY ROW_NUMBER() OVER (PARTITION BY "{AN}_{ATR}_{AN}_ID" ORDER BY "{AN}_{ATR}_ChangedAt" DESC) = 1
) cur ON cur."{AN}_{ATR}_{AN}_ID" = s.id
WHERE cur."{AN}_{ATR}_{AN}_ID" IS NULL
   OR (s.changed_at > cur."{AN}_{ATR}_ChangedAt"
       AND s.val IS DISTINCT FROM cur."{AN}_{ATR}_{AnchorDesc}_{AttrDesc}")
ORDER BY s.id;
```

This takes one current value per key from a snapshot-style source. If the source holds several versions per key, keep them all, order by `changed_at` together with the latest stored row, and insert only the rows whose value differs from the previous one (`LAG`).

**Knotted attributes**: same patterns, but join `{source}` to the knot on the value and insert the knot ID.

**Ties**: map each role through its identifier attribute (or knot) and insert combinations not yet present on the identifier roles. For historized ties, apply the same restatement check as for historized attributes.

**Nexuses**: a nexus's role columns (e.g., AN_ID, KS_ID) are typically not unique per row, so you **cannot** join back from the nexus table to the source to load attributes. Build a staging table with IDs drawn from the **sequence**, and only for events whose source key is not loaded yet:

```sql
CREATE OR REPLACE TEMPORARY TABLE {nx}_staging AS
SELECT
    {db}.{sch}."{NX}_{Descriptor}_ID_SEQ".nextval AS "{NX}_ID",
    src.{event_key_col} AS event_key,
    anchor_map."{AN}_KOD_{AN}_ID" AS anchor_fk,
    knot_map."{KNT}_ID" AS knot_fk,
    src.{value_col} AS val
FROM {source} src
JOIN {db}.{sch}."{AN}_KOD_{AnchorDesc}_Code" anchor_map ON anchor_map."{AN}_KOD_{AnchorDesc}_Code" = src.{source_code}
LEFT JOIN {db}.{sch}."{KNT}_{KnotDesc}" knot_map ON knot_map."{KNT}_{KnotDesc}" = src.{source_value}
WHERE NOT EXISTS (
    SELECT 1 FROM {db}.{sch}."{NX}_KOD_{Descriptor}_Code" e
    WHERE e."{NX}_KOD_{Descriptor}_Code" = src.{event_key_col}
);
```

Then, in the same `BEGIN ... END` block, INSERT into the nexus, its identifier attribute, ALL its other attribute tables, and its ties (`WHERE` the optional FK `IS NOT NULL`) from the staging table, using `{NX}_ID`. Drop the staging table at the end. Drawing IDs from the sequence (never `ROW_NUMBER()`) keeps them unique across runs and keeps the sequence in step with the table.

After loading, run the integrity checks. Constraints are not enforced, and `RELY` makes the optimizer trust them, so duplicates would give wrong query results. For a model made by the generator, `SELECT * FROM {schema}.IntegrityViolations;` (or the `ic_{table}` view of what was loaded) must return no rows (`references/generator.md`, *Integrity checks*); for a model written by hand, use the queries in `ddl-patterns.md` (section 11).

**⚠️ STOP**: Present the task graph design and load SQL for approval before creating tasks.

### L3: Task Creation Rules

- **No schedule on the root task** unless the user asks for one. Run on demand: `EXECUTE TASK {root_task};`
- **Use warehouse-based tasks** (not serverless) with a named warehouse.
- **Use BEGIN...END blocks** for multi-statement task bodies (Snowflake Scripting).
- **Use fully-qualified three-part names** (`database.schema.object`) for every table and sequence in task bodies. Do not rely on the task's session context for name resolution.
- **Resume child tasks before running the root.** Run `ALTER TASK {child} RESUME` for each child, which works whether or not the root has a schedule. `SELECT SYSTEM$TASK_DEPENDENTS_ENABLE('{root_task}')` resumes the whole graph including the root, so only use it when the root has a schedule and should start running on it.
- **Verify after execution** by checking task history, row counts and the integrity checks:

```sql
-- Check task graph results
SELECT name, state, error_message
FROM TABLE({database}.INFORMATION_SCHEMA.TASK_HISTORY(
    RESULT_LIMIT => 20,
    SCHEDULED_TIME_RANGE_START => DATEADD('minute', -10, CURRENT_TIMESTAMP())
))
WHERE database_name = '{DATABASE}'
ORDER BY scheduled_time;
```

---

## Snowflake-Specific Pitfalls

### Constraints Are Not Enforced
Snowflake enforces only `NOT NULL`. PK, UNIQUE and FK constraints are metadata. Declare them with `RELY` for join elimination, and make every load filter out existing rows. See `ddl-patterns.md`, section 0.

### Identifier Quoting
Write every name from the model in double quotes (`"AC_Actor"`, `"AC_NAM_ChangedAt"`), in DDL, queries and loads. The generator does, and it keeps the case and allows national characters (ö, å, ä), which an unquoted identifier does not: unquoted names are read as upper case and may hold only A-Z, digits, `_` and `$`. A quoted name is case sensitive: refer to a generated table as `public."AC_Actor"`, never `AC_Actor` (that is `AC_ACTOR`). The schema is not quoted. A database made without quotes has upper case names; classify its objects by kind and structure ([Q1](#q1-identify-model)). See `references/naming-conventions.md`.
### Sequences, Not IDENTITY
Anchors and nexuses use a sequence default, never `IDENTITY`. Snowflake IDENTITY columns reject explicit values, and loads must insert IDs drawn from the sequence into several tables.

### Time Zones
Store every `ChangedAt` as `timestamp_ntz(9)` in UTC and use `sysdate()` for "now". Interactive sessions and task sessions can have different `TIMEZONE` settings, so `current_timestamp()` cast to NTZ would give inconsistent results.

### Decimal Format in Source Data
Source data may use comma as decimal separator (e.g., `"8,000"` for 8.0). Use `TRY_TO_DECIMAL(REPLACE(value, ',', '.'), precision, scale)` when loading numeric attributes from text columns.

---

## Common Patterns

### Metadata Column
Every table has a `Metadata_XX int not null` column. This is used for tracking the source or batch of each row. Always include it.

### Historized Attributes
The `ChangedAt` column is part of the primary key. The latest perspective keeps the row with the highest `ChangedAt` per entity (`QUALIFY ROW_NUMBER()`). Point-in-time does the same after filtering `ChangedAt <= changingTimepoint`.

### Knotted Attributes
Store only the knot's FK. The perspective LEFT JOINs the knot table to bring in the readable value.

### Foreign Keys
All role columns and attribute owner columns have `RELY` FK constraints referencing the parent anchor/nexus/knot.

## Stopping Points

- ✋ After R1: Source discovery confirmed
- ✋ After R2/G2: Model design approved
- ✋ After G3: DDL reviewed
- ✋ After E2: Extension DDL approved
- ✋ After L2: Task graph and load SQL approved
- ✋ Before any truncate / full rebuild: explicit confirmation

## Output

Snowflake DDL scripts and load task graphs ready to execute, or query guidance for existing models.
