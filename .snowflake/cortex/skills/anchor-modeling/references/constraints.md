# Constraints beyond the keys

Load this when the user states a rule that the model should enforce: "stage names are unique", "no two performances on the same stage on the same day", "an order must not...", or asks how to validate data. The principle is the author's (*Sources* at the end); the SQL is for Snowflake and was run there (the view below, on a small example).

## What the generated model already checks

On the standard Snowflake tables used here, `NOT NULL` and `CHECK` are enforced, but primary, unique and foreign keys are not. The generated **integrity checks** report key violations: `IntegrityViolations` for the model, `ic_<table>` for one table (`generator.md`, *Integrity checks*). They also cover restatements and, in bi and crt, temporal duplicates and contradicting assertions. What is left is the user's own rules.

## The principle: every time slice

In a historized model **a rule has to hold in every time slice**, not in the rows that happen to be in the table. A stage name that is unique today and was used by another stage last year breaks the rule only if both were in effect at once. So:

- **A row-local `CHECK` can enforce a value range, not a rule across temporal slices.** For temporal uniqueness, check affected intervals or all their change boundaries, not only the inserted row's time: a backfill can overlap a version that starts later. Snowflake has no table triggers.
- **So in Snowflake the rule is a violation view** in the shape of the generated checks, which the load runs after every batch and treats as a failure. A rule over a non-historized attribute needs only the latest view.
- **Do not call a point-in-time function once for each row of another table** (`FROM t, TABLE(p...(t.at))`): Snowflake cannot evaluate it (`generator.md`). The view below needs no function.

## Pattern: a value that must be unique

For uniqueness of the stage name (`ST_NAM_Stage_Name`: owner `ST_NAM_ST_ID`, changing time `ST_NAM_ChangedAt`, value `ST_NAM_Stage_Name`). A row of the view is a violation: two different stages have the name at the same time. A version of another stage is in effect at a time when no later version of that stage has started by then.

```sql
CREATE OR REPLACE VIEW "ic_rule_ST_NAM_Stage_Name_unique" (Construct, Violation, ViolationKey, Occurrences) COPY GRANTS AS
SELECT
    'ST_NAM_Stage_Name',
    'stage name used by two stages at the same time',
    OBJECT_CONSTRUCT('ST_NAM_ST_ID', i."ST_NAM_ST_ID", 'ST_NAM_ChangedAt', i."ST_NAM_ChangedAt", 'ST_NAM_Stage_Name', i."ST_NAM_Stage_Name"),
    COUNT(DISTINCT o."ST_NAM_ST_ID")
FROM "ST_NAM_Stage_Name" i
JOIN "ST_NAM_Stage_Name" o
  ON  o."ST_NAM_Stage_Name" = i."ST_NAM_Stage_Name"
  AND o."ST_NAM_ST_ID" <> i."ST_NAM_ST_ID"
  AND o."ST_NAM_ChangedAt" <= i."ST_NAM_ChangedAt"
WHERE NOT EXISTS (
    SELECT 1 FROM "ST_NAM_Stage_Name" n
    WHERE n."ST_NAM_ST_ID" = o."ST_NAM_ST_ID" AND n."ST_NAM_ChangedAt" > o."ST_NAM_ChangedAt" AND n."ST_NAM_ChangedAt" <= i."ST_NAM_ChangedAt"
)
GROUP BY i."ST_NAM_ST_ID", i."ST_NAM_ChangedAt", i."ST_NAM_Stage_Name";
```

The query scans the starts of **all** versions, not just new rows: an overlap starts at one version's boundary. On a small example it reported a conflict while both stages held a name and did not report reuse after the first stage changed it. Schema-qualify the names as the model's are. This view is for a uni-temporal model. In bi and crt the assembled view lists assertions, not what is in effect. Use `n` for the present, `p` for a chosen changing time, and `t` to also set an assertion cutoff. A rule over every slice needs both timelines and, in crt, a specified positor or reconciliation policy; the uni query does not implement that.

## A rule across a tie

*No two performances on the same stage on the same day.* The date is an attribute of the performance and the stage is on a tie (`PE_wasHeld_ST_at`). Join the tie to the date attribute twice, to the stage and to the other performance on that stage, with the same day (`DATE_TRUNC('day', ...)` of both) and a different performance identity; use the latest views for non-historized parts, and the same "in effect at that time" test as above for historized ones. The view then does not depend on the order in which the tie and the attributes were loaded, which a trigger on one table would (the author loads identities, then the tie, then the attributes, in one transaction, or adds a second trigger on the tie). This view was not run.

## How to add a rule

1. **Ask what the rule means in time**, in the user's words, with the case that must fail: "Can two stages have the same name at the same time? Can a closed stage give its name to a new one?" A rule that holds only *now* and a rule that holds *in every slice* are different, and the second is stricter.
2. **Write the view** in the shape above (`ic_rule_<name>`, four columns) and keep it in its own script next to the model XML, **not** in the generated script: the generator does not know it, so it survives regeneration only as a separate script.
3. **Run it before publication.** Add it to the validation next to `IntegrityViolations` (`metadata.md`). When it finds rows, show them (`ViolationKey` says which) and block publication; do not delete rows without asking.
4. **Test normal use, an equal-start conflict, a backfill whose conflict starts later, and reuse at the preceding interval's exclusive end.** First store `Orion` for stage B at time 20; then load `Orion` for stage A at time 10. The view must flag the overlap beginning at 20.

## Sources

*Implementing constraints* (`https://www.anchormodeling.com/implementing-constraints/`, with its SQL Server script), *Modified Trigger Logic*, *Temporal Entity Integrity*.
