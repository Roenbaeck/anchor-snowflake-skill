# Semantic Views over an Anchor Model

A Snowflake semantic view gives business names, dimensions and metrics to data that a model holds. This is how to build one on an Anchor model, including **as of a point in time**.

**What is verified and what is not.** The pattern in *The pattern* below was found by trial and error in a Snowflake account and works there. Snowflake's own documentation of `CREATE SEMANTIC VIEW` does not say that a logical table may be a query over a table function, so it can change; if it stops working, fall back to a perspective view (see *Other sources for a logical table*). Everything marked **not tested** is guidance that has not been run.

## The pattern

A logical table does not have to name a table or a view: it can be a `SELECT` over `TABLE(...)` of a **point-in-time perspective** (`p` prefix, a table function that takes a timestamp). The timestamp is an expression in the definition.

```sql
CREATE OR REPLACE SEMANTIC VIEW ANCHOR_EXAMPLE_UNI.PUBLIC.test_sv
  TABLES (
    actors AS (
      SELECT
        "AC_ID",
        "AC_NAM_Actor_Name" AS actor_name,
        "AC_GEN_GEN_Gender" AS gender,
        "AC_PLV_PLV_ProfessionalLevel" AS professional_level
      FROM TABLE(ANCHOR_EXAMPLE_UNI.PUBLIC."pAC_Actor"(CURRENT_TIMESTAMP::TIMESTAMP_NTZ))
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
    actors.actor_count AS COUNT("AC_ID")
      COMMENT = 'Total number of actors'
  )
  COMMENT = 'Semantic view over Anchor Model point-in-time function for Actors'
```

Query it with the `SEMANTIC_VIEW` clause (clause order in the definition is `TABLES`, `RELATIONSHIPS`, `DIMENSIONS`, `METRICS`):

```sql
SELECT * FROM SEMANTIC_VIEW(
  ANCHOR_EXAMPLE_UNI.PUBLIC.test_sv
  DIMENSIONS actors.gender_dim
  METRICS actors.actor_count
);
```

## The point in time from a session variable

A **session variable** in place of the fixed timestamp lets whoever queries choose the point in time. This was also found by trial and error and works in the account where it was tried:

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
    actors.actor_count AS COUNT("AC_ID")
      COMMENT = 'Total number of actors'
  )
  COMMENT = 'Semantic view with session variable timestamp'
```

With the variable set, the view is queried as usual, and this was run and works:

```sql
SELECT * FROM SEMANTIC_VIEW(
  ANCHOR_EXAMPLE_UNI.PUBLIC.test_sv_var
  METRICS actors.actor_count
);
```

**The variable is read when the view is queried.** `SELECT GET_DDL('SEMANTIC_VIEW', 'ANCHOR_EXAMPLE_UNI.PUBLIC.test_sv_var')` returns the definition with `$pit_timestamp` in it, not a value, so nothing is substituted when the view is created. To see it in the results too, query the view, change the variable, and query again: a model with history gives different answers for two dates on either side of a change.

```sql
SET pit_timestamp = '2020-01-01'::TIMESTAMP_NTZ;
SELECT * FROM SEMANTIC_VIEW(ANCHOR_EXAMPLE_UNI.PUBLIC.test_sv_var METRICS actors.actor_count);
SET pit_timestamp = SYSDATE();
SELECT * FROM SEMANTIC_VIEW(ANCHOR_EXAMPLE_UNI.PUBLIC.test_sv_var METRICS actors.actor_count);
```

What to keep in mind:

- A variable belongs to a **session**, and **a query in a session where it has not been set fails**: `SQL compilation error: ... Session variable '$PIT_TIMESTAMP' does not exist`. It is an error, not an empty result, and there is no default to fall back on (a `COALESCE($pit_timestamp, SYSDATE())` cannot help, since the error comes before it is evaluated). Everyone who queries the view must `SET` it in their own session first, even to `SYSDATE()` for the current state. A client that opens its own session (a BI tool, a service such as Cortex Analyst) will therefore get the error until something sets the variable for it; if such a client is the audience, use the fixed or current point in time instead.
- Set it with the type that the function takes: `TIMESTAMP_NTZ`, in UTC (`SYSDATE()` for now).

## How to build one

1. **Take the column names from the function, not from the naming rules.** Names of knotted columns in particular differ between naming conventions. Read them off the function: `DESCRIBE FUNCTION {schema}."p{AN}_{Descriptor}"(TIMESTAMP_NTZ)` or `SELECT * FROM TABLE({schema}."p{AN}_{Descriptor}"(SYSDATE())) LIMIT 0`.
2. **Quote the generated names exactly** inside the logical table's `SELECT` (`"AC_NAM_Actor_Name"`, `"pAC_Actor"`): they are case sensitive (see `naming-conventions.md`). Give every column an **unquoted alias in lower case** (`AS actor_name`); dimensions and metrics refer to those aliases, and the semantic view is then easy to use.
3. **One logical table per anchor or nexus**, with the identity column as `PRIMARY KEY`. Its attributes become dimensions (or, if numeric and meant to be summed or averaged, the argument of a metric: `SUM(revenue)`), and a count of the identity is the usual first metric.
4. **A knotted attribute** is the knot's value column of the perspective (`AC_GEN_GEN_Gender` above), not the knot's identity.
5. **The point in time.** The argument has to be an expression. With an expression such as `SYSDATE()` or a fixed `'2024-12-31'::TIMESTAMP_NTZ` the semantic view has one point in time; with a session variable (above) the one who queries chooses it. For the current time, `SYSDATE()` (UTC, which is what a `ChangedAt` holds, and what the `n` perspectives use) is the choice that agrees with the model; the example above used `CURRENT_TIMESTAMP::TIMESTAMP_NTZ`, which is the session's own time zone. **Not tested:** `SYSDATE()` in a semantic view. For something that varies by period, make a snapshot table with an as-of column and use it as the logical table, with the as-of column as a dimension (**not tested**).
6. **Name** the semantic view and give it a `COMMENT`; use the descriptions of the model as the `COMMENT` of the logical tables and the dimensions.

## Other sources for a logical table (not tested)

- The **now** perspective `n{AN}_{Descriptor}` and the **latest** perspective `l{AN}_{Descriptor}` are ordinary views: `actors AS ANCHOR_EXAMPLE_UNI.PUBLIC."nAC_Actor"`. This is the safest source for the current state, and no timestamp is needed.
- **Ties** relate the logical tables of the anchors that they join: a `RELATIONSHIPS` clause between two logical tables on the columns of the tie's roles. The syntax of a relationship over a point-in-time function has not been tried; build the logical table of the tie the same way (a query over `TABLE("p{tie_name}"(...))`), then write the relationship, and run it before relying on it.

## Limits

- A fixed point in time is one per semantic view. A session variable moves the choice to the session, which a client that opens its own session does not have (above).
