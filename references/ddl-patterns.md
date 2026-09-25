# Snowflake DDL Patterns for Anchor Modeling

All patterns are extracted from the working `ANCHOR_EXAMPLE.PUBLIC` database. Use these as templates.

## 1. Knot Table

```sql
CREATE TABLE IF NOT EXISTS {schema}.{KNT}_{Descriptor} (
    {KNT}_ID {identity_type} not null,
    {KNT}_{Descriptor} {data_type} not null,
    Metadata_{KNT} int not null,
    constraint pk{KNT}_{Descriptor} primary key (
        {KNT}_ID
    ),
    constraint uq{KNT}_{Descriptor} unique (
        {KNT}_{Descriptor}
    )
) CLUSTER BY ({KNT}_ID);
COMMENT ON TABLE {schema}.{KNT}_{Descriptor} IS '{description}';
```

Example (Rating knot):
```sql
CREATE TABLE IF NOT EXISTS public.RAT_Rating (
    RAT_ID tinyint not null,
    RAT_Rating varchar(42) not null,
    Metadata_RAT int not null,
    constraint pkRAT_Rating primary key (RAT_ID),
    constraint uqRAT_Rating unique (RAT_Rating)
) CLUSTER BY (RAT_ID);
```

If `checksum` is enabled, add a computed column:
```sql
    {KNT}_Checksum numeric(19,0) default hash({KNT}_{Descriptor}),
```
And make the unique constraint on the checksum instead of the value.

## 2. Anchor Table

```sql
CREATE SEQUENCE IF NOT EXISTS {schema}.{AN}_{Descriptor}_ID_SEQ START 1 INCREMENT 1;
CREATE TABLE IF NOT EXISTS {schema}.{AN}_{Descriptor} (
    {AN}_ID {identity_type} default {schema}.{AN}_{Descriptor}_ID_SEQ.nextval not null,
    Metadata_{AN} int not null,
    constraint pk{AN}_{Descriptor} primary key (
        {AN}_ID
    )
) CLUSTER BY ({AN}_ID);
COMMENT ON TABLE {schema}.{AN}_{Descriptor} IS '{description}';
```

Example (Actor anchor):
```sql
CREATE SEQUENCE IF NOT EXISTS public.AC_Actor_ID_SEQ START 1 INCREMENT 1;
CREATE TABLE IF NOT EXISTS public.AC_Actor (
    AC_ID int default public.AC_Actor_ID_SEQ.nextval not null,
    Metadata_AC int not null,
    constraint pkAC_Actor primary key (AC_ID)
) CLUSTER BY (AC_ID);
```

## 3. Nexus Table

Uses a sequence (not IDENTITY) so that explicit IDs can be inserted during bulk loads with staging tables.

```sql
CREATE SEQUENCE IF NOT EXISTS {schema}.{NX}_{Descriptor}_ID_SEQ START 1 INCREMENT 1;
CREATE TABLE IF NOT EXISTS {schema}.{NX}_{Descriptor} (
    {NX}_ID {identity_type} default {schema}.{NX}_{Descriptor}_ID_SEQ.nextval not null,
    {role1_type}_ID_{role1_name} {role1_id_type} not null,
    {role2_type}_ID_{role2_name} {role2_id_type} not null,
    -- ... more roles ...
    constraint {NX}_{Descriptor}_fk{role1_type}_{role1_name} foreign key (
        {role1_type}_ID_{role1_name}
    ) references {schema}.{role1_table}({role1_type}_ID),
    -- ... more FK constraints ...
    Metadata_{NX} int not null,
    constraint pk{NX}_{Descriptor} primary key (
        {NX}_ID
    )
) CLUSTER BY ({NX}_ID);
COMMENT ON TABLE {schema}.{NX}_{Descriptor} IS '{description}';
```

Example (Event nexus):
```sql
CREATE SEQUENCE IF NOT EXISTS public.EV_Event_ID_SEQ START 1 INCREMENT 1;
CREATE TABLE IF NOT EXISTS public.EV_Event (
    EV_ID int default public.EV_Event_ID_SEQ.nextval not null,
    ST_ID_wasHeldAt int not null,
    PR_ID_wasPlayed int not null,
    ETY_ID_of tinyint not null,
    constraint EV_Event_fkST_wasHeldAt foreign key (ST_ID_wasHeldAt) references public.ST_Stage(ST_ID),
    constraint EV_Event_fkPR_wasPlayed foreign key (PR_ID_wasPlayed) references public.PR_Program(PR_ID),
    constraint EV_Event_fkETY_of foreign key (ETY_ID_of) references public.ETY_EventType(ETY_ID),
    Metadata_EV int not null,
    constraint pkEV_Event primary key (EV_ID)
) CLUSTER BY (EV_ID);
```

> **Why not IDENTITY?** Snowflake IDENTITY columns do not accept explicit values on INSERT. During bulk loading, a staging table assigns explicit IDs that must be inserted into both the nexus table and all its attribute tables. A sequence default allows this; IDENTITY does not.

## 4. Attribute Tables

### 4a. Static Attribute

```sql
CREATE TABLE IF NOT EXISTS {schema}.{AN}_{ATR}_{AnchorDesc}_{AttrDesc} (
    {AN}_{ATR}_{AN}_ID {anchor_id_type} not null,
    {AN}_{ATR}_{AnchorDesc}_{AttrDesc} {data_type} not null,
    Metadata_{AN}_{ATR} int not null,
    constraint fk{AN}_{ATR}_{AnchorDesc}_{AttrDesc} foreign key (
        {AN}_{ATR}_{AN}_ID
    ) references {schema}.{AN}_{AnchorDesc}({AN}_ID),
    constraint pk{AN}_{ATR}_{AnchorDesc}_{AttrDesc} primary key (
        {AN}_{ATR}_{AN}_ID
    )
) CLUSTER BY ({AN}_{ATR}_{AN}_ID);
```

Example (Program Name — static):
```sql
CREATE TABLE IF NOT EXISTS public.PR_NAM_Program_Name (
    PR_NAM_PR_ID int not null,
    PR_NAM_Program_Name varchar(42) not null,
    Metadata_PR_NAM int not null,
    constraint fkPR_NAM_Program_Name foreign key (PR_NAM_PR_ID) references public.PR_Program(PR_ID),
    constraint pkPR_NAM_Program_Name primary key (PR_NAM_PR_ID)
) CLUSTER BY (PR_NAM_PR_ID);
```

### 4b. Historized Attribute

Same as static but adds `ChangedAt` to PK:

```sql
CREATE TABLE IF NOT EXISTS {schema}.{AN}_{ATR}_{AnchorDesc}_{AttrDesc} (
    {AN}_{ATR}_{AN}_ID {anchor_id_type} not null,
    {AN}_{ATR}_{AnchorDesc}_{AttrDesc} {data_type} not null,
    {AN}_{ATR}_ChangedAt {time_type} not null,
    Metadata_{AN}_{ATR} int not null,
    constraint fk{AN}_{ATR}_{AnchorDesc}_{AttrDesc} foreign key (
        {AN}_{ATR}_{AN}_ID
    ) references {schema}.{AN}_{AnchorDesc}({AN}_ID),
    constraint pk{AN}_{ATR}_{AnchorDesc}_{AttrDesc} primary key (
        {AN}_{ATR}_{AN}_ID,
        {AN}_{ATR}_ChangedAt
    )
) CLUSTER BY ({AN}_{ATR}_{AN}_ID, {AN}_{ATR}_ChangedAt);
```

Example (Actor Name — historized):
```sql
CREATE TABLE IF NOT EXISTS public.AC_NAM_Actor_Name (
    AC_NAM_AC_ID int not null,
    AC_NAM_Actor_Name varchar(42) not null,
    AC_NAM_ChangedAt datetime not null,
    Metadata_AC_NAM int not null,
    constraint fkAC_NAM_Actor_Name foreign key (AC_NAM_AC_ID) references public.AC_Actor(AC_ID),
    constraint pkAC_NAM_Actor_Name primary key (AC_NAM_AC_ID, AC_NAM_ChangedAt)
) CLUSTER BY (AC_NAM_AC_ID, AC_NAM_ChangedAt);
```

### 4c. Knotted Static Attribute

FK to knot instead of value column:

```sql
CREATE TABLE IF NOT EXISTS {schema}.{AN}_{ATR}_{AnchorDesc}_{AttrDesc} (
    {AN}_{ATR}_{AN}_ID {anchor_id_type} not null,
    {AN}_{ATR}_{KNT}_ID {knot_id_type} not null,
    Metadata_{AN}_{ATR} int not null,
    constraint fk{AN}_{ATR}_{AnchorDesc}_{AttrDesc} foreign key (
        {AN}_{ATR}_{AN}_ID
    ) references {schema}.{AN}_{AnchorDesc}({AN}_ID),
    constraint fk_knotref_{AN}_{ATR}_{KNT} foreign key (
        {AN}_{ATR}_{KNT}_ID
    ) references {schema}.{KNT}_{KnotDesc}({KNT}_ID),
    constraint pk{AN}_{ATR}_{AnchorDesc}_{AttrDesc} primary key (
        {AN}_{ATR}_{AN}_ID
    )
) CLUSTER BY ({AN}_{ATR}_{AN}_ID);
```

Example (Actor Gender — knotted static):
```sql
CREATE TABLE IF NOT EXISTS public.AC_GEN_Actor_Gender (
    AC_GEN_AC_ID int not null,
    AC_GEN_GEN_ID number(1,0) not null,
    Metadata_AC_GEN int not null,
    constraint fkAC_GEN_Actor_Gender foreign key (AC_GEN_AC_ID) references public.AC_Actor(AC_ID),
    constraint fk_AC_GEN_GEN foreign key (AC_GEN_GEN_ID) references public.GEN_Gender(GEN_ID),
    constraint pkAC_GEN_Actor_Gender primary key (AC_GEN_AC_ID)
) CLUSTER BY (AC_GEN_AC_ID);
```

### 4d. Knotted Historized Attribute

FK to knot + ChangedAt in PK:

```sql
CREATE TABLE IF NOT EXISTS {schema}.{AN}_{ATR}_{AnchorDesc}_{AttrDesc} (
    {AN}_{ATR}_{AN}_ID {anchor_id_type} not null,
    {AN}_{ATR}_{KNT}_ID {knot_id_type} not null,
    {AN}_{ATR}_ChangedAt {time_type} not null,
    Metadata_{AN}_{ATR} int not null,
    constraint fk{AN}_{ATR}_{AnchorDesc}_{AttrDesc} foreign key (
        {AN}_{ATR}_{AN}_ID
    ) references {schema}.{AN}_{AnchorDesc}({AN}_ID),
    constraint fk_knotref_{AN}_{ATR}_{KNT} foreign key (
        {AN}_{ATR}_{KNT}_ID
    ) references {schema}.{KNT}_{KnotDesc}({KNT}_ID),
    constraint pk{AN}_{ATR}_{AnchorDesc}_{AttrDesc} primary key (
        {AN}_{ATR}_{AN}_ID,
        {AN}_{ATR}_ChangedAt
    )
) CLUSTER BY ({AN}_{ATR}_{AN}_ID, {AN}_{ATR}_ChangedAt);
```

## 5. Tie Table

### 5a. Static Tie

```sql
CREATE TABLE IF NOT EXISTS {schema}.{tie_name} (
    {type1}_ID_{role1} {id_type} not null,
    {type2}_ID_{role2} {id_type} not null,
    -- knot roles if any:
    {knt}_ID_{knot_role} {knot_id_type} not null,
    Metadata_{tie_short} int not null,
    constraint pk{tie_name} primary key (
        {identifier_role_columns}
    ),
    constraint {tie_name}_fk{type1}_{role1} foreign key ({type1}_ID_{role1})
        references {schema}.{type1_table}({type1}_ID),
    constraint {tie_name}_fk{type2}_{role2} foreign key ({type2}_ID_{role2})
        references {schema}.{type2_table}({type2}_ID)
) CLUSTER BY ({identifier_role_columns});
```

### 5b. Historized Tie

Adds `{tie_name}_ChangedAt` to PK:

```sql
CREATE TABLE IF NOT EXISTS {schema}.{tie_name} (
    {type1}_ID_{role1} {id_type} not null,
    {type2}_ID_{role2} {id_type} not null,
    {tie_name}_ChangedAt {time_type} not null,
    Metadata_{tie_short} int not null,
    constraint pk{tie_name} primary key (
        {identifier_role_columns},
        {tie_name}_ChangedAt
    ),
    -- FK constraints ...
) CLUSTER BY ({identifier_role_columns});
```

## 6. Latest Perspective (l prefix)

A view joining the anchor/nexus to all its attributes and their knots, picking `max(ChangedAt)` for historized attributes.

```sql
CREATE OR REPLACE VIEW {schema}.l{AN}_{Descriptor} AS
SELECT
    {AN}.{AN}_ID,
    {AN}.Metadata_{AN},
    -- For each attribute:
    {ATR}.{AN}_{ATR}_{AN}_ID,
    {ATR}.Metadata_{AN}_{ATR},
    {ATR}.{AN}_{ATR}_ChangedAt,          -- if historized
    {ATR}.{AN}_{ATR}_{AnchorDesc}_{AttrDesc},  -- or knot value via join
    k{KNT}.{KNT}_{KnotDesc} AS {AN}_{ATR}_{KNT}_{KnotDesc},  -- if knotted
    {ATR}.{AN}_{ATR}_{KNT}_ID            -- if knotted
FROM
    {schema}.{AN}_{Descriptor} {AN}
-- For each attribute, LEFT JOIN:
LEFT JOIN {schema}.{AN}_{ATR}_{AnchorDesc}_{AttrDesc} {ATR}
    ON {ATR}.{AN}_{ATR}_{AN}_ID = {AN}.{AN}_ID
    -- If historized, add max(ChangedAt) subquery:
    AND {ATR}.{AN}_{ATR}_ChangedAt = (
        SELECT max(sub.{AN}_{ATR}_ChangedAt)
        FROM {schema}.{AN}_{ATR}_{AnchorDesc}_{AttrDesc} sub
        WHERE sub.{AN}_{ATR}_{AN}_ID = {AN}.{AN}_ID
    )
-- If knotted, LEFT JOIN the knot:
LEFT JOIN {schema}.{KNT}_{KnotDesc} k{KNT}
    ON k{KNT}.{KNT}_ID = {ATR}.{AN}_{ATR}_{KNT}_ID
;
```

Add `COMMENT` on the view matching the anchor/nexus description, and `COMMENT` on knotted/attributed columns.

## 7. Point-in-Time Perspective (p prefix)

A table function taking a `changingTimepoint timestamp_ntz(9)` parameter:

```sql
CREATE OR REPLACE FUNCTION {schema}.p{AN}_{Descriptor} (
    changingTimepoint timestamp_ntz(9)
)
RETURNS TABLE ( /* all columns from latest perspective */ )
LANGUAGE SQL
AS $$
    SELECT /* same joins as latest, but max(ChangedAt) WHERE ChangedAt <= changingTimepoint */
$$;
```

## 8. Now Perspective (n prefix)

```sql
CREATE OR REPLACE VIEW {schema}.n{AN}_{Descriptor} AS
SELECT * FROM TABLE({schema}.p{AN}_{Descriptor}(current_timestamp()::timestamp_ntz(9)));
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
        h{ATR}.{AN}_{ATR}_ChangedAt::timestamp_ntz(9) AS inspectedTimepoint,
        '{ATR}' AS mnemonic,
        p{AN}.*
    FROM
        {schema}.{AN}_{ATR}_{AnchorDesc}_{AttrDesc} h{ATR},
        TABLE({schema}.p{AN}_{Descriptor}(h{ATR}.{AN}_{ATR}_ChangedAt::timestamp_ntz(9))) p{AN}
    WHERE
        (selection IS NULL OR selection LIKE '%{ATR}%')
    AND h{ATR}.{AN}_{ATR}_ChangedAt BETWEEN intervalStart AND intervalEnd
    AND p{AN}.{AN}_ID = h{ATR}.{AN}_{ATR}_{AN}_ID
    UNION
    -- ... repeat for each historized attribute ...
$$;
```

## 10. Tie Perspectives

Ties also get latest, point-in-time, now, and difference perspectives with the same patterns, joining tie roles to their anchor/knot tables.
