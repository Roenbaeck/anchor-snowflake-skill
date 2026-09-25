# Snowflake DDL Patterns for Anchor Modeling

Templates for uni-temporal Anchor models on Snowflake. Examples use the theatre domain (actors, programs, stages, events). If an `ANCHOR_EXAMPLE.PUBLIC` database exists in the account, it contains a working model built from these patterns.

## 0. General Rules

### Constraints are declared, not enforced

Snowflake enforces only `NOT NULL`. `PRIMARY KEY`, `UNIQUE` and `FOREIGN KEY` are metadata. Consequences:

- **Loads must guarantee uniqueness themselves** (see the loading patterns in SKILL.md). Nothing stops a duplicate row.
- **Declare every PK, UNIQUE and FK constraint with `RELY`.** This tells the optimizer it may trust them, which enables **join elimination**: a perspective query that selects only a few columns skips the LEFT JOINs to the attribute tables it does not need. This is what keeps wide Anchor perspectives cheap.
- Because the optimizer trusts `RELY` constraints, a load that breaks them can produce wrong query results. Run the duplicate checks in section 11 after loading.

### Data types

- All Snowflake integer types (`int`, `smallint`, `tinyint`, `bigint`) are synonyms for `NUMBER(38,0)`. Use `int` for anchor, nexus and knot identities unless there is a reason not to.
- Use `timestamp_ntz(9)` for every `ChangedAt` column and store values **in UTC**. Perspectives compare against `sysdate()` (UTC, NTZ) so results do not depend on the session time zone, and task sessions and interactive sessions agree.

### Clustering

Do **not** add `CLUSTER BY` by default. Automatic clustering consumes credits and gives nothing on small tables; knots and most attribute tables never reach a size where it helps. Add `CLUSTER BY ({owner_id_column})` only to individual tables that are very large (multi-terabyte) and show poor pruning in query profiles.

### Idempotent DDL

`CREATE TABLE IF NOT EXISTS` silently does nothing when a table with that name already exists, **even if its structure differs**. When extending a model, check existing definitions (`DESCRIBE TABLE`) rather than assuming a re-run applied changes. Anchor Modeling adds new tables instead of altering existing ones, so this is normally safe. Perspectives are the exception, and they use `CREATE OR REPLACE ... COPY GRANTS`.

## 1. Knot Table

```sql
CREATE TABLE IF NOT EXISTS {schema}.{KNT}_{Descriptor} (
    {KNT}_ID int not null,
    {KNT}_{Descriptor} {data_type} not null,
    Metadata_{KNT} int not null,
    constraint pk{KNT}_{Descriptor} primary key (
        {KNT}_ID
    ) rely,
    constraint uq{KNT}_{Descriptor} unique (
        {KNT}_{Descriptor}
    ) rely
);
COMMENT ON TABLE {schema}.{KNT}_{Descriptor} IS '{description}';
```

Example (Rating knot):
```sql
CREATE TABLE IF NOT EXISTS public.RAT_Rating (
    RAT_ID int not null,
    RAT_Rating varchar(42) not null,
    Metadata_RAT int not null,
    constraint pkRAT_Rating primary key (RAT_ID) rely,
    constraint uqRAT_Rating unique (RAT_Rating) rely
);
```

**Checksum option** (for long knot values): Snowflake column defaults cannot reference other columns, so a checksum cannot be computed by the table. Add a plain column and fill it during loading:
```sql
    {KNT}_Checksum number(19,0) not null,   -- loaded as hash({KNT}_{Descriptor})
```
Put the unique constraint on `{KNT}_Checksum` instead of the value, and look up knot IDs by `hash(source_value)` during loads.

## 2. Anchor Table

```sql
CREATE SEQUENCE IF NOT EXISTS {schema}.{AN}_{Descriptor}_ID_SEQ START 1 INCREMENT 1;
CREATE TABLE IF NOT EXISTS {schema}.{AN}_{Descriptor} (
    {AN}_ID int default {schema}.{AN}_{Descriptor}_ID_SEQ.nextval not null,
    Metadata_{AN} int not null,
    constraint pk{AN}_{Descriptor} primary key (
        {AN}_ID
    ) rely
);
COMMENT ON TABLE {schema}.{AN}_{Descriptor} IS '{description}';
```

Example (Actor anchor):
```sql
CREATE SEQUENCE IF NOT EXISTS public.AC_Actor_ID_SEQ START 1 INCREMENT 1;
CREATE TABLE IF NOT EXISTS public.AC_Actor (
    AC_ID int default public.AC_Actor_ID_SEQ.nextval not null,
    Metadata_AC int not null,
    constraint pkAC_Actor primary key (AC_ID) rely
);
```

## 3. Nexus Table

Uses a sequence (not IDENTITY) so that IDs drawn from the sequence during a load can be inserted explicitly into the nexus and all its attribute tables.

```sql
CREATE SEQUENCE IF NOT EXISTS {schema}.{NX}_{Descriptor}_ID_SEQ START 1 INCREMENT 1;
CREATE TABLE IF NOT EXISTS {schema}.{NX}_{Descriptor} (
    {NX}_ID int default {schema}.{NX}_{Descriptor}_ID_SEQ.nextval not null,
    {role1_type}_ID_{role1_name} int not null,
    {role2_type}_ID_{role2_name} int not null,
    -- ... more roles ...
    Metadata_{NX} int not null,
    constraint {NX}_{Descriptor}_fk{role1_type}_{role1_name} foreign key (
        {role1_type}_ID_{role1_name}
    ) references {schema}.{role1_table}({role1_type}_ID) rely,
    -- ... more FK constraints ...
    constraint pk{NX}_{Descriptor} primary key (
        {NX}_ID
    ) rely
);
COMMENT ON TABLE {schema}.{NX}_{Descriptor} IS '{description}';
```

Example (Event nexus):
```sql
CREATE SEQUENCE IF NOT EXISTS public.EV_Event_ID_SEQ START 1 INCREMENT 1;
CREATE TABLE IF NOT EXISTS public.EV_Event (
    EV_ID int default public.EV_Event_ID_SEQ.nextval not null,
    ST_ID_wasHeldAt int not null,
    PR_ID_wasPlayed int not null,
    ETY_ID_of int not null,
    Metadata_EV int not null,
    constraint EV_Event_fkST_wasHeldAt foreign key (ST_ID_wasHeldAt) references public.ST_Stage(ST_ID) rely,
    constraint EV_Event_fkPR_wasPlayed foreign key (PR_ID_wasPlayed) references public.PR_Program(PR_ID) rely,
    constraint EV_Event_fkETY_of foreign key (ETY_ID_of) references public.ETY_EventType(ETY_ID) rely,
    constraint pkEV_Event primary key (EV_ID) rely
);
```

> **Why not IDENTITY?** Snowflake IDENTITY columns do not accept explicit values on INSERT. A load stages rows with IDs taken from the sequence (`seq.nextval`) and inserts those same IDs into the nexus and all its attribute tables. A sequence default allows this; IDENTITY does not.

Give every nexus a static **identifier attribute** holding the source key of the event (like anchors). Incremental loads use it to recognise events that are already loaded.

## 4. Attribute Tables

### 4a. Static Attribute

```sql
CREATE TABLE IF NOT EXISTS {schema}.{AN}_{ATR}_{AnchorDesc}_{AttrDesc} (
    {AN}_{ATR}_{AN}_ID int not null,
    {AN}_{ATR}_{AnchorDesc}_{AttrDesc} {data_type} not null,
    Metadata_{AN}_{ATR} int not null,
    constraint fk{AN}_{ATR}_{AnchorDesc}_{AttrDesc} foreign key (
        {AN}_{ATR}_{AN}_ID
    ) references {schema}.{AN}_{AnchorDesc}({AN}_ID) rely,
    constraint pk{AN}_{ATR}_{AnchorDesc}_{AttrDesc} primary key (
        {AN}_{ATR}_{AN}_ID
    ) rely
);
```

Example (Program Name, static):
```sql
CREATE TABLE IF NOT EXISTS public.PR_NAM_Program_Name (
    PR_NAM_PR_ID int not null,
    PR_NAM_Program_Name varchar(42) not null,
    Metadata_PR_NAM int not null,
    constraint fkPR_NAM_Program_Name foreign key (PR_NAM_PR_ID) references public.PR_Program(PR_ID) rely,
    constraint pkPR_NAM_Program_Name primary key (PR_NAM_PR_ID) rely
);
```

### 4b. Historized Attribute

Same as static but adds `ChangedAt` to the PK:

```sql
CREATE TABLE IF NOT EXISTS {schema}.{AN}_{ATR}_{AnchorDesc}_{AttrDesc} (
    {AN}_{ATR}_{AN}_ID int not null,
    {AN}_{ATR}_{AnchorDesc}_{AttrDesc} {data_type} not null,
    {AN}_{ATR}_ChangedAt timestamp_ntz(9) not null,
    Metadata_{AN}_{ATR} int not null,
    constraint fk{AN}_{ATR}_{AnchorDesc}_{AttrDesc} foreign key (
        {AN}_{ATR}_{AN}_ID
    ) references {schema}.{AN}_{AnchorDesc}({AN}_ID) rely,
    constraint pk{AN}_{ATR}_{AnchorDesc}_{AttrDesc} primary key (
        {AN}_{ATR}_{AN}_ID,
        {AN}_{ATR}_ChangedAt
    ) rely
);
```

Example (Actor Name, historized):
```sql
CREATE TABLE IF NOT EXISTS public.AC_NAM_Actor_Name (
    AC_NAM_AC_ID int not null,
    AC_NAM_Actor_Name varchar(42) not null,
    AC_NAM_ChangedAt timestamp_ntz(9) not null,
    Metadata_AC_NAM int not null,
    constraint fkAC_NAM_Actor_Name foreign key (AC_NAM_AC_ID) references public.AC_Actor(AC_ID) rely,
    constraint pkAC_NAM_Actor_Name primary key (AC_NAM_AC_ID, AC_NAM_ChangedAt) rely
);
```

### 4c. Knotted Static Attribute

FK to the knot instead of a value column:

```sql
CREATE TABLE IF NOT EXISTS {schema}.{AN}_{ATR}_{AnchorDesc}_{AttrDesc} (
    {AN}_{ATR}_{AN}_ID int not null,
    {AN}_{ATR}_{KNT}_ID int not null,
    Metadata_{AN}_{ATR} int not null,
    constraint fk{AN}_{ATR}_{AnchorDesc}_{AttrDesc} foreign key (
        {AN}_{ATR}_{AN}_ID
    ) references {schema}.{AN}_{AnchorDesc}({AN}_ID) rely,
    constraint fk_{AN}_{ATR}_{KNT} foreign key (
        {AN}_{ATR}_{KNT}_ID
    ) references {schema}.{KNT}_{KnotDesc}({KNT}_ID) rely,
    constraint pk{AN}_{ATR}_{AnchorDesc}_{AttrDesc} primary key (
        {AN}_{ATR}_{AN}_ID
    ) rely
);
```

Example (Actor Gender, knotted static):
```sql
CREATE TABLE IF NOT EXISTS public.AC_GEN_Actor_Gender (
    AC_GEN_AC_ID int not null,
    AC_GEN_GEN_ID int not null,
    Metadata_AC_GEN int not null,
    constraint fkAC_GEN_Actor_Gender foreign key (AC_GEN_AC_ID) references public.AC_Actor(AC_ID) rely,
    constraint fk_AC_GEN_GEN foreign key (AC_GEN_GEN_ID) references public.GEN_Gender(GEN_ID) rely,
    constraint pkAC_GEN_Actor_Gender primary key (AC_GEN_AC_ID) rely
);
```

### 4d. Knotted Historized Attribute

FK to the knot, plus `ChangedAt` in the PK:

```sql
CREATE TABLE IF NOT EXISTS {schema}.{AN}_{ATR}_{AnchorDesc}_{AttrDesc} (
    {AN}_{ATR}_{AN}_ID int not null,
    {AN}_{ATR}_{KNT}_ID int not null,
    {AN}_{ATR}_ChangedAt timestamp_ntz(9) not null,
    Metadata_{AN}_{ATR} int not null,
    constraint fk{AN}_{ATR}_{AnchorDesc}_{AttrDesc} foreign key (
        {AN}_{ATR}_{AN}_ID
    ) references {schema}.{AN}_{AnchorDesc}({AN}_ID) rely,
    constraint fk_{AN}_{ATR}_{KNT} foreign key (
        {AN}_{ATR}_{KNT}_ID
    ) references {schema}.{KNT}_{KnotDesc}({KNT}_ID) rely,
    constraint pk{AN}_{ATR}_{AnchorDesc}_{AttrDesc} primary key (
        {AN}_{ATR}_{AN}_ID,
        {AN}_{ATR}_ChangedAt
    ) rely
);
```

## 5. Tie Table

`{tie_name}` is the full tie name composed from roles, e.g. `AC_part_PR_in_RAT_got`.

### 5a. Static Tie

```sql
CREATE TABLE IF NOT EXISTS {schema}.{tie_name} (
    {type1}_ID_{role1} int not null,
    {type2}_ID_{role2} int not null,
    -- knot roles if any:
    {KNT}_ID_{knot_role} int not null,
    Metadata_{tie_name} int not null,
    constraint pk{tie_name} primary key (
        {identifier_role_columns}
    ) rely,
    constraint {tie_name}_fk{type1}_{role1} foreign key ({type1}_ID_{role1})
        references {schema}.{type1_table}({type1}_ID) rely,
    constraint {tie_name}_fk{type2}_{role2} foreign key ({type2}_ID_{role2})
        references {schema}.{type2_table}({type2}_ID) rely,
    constraint {tie_name}_fk{KNT}_{knot_role} foreign key ({KNT}_ID_{knot_role})
        references {schema}.{KNT}_{KnotDesc}({KNT}_ID) rely
);
COMMENT ON TABLE {schema}.{tie_name} IS '{description}';
```

Example (actor part in program, got rating):
```sql
CREATE TABLE IF NOT EXISTS public.AC_part_PR_in_RAT_got (
    AC_ID_part int not null,
    PR_ID_in int not null,
    RAT_ID_got int not null,
    Metadata_AC_part_PR_in_RAT_got int not null,
    constraint pkAC_part_PR_in_RAT_got primary key (AC_ID_part, PR_ID_in) rely,
    constraint AC_part_PR_in_RAT_got_fkAC_part foreign key (AC_ID_part) references public.AC_Actor(AC_ID) rely,
    constraint AC_part_PR_in_RAT_got_fkPR_in foreign key (PR_ID_in) references public.PR_Program(PR_ID) rely,
    constraint AC_part_PR_in_RAT_got_fkRAT_got foreign key (RAT_ID_got) references public.RAT_Rating(RAT_ID) rely
);
```

### 5b. Historized Tie

Adds `{tie_name}_ChangedAt` to the PK:

```sql
CREATE TABLE IF NOT EXISTS {schema}.{tie_name} (
    {type1}_ID_{role1} int not null,
    {type2}_ID_{role2} int not null,
    {tie_name}_ChangedAt timestamp_ntz(9) not null,
    Metadata_{tie_name} int not null,
    constraint pk{tie_name} primary key (
        {identifier_role_columns},
        {tie_name}_ChangedAt
    ) rely,
    constraint {tie_name}_fk{type1}_{role1} foreign key ({type1}_ID_{role1})
        references {schema}.{type1_table}({type1}_ID) rely,
    constraint {tie_name}_fk{type2}_{role2} foreign key ({type2}_ID_{role2})
        references {schema}.{type2_table}({type2}_ID) rely
    -- plus FK constraints for any knot roles, as in 5a
);
```

## 6. Latest Perspective (l prefix)

A view joining the anchor/nexus to all its attributes and their knots.

- **Static attributes** are joined directly on their PK. With `RELY` constraints, Snowflake can eliminate these joins when their columns are not selected.
- **Historized attributes** are joined through a derived table that keeps the latest row per owner with `QUALIFY`. Do not use a correlated `max(ChangedAt)` subquery in the join condition: Snowflake's support for correlated subqueries is limited, and `QUALIFY` is the idiomatic pattern.

```sql
CREATE OR REPLACE VIEW {schema}.l{AN}_{Descriptor} COPY GRANTS
COMMENT = '{description}'
AS
SELECT
    {AN}.{AN}_ID,
    {AN}.Metadata_{AN},
    -- For each attribute:
    {ATR}.{AN}_{ATR}_{AN}_ID,
    {ATR}.Metadata_{AN}_{ATR},
    {ATR}.{AN}_{ATR}_ChangedAt,                        -- if historized
    {ATR}.{AN}_{ATR}_{AnchorDesc}_{AttrDesc},          -- if not knotted
    k{ATR}.{KNT}_{KnotDesc} AS {AN}_{ATR}_{KNT}_{KnotDesc},  -- if knotted
    {ATR}.{AN}_{ATR}_{KNT}_ID                          -- if knotted
FROM
    {schema}.{AN}_{Descriptor} {AN}
-- Static attribute: plain LEFT JOIN
LEFT JOIN {schema}.{AN}_{ATR}_{AnchorDesc}_{AttrDesc} {ATR}
    ON {ATR}.{AN}_{ATR}_{AN}_ID = {AN}.{AN}_ID
-- Historized attribute: latest row per owner
LEFT JOIN (
    SELECT *
    FROM {schema}.{AN}_{ATR}_{AnchorDesc}_{AttrDesc}
    QUALIFY ROW_NUMBER() OVER (
        PARTITION BY {AN}_{ATR}_{AN}_ID
        ORDER BY {AN}_{ATR}_ChangedAt DESC
    ) = 1
) {ATR}
    ON {ATR}.{AN}_{ATR}_{AN}_ID = {AN}.{AN}_ID
-- Knotted attribute: LEFT JOIN the knot (aliased per attribute; one knot can serve several attributes)
LEFT JOIN {schema}.{KNT}_{KnotDesc} k{ATR}
    ON k{ATR}.{KNT}_ID = {ATR}.{AN}_{ATR}_{KNT}_ID
;
```

Add column-level comments on knotted and attribute value columns with a column list in the `CREATE VIEW` (`CREATE OR REPLACE VIEW v (col1 COMMENT '...', col2 COMMENT '...') COPY GRANTS ...`). Comments added afterwards with `ALTER VIEW` are lost the next time the view is replaced.

## 7. Point-in-Time Perspective (p prefix)

A table function taking `changingTimepoint timestamp_ntz(9)` (UTC). Same joins as the latest perspective, but each historized attribute is first filtered to `ChangedAt <= changingTimepoint`:

```sql
CREATE OR REPLACE FUNCTION {schema}.p{AN}_{Descriptor} (
    changingTimepoint timestamp_ntz(9)
)
RETURNS TABLE (
    {AN}_ID int,
    Metadata_{AN} int,
    -- ... one entry per column of the latest perspective, same names and types ...
    {AN}_{ATR}_{AN}_ID int,
    Metadata_{AN}_{ATR} int,
    {AN}_{ATR}_ChangedAt timestamp_ntz(9),
    {AN}_{ATR}_{AnchorDesc}_{AttrDesc} {data_type}
)
LANGUAGE SQL
AS $$
    SELECT
        {AN}.{AN}_ID,
        {AN}.Metadata_{AN},
        {ATR}.{AN}_{ATR}_{AN}_ID,
        {ATR}.Metadata_{AN}_{ATR},
        {ATR}.{AN}_{ATR}_ChangedAt,
        {ATR}.{AN}_{ATR}_{AnchorDesc}_{AttrDesc}
    FROM
        {schema}.{AN}_{Descriptor} {AN}
    LEFT JOIN (
        SELECT *
        FROM {schema}.{AN}_{ATR}_{AnchorDesc}_{AttrDesc}
        WHERE {AN}_{ATR}_ChangedAt <= changingTimepoint
        QUALIFY ROW_NUMBER() OVER (
            PARTITION BY {AN}_{ATR}_{AN}_ID
            ORDER BY {AN}_{ATR}_ChangedAt DESC
        ) = 1
    ) {ATR}
        ON {ATR}.{AN}_{ATR}_{AN}_ID = {AN}.{AN}_ID
    -- static and knotted attributes/knots joined exactly as in the latest perspective
$$;
```

## 8. Now Perspective (n prefix)

`sysdate()` returns the current time in UTC as `timestamp_ntz`, matching how `ChangedAt` is stored. Do not use `current_timestamp()::timestamp_ntz`, which gives the session's local wall-clock time.

```sql
CREATE OR REPLACE VIEW {schema}.n{AN}_{Descriptor} COPY GRANTS
COMMENT = '{description}'
AS
SELECT * FROM TABLE({schema}.p{AN}_{Descriptor}(sysdate()));
```

## 9. Difference Perspective (d prefix)

A table function taking `intervalStart` and `intervalEnd`:

```sql
CREATE OR REPLACE FUNCTION {schema}.d{AN}_{Descriptor} (
    intervalStart timestamp_ntz(9),
    intervalEnd timestamp_ntz(9),
    selection varchar DEFAULT NULL
)
RETURNS TABLE (
    inspectedTimepoint timestamp_ntz(9),
    mnemonic varchar,
    /* all columns from latest perspective */
)
LANGUAGE SQL
AS $$
    -- UNION of one SELECT per historized attribute:
    SELECT DISTINCT
        h{ATR}.{AN}_{ATR}_ChangedAt AS inspectedTimepoint,
        '{ATR}' AS mnemonic,
        p{AN}.*
    FROM
        {schema}.{AN}_{ATR}_{AnchorDesc}_{AttrDesc} h{ATR},
        TABLE({schema}.p{AN}_{Descriptor}(h{ATR}.{AN}_{ATR}_ChangedAt)) p{AN}
    WHERE
        (selection IS NULL OR selection LIKE '%{ATR}%')
    AND h{ATR}.{AN}_{ATR}_ChangedAt BETWEEN intervalStart AND intervalEnd
    AND p{AN}.{AN}_ID = h{ATR}.{AN}_{ATR}_{AN}_ID
    UNION
    -- ... repeat for each historized attribute ...
$$;
```

## 10. Tie Perspectives

Ties get latest, point-in-time, now and difference perspectives with the same patterns. For a historized tie, the latest perspective keeps one row per identifier-role combination:

```sql
CREATE OR REPLACE VIEW {schema}.l{tie_name} COPY GRANTS AS
SELECT *
FROM {schema}.{tie_name}
QUALIFY ROW_NUMBER() OVER (
    PARTITION BY {identifier_role_columns}
    ORDER BY {tie_name}_ChangedAt DESC
) = 1;
```

Join knot roles to their knot tables to expose readable values, as in attribute perspectives.

## 11. Integrity Checks

Constraints are not enforced, so verify after every load. Each query should return no rows:

```sql
-- Duplicate identities (anchor, nexus, knot)
SELECT {AN}_ID FROM {schema}.{AN}_{Descriptor} GROUP BY 1 HAVING count(*) > 1;

-- Duplicate attribute keys (add ChangedAt for historized attributes)
SELECT {AN}_{ATR}_{AN}_ID, {AN}_{ATR}_ChangedAt
FROM {schema}.{AN}_{ATR}_{AnchorDesc}_{AttrDesc}
GROUP BY 1, 2 HAVING count(*) > 1;

-- Orphaned attribute rows
SELECT a.{AN}_{ATR}_{AN}_ID
FROM {schema}.{AN}_{ATR}_{AnchorDesc}_{AttrDesc} a
LEFT JOIN {schema}.{AN}_{Descriptor} an ON an.{AN}_ID = a.{AN}_{ATR}_{AN}_ID
WHERE an.{AN}_ID IS NULL;

-- Duplicate knot values
SELECT {KNT}_{Descriptor} FROM {schema}.{KNT}_{Descriptor} GROUP BY 1 HAVING count(*) > 1;
```
