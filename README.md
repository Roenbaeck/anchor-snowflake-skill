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

The skill lives in `.snowflake/cortex/skills/anchor-modeling/`, where Cortex Code in a Snowflake workspace looks for skills:

| File | Purpose |
|------|---------|
| `SKILL.md` | Entry point: intent detection and step-by-step workflows |
| `references/constructs.md` | Anchor Modeling theory and construct definitions |
| `references/ddl-patterns.md` | Snowflake DDL and perspective templates, integrity checks |
| `references/naming-conventions.md` | Naming rules and Snowflake identifier-case behaviour |

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

## Requirements

- Cortex Code in a Snowflake workspace (see Installation), or another agent host with a tool that executes Snowflake SQL. Without such a tool, the skill still produces SQL for you to run yourself.
- A Snowflake role that can create schemas, tables, sequences, views, functions and tasks in the target database, plus a warehouse for the load tasks.
- Optional: an `ANCHOR_EXAMPLE.PUBLIC` database containing a sample theatre-domain model. The skill uses it as a live reference if it exists.

## Limitations

- Uni-temporal models only. Concurrent-reliance-temporal and bitemporal models are explained but not generated.
- Snowflake does not enforce PK/UNIQUE/FK constraints. The skill declares them with `RELY` (for join elimination) and relies on idempotent loads plus post-load integrity checks.

## License

MIT, see [LICENSE](LICENSE).
