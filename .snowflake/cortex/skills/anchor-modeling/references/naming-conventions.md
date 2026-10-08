# Anchor Modeling Naming Conventions (Snowflake)

## Identifier Quoting and Case

**Every name that comes from the model is written inside double quotes**, in the DDL that the generator makes and in the DDL, queries and loads that this skill writes: tables, views, functions, columns, constraints, sequences and aliases.

```sql
CREATE TABLE IF NOT EXISTS public."FTG_Företag" (
    "FTG_ID" tinyint not null,
    "FTG_Företag" varchar(50) not null,
    constraint "pkFTG_Företag" primary key ("FTG_ID") RELY
);
SELECT "FTG_Företag" FROM public."FTG_Företag";
```

Why: Snowflake reads an **unquoted** identifier as upper case and accepts only the letters A-Z, digits, underscores and dollar signs in one. So `FTG_Företag` is a syntax error (`ö`), and `AVD_Avdelning` would be created as `AVD_AVDELNING`. A quoted identifier keeps its case and can hold national characters.

Consequences:

- A quoted identifier is **case sensitive**, and `"AC_Actor"` is not `AC_ACTOR`. A statement that refers to a table of the model has to quote the name exactly as it was created: `SELECT * FROM public."AC_Actor"`. Unquoted `AC_Actor` means `AC_ACTOR`, which does not exist. Copy names from the generated DDL; do not retype them in another case, and do not mix quoted and unquoted forms for one object.
- `SHOW TABLES`, `INFORMATION_SCHEMA` and query results return the names as they were created, so a perspective's prefix can be told from a mnemonic by case again (`lAC_Actor`). A database made by an earlier version of the generator or by hand without quotes has upper case names, and for that one the classification rules in SKILL.md (Q1) still apply: classify by kind and structure.
- **The schema is written as it is** (`public`, `knots`), unquoted, so it is read as upper case and matches a schema that was created without quotes. A schema name that has to be quoted (national characters, say) is written quoted by the generator, and the schema then has to be created with exactly that name.
- The names that the generator itself fixes (`IntegrityViolations`, `ANCHOR_GENERATE`) are not quoted, so they are case insensitive.
- Switching a database that was made with unquoted names to the quoted form makes a **second set of objects**: `CREATE TABLE IF NOT EXISTS public."AC_Actor"` does not see `AC_ACTOR`. Drop the old objects or rename them (`ALTER TABLE ... RENAME TO "AC_Actor"`) first.

If an audience wants all-ASCII names, transliterate them (ö→o, å→a, ä→a) in the model; they are still quoted.

## Mnemonics

| Construct | Length | Scope | Case | Examples |
|-----------|--------|-------|------|----------|
| Anchor | 2 letters | Unique in model | UPPERCASE | AC, ST, PR, EV, PN |
| Nexus | 2 letters | Unique in model (same pool as anchors) | UPPERCASE | EV |
| Attribute | 3 letters | Unique within its anchor/nexus | UPPERCASE | NAM, GEN, PLV, DAT |
| Knot | 3 letters | Unique among all knots | UPPERCASE | GEN, RAT, PLV, UTL |

## Descriptors

Always PascalCase. Examples: `Actor`, `Stage`, `ProfessionalLevel`, `EventType`.

## Table Names

| Object | Pattern | Example |
|--------|---------|---------|
| Knot | `{KNT}_{Descriptor}` | `RAT_Rating` |
| Anchor | `{AN}_{Descriptor}` | `AC_Actor` |
| Nexus | `{NX}_{Descriptor}` | `EV_Event` |
| Static attribute | `{AN}_{ATR}_{AnchorDesc}_{AttrDesc}` | `PR_NAM_Program_Name` |
| Historized attribute | Same as static | `AC_NAM_Actor_Name` |
| Knotted attribute | Same pattern, but stores knot FK | `AC_GEN_Actor_Gender` |
| Tie | `{type}_{role}` per role, joined by `_` | `AC_part_PR_in_RAT_got` |
| Sequence | `{AN}_{Descriptor}_ID_SEQ` | `AC_Actor_ID_SEQ` |

## Column Names

| Column | Pattern | Example |
|--------|---------|---------|
| Anchor/nexus ID | `{AN}_ID` | `AC_ID` |
| Knot ID | `{KNT}_ID` | `GEN_ID` |
| Knot value | `{KNT}_{Descriptor}` | `GEN_Gender` |
| Attribute owner FK | `{AN}_{ATR}_{AN}_ID` | `AC_NAM_AC_ID` |
| Attribute value | `{AN}_{ATR}_{AnchorDesc}_{AttrDesc}` | `AC_NAM_Actor_Name` |
| Attribute knot FK | `{AN}_{ATR}_{KNT}_ID` | `AC_GEN_GEN_ID` |
| ChangedAt | `{AN}_{ATR}_ChangedAt` | `AC_NAM_ChangedAt` |
| Metadata | `Metadata_{AN}` or `Metadata_{AN}_{ATR}` | `Metadata_AC`, `Metadata_AC_NAM` |
| Tie role column | `{type}_ID_{role}` | `AC_ID_part`, `PR_ID_in` |
| Tie ChangedAt | `{tie_name}_ChangedAt` | `ST_at_PR_isPlaying_ChangedAt` |
| Tie Metadata | `Metadata_{tie_name}` (full tie name) | `Metadata_ST_at_PR_isPlaying` |
| Nexus role column | `{type}_ID_{role}` | `ST_ID_wasHeldAt` |

## Tie Names

Composed from roles: `{type}_{role}` for each role, joined by `_`.

Example: Actors having parts in programs with ratings → `AC_part_PR_in_RAT_got`

Role names are camelCase and should read as a sentence with the anchor names:
"Actor **part** Program **in** Rating **got**"

## View Names (Perspectives)

| Perspective | Prefix | Pattern | Example |
|-------------|--------|---------|---------|
| Latest | `l` | `l{AN}_{Descriptor}` | `lAC_Actor` |
| Point-in-time | `p` | `p{AN}_{Descriptor}` (function) | `pAC_Actor` |
| Now | `n` | `n{AN}_{Descriptor}` | `nAC_Actor` |
| Difference | `d` | `d{AN}_{Descriptor}` (function) | `dAC_Actor` |

Same prefixes apply to tie perspectives: `lAC_part_PR_in_RAT_got`, `pST_at_PR_isPlaying`, etc.

## Constraint Names

| Constraint | Pattern | Example |
|-----------|---------|---------|
| Primary key | `pk{TableName}` | `pkAC_Actor` |
| Unique | `uq{TableName}` | `uqRAT_Rating` |
| Foreign key (attribute → owner) | `fk{TableName}` | `fkAC_NAM_Actor_Name` |
| Foreign key (attribute → knot) | `fk_{AN}_{ATR}_{KNT}` | `fk_AC_GEN_GEN` |
| Foreign key (tie role) | `{TieName}_fk{type}_{role}` | `AC_part_PR_in_RAT_got_fkAC_part` |
| Foreign key (nexus role) | `{NexusName}_fk{type}_{role}` | `EV_Event_fkST_wasHeldAt` |

All constraints are declared with `RELY` (see `ddl-patterns.md`, section 0).

## Schema

Default schema is `public` (matching Anchor Modeling's default `dbo` capsule). Use capsule names for separate schemas when the model is partitioned.

## Comments

Every table and view should have a `COMMENT` describing what it represents. Perspectives inherit the comment from their anchor/nexus/tie. Describe what an instance IS, not how it's stored.

Knotted and attributed columns in perspectives should also have column-level comments.
