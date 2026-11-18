# Anchor Modeling Constructs

## The Idea

Anchor Modeling splits a domain into sixth-normal-form pieces, except where a relationship must be recorded as a whole (see ties and nexuses below):

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
- For recurring loads, define how source rows find the same identity. A static identifier attribute works for one stable value; composite, changing or upstream-resolved keys work too (`identity-and-keys.md`).

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

A relationship with at least two anchor or nexus roles, possibly also knot roles. Each participant has a role.

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

**Atomic ties:** Several non-identifier roles can belong in one tie if the business requires them to be recorded together. Such a tie can be in fifth rather than sixth normal form. Split the roles into separate ties if they can be independently missing or changing; do not split a relationship that must always be recorded as a whole.

#### Flavors

- **Static tie**: relationship holds once established.
- **Historized tie**: keep versions of the relationship by changing time. An ending needs a recorded state or other explicit domain fact; the absence of another row does not end it.
- **Knotted tie**: carries a knot value.

### Nexus

An event-like entity with its own identity, roles, and properties.

- Like an anchor: has `{mnemonic}_ID` and can have attributes.
- Like a tie: has role columns referencing anchors and knots.
- Required, stable roles are stored together on the nexus table itself. A nexus with multiple such roles is in fifth rather than sixth normal form; separate ties would let them be recorded independently.
- Roles are never identifiers (nexus ID identifies it).
- Nexus is immutable once recorded.

Example: Event nexus `EV_Event` with `EV_ID`, `ST_ID_wasHeldAt`, `PR_ID_wasPlayed`, `ETY_ID_of`.

**Chronicle**: An attribute with a `chronicle` ordinal places the nexus in time. Every nexus should have at least one.

**Matching**: A recurring load needs a repeatable way to recognize the event. Use a static identifier attribute for a stable event key, or a suitable key route or upstream mapping (`identity-and-keys.md`).

**Use a nexus when** a happening has its own identity and its roles must be recorded together and stay fixed. If a participant may be unknown or change independently, represent it in a separate tie to the happening. An anchor with ties also works when the participants need not be recorded as a whole.
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
5. **By default, keep independently changing roles separate.** A historized tie typically has at most one non-identifier role. Keep several together only when a new version must state the entire relationship at once. For independently changing knot roles, consider an anchor with knotted attributes.
6. **A knot models a state** only if the knot's values are *mutually exclusive and exhaustive*: exactly one applies at any time (`Not married` and `Married`; `Resigned` and `Active`). Leaving one state is entering another. Do not repurpose a stored knot value. If a value needs independent properties (a fee per professional level), model it as an anchor with attributes.
7. **Evolution is by extension.** A new property or relationship is a new table. A new allowed value in an existing state domain is not a new construct; when the domain allows it, add a knot row without changing the meaning of existing rows.
8. **No NULL, no update.** A property that is unknown has no row. An entity enters the warehouse with only an identity. A reference to something not yet loaded: create the referred anchor from what the record tells you, or add no tie row until the information arrives (a historized tie may hold the likely one and be corrected later).

## Perspectives

Views that join pieces back into wide rows:

| Prefix | Type | Shows |
|--------|------|-------|
| `l` | Latest | Most recent version in changing-time order, including future-effective values |
| `p` | Point-in-time | Values as of a given timestamp (table function) |
| `n` | Now | Values as of the current time (view) |
| `d` | Difference | Every change between two timestamps (table function) |

For what is in effect now, use `n` or `p(now)` rather than `l`. For what was believed at a past assertion time, inspect the `t` perspective. For semantic views, including as of a point in time, see `semantic-views.md`.

## Metadata Column

When enabled, `Metadata_…` columns can link rows to the batch that recorded them (`metadata.md`).

## Temporalization

- **uni** (uni-temporal): changing time only.
- **bi** (bitemporal): changing time and assertion (positing) time.
- **crt** (concurrent-reliance-temporal): changing time plus assertions with positing time, positor, and reliability.

Recording time is separate from positing time (`time.md`).

Constructs are the same in all three; only generated tables/perspectives differ.

**Scope of this skill:** the hand-written DDL templates and the loading patterns cover **uni-temporal** models only. The Anchor generator (`generator.md`) makes the DDL for uni, bitemporal and concurrent-reliance-temporal models. For a bitemporal or concurrent-reliance-temporal request, use the generator; the loading patterns for them are not written yet, so ask the user before improvising a load.
