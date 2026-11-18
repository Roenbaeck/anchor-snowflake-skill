# Warehouse practice

Load this when a user asks where an Anchor model fits in a warehouse or lake, whether to keep bronze/silver/gold layers or data marts, how to organise loads, how an agent or a BI tool should query the model, or why the model should not copy a source system's tables. These are the author's positions, from his writing (*Sources* at the end). Give them as his positions, with their reasons; they are not laws, and the user may already have layers that work.

## Loading discipline

- **Keep recurring loads append-only.** Record changes and assertions without overwriting existing history. This is the author's "zero update strategy" (2007); on SQL Server it also reduced locking. A batch tag helps identify inserted rows for recovery, but later dependencies can make deleting those rows unsafe (`metadata.md`).
- **Large volumes go in with `COPY INTO` from a stage and set-based inserts**, not row by row (`SKILL.md`, *Data Loading Workflow*; `performance.md`). Parse awkward sources (a large XML, say) outside the database into flat files first: the author's `unstruct` does it for simple XML, and is early software with little error handling, so check its output.
- **Never change a posit in place.** Where the model is bitemporal or concurrent, a mistake is corrected by retracting the wrong version and adding the right one, never by overwriting; history keeps both.

## Layers: the author's current position (2026-11-17)

A warehouse has two fixed points: the **landing zone** (what arrives) and the **business-facing model** (what is asked). His position, in *The Layers have been Slain*:

1. **Do not build persisted layers between them** (bronze, silver, gold, staging, cleansed, integration). They serve the people who maintain the warehouse, not the people who use it, and each one is something to keep correct. Load from the landing zone **straight into the Anchor model**, with the loads scheduled as a dependency graph.
2. **Keep the normalised, historized model in the middle.** Skipping it and letting agents read raw landing files is, in his words, not a good idea. The model is where identity, history and "what we knew when" are settled once.
3. **Do not build separate data marts** as the presentation layer. A mart is a simplified copy of a richer model. Put a **semantic layer** over the model's perspectives instead (views with business names and the model's descriptions as comments; a semantic view, which Snowflake has: `semantic-views.md`), and let people and agents query it. (The Avito warehouse of 2015, on Vertica, did build denormalized marts for its BI software, because business users did not want fully historized versions; this position is later and replaces them.)
4. **Expect the work to move**, from moving data between layers to semantic questions: reconciling two valid definitions of the same word.

The older post, *The Slayers of Layers* (2019), gives the test to apply to every layer: each is a cost of upkeep, tolerable while the technology is immature; ask what it buys and count what it costs. When a user already has layers, do not tear them down on this account: say what the position is, ask what each layer is for, and suggest removing the ones nobody can justify. The post's claims about speed ("minutes instead of days") are the author's impression, not a measurement.

## Do not copy the source

- **Model the organisation's own concepts and goals, not the source schemas.** Copying application schemas reproduces their problems and ties the model to one application. Ask what the business is trying to do, in its words (`modeling-practice.md`). Do not adopt a third-party application's whole model either: take the part the business needs, and expect to revise it.
- **Hold the history, not just the current state.** Applications overwrite; regulations and questions about the past need what was there.
- **Do not build JBOT** (just a bunch of tables): independent tables with no enforced model, in a lake or anywhere. They are easy to start, they need no design, and they drift until nobody can maintain them.
- **Business rules are descriptive, not physical.** The model holds what things are and when; how they are *interpreted* (a customer, active, a margin) lives in views and queries over it, so a changed definition is a changed view, not a migration.
- **Dimensions and facts are views over the model**, and temporal: history stays in the anchor model, a dimension is built for a time or a range of times (`performance.md`, twine), not as slowly changing dimension types that give write-time or read-time anomalies.

## Coming from Data Vault

*Anchor for Vaulters* (the author's mapping, for users who know Data Vault):

| Data Vault | Anchor |
|---|---|
| Hub (business key) | **Anchor**, with a surrogate identity. Match the business key through an attribute or a key route (`identity-and-keys.md`) |
| Satellite | **Attributes**: one table per column, each historized or not as its own column changes, so a change to one column writes one small row, and a column that is added later is a new table |
| Link | **Tie**, with its cardinality and identifier roles stated in the model; a link's own descriptive data is attributes of a nexus or of an anchor made for it |
| Reference table | **Knot** |

What differs, so that nothing is promised that is not so: **no hashing of keys** (`identity-and-keys.md`); the model is a **normalised** (6NF) one rather than a raw copy of the source, so **a reload needs the source** and nothing is kept as "the raw data"; loads are slower than appending to a vault's wide satellites, and the model is queried through the perspectives instead of join logic written for each satellite; and the performance claims **need a recent database version** (`performance.md`). Do not tear a working vault down on this account: the mapping is for people who want to see what they would be building.

## When an agent or a tool queries the model

- **Define the question first.** *How many customers do we have?* has several answers: customers as of when, by which definition (billed, active, ever), counting which identity. An actuary and an accountant can both be right. Ask which they mean, state the definition in the answer, and give the same answer to the same question tomorrow: put the agreed definitions in views or a semantic layer, with a comment, once.
- **A model generated twice from the same description will differ.** In the author's experiment with a language model, a bakery gave a different model on each run. So keep the model XML and the list of assumptions as the record, change it by editing it, and do not regenerate it from a description each time.
- **Say which cutoff the answer uses.** Use changing time for what was in effect, changing and assertion time for what someone believed, and recording or publication time for what the warehouse had available (`time.md`).
- **Give agents the model's descriptions.** They are the comments the generator writes on every table and column, and they are what lets an agent read the model without guessing.

## Sources

*The Layers have been Slain* (2026-11-17), *The Slayers of Layers*, *The Return of the JBOT*, *Towards a Model-Driven Organization* (parts 1 and 2), *Temporal Dimensional Modeling*, *Metadata driven Anchor DW Automation*, *Unstruct*, *The Intelligent Lake*, and the 2007 cheat sheet (the zero update strategy), *Anchor for Vaulters*, *Big Data Normalization for MPP Databases*, at `https://www.anchormodeling.com/`.
