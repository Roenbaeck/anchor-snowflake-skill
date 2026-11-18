# Performance

Load this when a user asks why a query is slow, plans a large load, asks how to query history over many dates, or wants to know whether Anchor's many narrow tables will be fast (Q2, L1). The principles are the author's (*Sources* at the end), mostly from tests on SQL Server; the as-of join below was run on Snowflake. Say which figure is from where, and measure on the user's data before promising anything.

## Why narrow tables are fast, and what it depends on

- **Table elimination.** A query that names a few attributes of an anchor needs only those attribute tables: the optimizer drops the left joins to the others. It works only when **primary and foreign keys are declared** and the query is written so that the join can be proved harmless (a left join from the anchor to the attribute on its key). The generated tables declare every key `RELY` for exactly this (`SKILL.md`, *Constraints Are Not Enforced*), so the optimizer trusts them: a load that breaks a key then gives wrong results, not an error, and the integrity checks matter (`generator.md`). Look at the **query profile** after rewriting a query: a rewrite that looks cleverer can lose the elimination.
- **Query the latest, now and point-in-time perspectives** and name only the columns needed. A `SELECT *` over a perspective reads every attribute table it joins.
- **Latest versions** are found with a `MAX` subselect or `QUALIFY ROW_NUMBER()`; the generated perspectives already do. The author's measurements of SQL Server's variants (`OUTER APPLY`, `TOP 1`, `ROW_NUMBER` with a redundant `GROUP BY`) do not carry over: do not copy them, test what the profile shows.
- **Bulk loading:** load files with `COPY INTO` from a stage and set-based inserts that filter what exists, not row by row (`SKILL.md`, *Data Loading Workflow*). The author's experience on SQL Server was that generated bulk packages were "magnitudes faster" than triggers on the latest views.
- **Do not call a point-in-time function once for each row of another table** (`FROM t, TABLE(p...(t.at))`): Snowflake cannot evaluate it. For many reference times use the twine below.

## As-of joins: many reference times against a long history

*For every row in a fact table, what was the value in effect at its time?* Do not use a `BETWEEN` range join (optimizers plan them badly) and be careful with a join that picks the latest row per row (`QUALIFY ROW_NUMBER() ... ORDER BY h.t DESC` after a `h.t <= r.t` join): it is correct and is the first thing to try, but it multiplies the rows before it picks. Large inputs want a **twine**: one ordered pass over the history rows and the reference rows together. The author reports a query that was cancelled after four hours that ran in seconds as a twine, each table scanned once.

```sql
-- history h(id, t, ...) and reference rows r(id, t): the version in effect at each r.t
WITH points AS (
    SELECT 'H' AS origin, id, t FROM h
    UNION ALL
    SELECT 'R', id, t FROM r
), carried AS (
    SELECT origin, id, t,
           MAX(CASE WHEN origin = 'H' THEN t END)
               OVER (PARTITION BY id ORDER BY t, CASE origin WHEN 'H' THEN 0 ELSE 1 END
                     ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS since
    FROM points
)
SELECT r.id, r.t, h.v                -- one exact key lookup in the history per reference row
FROM carried c
JOIN r ON r.id = c.id AND r.t = c.t AND c.origin = 'R'
LEFT JOIN h ON h.id = c.id AND h.t = c.since;
```

(A sketch of the idea, not the author's code: the history rows sort first at equal times, so a version that starts at `t` counts at `t`. Both the history and the reference need one row per identity and time, or rows join twice; `since` is null for a reference that comes before any history. For an attribute of an anchor, `h` is its table or the knot-resolved values; `id` is the owner column and `t` the `ChangedAt`.) It was checked on Snowflake against the row-number join on 8,000 reference rows over about 20,000 history rows, with the same result for every row; speed was not measured, so compare the two on the user's data. Use it also to **load temporal dimensions**: history stays in the model, and the twine finds the historically correct relation, instead of slowly changing dimension types.

## Sources

*Let's Twine Again*, *Information in Effect and Performance*, *Four features for performance*, *Optimizing BETWEEN joins*, *Temporal Dimensional Modeling*, *Performance improvements*, and the 2007 cheat sheet (indexing, collapsing views), at `https://www.anchormodeling.com/`.
