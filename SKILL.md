---
name: anchor-modeling
description: "Full-lifecycle Anchor Modeling on Snowflake. Use for **ALL** requests involving Anchor Modeling: reverse-engineering an existing database into an Anchor model, generating Anchor model DDL, querying Anchor model databases, extending models with new constructs, or explaining Anchor model patterns. Triggers: anchor model, anchor modeling, 6NF, sixth normal form, anchor table, historized attribute, knot table, tie table, nexus table, perspective view, temporal model, bitemporal, uni-temporal, reverse engineer to anchor, create anchor model."
---

# Anchor Modeling

Anchor Modeling is a database design technique that decomposes a domain into sixth-normal-form tables: identity tables (anchors), single-property tables (attributes), lookup tables (knots), relationship tables (ties), and event tables (nexuses). It targets Snowflake SQL exclusively.

## Setup

1. **Load** `references/constructs.md` for Anchor Modeling theory and construct definitions.
2. **Load** `references/ddl-patterns.md` for Snowflake DDL templates.
3. **Load** `references/naming-conventions.md` for naming rules.
4. The database `ANCHOR_EXAMPLE.PUBLIC` contains a working theatre-domain model you can reference.

## Intent Detection

| Intent | Triggers | Go to |
|--------|----------|-------|
| REVERSE-ENGINEER | "reverse engineer", "convert to anchor", "anchor model from", "create anchor model from database", "from staged files" | [Step R1](#r1-discover-source) |
| GENERATE | "generate DDL", "create anchor model", "build anchor model", "new anchor model" | [Step G1](#g1-gather-domain) |
| QUERY | "query anchor", "how to query", "perspective", "latest view", "point in time" | [Step Q1](#q1-identify-model) |
| EXTEND | "add attribute", "add anchor", "add tie", "add knot", "extend model", "new property" | [Step E1](#e1-identify-target) |
| LOAD | "load data", "populate", "insert data", "load from source", "fill anchor model" | [Step L1](#l1-plan-loading-order) |
| EXPLAIN | "explain anchor", "what is a knot", "how does", "anchor modeling concept" | [Step X1](#x1-explain) |

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

**Rules for mapping:**

- Each entity with a surrogate or natural key → **Anchor**. Use a 2-letter mnemonic derived from the entity name.
- **Every anchor MUST have at least one identifier attribute** — a static attribute storing the source natural key (e.g., employee number, project code). Without this, there is no way to map source data to surrogate IDs during loading. Use a KOD (code) or similar mnemonic.
- Each column on an entity → **Attribute** (3-letter mnemonic). Decide:
  - Static vs historized: ask user which properties change over time.
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

**⚠️ STOP**: Confirm understanding of the domain before designing.

### G2: Design Model

**Goal:** Assign mnemonics and structure.

**Actions:**

1. Assign 2-letter mnemonics for anchors and nexuses.
2. Assign 3-letter mnemonics for attributes and knots.
3. Design ties with role names that read as a sentence.
4. Decide historization for each attribute and tie.
5. Present the model summary table (as in R2).

**⚠️ STOP**: Get approval on the model design.

### G3: Generate DDL

**Goal:** Produce Snowflake SQL DDL.

**Actions:**

Load `references/ddl-patterns.md` and generate in this order:

1. **Database and schema** (if new)
2. **Knots** — lookup tables with identity + value + Metadata column
3. **Anchors** — identity-only tables with sequences
4. **Nexuses** — identity + role FK columns
5. **Attributes** — one table per property, four flavors:
   - Static: `{anchor}_{attr}_{AnchorDesc}_{AttrDesc}` with FK + value + Metadata
   - Historized: adds `ChangedAt` column to PK
   - Knotted static: FK to knot instead of value column
   - Knotted historized: FK to knot + `ChangedAt`
6. **Ties** — relationship tables with role columns
7. **Latest perspectives** (`l` prefix) — views joining anchor/nexus to all attributes
8. **Point-in-time perspectives** (`p` prefix) — table functions taking a timestamp
9. **Now perspectives** (`n` prefix) — views calling the point-in-time function with `current_timestamp()`
10. **Difference perspectives** (`d` prefix) — table functions for change intervals

Apply naming conventions from `references/naming-conventions.md`. Add `COMMENT` on every table and view using descriptions. Add `CLUSTER BY` on primary key columns.

**⚠️ STOP**: Present the generated DDL for review before execution.

### G4: Execute DDL

Execute the approved DDL using `snowflake_sql_execute`, one statement at a time in dependency order. Verify each object was created successfully.

---

## Query Workflow

### Q1: Identify Model

**Goal:** Find and understand the Anchor model database.

**Actions:**

1. Ask which database/schema contains the Anchor model, or discover it via `SHOW TABLES`.
2. Classify objects by naming convention:
   - 2-char prefix + `_` + PascalCase = anchor
   - 3-char prefix attributes, knots
   - Multi-prefix with roles = ties
   - `l`/`p`/`n`/`d` prefix = perspectives

### Q2: Write Query

**Goal:** Help the user query the model.

**Rules:**

- **For current-state queries**: Use latest perspectives (`lXX_...`). These join everything together.
- **For point-in-time queries**: Use point-in-time perspective functions (`pXX_...(timestamp)`).
- **For change tracking**: Use difference perspectives (`dXX_...(start, end)`).
- **For simple lookups**: Query individual tables directly.
- Always explain what the perspective joins under the hood.

---

## Extend Workflow

### E1: Identify Target

**Goal:** Find the existing model and understand what to add.

**Actions:**

1. Identify the target database/schema.
2. Ask what the user wants to add: new anchor, attribute, knot, tie, or nexus.
3. Introspect existing objects to understand current mnemonics and avoid conflicts.

### E2: Design Extension

**Goal:** Design the new construct(s) following existing conventions.

**Rules:**

- New attributes on existing anchors: pick a 3-letter mnemonic not yet used on that anchor.
- New anchors: pick a 2-letter mnemonic not yet used in the model.
- New ties: compose from existing anchor/knot mnemonics + role names.
- Generate DDL for the new objects AND update affected perspectives.

**⚠️ STOP**: Present the extension DDL for approval.

### E3: Apply Extension

Execute DDL. Recreate affected perspective views with `CREATE OR REPLACE`.

---

## Explain Workflow

### X1: Explain

**Goal:** Teach the user about Anchor Modeling concepts.

Load `references/constructs.md` and explain the requested concept. Use `ANCHOR_EXAMPLE.PUBLIC` for concrete examples. Key points to cover:

- **Why Anchor Modeling**: non-destructive schema evolution, no ALTER TABLE, history built in, narrow tables favor columnar engines like Snowflake.
- **When to use what**: anchor vs knot, tie vs nexus, static vs historized.
- **Perspectives**: latest for dashboards, point-in-time for audits, difference for CDC.

---

## Data Loading Workflow

**Always implement loading as a Snowflake task graph (DAG).** This makes the load repeatable, captures the dependency order, and enables parallel execution where the graph allows it.

### L1: Plan the Task Graph

Design a DAG with these dependency layers:

```
ROOT_TASK (no schedule — on-demand via EXECUTE TASK)
    ├─→ LOAD_KNOTS ─────────────────┐  (all knots in one task)
    └─→ LOAD_ANCHORS ──┐            │  (all anchors + identifier attributes)
                        │            │
                        ├─→ LOAD_ANCHOR_DESCRIPTIONS  (after LOAD_ANCHORS)
                        │            │
                        └────────────┼─→ LOAD_{nexus1}  (after both KNOTS and ANCHORS)
                                     ├─→ LOAD_{nexus2}
                                     └─→ LOAD_{nexus3}
```

**Parallelism rules:**
- LOAD_KNOTS and LOAD_ANCHORS run **in parallel** (no cross-dependencies).
- LOAD_ANCHOR_DESCRIPTIONS depends only on LOAD_ANCHORS.
- Each nexus load task depends on **both** LOAD_KNOTS and LOAD_ANCHORS (fan-in), so it waits for both to finish.
- Multiple nexus load tasks run **in parallel** with each other.

**Root task** truncates all tables in reverse FK dependency order (ties → nexus attributes → nexuses → anchor attributes → anchors → knots).

**Each nexus load task** is a single BEGIN...END block that:
1. Creates a temporary staging table with explicit ROW_NUMBER() IDs, joining source to all anchor identifier and knot lookup tables.
2. INSERTs into the nexus table from the staging table.
3. INSERTs into every attribute table (static and knotted) from the staging table.
4. INSERTs into tie tables (WHERE the optional FK IS NOT NULL) from the staging table.
5. Drops the staging table.

### L2: Staging Table Pattern (Critical)

A nexus's role columns (e.g., AN_ID, KS_ID) are typically not unique per row, so you **cannot** join back from the nexus table to the source to load attributes. Instead, build a staging table upfront:

```sql
CREATE OR REPLACE TEMPORARY TABLE {nx}_staging AS
SELECT
    ROW_NUMBER() OVER (ORDER BY src.source_pk) AS {nx}_id,
    anchor_map.ANCHOR_ID AS anchor_fk,
    knot_map.KNOT_ID AS knot_fk,
    src.value_column AS value
FROM source_table src
JOIN identifier_attribute_table anchor_map ON anchor_map.code = src.source_code
LEFT JOIN knot_table knot_map ON knot_map.value = src.source_value;
```

Then INSERT into the nexus and ALL its attribute/tie tables from the same staging table using the explicit `{nx}_id`. This avoids the cross-join explosion that occurs when correlating nexus rows back to source rows by non-unique FK combinations.

### L3: Task Creation Rules

- **No schedule on root task.** Run on demand: `EXECUTE TASK {root_task};`
- **Use warehouse-based tasks** (not serverless) with a named warehouse.
- **Use BEGIN...END blocks** for multi-statement task bodies (Snowflake Scripting).
- **Use fully-qualified table names** (database.schema.table) in task bodies for nexus INSERT targets. Tables created via `CREATE OR REPLACE` in an interactive session may not resolve by schema-qualified name alone from a task context.
- **Resume children before root.** Child tasks must be resumed (`ALTER TASK ... RESUME`) before executing the root. Use `SELECT SYSTEM$TASK_DEPENDENTS_ENABLE('{root_task}')` if the root has a schedule, or resume each child individually if the root is on-demand only.
- **Verify after execution** by checking task history and row counts:

```sql
-- Check task graph results
SELECT name, state, error_message
FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY(
    RESULT_LIMIT => 20,
    SCHEDULED_TIME_RANGE_START => DATEADD('minute', -10, CURRENT_TIMESTAMP())
))
WHERE database_name = '{database}'
ORDER BY scheduled_time;
```

**⚠️ STOP**: Present the task graph design for approval before creating tasks.

---

## Snowflake-Specific Pitfalls

### Unicode Characters in Identifiers
Snowflake identifiers with non-ASCII characters (ö, å, ä, ü, etc.) MUST be double-quoted. This affects table names, column names, constraint names, and all references in views/functions. Example: `"KS_Kostnadsställe"`, `"FT_OVR_FlexTid_Övertid"`. Unquoted identifiers with these characters cause syntax errors.

**Recommendation:** When designing the model, prefer ASCII-only identifiers if the audience is international. If preserving native-language names is important, consistently double-quote all affected identifiers in DDL and queries.

### Use Sequences, Not IDENTITY, for Nexuses
The DDL patterns use `IDENTITY(1,1)` for nexus tables, but **IDENTITY columns do not allow explicit value insertion** in Snowflake. This makes bulk loading with staging tables impossible. **Always use a SEQUENCE default instead** (same pattern as anchor tables). The `references/ddl-patterns.md` templates reflect this.

### Fully-Qualified Names in Task Bodies
Tables created via `CREATE OR REPLACE` in an interactive session may not resolve by schema-qualified name (e.g., `public.MY_TABLE`) when referenced from a task body. **Always use fully-qualified three-part names** (`database.schema.table`) for INSERT targets in task bodies. If a task fails with "table does not exist" for a table you know exists, recreate the table in the same session context where the tasks were created, or switch to fully-qualified names.

### Decimal Format in Source Data
Source data may use comma as decimal separator (e.g., `"8,000"` for 8.0). Use `TRY_TO_DECIMAL(REPLACE(value, ',', '.'), precision, scale)` when loading numeric attributes from text columns.

---

## Common Patterns

### Metadata Column
Every table has a `Metadata_XX int not null` column. This is used for tracking the source or batch of each row. Always include it.

### Historized Attributes
The `ChangedAt` column is part of the primary key. The latest perspective picks `max(ChangedAt)` per entity. Point-in-time uses `max(ChangedAt) WHERE ChangedAt <= @timepoint`.

### Knotted Attributes
Store only the knot's FK. The perspective LEFT JOINs the knot table to bring in the readable value.

### Foreign Keys
All role columns and attribute owner columns have FK constraints referencing the parent anchor/nexus/knot.

## Stopping Points

- ✋ After R1: Source discovery confirmed
- ✋ After R2/G2: Model design approved
- ✋ After G3: DDL reviewed
- ✋ After L2: Loading plan approved
- ✋ After E2: Extension DDL approved

## Output

Snowflake DDL scripts ready to execute, or query guidance for existing models.
