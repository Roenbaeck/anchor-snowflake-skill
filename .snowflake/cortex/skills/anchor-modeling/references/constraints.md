# Constraints beyond the keys

Load this when the user states a rule that the model should enforce: "stage names are unique", "no two performances on the same stage on the same day", "an order must not...", or asks how to validate data. The principle is the author's (*Sources* at the end); the SQL is for Snowflake and was run there (the view below, on a small example).

## What the generated model already checks

Do not rebuild these. Snowflake enforces only `NOT NULL`: primary, unique and foreign keys are metadata (declared `RELY`), and the generated **integrity checks** find the rows that break them: `IntegrityViolations` for the model, `ic_<table>` for one table (`generator.md`, *Integrity checks*). They cover duplicate keys, references to nothing, restatements, and in bi and crt temporal duplicates and contradicting assertions. What is left is the user's own rules: a value that must be unique across things, a combination that must not repeat, a range, a rule that spans tables.

## The principle: every time slice

In a historized model **a rule has to hold in every time slice**, not in the rows that happen to be in the table. A stage name that is unique today and was used by another stage last year breaks the rule only if both were in effect at once. So:

- **A table constraint cannot say it**, and Snowflake would not enforce one anyway. In databases that have triggers, the author checks a rule when a row is inserted, only for the slices that row touches, with an `AFTER INSERT` trigger over the point-in-time perspective, rolling back on a violation, and not with `CHECK` constraints (the number of time points to check is very large). Snowflake has no triggers on tables.
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

On a small example it reported exactly the conflict that was there (a name taken by a second stage while the first still had it) and nothing for a name reused after its first holder had changed it. Schema-qualify the names as the model's are. This view is for a uni-temporal model. In bi and crt the assembled view `<attribute name>` lists every assertion (posit and annex rows together), not what is in effect. For a rule over what is in effect in bi or crt, check the perspective, the latest (`l`) or now (`n`) view for the present, or a point-in-time view called with a constant time; a rule over every slice needs the shape above over what the perspective shows, and has not been run.

## A rule across a tie

*No two performances on the same stage on the same day.* The date is an attribute of the performance and the stage is on a tie (`PE_wasHeld_ST_at`). Join the tie to the date attribute twice, to the stage and to the other performance on that stage, with the same day (`DATE_TRUNC('day', ...)` of both) and a different performance identity; use the latest views for non-historized parts, and the same "in effect at that time" test as above for historized ones. The view then does not depend on the order in which the tie and the attributes were loaded, which a trigger on one table would (the author loads identities, then the tie, then the attributes, in one transaction, or adds a second trigger on the tie). This view was not run.

## How to add a rule

1. **Ask what the rule means in time**, in the user's words, with the case that must fail: "Can two stages have the same name at the same time? Can a closed stage give its name to a new one?" A rule that holds only *now* and a rule that holds *in every slice* are different, and the second is stricter.
2. **Write the view** in the shape above (`ic_rule_<name>`, four columns) and keep it in its own script next to the model XML, **not** in the generated script: the generator does not know it, so it survives regeneration only as a separate script.
3. **Run it from the load.** Add it to the finalizer's check next to `IntegrityViolations` (`metadata.md`, *Finalizer*), so that a batch that breaks the rule is marked, not silently accepted. When it finds rows, show them (`ViolationKey` says which) and stop; do not delete rows without asking.
4. **Test it with three loads:** one that must pass, one that must be flagged, and one that reuses a value after it stopped being in effect (passes).
5. **Rules of other shapes** (a value range, a value that must exist, a rule over several tables) have the same form: a violation view that finds the rows that break them in some slice. These shapes are not in the author's examples and have not been run.

## Sources

*Implementing constraints* (`https://www.anchormodeling.com/implementing-constraints/`, with its SQL Server script), *Modified Trigger Logic*, *Temporal Entity Integrity*.
