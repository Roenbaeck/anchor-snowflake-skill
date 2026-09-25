# Anchor Modeling Naming Conventions (Snowflake)

## Identifier Case

Snowflake stores **unquoted** identifiers in UPPERCASE. `AC_Actor` is created as `AC_ACTOR`, and `lAC_Actor` as `LAC_ACTOR`. Consequences:

- Writing PascalCase in DDL is fine and keeps scripts readable, and unquoted references are case-insensitive. But `SHOW TABLES`, `INFORMATION_SCHEMA` and query results return the uppercase form.
- The perspective prefix is no longer distinguishable by case: `LAC_ACTOR` looks like a construct with mnemonic `LAC`. Never classify objects by name alone. See the classification rules in SKILL.md (Q1).
- Only quoted identifiers (`"AC_Actor"`) keep their case, and they must then be quoted with exactly that case everywhere. Do not mix quoted and unquoted forms for the same object.

## Unicode Characters Warning

Snowflake identifiers containing non-ASCII characters (ö, å, ä, ü, é, etc.) MUST be double-quoted in all DDL and DML. This includes table names, column names, constraint names, sequence names, and all references in views and functions. Unquoted non-ASCII identifiers cause `syntax error ... unexpected` errors.

**Options:**
1. **ASCII-only identifiers** (recommended for international teams): transliterate special characters (ö→o, å→a, ä→a). Keeps all identifiers unquoted and case-insensitive.
2. **Quoted identifiers** (preserves native language): double-quote every identifier with special characters. Quoted identifiers are case-sensitive, so all references must match the exact case used at creation.

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
