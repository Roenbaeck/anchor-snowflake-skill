# Performance

Load this when a user asks why a query is slow, plans a large load, asks how to query history over many dates, or wants to know whether Anchor's many narrow tables will be fast (Q2, L1). The principles are the author's (*Sources* at the end). His measurements are from SQL Server (2008 to 2017) and Vertica, none from Snowflake; the as-of join below was run on Snowflake, but its speed was not measured. Say which figure is from where, and measure on the user's data before promising anything.

## When a narrow-table model wins, and when it does not

The author's own experiment (288 tests, SQL Server 2008) compared an anchor database, queried through its latest view, with one wide table holding the same data. The more of these hold, the better the anchor model does, and one held strongly enough can be enough: **history** is kept, the data is **sparse**, values come from **few distinct values** (knots), there are **few identities relative to the data**, there are **many rows**, there are **many attributes** in the model, **few of them are queried at once**, and the query's **conditions bound the result**. The other side of the same results:

- A query that reads **all the attributes with no conditions** was slower, from about 2 to 26 times. If a user's main workload is wide extracts of everything, say so, and put the extract in a table or dynamic table that is refreshed instead of a view over the narrow tables.
- The narrow model is *larger* than the wide table when values are tiny, because the key repeats in every attribute table (6:1 in the worst scenario), and smaller with sparsity, history and shared values (1:2 in the best).
- Where it wins, it wins up to about 14 times. The experiment covers one DBMS and one machine. Fewer I/O and table elimination are the reasons the author gives, which suits I/O-bound warehouses; Snowflake's columnar storage already reads only the columns a query names, so **do not promise the same margin there**: test.

## Why narrow tables can be fast on Snowflake, and what it depends on

- **Join elimination.** The optimizer can skip joins when it can prove they do not affect the result. The generated DDL uses `RELY` to let Snowflake trust declared keys. Validate the candidate before consumers can query it: an invalid key can produce a wrong result rather than an error (`metadata.md`). Check the query profile after a rewrite.
- **Query the latest, now and point-in-time perspectives** and name only the columns needed. A `SELECT *` over a perspective reads every attribute table it joins.
- **Latest versions** are found with a `MAX` subselect or `QUALIFY ROW_NUMBER()`; the generated perspectives already do. The author's measurements of SQL Server's variants (`OUTER APPLY`, `TOP 1`, `ROW_NUMBER` with a redundant `GROUP BY`) do not carry over: do not copy them, test what the profile shows.
- **No indexes to design.** The author's advice on clustered keys (owner ascending, time descending) and partitioning is for SQL Server. Snowflake keeps micro-partitions itself; a clustering key on a very large attribute or tie table is something to measure, not a default.
- **Bulk loading:** load files with `COPY INTO` from a stage and set-based inserts that filter what exists, not row by row (`SKILL.md`, *Data Loading Workflow*). The author's experience on SQL Server was that generated bulk packages were "magnitudes faster" than triggers on the latest views (no figures).
- **Do not call a point-in-time function once for each row of another table** (`FROM t, TABLE(p...(t.at))`): Snowflake cannot evaluate the generated function this way. For many reference times, try `ASOF JOIN` or a twine.

## As-of joins: many reference times against a long history

*For every row of a table, what was the value in effect at that row's time?* Two situations, and the technique differs.

- **The time is a constant** ("as of 2021-06-30"): use the point-in-time perspective of each table, then join as usual. For an outer join, say *why* a value is missing instead of showing a bare null: no history at all for the identity, or history that starts only after the time asked about.
- **The time comes from another table** (a purchase date for each purchase): this is a *temporal dependent join*. For a single resolved history, start with Snowflake's native [`ASOF JOIN`](https://docs.snowflake.com/en/sql-reference/constructs/asof-join), which retains each driving row and finds the closest history row at or before it:

  ```sql
  SELECT r.id, r.t, h.v
  FROM r
  ASOF JOIN h
    MATCH_CONDITION (r.t >= h.t)
    ON r.id = h.id;
  ```

  Validate one history row per `(id, t)`; Snowflake can choose nondeterministically between tied matches. For bi or crt, resolve the assertion cutoff and positor before applying this single-history example; do not use raw posits and annexes as `h`. For explicit interval boundaries or several histories, compare an interval join or the twine below and profile on the user's data. The `ASOF JOIN` example uses documented syntax and was not benchmarked here.

**The twine** replaces the inequality join with one ordered pass over the history rows and the reference rows together: tag each row with the timeline it came from, put both on one line sorted by (identity, time), and let a window function carry the latest history time forward to each reference row. No end dates are needed, which suits insert-only warehouses. The author reports a query that was cancelled after four hours that ran in seconds as a twine (SQL Server).

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

(A sketch of the idea, not the author's code: the history rows sort first at equal times, so a version that starts at `t` counts at `t`. Both the history and the reference need one row per identity and time, or rows join twice; `since` is null for a reference that comes before any history. For an attribute of an anchor, `h` is its table or the knot-resolved values; `id` is the owner column and `t` the `ChangedAt`.) It was checked on Snowflake against the row-number join on 8,000 reference rows over about 20,000 history rows, with the same result for every row; speed was not measured, so compare the two on the user's data.

Rules for choosing and writing one:

1. **Pick the variant by cardinality.** One driving row per identity and many history rows: the author's *specialized* twine takes the value of the *preceding row* of the sorted line (`LAG` over the same partition and order), so no second join is needed. Many driving rows per identity (the sketch above, his *generalized* twine): carry the history time forward with the conditional running `MAX`, then fetch the value by a key lookup. **In his many-to-many test (4 million against 4 million rows, SQL Server) the twine was not faster** (49% against 51% of the batch cost): compare on the user's data.
2. **The cost is the sort.** The combined set is sorted by (identity, time); in his one-to-many test the sort was a third of the plan. On Snowflake, watch the profile for spilling to local or remote storage on large inputs, and for an `UNION ALL` that is not the cheap part you expect.
3. **Several timelines in one pass:** give each history table its own timeline number, the driving rows number 0, one conditional running `MAX` for each, and one key lookup for each. Cast the time types to the **most granular** one first, or the `UNION ALL` may fail or convert implicitly.
4. **Ties at equal times** are decided by the sort: make the order explicit as above, and decide whether a change at the same time as the reference counts (it should).

## Temporal dimensions

A dimension is an immutable key table plus one or more history tables (key, valid-from, properties); the fact references the key. Nothing is ever updated, and facts and dimensions stay independent in time. The author's paper (SQL Server 2017, 52 million facts, 21.5 million dimension rows) asks three questions: **TIY** (facts show the *current* value), **YIT** (the value *at a chosen date*) and **TOY** (the value in effect *when the fact happened*). TIY and YIT are a point-in-time view; TOY is a twine, not a per-fact function call (that estimated billions of operations and one run was halted after four hours; the twine took 53.5 s). Slowly changing dimension types answer one question each: type 1 only today, type 2 historically correct values but slow writes (a twine is needed to find the version in effect when loading too), types 5 and 7 fast reads but heavy writes. Where current *and* arbitrary-date values are both common and writes must stay fast, keep the history in the model and build the dimension as a view, or a table refreshed from it. His timings are SQL Server's, so use them for the shape of the trade-off, not the numbers.

## Sources

*Let's Twine Again*, *Information in Effect and Performance*, *Temporal joins and twines* (2023 slides), *Turning Horrible Joins into Wonderful Speedups*, *Temporal Dimensional Modeling* (paper), *Anchor Modeling* (the 2010 DKE paper, §7 and §8), *Four features for performance*, *Optimizing BETWEEN joins*, *Performance improvements*, and the 2007 cheat sheet (indexing, collapsing views), at `https://www.anchormodeling.com/` and `.../publications/`.
