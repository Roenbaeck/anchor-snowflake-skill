# Anchor Modeling Constructs

## The Idea

Anchor Modeling splits a domain into sixth-normal-form pieces:

- **Things** with identity → their own table (anchor), holding nothing but that identity.
- **Every property** → its own table (attribute).
- **Every relationship** → its own table (tie).

The model grows by adding new tables, never altering existing ones. Old queries keep working. Data is append-only — changes are new rows with later timestamps, not overwrites.

Many narrow tables favor columnar engines like Snowflake: queries read only the columns they need, and selective conditions on one narrow table cut rows early.

Perspectives (views) join the pieces back together for convenience.

## Constructs

### Anchor

A thing with an identity: a person, actor, stage, program.

- `mnemonic`: 2-letter code, unique in model. Prefixes all generated names.
- `descriptor`: PascalCase readable name.
- `identity`: data type of surrogate ID.
- Generated table: `{mnemonic}_{descriptor}` with one column `{mnemonic}_ID`.
- Use a sequence for auto-generated IDs.

**Use when** the thing has a persistent identity and other things relate to it.
**Don't use** for value lists (that's a knot).

### Attribute

A single property of an anchor or nexus. Each gets its own table.

- `mnemonic`: 3 letters, unique within its anchor/nexus.
- Has either `dataRange` (data type for own values) or `knotRange` (reference to a knot), never both.
- `timeRange` makes it historized: every change kept with timestamp.

Four flavors:

| Flavor | Has | Example |
|--------|-----|---------|
| Static | dataRange only | Program name |
| Historized | dataRange + timeRange | Actor name |
| Knotted static | knotRange only | Actor gender |
| Knotted historized | knotRange + timeRange | Actor professional level |

Generated table: `{anchor}_{attr}_{AnchorDesc}_{AttrDesc}` with columns:
- `{anchor}_{attr}_{anchor}_ID` — FK to owner
- `{anchor}_{attr}_{AnchorDesc}_{AttrDesc}` — value (or knot FK)
- `{anchor}_{attr}_ChangedAt` — if historized
- `Metadata_{anchor}_{attr}` — metadata/batch tracking

### Knot

A small, shared set of stable values: genders, ratings, event types.

- `mnemonic`: 3 letters, unique among knots.
- `identity`: usually small type (tinyint).
- `dataRange`: type of the values themselves.
- Used via `knotRange` on attributes, or via roles in ties/nexuses.

Generated table: `{mnemonic}_{descriptor}` with `{mnemonic}_ID` and `{mnemonic}_{descriptor}` value column.

**Use when** values form a closed, stable list referenced by many rows.
**Use plain attribute** when values are open-ended.

### Tie

A relationship between two or more anchors, possibly including knots. Each participant has a role.

Generated name: `{type}_{role}` for each role, joined by `_`.
Example: `AC_part_PR_in_RAT_got`.

Columns: `{type}_ID_{role}` for each role.

#### Cardinality

The `identifier` flag on roles forms the primary key:

| Identifier roles | Cardinality |
|---|---|
| All anchor roles | Many-to-many |
| Some anchor roles | Many-to-one (from identifier side) |
| None | One-to-one |

A knot role as identifier → same anchor pair can relate once per knot value.
A knot role as non-identifier → value of the relationship.

#### Flavors

- **Static tie**: relationship holds once established.
- **Historized tie**: relationship can begin/end. `ChangedAt` added to PK.
- **Knotted tie**: carries a knot value.

### Nexus

An event-like entity with its own identity, roles, and properties.

- Like an anchor: has `{mnemonic}_ID` and can have attributes.
- Like a tie: has role columns referencing anchors and knots.
- Roles stored on the nexus table itself (not separate).
- Roles are never identifiers (nexus ID identifies it).
- Nexus is immutable once recorded.

Example: Event nexus `EV_Event` with `EV_ID`, `ST_ID_wasHeldAt`, `PR_ID_wasPlayed`, `ETY_ID_of`.

**Chronicle**: An attribute with a `chronicle` ordinal places the nexus in time. Every nexus should have at least one.

**Use nexus when** a relationship is a happening: has its own properties, other things refer to it, or same anchors participate many times.
**Use tie when** the relationship is fully described by who takes part.

### Role

How a participant takes part in a tie or nexus.

- `role`: camelCase name, unique within the tie.
- `type`: mnemonic of the anchor/nexus/knot.
- `identifier`: makes role part of the key.

Choose names that read as a sentence: "actor **part** program **in**, **got** rating".

## Perspectives

Views that join pieces back into wide rows:

| Prefix | Type | Shows |
|--------|------|-------|
| `l` | Latest | Most recent value of everything |
| `p` | Point-in-time | Values as of a given timestamp (table function) |
| `n` | Now | Values as of current_timestamp (view) |
| `d` | Difference | Every change between two timestamps (table function) |

For querying and semantic models, latest perspectives (`l` prefix) are the starting point.

## Metadata Column

Every table has `Metadata_XX int not null` for tracking source/batch of each row.

## Temporalization

- **uni** (uni-temporal): changing time only.
- **crt** (concurrent-reliance-temporal): also records when/by whom/reliability.
- **bi** (bitemporal): changing time + positing time.

Constructs are the same in all three; only generated tables/perspectives differ.
