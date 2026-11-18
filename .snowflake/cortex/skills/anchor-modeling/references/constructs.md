# Anchor Modeling Constructs

## The Idea

Anchor Modeling splits a domain into sixth-normal-form pieces:

- **Things** with identity → their own table (anchor), holding nothing but that identity.
- **Every property** → its own table (attribute).
- **Every relationship** → its own table (tie).

The model grows by adding new tables, never altering existing ones. Old queries keep working. Data is append-only — changes are new rows with later timestamps, not overwrites. Loads must therefore be incremental: truncating and reloading from a source that only holds current values destroys the history.

Many narrow tables favor columnar engines like Snowflake: queries read only the columns they need, and selective conditions on one narrow table cut rows early.

Perspectives (views) join the pieces back together for convenience.

## Constructs

### Anchor

A thing with an identity: a person, actor, stage, program.

- `mnemonic`: 2-letter code, unique in model. Prefixes all generated names.
- `descriptor`: PascalCase readable name.
- `identity`: data type of surrogate ID.
- Generated table: `{mnemonic}_{descriptor}` with `{mnemonic}_ID` and `Metadata_{mnemonic}`.
- Use a sequence for auto-generated IDs.
- Give it a static identifier attribute holding the source natural key, so loads can map source rows to surrogate IDs.

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
- `identity`: integer; in Snowflake all integer types are `NUMBER(38,0)`, so use `int`.
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

**Identifier**: Like an anchor, a nexus should have a static attribute holding the source key of the event, so incremental loads can tell which events are already loaded.

**Use nexus when** a relationship is a happening: has its own properties, other things refer to it, or same anchors participate many times.
**Use tie when** the relationship is fully described by who takes part.

### Role

How a participant takes part in a tie or nexus.

- `role`: camelCase name, unique within the tie.
- `type`: mnemonic of the anchor/nexus/knot.
- `identifier`: makes role part of the key.

Choose names that read as a sentence: "actor **part** program **in**, **got** rating".

## Design guidelines (the author's)

From the 2010 paper *Anchor Modeling* (§4), in the order that a modeling session meets them. They use the paper's words, *knotted* and *historized*.

1. **Core entities and happenings are anchors (or nexuses).** A happening is a tie only if it has no properties of its own.
2. **Attributes:** historized if its versions matter, otherwise static; knotted if the values are categories or a small fixed set. The two combine (a knotted historized attribute).
3. **Ties:** historized if the relationship may change over time, otherwise static; knotted if the relationships belong to categories. The two combine.
4. **Roles that may be unknown for a while:** use several smaller ties in which every role is known, rather than one large tie with a hole in it (there is no NULL).
5. **A historized tie should have at most one role outside its identifier.** If it has more, and the roles do not change together, every new version repeats the rest of the identifier. Move the knot roles to an anchor with knotted attributes and tie to that anchor instead: the same for a knotted tie whose knot roles do not change synchronously.
6. **A knot models a state** only if the knot's values are *mutually exclusive and exhaustive*: exactly one is true at any time (`Not married` and `Married`; `Resigned` and `Active`). That is how something *ends* without a NULL or a delete: a historized attribute or tie only has the start of an interval, so leaving a state is entering another. A knot may never change once loaded (it is a collapsed anchor); if a knot starts to have properties of its own (a fee per professional level), unfold it into an anchor with attributes.
7. **Evolution is by extension.** A new property, relationship or state is a new table. Do not alter or drop existing tables to make room.
8. **No NULL, no update.** A property that is unknown has no row. An entity enters the warehouse with only an identity. A reference to something not yet loaded: create the referred anchor from what the record tells you, or add no tie row until the information arrives (a historized tie may hold the likely one and be corrected later).

## Perspectives

Views that join pieces back into wide rows:

| Prefix | Type | Shows |
|--------|------|-------|
| `l` | Latest | Most recent value of everything |
| `p` | Point-in-time | Values as of a given timestamp (table function) |
| `n` | Now | Values as of current_timestamp (view) |
| `d` | Difference | Every change between two timestamps (table function) |

For querying, latest perspectives (`l` prefix) are the starting point. For semantic views, including as of a point in time, see `semantic-views.md`.

## Metadata Column

Every table has `Metadata_XX int not null` for tracking source/batch of each row.

## Temporalization

- **uni** (uni-temporal): changing time only.
- **crt** (concurrent-reliance-temporal): also records when/by whom/reliability.
- **bi** (bitemporal): changing time + positing time.

Constructs are the same in all three; only generated tables/perspectives differ.

**Scope of this skill:** the hand-written DDL templates and the loading patterns cover **uni-temporal** models only. The Anchor generator (`generator.md`) makes the DDL for uni, bitemporal and concurrent-reliance-temporal models. For a bitemporal or concurrent-reliance-temporal request, use the generator; the loading patterns for them are not written yet, so ask the user before improvising a load.
