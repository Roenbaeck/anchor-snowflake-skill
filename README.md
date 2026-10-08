# Anchor Modeling skill for Snowflake

An agent skill for [Anchor Modeling](https://www.anchormodeling.com/) on Snowflake. It covers the full lifecycle of an Anchor model (loading is written for uni-temporal models):

- **Reverse-engineer** an existing database or staged files into an Anchor model
- **Generate** DDL: knots, anchors, nexuses, attributes, ties, and latest / point-in-time / now / difference perspectives, by hand for a uni-temporal model or with the Anchor generator (below) for uni-, bi- and concurrent-reliance-temporal models, with equivalence and checksums
- **Load** data incrementally with a Snowflake task graph
- **Query** existing Anchor models through their perspectives
- **Extend** a model with new constructs, without altering existing tables
- **Explain** Anchor Modeling concepts

The naming conventions follow the [Anchor Modeler](https://github.com/Roenbaeck/anchor).

## Contents

The skill lives in `.snowflake/cortex/skills/anchor-modeling/`, where Cortex Code in a Snowflake workspace looks for skills:

| File | Purpose |
|------|---------|
| `SKILL.md` | Entry point: intent detection and step-by-step workflows |
| `references/constructs.md` | Anchor Modeling theory and construct definitions |
| `references/ddl-patterns.md` | Snowflake DDL and perspective templates, integrity checks |
| `references/naming-conventions.md` | Naming rules and Snowflake identifier-case behaviour |
| `references/generator.md` | The Anchor generator: install, use, errors, limits |
| `references/model-xml.md` | The model XML that the generator reads, and how to write it |
| `references/semantic-views.md` | Semantic views on an Anchor model, as of a point in time too |
| `generator/anchor_generator.sql` | The generator, as one SQL script to run once per account. Built from the [Anchor repository](https://github.com/Roenbaeck/anchor) (`tools/build-snowflake-generator.ps1`); the first lines say from which commit. Do not edit it |
| `generator/example-model.xml`, `generator/minimal-model.xml` | A model as the Anchor Modeler saves it, and a small one to start from |

## Installation (Snowflake workspace)

Create a Git-backed workspace in Snowsight that points to this repository. Cortex Code in that workspace then picks up the skill, and `/anchor-modeling` becomes available.

1. **Create an API integration for GitHub** (once per account; requires `ACCOUNTADMIN` or a role with `CREATE INTEGRATION`):

   ```sql
   CREATE API INTEGRATION IF NOT EXISTS github_api_integration
     API_PROVIDER = git_https_api
     API_ALLOWED_PREFIXES = ('https://github.com/Roenbaeck')
     ENABLED = TRUE;

   GRANT USAGE ON INTEGRATION github_api_integration TO ROLE <your_role>;
   ```

   The repository is public, so no credentials are needed to read it. To push changes back (for example from a fork), also set up authentication: a personal access token stored in a Snowflake secret, or OAuth through the Snowflake GitHub app.

2. **Create the workspace**: in Snowsight, go to **Projects » Workspaces**, choose to create a workspace **From Git repository**, and enter:
   - Repository URL: `https://github.com/Roenbaeck/anchor-snowflake-skill`
   - API integration: `github_api_integration`

3. **Use the skill**: open Cortex Code in the new workspace and type `/anchor-modeling`. The skill also triggers on its own when you ask about Anchor Modeling.

To get later updates to the skill, pull from the repository inside the workspace.

**Other agent hosts:** copy the `anchor-modeling` folder into that host's skills directory, keeping `SKILL.md` and `references/` together.

## The Anchor generator (optional)

The skill can write DDL by hand for uni-temporal models. With the generator installed it gives the same DDL as the Anchor Modeler's *Generate SQL*, for all three temporalizations. The generator is four objects in one schema (a template engine, a model reader, the templates, and a function that ties them together), all running inside Snowflake. A model made by the generator also gets integrity checks (`IntegrityViolations` and a view per table), which matter because Snowflake does not enforce keys.

**You do not have to install it yourself.** Ask Cortex Code to "install the Anchor generator": it checks whether it is there, asks which database and schema to use, and installs it with a Git repository in Snowflake (this uses the API integration from *Installation* above) or, if there is none, with the Snowflake CLI. If neither is possible it tells you the two steps to do by hand. The steps it runs are in `references/generator.md`.

To do it by hand, run `generator/anchor_generator.sql` in a schema of your choice (in a Snowsight workspace: open the file, pick the schema, *Run all*). Then:

```sql
SELECT ANCHOR_GENERATE($$<the model XML>$$, 'uni');   -- 'uni', 'bi', 'crt', or NULL for what the model says
```

returns the whole script for the model. Cortex Code does this itself when you ask it to generate an Anchor model and the generator is installed. See `references/generator.md` and `references/model-xml.md`.

To update the generator, ask Cortex Code to update it, or fetch the repository again and run the script again; it replaces the four objects.

## Requirements

- Cortex Code in a Snowflake workspace (see Installation), or another agent host with a tool that executes Snowflake SQL. Without such a tool, the skill still produces SQL for you to run yourself.
- A Snowflake role that can create schemas, tables, sequences, views, functions and tasks in the target database, plus a warehouse for the load tasks.
- Optional: an `ANCHOR_EXAMPLE.PUBLIC` database containing a sample theatre-domain model. The skill uses it as a live reference if it exists.

## Limitations

- The loading patterns are written for uni-temporal models. Bitemporal and concurrent-reliance-temporal models are generated by the generator, but loading them is not covered yet.
- Snowflake does not enforce PK/UNIQUE/FK constraints. The skill declares them with `RELY` (for join elimination) and relies on idempotent loads plus post-load integrity checks.

## License

MIT, see [LICENSE](LICENSE).
