# Anchor Modeling skill for Snowflake

An agent skill for [Anchor Modeling](https://www.anchormodeling.com/) on Snowflake. It covers the full lifecycle of a uni-temporal Anchor model:

- **Reverse-engineer** an existing database or staged files into an Anchor model
- **Generate** DDL: knots, anchors, nexuses, attributes, ties, and latest / point-in-time / now / difference perspectives
- **Load** data incrementally with a Snowflake task graph
- **Query** existing Anchor models through their perspectives
- **Extend** a model with new constructs, without altering existing tables
- **Explain** Anchor Modeling concepts

The naming conventions follow the [Anchor Modeler](https://github.com/Roenbaeck/anchor).

## Contents

| File | Purpose |
|------|---------|
| `SKILL.md` | Entry point: intent detection and step-by-step workflows |
| `references/constructs.md` | Anchor Modeling theory and construct definitions |
| `references/ddl-patterns.md` | Snowflake DDL and perspective templates, integrity checks |
| `references/naming-conventions.md` | Naming rules and Snowflake identifier-case behaviour |

## Installation

Copy (or clone) this repository into the skills directory of your agent host, so that `SKILL.md` sits at the root of a folder named after the skill, e.g. `.../skills/anchor-modeling/SKILL.md`.

## Requirements

- An agent host with a tool that executes Snowflake SQL (for example `snowflake_sql_execute`). Without one, the skill still produces SQL for you to run yourself.
- A Snowflake role that can create schemas, tables, sequences, views, functions and tasks in the target database, plus a warehouse for the load tasks.
- Optional: an `ANCHOR_EXAMPLE.PUBLIC` database containing a sample theatre-domain model. The skill uses it as a live reference if it exists.

## Limitations

- Uni-temporal models only. Concurrent-reliance-temporal and bitemporal models are explained but not generated.
- Snowflake does not enforce PK/UNIQUE/FK constraints. The skill declares them with `RELY` (for join elimination) and relies on idempotent loads plus post-load integrity checks.

## License

MIT, see [LICENSE](LICENSE).
