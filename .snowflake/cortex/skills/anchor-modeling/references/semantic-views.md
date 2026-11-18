# Semantic Views over an Anchor Model

A Snowflake semantic view gives business names, dimensions and metrics to data that a model holds. This is how to build one on an Anchor model, including **as of a point in time**.

**What is verified and what is not.** A logical table over a point-in-time table function was tried in a Snowflake account. The revised metric and UTC variant below have not been retested. Snowflake's documentation of `CREATE SEMANTIC VIEW` does not say that a logical table may be a query over a table function. Prefer the ordinary `n` perspective for current state, or an ordinary view over a function with a fixed cutoff if the direct table-function pattern is unavailable.

## The pattern

This uni-temporal example uses a `SELECT` over `TABLE(...)` of a **point-in-time perspective**. It measures **actors with a known name at the cutoff**, not whether all actors existed then. A perspective driven by the anchor can retain an identity even when no name version is in effect.

```sql
CREATE OR REPLACE SEMANTIC VIEW ANCHOR_EXAMPLE_UNI.PUBLIC.test_sv
  TABLES (
    actors AS (
      SELECT
        "AC_ID",
        "AC_NAM_Actor_Name" AS actor_name,
        "AC_GEN_GEN_Gender" AS gender,
        "AC_PLV_PLV_ProfessionalLevel" AS professional_level
      FROM TABLE(ANCHOR_EXAMPLE_UNI.PUBLIC."pAC_Actor"(SYSDATE()))
    )
      PRIMARY KEY ("AC_ID")
      COMMENT = 'Actors at current point in time'
  )
  DIMENSIONS (
    actors.actor_name_dim AS actor_name
      COMMENT = 'Name of the actor',
    actors.gender_dim AS gender
      COMMENT = 'Gender of the actor',
    actors.professional_level_dim AS professional_level
      COMMENT = 'Professional level of the actor'
  )
  METRICS (
    actors.named_actor_count AS COUNT(actor_name)
      COMMENT = 'Actors with a known name at the cutoff'
  )
  COMMENT = 'Semantic view over Anchor Model point-in-time function for Actors'
```

Query it with the `SEMANTIC_VIEW` clause (clause order in the definition is `TABLES`, `RELATIONSHIPS`, `DIMENSIONS`, `METRICS`):

```sql
SELECT * FROM SEMANTIC_VIEW(
  ANCHOR_EXAMPLE_UNI.PUBLIC.test_sv
  DIMENSIONS actors.gender_dim
  METRICS actors.named_actor_count
);
```

## The point in time from a session variable

A **session variable** in place of the fixed timestamp lets whoever queries choose the point in time. This was tried with the original metric; recheck the revised example in your account:

```sql
SET pit_timestamp = '2024-06-15 12:00:00'::TIMESTAMP_NTZ;

CREATE OR REPLACE SEMANTIC VIEW ANCHOR_EXAMPLE_UNI.PUBLIC.test_sv_var
  TABLES (
    actors AS (
      SELECT
        "AC_ID",
        "AC_NAM_Actor_Name" AS actor_name,
        "AC_GEN_GEN_Gender" AS gender,
        "AC_PLV_PLV_ProfessionalLevel" AS professional_level
      FROM TABLE(ANCHOR_EXAMPLE_UNI.PUBLIC."pAC_Actor"($pit_timestamp))
    )
      PRIMARY KEY ("AC_ID")
      COMMENT = 'Actors at a given point in time'
  )
  DIMENSIONS (
    actors.actor_name_dim AS actor_name
      COMMENT = 'Name of the actor',
    actors.gender_dim AS gender
      COMMENT = 'Gender of the actor'
  )
  METRICS (
    actors.named_actor_count AS COUNT(actor_name)
      COMMENT = 'Actors with a known name at the cutoff'
  )
  COMMENT = 'Semantic view with session variable timestamp'
```

With the variable set, query the view as usual:

```sql
SELECT * FROM SEMANTIC_VIEW(
  ANCHOR_EXAMPLE_UNI.PUBLIC.test_sv_var
  METRICS actors.named_actor_count
);
```

**The variable is read when the view is queried.** `SELECT GET_DDL('SEMANTIC_VIEW', 'ANCHOR_EXAMPLE_UNI.PUBLIC.test_sv_var')` returns the definition with `$pit_timestamp` in it. Test with an actor whose first name version falls between the two cutoffs. The actor should contribute to the count only at the later cutoff. A name change between two known names need not change the count.

```sql
SET pit_timestamp = '2020-01-01'::TIMESTAMP_NTZ;
SELECT * FROM SEMANTIC_VIEW(ANCHOR_EXAMPLE_UNI.PUBLIC.test_sv_var METRICS actors.named_actor_count);
SET pit_timestamp = SYSDATE();
SELECT * FROM SEMANTIC_VIEW(ANCHOR_EXAMPLE_UNI.PUBLIC.test_sv_var METRICS actors.named_actor_count);
```

What to keep in mind:

- A variable belongs to a **session**, and **a query in a session where it has not been set fails**: `SQL compilation error: ... Session variable '$PIT_TIMESTAMP' does not exist`. It is an error, not an empty result, and there is no default to fall back on (a `COALESCE($pit_timestamp, SYSDATE())` cannot help, since the error comes before it is evaluated). Everyone who queries the view must `SET` it in their own session first, even to `SYSDATE()` for the current state. A client that opens its own session (a BI tool, a service such as Cortex Analyst) will therefore get the error until something sets the variable for it; if such a client is the audience, use the fixed or current point in time instead.
- Set it with the type that the function takes: `TIMESTAMP_NTZ`, in UTC (`SYSDATE()` for now).

## How to build one

1. **Take the column names from the function, not from the naming rules.** Names of knotted columns in particular differ between naming conventions. Read them off the function: `DESCRIBE FUNCTION {schema}."p{AN}_{Descriptor}"(TIMESTAMP_NTZ)` or `SELECT * FROM TABLE({schema}."p{AN}_{Descriptor}"(SYSDATE())) LIMIT 0`.
2. **Quote the generated names exactly** inside the logical table's `SELECT` (`"AC_NAM_Actor_Name"`, `"pAC_Actor"`): they are case sensitive (see `naming-conventions.md`). Give every column an **unquoted alias in lower case** (`AS actor_name`); dimensions and metrics refer to those aliases, and the semantic view is then easy to use.
3. **One logical table per anchor or nexus.** Use its actual key: identity for uni or bi, or identity and positor in crt unless you first select a positor. Define what qualifies an identity (for example, in service at the cutoff) before counting it. Add a predicate on the relevant domain facts rather than counting all anchor IDs; a batch load time is not evidence that the thing existed or qualified.
4. **A knotted attribute** is the knot's value column of the perspective (`AC_GEN_GEN_Gender` above), not the knot's identity.
5. **The point in time.** Use `SYSDATE()` for UTC now; `CURRENT_TIMESTAMP()::TIMESTAMP_NTZ` uses the session's local clock. A fixed UTC timestamp sets one changing-time cutoff per view, while a session variable lets the reader choose it. For an assertion cutoff use the `t` perspective and inspect its arguments. For many cutoffs, start with the as-of join patterns in `performance.md`.
6. **Name** the semantic view and give it a `COMMENT`; use the descriptions of the model as the `COMMENT` of the logical tables and the dimensions.

## Slicing a nexus by its anchors (relationships)

A nexus measured by things it refers to (cost center, project) needs those things as logical tables and `RELATIONSHIPS` between them. **Tested on Snowflake (2026-11-18) over base tables of this shape**: a nexus `HT_Ledger` with a required role to `KS_Cost`, and an optional role to `PR_Project` held in the tie `HT_PR_Ledger_in_Project`, whose key is the nexus role (a transaction belongs to at most one project). Five transactions, three of them with a project.

```sql
CREATE OR REPLACE SEMANTIC VIEW {db}.{sch}.SV_LEDGER
  TABLES (
    ht   AS {db}.{sch}."HT_Ledger"              PRIMARY KEY ("HT_ID")    COMMENT = 'Ledger transactions',
    ks   AS {db}.{sch}."KS_Cost"                PRIMARY KEY ("KS_ID")    COMMENT = 'Cost centers',
    pr   AS {db}.{sch}."PR_Project"             PRIMARY KEY ("PR_ID")    COMMENT = 'Projects',
    htpr AS {db}.{sch}."HT_PR_Ledger_in_Project" PRIMARY KEY ("HT_ID_in") COMMENT = 'Which project a transaction belongs to, if any'
  )
  RELATIONSHIPS (
    ht_to_ks AS ht   ("KS_ID_at")  REFERENCES ks,     -- a required role: nexus to anchor
    ht_to_htpr AS htpr ("HT_ID_in") REFERENCES ht,     -- an optional role: nexus ...
    htpr_to_pr AS htpr ("PR_ID_for") REFERENCES pr     -- ... through its tie to the anchor
  )
  DIMENSIONS (
    ks.cost_center AS "KS_Name" COMMENT = 'Name of the cost center',
    pr.project     AS "PR_Name" COMMENT = 'Name of the project'
  )
  METRICS ( ht.total_amount AS SUM("HT_Amount") COMMENT = 'Sum of transaction amounts' )
  COMMENT = 'Ledger by cost center and project';
```

`SEMANTIC_VIEW(... DIMENSIONS pr.project METRICS ht.total_amount)` gave Alpha 300, Beta 300 and **an empty group of 900**: the transactions with no tie row are counted, not lost, and the sum over every group equals the sum of all rows. By cost center it gave North 800, South 700; by both, North/Alpha 300, North/(none) 500, South/Beta 300, South/(none) 400. Check the same way on a real model: the groups must add up to the total.

What was **not** tested: logical tables over the `l`, `n` or `p` perspectives of a generated model in a relationship (the `l` views were used for a single logical table in a built model, without relationships), relationships in a model with a historized tie, and a point in time together with relationships. If a relationship is rejected on a perspective, fall back to one logical table that joins what is needed in its `SELECT`, and say so.

## Other sources for a logical table (not tested)

- The **now** perspective `n{AN}_{Descriptor}` is an ordinary view: `actors AS ANCHOR_EXAMPLE_UNI.PUBLIC."nAC_Actor"`. Use it for values in effect now. The **latest** perspective `l{AN}_{Descriptor}` may include future-effective values.
- **Ties over a point-in-time function** (relationships over a table function; plain tables are tested, see above): a `RELATIONSHIPS` clause between two logical tables on the columns of the tie's roles. The syntax of a relationship over a point-in-time function has not been tried; build the logical table of the tie the same way (a query over `TABLE("p{tie_name}"(...))`), then write the relationship, and run it before relying on it.

## Limits

- A fixed point in time is one per semantic view. A session variable moves the choice to the session, which a client that opens its own session does not have (above).
