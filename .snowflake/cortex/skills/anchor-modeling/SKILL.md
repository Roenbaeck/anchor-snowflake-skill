---
name: anchor-modeling
description: "Full-lifecycle Anchor Modeling on Snowflake. Use for **ALL** requests involving Anchor Modeling: reverse-engineering an existing database into an Anchor model, generating Anchor model DDL, loading data into an Anchor model, querying Anchor model databases, extending models with new constructs, building semantic views on an Anchor model, adding constraints and business rules, advising on time, keys and warehouse layers, or explaining Anchor model patterns. Triggers: anchor model, anchor modeling, 6NF, sixth normal form, anchor table, historized attribute, knot table, tie table, nexus table, perspective view, temporal model, bitemporal, uni-temporal, reverse engineer to anchor, create anchor model, semantic view on anchor model, unique constraint across time, business rule on a historized model, anchor model layers, hash keys versus sequence identity."
---

# Anchor Modeling

Anchor Modeling is a database design technique that decomposes a domain into sixth-normal-form tables, except where a relationship must be recorded as a whole: identity tables (anchors), single-property tables (attributes), lookup tables (knots), relationship tables (ties), and event tables (nexuses). This skill targets Snowflake SQL exclusively.

## Setup

Reference files. Load each one when a step says so, not all up front:

- `references/constructs.md`: Anchor Modeling theory and construct definitions, and the design guidelines (when to historize, knot, tie, or use a state).
- `references/modeling-practice.md`: how to ask the user about the domain and what to check before presenting a model. Load it at R1, R2, G1, G2 and E2.
- `references/ddl-patterns.md`: Snowflake DDL templates, perspective templates, integrity checks.
- `references/naming-conventions.md`: naming rules and identifier-case behaviour.
- `references/generator.md`: the Anchor generator, a set of Snowflake functions that makes the whole DDL script from the model XML (uni, bi and crt). Load it when generating DDL.
- `references/model-xml.md`: the model XML that the generator reads, and how to write it. Load it with `generator.md`.
- `references/semantic-views.md`: semantic views on an Anchor model, including as of a point in time. Load it when the user wants one.
- `references/metadata.md`: what the Metadata column is for, batch tracking infrastructure, and how to roll back a load. Load it when building a task graph or when the user asks about Metadata.
- `references/time.md`: changing, assertion, recording, happening and structuring time; timestamp sources and proxies; corrections, retractions and reliability. Load it at R1, G1, G2, L1 and when a question is about *when* or *as of*.
- `references/identity-and-keys.md`: surrogate identities, why not to hash natural keys, changing keys, early and late identity management. Load it at R2, G2 and L1.
- `references/constraints.md`: rules beyond the keys (a value that must be unique, a combination that must not repeat), which must hold in every time slice, as violation views. Load it when the user states such a rule.
- `references/performance.md`: when narrow tables win and when not, why they are fast and what it depends on, and as-of joins over many dates (the twine), temporal dimensions. Load it when a query is slow or a load is large.
- `references/warehouse-practice.md`: where the model fits (landing zone, model, semantic layer; no persisted layers between), a mapping from Data Vault, loading discipline, and how an agent should answer questions over the model. Load it when asked about layers, marts, or how an agent should query the model.

**Environment:**

- Run SQL with the Snowflake SQL execution tool the host provides (for example `snowflake_sql_execute`). If no such tool is available, hand the SQL to the user to run.
- If an `ANCHOR_EXAMPLE.PUBLIC` database exists in the account, it holds a working theatre-domain model you can use as a live reference. Check with `SHOW SCHEMAS IN DATABASE ANCHOR_EXAMPLE` before relying on it. If it is missing, use the examples in the reference files.

**Scope:** the hand-written templates in `references/ddl-patterns.md` cover **uni-temporal** models. The Anchor generator (`references/generator.md`) makes DDL for **uni, bitemporal (bi) and concurrent-reliance-temporal (crt)** models; it is installed once per account. For a bitemporal or concurrent-reliance-temporal request, use the generator if it is installed. If it is not, explain the concepts, say that the generator is needed for them (`generator.md` says how to install it), and ask before improvising DDL.

**Modeling:** a model states what the business means, and a generated model can be valid and still say something else. Whenever you design or change a model (R, G and E steps), follow `references/modeling-practice.md`: settle no question of meaning silently, ask in the user's own words and not in Anchor terms (the person answering may not know them), and show sentences, examples and assumptions for the user to confirm, not only a table of constructs.

## Intent Detection

| Intent | Triggers | Go to |
|--------|----------|-------|
| REVERSE-ENGINEER | "reverse engineer", "convert to anchor", "anchor model from", "create anchor model from database", "from staged files" | [Step R1](#r1-discover-source) |
| GENERATE | "generate DDL", "create anchor model", "build anchor model", "new anchor model" | [Step G1](#g1-gather-domain) |
| QUERY | "query anchor", "how to query", "perspective", "latest view", "point in time" | [Step Q1](#q1-identify-model) |
| EXTEND | "add attribute", "add anchor", "add tie", "add knot", "extend model", "new property" | [Step E1](#e1-identify-target) |
| LOAD | "load data", "populate", "insert data", "load from source", "fill anchor model" | [Step L1](#l1-plan-the-task-graph) |
| SEMANTIC VIEW | "semantic view", "semantic model", "Cortex Analyst", "metrics and dimensions on the anchor model" | [Step S1](#s1-choose-what-to-expose) |
| CONSTRAIN | "unique", "must be unique", "constraint", "business rule", "validate", "must not", "no two" | [Step C1](#c1-state-the-rule) |
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
4. Test candidate keys, cardinalities and lookup lists with aggregates (`references/modeling-practice.md`); a sample is not enough. For each timestamp, distinguish happening, changing, assertion and recording time, and confirm precision and zone (`references/time.md`). A recording timestamp alone cannot tell when a fact changed.

**⚠️ STOP**: Present the discovered entities and relationships, with what the data showed (and the surprises, as questions), to the user for confirmation before modeling.

### R2: Propose Anchor Model

**Goal:** Map the source structure to Anchor Modeling constructs.

Load `references/constructs.md`, `references/modeling-practice.md` and `references/identity-and-keys.md`.

**Rules for mapping:**

- A thing with a persistent identity → **anchor** (2-letter mnemonic). Give it a surrogate identity; a source key is a way to find it, never a hash standing in for it.
- For recurring loads of anchors and nexuses, settle the identity match: a static identifier attribute for one stable value, a key route over the model (possibly historized), or a trusted upstream mapping. Confirm source scope, reuse and ambiguity (`references/identity-and-keys.md`).
- Each column on an entity → **Attribute** (3-letter mnemonic). Decide:
  - Static vs historized: a snapshot of current values shows nothing about change, so ask the user which properties change over time, in their words (practice file, Questions in plain language).
  - Use a knotted attribute for values in a shared, stable domain. The size of an observed lookup list is evidence, not the criterion.
- Foreign keys and join tables are evidence for **ties**; confirm cardinality with the user.
- An independently identifiable happening can be a **nexus** when its roles are required and stable. Keep optional or independently changing participants in separate ties.
- **Knot vs anchor for code+description pairs:** If the lookup item needs separately modeled properties, such as a code and a description, use an anchor; historize the properties whose history matters.

Check the model against the list in the practice file (Check before you present). Present it as the practice file says (Presenting the model): sentences in the user's words, examples with the case the model does not allow, assumptions, open questions with your proposed answers, and then the construct table:

```
| Construct | Mnemonic | Descriptor       | Notes                    |
|-----------|----------|------------------|--------------------------|
| Anchor    | AC       | Actor            | From source actors table |
| Attribute | KOD      | Code (on AC)     | Static, identifier       |
| Attribute | NAM      | Name (on AC)     | Historized               |
| Knot      | GEN      | Gender           | M/F/Other                |
| Tie       | —        | AC_part_PR_in    | Many-to-many             |
```

**⚠️ STOP**: Get user approval on the proposed model before generating DDL. The approval is of the sentences, the examples, the assumptions and the answers to the open questions, not only of the table.

### R3: Generate DDL

Proceed to [Step G3](#g3-generate-ddl) with the approved model.

---

## Generate Workflow

### G1: Gather Domain

**Goal:** Understand what the user wants to model, in their words.

Load `references/modeling-practice.md`, and `references/time.md` for the questions about history and dates.

**Actions:**

1. Ask the user to describe their domain: what are the main things, what do they know about each, how do they relate? If there is a document, a spreadsheet or a system to look at, use it, and test what you read in it against the data ([R1](#r1-discover-source), step 4).
2. From what they said, work out what they have not settled: which things have an identity and how two are told apart, which properties change and whose history is wanted, which values are a stable shared set, how many of one thing relate to how many of another, which relationships are happenings with details of their own, and whether to distinguish what was in effect, what someone believed, and what the warehouse had received.
3. Ask about those, as the practice file says: in the user's words and not in Anchor terms, with an example from their domain and your own guess in each question, the costliest first, at most five in a round. Assume the cheap ones and say so.

**⚠️ STOP**: Confirm the understanding of the domain, as sentences, before designing.

### G2: Design Model

**Goal:** Assign mnemonics and structure.

Load `references/constructs.md`, `references/modeling-practice.md`, `references/time.md` and `references/identity-and-keys.md`.

**Actions:**

1. Assign 2-letter mnemonics for anchors and nexuses.
2. Assign 3-letter mnemonics for attributes and knots.
3. Design ties with role names that read as a sentence.
4. Decide historization for each attribute and tie, from what the user said; where they did not say, use the cautious default and list it as an assumption.
5. Write a description for every construct from the confirmed sentence, in the user's words (it becomes the table's `COMMENT`).
6. Check the model against the list in the practice file (Check before you present), and fix what it finds.
7. Present the model as the practice file says (Presenting the model): sentences, examples with the case the model does not allow, assumptions, open questions with your proposed answers, then the summary table (as in R2).

**⚠️ STOP**: Get approval on the model design: the sentences, the examples, the assumptions and the answers, not only the table.

### G3: Generate DDL

**Goal:** Produce Snowflake SQL DDL.

**Choose how to generate.** Check whether the Anchor generator is installed:

```sql
SHOW USER FUNCTIONS LIKE 'ANCHOR_GENERATE' IN ACCOUNT;
```

- **Installed: use it.** Load `references/generator.md` and `references/model-xml.md`. Write the model XML from the approved design (or take the one the user has from the Anchor Modeler), call `ANCHOR_GENERATE`, and present the script. If it answers `The model has N problems, so nothing was generated`, correct the model as the list says and call it again (a generator installed before that check existed gives names such as `GEN_undefined` for incomplete `<metadata>`: search the result for `undefined`, do not run it, and offer to update the generator). This is the only way to get bitemporal and concurrent-reliance-temporal models, equivalence and checksums, and it gives the same DDL as the Anchor Modeler.
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

- Declare PK, UNIQUE and FK constraints. Use `RELY` for join elimination only with validated data; see `ddl-patterns.md`, section 0.
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

**Before the query:** define the question (`references/warehouse-practice.md`): what was in effect, what someone believed, or what had been received or published (`references/time.md`). A changing-time filter resolves versions; for a historical count, also filter on the domain facts that make an identity exist or qualify then. For many dates or a slow query, read `references/performance.md`.

**Rules:**

- **For what is in effect now**: Use `nXX_...` or `pXX_...(SYSDATE())`. `lXX_...` has no changing-time cutoff and may show future-effective data.
- **For a chosen changing time**: Use `pXX_...(timestamp)` with a UTC timestamp. For an assertion cutoff as well, inspect the `t` signature and select the positor in crt.
- **For change tracking**: Use difference perspectives (`dXX_...(start, end)`).
- **For simple lookups**: Query individual tables directly.
- Always explain what the perspective joins under the hood.

---

## Semantic View Workflow

Load `references/semantic-views.md`, which holds the tested pattern and says what is not tested.

### S1: Choose What to Expose

**Goal:** Know which anchors and attributes, and as of when.

1. Ask which database and schema hold the model, and which anchors (or nexuses) and which of their attributes to expose. Keep it to one subject area. For every metric, ask for its **definition** in the user's words (what counts as a customer or an order, as of when, which identity is counted): a semantic view is where an agreed definition is kept once, and two valid definitions are two metrics with their own names (`references/warehouse-practice.md`).
2. Ask as of when: **now** (the current state), a **fixed** point in time, or **chosen per session** with a session variable (`$pit_timestamp`, set with `SET` by whoever queries: a session that has not set it gets a compilation error, so it does not suit a client that opens its own session). A fixed or current point in time is one per semantic view.
3. For each anchor, read the column names of its point-in-time function (`SELECT * FROM TABLE({db}.{sch}."p{AN}_{Descriptor}"(SYSDATE())) LIMIT 0`) instead of building them from the naming rules.

### S2: Design

Use one logical table per anchor or nexus, based on `n` for current state or `p` for a chosen time. Declare `PRIMARY KEY` at the actual grain (identity with a chosen positor, or identity and positor in crt). Before counting, filter to the identities that exist or qualify at the chosen time; a bare count of anchor IDs is not a historical headcount. Use the descriptions as comments. Ties as semantic relationships are not tested (see the reference).

**⚠️ STOP**: Present the semantic view for review before creating it.

### S3: Create and Check

Run the `CREATE OR REPLACE SEMANTIC VIEW`, then check it with a query: `SELECT * FROM SEMANTIC_VIEW({view} DIMENSIONS {table}.{dimension} METRICS {table}.{metric});`. If Snowflake rejects the logical table, report the error. Use `n` only for a current-state request; for a fixed historical cutoff, try an ordinary view over the same cutoff. Stop if the fallback cannot preserve the requested semantics.

---

## Constrain Workflow

### C1: State the Rule

Load `references/constraints.md` and `references/modeling-practice.md`. Ask what the rule means **in time**, in the user's words, with the case that must fail ("Can two stages have the same name at the same time? Can a closed stage give its name to a new one?"), and what should happen when a load breaks it (flag the batch, or stop). Check first that the generated integrity checks (`generator.md`) do not already cover it.

**⚠️ STOP**: Confirm the rule as sentences and the case that must fail.

### C2: Design the Check

Snowflake has no triggers on tables, so the rule is a violation view in the shape of the generated checks (`Construct`, `Violation`, `ViolationKey`, `Occurrences`), kept in its own script next to the model XML, not in the generated one. Use `references/constraints.md` for the pattern, and say which parts of it have been run.

**⚠️ STOP**: Approval of the view.

### C3: Create and Test

In a scratch schema, test a load that must pass, an equal-start conflict, a backfill whose conflict starts later, and legitimate reuse at the preceding interval's exclusive end. Add the view to the task graph's validation (`references/metadata.md`) so a breaking batch cannot publish.

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

Load `references/ddl-patterns.md`, `references/naming-conventions.md` and `references/modeling-practice.md`. A new construct raises the same questions of meaning as in a new model (whether it is a thing or a property, how many relate, whether it changes), and gets them asked and confirmed the same way, in plain words.

**Rules:**

- New attributes on existing anchors: pick a 3-letter mnemonic not yet used on that anchor.
- New anchors and nexuses: pick a 2-letter mnemonic not yet used in the model, and define an identity match if they will be loaded repeatedly.
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

Load `references/constructs.md` (and `references/time.md` for anything about history, `references/warehouse-practice.md` for where the model fits) and explain the requested concept. Use `ANCHOR_EXAMPLE.PUBLIC` for concrete examples if it exists, otherwise the examples in the reference files. Key points to cover:

- **Why Anchor Modeling**: non-destructive schema evolution, no ALTER TABLE, history built in, narrow tables favor columnar engines like Snowflake.
- **When to use what**: anchor vs knot, tie vs nexus, static vs historized.
- **Perspectives**: now for what is in effect, latest without a changing-time cutoff, point-in-time for a chosen changing time, difference for changes, and time-traveling for assertion cutoffs.

---

## Data Loading Workflow

Load `references/ddl-patterns.md`, `references/naming-conventions.md`, `references/metadata.md`, `references/time.md` (where the times come from) and `references/identity-and-keys.md` (how source rows find their identities). For large volumes, `references/performance.md`.

**Always implement loading as a Snowflake task graph (DAG).** This makes the load repeatable, captures the dependency order, and enables parallel execution where the graph allows it.

**Loads are incremental and idempotent.** A recurring load never truncates. Plan replay and conflict detection before inserting. Snowflake standard tables do not enforce PK/UNIQUE/FK constraints: deduplicate the source, stop on conflicting candidates, and serialize competing writes. An anti-join alone is insufficient.

### L1: Plan the Task Graph

Design a DAG in which **every task waits only for what it needs**. Names are made from mnemonics (ASCII), so the task names need no quoting.

```
LOAD_ROOT                      (no work; run with EXECUTE TASK unless the user wants a schedule)
 ├─→ LOAD_KNOTS                      (all knots, in one task: they are small)
 ├─→ LOAD_{AN1}                      (new anchor identities + the identifier attribute of anchor 1)
 │     ├─→ LOAD_{AN1}_{ATR1}         (one task per attribute: static, historized, knotted)
 │     ├─→ LOAD_{AN1}_{ATR2}  ←──────┤ AFTER LOAD_{AN1} and, for a knotted attribute, LOAD_KNOTS
 │     └─→ ...
 ├─→ LOAD_{AN2}  ...  (anchors are independent of each other, so they load in parallel)
 ├─→ LOAD_{tie1}                     (AFTER the anchors of its roles, and LOAD_KNOTS if it has a knot role)
 ├─→ LOAD_{NX1}                      (AFTER the anchors, knots and other nexuses of its roles)
 │     └─→ LOAD_{NX1}_{ATR}          (AFTER LOAD_{NX1}, and LOAD_KNOTS if knotted)
 └─→ ...
LOAD_FINAL                     (finalizer: the integrity checks)
```

**Dependencies, precisely:**
- `LOAD_KNOTS` and every `LOAD_{AN}` run in **parallel**, directly after the root.
- An attribute needs the identities of its anchor, and the knot identities if it is knotted: `AFTER LOAD_{AN}` or `AFTER LOAD_{AN}, LOAD_KNOTS`. **All attributes of an anchor run in parallel with each other** (each writes its own table). Nexus attributes work the same way after their nexus.
- A tie needs the identities of every anchor and nexus in its roles, and the knots if it has a knot role: `AFTER LOAD_{AN1}, LOAD_{AN2}, LOAD_KNOTS`. A tie does not wait for the attributes of the anchors.
- A nexus loads after the anchors, knots and nexuses in its roles, then its identifier attribute, then its other attributes (as for an anchor).
- Only the identifier attribute of an anchor or nexus is part of the anchor's own task: the other attributes map source rows to identities through it.

**Finalizer.** `CREATE TASK LOAD_FINAL WAREHOUSE = {wh} FINALIZE = LOAD_ROOT AS ...` runs after the other tasks complete or fail (not if the root is skipped). Use it to log validation results, not to infer execution success. Publish only after all expected work and validation have succeeded (`references/metadata.md`).

**Limits of a task graph (Snowflake):** at most **1,000 tasks** per graph, **100 child tasks** per task and **100 predecessors** per task. Count the tasks before creating them (root, knots, anchors, attributes, ties, nexuses, nexus attributes, finalizer) and then:
- more than 99 anchors and ties directly under the root: put a layer of grouping tasks (no work) between the root and them;
- an anchor with more than 100 attributes, or a tie with more than 100 predecessors: group;
- more than 1,000 tasks in all: **one task per anchor for its attributes** (`LOAD_{AN}_ATTRIBUTES`, whose body runs the attribute statements one after the other) instead of one task per attribute. Say which of the two you use, and why.

**Compute.** Parallel tasks on one warehouse share it, so a graph with many parallel tasks queues unless the warehouse is large enough or multi-cluster. Serverless tasks (`USER_TASK_MANAGED_INITIAL_WAREHOUSE_SIZE` instead of `WAREHOUSE`) are managed per task. Ask the user which they want; the default is a named warehouse (L3).

**Root task** associates the Snowflake graph-run ID with a batch ID in `LOAD_BATCH_LOG`. Child tasks resolve that same batch ID from the runtime graph-run ID; configure no overlap and serialize other writers to the same tables (`references/metadata.md`).

**Initial load / full rebuild** is not part of the graph. If the user wants to start over, provide a separate truncate script: ties → nexus attributes → nexuses → anchor attributes → anchors → knots. Reset the sequences only if the tables are empty. Warn explicitly that it destroys all history, and run it only after the user has confirmed.

**Only the uni-temporal patterns below are written.** For bitemporal and concurrent-reliance-temporal models the graph has the same shape (posits instead of rows), but there are no load statements yet: tell the user.
### L2: Load Patterns

Replace `{db}.{sch}` with the fully-qualified schema and `{md}` with `:md` in task bodies. Every child task resolves `:md` from its graph-run ID (`references/metadata.md`). The examples assume metadata is enabled. Wrap task bodies in `BEGIN ... END;`; use an explicit transaction for DML that must commit as one unit, with staging DDL outside that transaction.

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

**Key routes**: the examples use a stable source key. For a route across attributes, ties or roles, resolve every branch and load the constructs it depends on first. Use the latest `key_…` view only when its time semantics fit the source. For reused or historized keys, match at the relevant changing time and within the right namespace (`references/identity-and-keys.md`). Stop on ambiguous or incomplete matches; do not treat them as new identities.

**Static attributes**: before inserting, require one distinct non-null candidate per owner and report any disagreement with an already stored value. Map source rows to IDs through the agreed lookup. The example uses a validated `{source}` and a static identifier attribute.

```sql
INSERT INTO {db}.{sch}."{AN}_{ATR}_{AnchorDesc}_{AttrDesc}" ("{AN}_{ATR}_{AN}_ID", "{AN}_{ATR}_{AnchorDesc}_{AttrDesc}", "Metadata_{AN}_{ATR}")
SELECT DISTINCT i."{AN}_KOD_{AN}_ID" AS owner_id, src.{value_col}, {md}
FROM {source} src
JOIN {db}.{sch}."{AN}_KOD_{AnchorDesc}_Code" i ON i."{AN}_KOD_{AnchorDesc}_Code" = src.{key_col}
WHERE src.{value_col} IS NOT NULL
  AND NOT EXISTS (
    SELECT 1 FROM {db}.{sch}."{AN}_{ATR}_{AnchorDesc}_{AttrDesc}" a
    WHERE a."{AN}_{ATR}_{AN}_ID" = i."{AN}_KOD_{AN}_ID"
  )
ORDER BY owner_id;
```

**Historized attributes, ordered snapshots only.** Prepare `{validated_snapshot}` as one non-null value and time per resolved owner (`id`, `val`, `changed_at`). If actual changing time is unavailable, use a stable observation timestamp, not a fresh `SYSDATE()` on retry (`references/time.md`). Recognize completed input replays and remove exact versions already stored. Run the preflight and write under the same writer exclusion:

```sql
-- Run against the snapshot after removing exact replays. Any result blocks this insert.
SELECT s.id, s.changed_at,
       CASE WHEN same_time."{AN}_{ATR}_{AN}_ID" IS NOT NULL
            THEN 'conflicting value at the same time' ELSE 'requires backfill' END AS reason
FROM {validated_snapshot} s
LEFT JOIN {db}.{sch}."{AN}_{ATR}_{AnchorDesc}_{AttrDesc}" same_time
  ON same_time."{AN}_{ATR}_{AN}_ID" = s.id
 AND same_time."{AN}_{ATR}_ChangedAt" = s.changed_at
WHERE EXISTS (
    SELECT 1 FROM {db}.{sch}."{AN}_{ATR}_{AnchorDesc}_{AttrDesc}" h
    WHERE h."{AN}_{ATR}_{AN}_ID" = s.id AND h."{AN}_{ATR}_ChangedAt" >= s.changed_at
);
```

Only when the preflight is empty, compare to the current maximum and insert changed values:

```sql
INSERT INTO {db}.{sch}."{AN}_{ATR}_{AnchorDesc}_{AttrDesc}" ("{AN}_{ATR}_{AN}_ID", "{AN}_{ATR}_{AnchorDesc}_{AttrDesc}", "{AN}_{ATR}_ChangedAt", "Metadata_{AN}_{ATR}")
SELECT s.id, s.val, s.changed_at, {md}
FROM {validated_snapshot} s
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

**Historized attributes, timestamped changes or backfills.** Keep the source changing times. Validate uniqueness in the incoming and stored data at `(id, changed_at)`: skip exact replays and stop on different values at the same time. For late versions, inspect the merged history of affected owners, including both neighbors of each candidate, and apply the construct's restatement policy. Do not silently drop a change or rewrite a stored boundary to satisfy it. If the backfill is not supported or needs reconciliation, report it and stop. With `A@10` and `C@30` stored, `B@20` belongs in the backfill, not in the snapshot filter.

**Knotted attributes**: same patterns, but join `{source}` to the knot on the value and insert the knot ID.

**Ties**: resolve each role and report conflicting combinations for the same identifier. For historized ties, apply the same ordered-snapshot or timestamped-change rules as for attributes.

**Nexuses**: the role columns need not uniquely identify the event. First validate one candidate per event key and ensure every required role resolves; report missing roles and contradictions with stored roles. In this example the source has already passed those checks. Stage IDs from the named sequence for events that do not match an existing identity:

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
JOIN {db}.{sch}."{KNT}_{KnotDesc}" knot_map ON knot_map."{KNT}_{KnotDesc}" = src.{source_value}
WHERE NOT EXISTS (
    SELECT 1 FROM {db}.{sch}."{NX}_KOD_{Descriptor}_Code" e
    WHERE e."{NX}_KOD_{Descriptor}_Code" = src.{event_key_col}
);
```

Insert the nexus and its identifier mapping in one DML transaction. Each dependent attribute or tie task then resolves the event through that mapping, including events loaded in earlier batches. This lets a late attribute or optional relationship load without creating a second event. Keep optional roles in separate ties; do not put nullable FKs in the nexus.

Before publication, check graph execution and integrity separately (`references/metadata.md`). Use the generated `IntegrityViolations` and custom rule views, or validate the hand-written model as in `ddl-patterns.md`. Check before readers can query invalid data under `RELY`; detecting violations after exposing the rows does not protect query results.

**⚠️ STOP**: Present the task graph design (the dependencies, the number of tasks against the limits in L1, and the compute) and the load SQL for approval before creating tasks.

### L3: Task Creation Rules

- **No schedule on the root task** unless the user asks for one. Run on demand: `EXECUTE TASK {root_task};`
- **Use warehouse-based tasks with a named warehouse by default**, and ask whether serverless tasks are wanted instead (see L1, Compute). The tasks of a graph run in parallel on the one warehouse, so say whether it is sized for that.
- **Use BEGIN...END blocks** for multi-statement task bodies (Snowflake Scripting).
- **Use fully-qualified three-part names** (`database.schema.object`) for every table and sequence in task bodies. Do not rely on the task's session context for name resolution.
- **Resume child tasks before running the root.** Run `ALTER TASK {child} RESUME` for each child, which works whether or not the root has a schedule. `SELECT SYSTEM$TASK_DEPENDENTS_ENABLE('{root_task}')` resumes the whole graph including the root, so only use it when the root has a schedule and should start running on it.
- **Verify before publication** by checking execution for the exact graph run, row counts and integrity (`references/metadata.md`). The following is for diagnostics, not a publication gate:

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

### Standard Tables Do Not Enforce Keys
`NOT NULL` and `CHECK` are enforced. PK, UNIQUE and FK constraints are metadata on standard Snowflake tables. Validate before publication, especially with `RELY` (`references/ddl-patterns.md`, section 0).

### Identifier Quoting
Write every name from the model in double quotes (`"AC_Actor"`, `"AC_NAM_ChangedAt"`), in DDL, queries and loads. The generator does, and it keeps the case and allows national characters (ö, å, ä), which an unquoted identifier does not: unquoted names are read as upper case and may hold only A-Z, digits, `_` and `$`. A quoted name is case sensitive: refer to a generated table as `public."AC_Actor"`, never `AC_Actor` (that is `AC_ACTOR`). The schema is not quoted. A database made without quotes has upper case names; classify its objects by kind and structure ([Q1](#q1-identify-model)). See `references/naming-conventions.md`.
### Sequences, Not IDENTITY
Use a named sequence so a load can allocate an identity once and use it in several tables. Explicit `IDENTITY` inserts are allowed, but can collide with later generated values.

### Time Zones
Store every `ChangedAt` as `timestamp_ntz(9)` in UTC and use `sysdate()` for "now". Never local time (daylight saving repeats an hour) and never a time of day alone (`references/time.md`). Interactive sessions and task sessions can have different `TIMEZONE` settings, so `current_timestamp()` cast to NTZ would give inconsistent results.

### Decimal Format in Source Data
Source data may use comma as decimal separator (e.g., `"8,000"` for 8.0). Use `TRY_TO_DECIMAL(REPLACE(value, ',', '.'), precision, scale)` when loading numeric attributes from text columns.

### Dates Are Not ChangedAt
An order date or a birth date is an attribute of the thing (happening time). `ChangedAt` is when a value started to hold (`references/time.md`).

### No Hashed Identities
Identities are sequence integers. A hash of a natural key that can change becomes another thing, and costs several times the space (`references/identity-and-keys.md`).

### Rules Over Time
A rule that applies in every time slice needs an interval or boundary check, not a row-local `CHECK`. Use a violation view (`references/constraints.md`).

---

## Common Patterns

### Metadata Column
The batch tracking patterns assume metadata is enabled. `Metadata_…` links a row to the batch that recorded it, not to an assertion time (`references/metadata.md`).

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
- ✋ After C1 and C2: the rule, and the view that checks it
- ✋ After L2: Task graph and load SQL approved
- ✋ Before any truncate / full rebuild: explicit confirmation

## Output

Snowflake DDL scripts and load task graphs ready to execute, or query guidance for existing models.
