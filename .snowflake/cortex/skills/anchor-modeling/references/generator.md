# The Anchor Generator

A set of Snowflake objects that turns an Anchor model, given as the XML that the [Anchor Modeler](https://github.com/Roenbaeck/anchor) saves, into the complete DDL script for it: the same script as the modeler's *Generate SQL*, for uni-, bi- and concurrent-reliance-temporal models, with equivalence, checksums, descriptions and all the perspectives. It runs inside Snowflake; nothing is installed outside it.

Use it when the user has, or can make, the model XML. Without it, generate the DDL by hand from `ddl-patterns.md`.

## Install

The script is `generator/anchor_generator.sql`, next to `SKILL.md`. It is built from the Anchor repository (`tools/build-snowflake-generator.ps1`), and its first lines say from which commit. It is not edited by hand. It creates its objects in the **current schema**, so choose one first, for example a schema called `ANCHOR_TOOLS`. The role needs `CREATE FUNCTION` and `CREATE TABLE` on it.

The script is about 450 KB in seven statements (the largest about 140 KB, within Snowflake's limit of 1 MB for the text of a statement). **That is too large to read into the conversation and send back as SQL, so do not try**: let Snowflake or a command-line tool read the file. There are three ways, in the order to prefer them.

### A. From a Git repository in Snowflake (nothing passes through the agent)

Snowflake fetches the skill's repository and runs the script from it. It needs an API integration for GitHub, which the README of the skill sets up once per account (step 1 of *Installation*); check with `SHOW API INTEGRATIONS;`. If there is none and the role cannot create one, use B or C, or ask the user to run that step.

```sql
CREATE SCHEMA IF NOT EXISTS {db}.ANCHOR_TOOLS;
USE SCHEMA {db}.ANCHOR_TOOLS;

CREATE GIT REPOSITORY IF NOT EXISTS ANCHOR_SKILL_REPO
  API_INTEGRATION = github_api_integration
  ORIGIN = 'https://github.com/Roenbaeck/anchor-snowflake-skill.git';
ALTER GIT REPOSITORY ANCHOR_SKILL_REPO FETCH;

EXECUTE IMMEDIATE FROM @ANCHOR_SKILL_REPO/branches/main/.snowflake/cortex/skills/anchor-modeling/generator/anchor_generator.sql;
```

Use the name of the user's integration if it is not `github_api_integration`. Run these in one session, so that `USE SCHEMA` still holds for the last statement. The repository object stays in the schema; to **update** the generator later, run `ALTER GIT REPOSITORY ANCHOR_SKILL_REPO FETCH;` and the `EXECUTE IMMEDIATE FROM` again. The script holds none of the Jinja or CLI template delimiters (`{{`, `{%`, `{#`, `<%`, `&{`), which the build checks. It does hold `&&` (in the engine), which a tool that reads `&` as a variable changes (see B), so the last statement of the script checks that it arrived intact.

### B. With the Snowflake CLI, when there is a shell

If the host has a shell and the Snowflake CLI (`snow --version`) is connected to the account, give it the file **with templating turned off**:

```
snow sql -f <path to>/generator/anchor_generator.sql --database <db> --schema ANCHOR_TOOLS --enable-templating NONE
```

`--enable-templating NONE` is not optional. By default the CLI reads `&` (and `<% %>`) in a script as the start of a variable, and the engine in the script has `&&` in it: without the flag the script arrives changed, and the generator fails when it is used with `JavaScript execution error: ... module is not defined in SISULATE`. Any other tool that runs the file has to be told not to read `&`, `{{ }}` or `<% %>` in it either. (Create the schema first, with `snow sql -q "CREATE SCHEMA IF NOT EXISTS <db>.ANCHOR_TOOLS"`.) The path is the `generator` folder next to `SKILL.md` in the workspace or checkout where the skill was found.

### C. By the user, in Snowsight

Ask the user to open `generator/anchor_generator.sql` in their workspace, select the database and schema (for example `ANCHOR_TOOLS`) in the context selector, and choose *Run all*. Give them the exact steps; do not paste the script.

### After installing

The last statement of the script runs the generator on a small model that is part of the script and returns one sentence: `The Anchor generator is installed and works: N characters of SQL for the example model`. That is the check; if it returns anything else, or an error, the script was changed on its way to Snowflake (see the table), and the install has to be done again in another way. Check also that the objects exist and that the engine runs; neither needs a model:

```sql
SHOW USER FUNCTIONS LIKE 'ANCHOR_GENERATE' IN SCHEMA {db}.ANCHOR_TOOLS;
SELECT COUNT(*) FROM {db}.ANCHOR_TOOLS.ANCHOR_TEMPLATE;                                        -- 43
SELECT {db}.ANCHOR_TOOLS.SISULATE('Hello $who$', '{"who":"Snowflake"}');                      -- Hello Snowflake
```

If a step fails, show the user the statement and the error message, and stop; Step G4 in `SKILL.md` says the same about generated SQL. Errors seen only when installing:

| Message | Cause |
|---------|-------|
| `Insufficient privileges` on a `CREATE` | the role lacks `CREATE FUNCTION`, `CREATE TABLE` or `CREATE GIT REPOSITORY` on the schema, or `USAGE` on the integration |
| `JavaScript execution error: ... module is not defined in SISULATE`, or the last statement says that the output is not what it should be | the text of the script was changed on its way to Snowflake, for example by the CLI reading `&` as a variable (`&&` became `&` in the engine). Install again: with `snow sql ... --enable-templating NONE` (B), from the Git repository (A), or in Snowsight (C) |
| `API integration ... does not exist` | name it as the user's integration is named, or set it up (README, Installation, step 1) |
| the script ran but the functions are in another schema | `USE SCHEMA` did not hold for the script; find them with `SHOW USER FUNCTIONS LIKE 'ANCHOR_GENERATE' IN ACCOUNT` and use the qualified name, or install again in one session |

Every name that comes from the model is written in double quotes, so national characters (ö, å, ä) and lower case letters work, and the names are case sensitive: query a generated table as `public."AC_Actor"`. The schema is written as it is. See `naming-conventions.md`. The four objects of the generator itself are not quoted.

The script creates four objects:

| Object | What it is |
|--------|------------|
| `SISULATE(template, bindings)` | the Sisula template engine, as a JavaScript function |
| `ANCHOR_BINDINGS(model_xml, temporalization)` | the model XML as the JSON that the templates read |
| `ANCHOR_TEMPLATE` | a table with the templates of uni, bi and crt, in the order they are rendered |
| `ANCHOR_GENERATE(model_xml, temporalization)` | the whole DDL script for the model, as one string |

Check whether it is installed before using it:

```sql
SHOW USER FUNCTIONS LIKE 'ANCHOR_GENERATE' IN ACCOUNT;
```

If it is there, note its database and schema from the result; call it by its qualified name from any session.

## Use

```sql
SELECT ANCHOR_GENERATE($$<the model XML>$$, 'uni');
```

- The first argument is the complete XML of the model (see `model-xml.md`). Dollar quoting avoids escaping the quotes in it; the XML of a model never contains `$$`.
- The second argument is `'uni'`, `'bi'` or `'crt'`. `NULL` takes the temporalization that the model says in its `<metadata>`. Anything else is an error.
- The result is one string: the script. Everything that matters about the model (naming, equivalence, whether there are metadata columns, restatement and so on) is decided by the `<metadata>` element and the constructs in the XML, not by arguments.

An agent should write the result to a file, show the user the script for review, and run it **one statement at a time** (a statement ends at the first semicolon that is outside a `$$ ... $$` body, since function bodies contain semicolons). In a Snowsight workspace, it is simpler to paste the script into a SQL file and use *Run all*. See Step G4 in `SKILL.md`.

To see what the generator thinks of a model without generating, look at the bindings:

```sql
SELECT ANCHOR_BINDINGS($$<the model XML>$$, NULL);
```

## What the script contains

In order, the templates for the temporalization produce:

1. Equivalence and default helpers (when the model has equivalence)
2. Knots, anchors, nexuses
3. Attributes, ties
4. Equivalence views for knots and attributes (when the model has equivalence)
5. Rewinders (the `r` functions, the state of an attribute at a time of change)
6. Perspectives for anchors, nexuses and ties: latest `l`, point-in-time `p`, now `n` and difference `d`
7. Integrity checks: a view `ic_{table}` for every table, and `IntegrityViolations` for the whole model (see below)
8. `COMMENT`s from the descriptions of the model

Tables are `CREATE TABLE IF NOT EXISTS`, so running the script again on a database that has the model adds what is new and changes nothing else. Views and functions are `CREATE OR REPLACE ... COPY GRANTS`, so grants on them survive. Every table has a `CLUSTER BY`, every key is declared `RELY`, and the default for *now* is `sysdate()` (UTC). Every generated identity takes its value from a sequence named `{table}_ID_SEQ` (never `IDENTITY`): knots, anchors, nexuses and, in bi and crt, the posit of every attribute and tie (for example `ST_NAM_Stage_Name_Posit_ID_SEQ`). A load can therefore draw an identity first and insert it explicitly, as the load patterns in `SKILL.md` do.

## Integrity checks

Snowflake does not enforce primary, unique or foreign keys, and the generated tables declare every key `RELY`, so the optimizer trusts them. A load that breaks a key gives wrong results, not an error. The generator therefore makes views that find the rows that break what the tables declare:

```sql
SELECT * FROM {schema}.IntegrityViolations;          -- the whole model; no rows is fine
SELECT * FROM {schema}."ic_{table}";                   -- one table, which reads only that table
```

`{schema}` is the model's default schema (the `encapsulation` setting) for `IntegrityViolations`, and the schema of the table for an `ic_` view. A row has four columns: `Construct` (the table), `Violation` (what is wrong), `ViolationKey` (the key of the rows, an object of column and value) and `Occurrences` (how many rows).

| `Violation` | Meaning |
|-------------|---------|
| `duplicate primary key` | the same primary key more than once |
| `duplicate unique key` | the same unique key more than once (for a tie with no identifier, the role is named) |
| `no row in {table} for {column}` | a foreign key that points at nothing |
| `restatement` | in a historized attribute or tie that may not store restatements (the `restatable` flag is false), a value that is the same as the one before it in changing time. uni only |

Run them after every load. `IntegrityViolations` reads every table, so on a large model prefer the `ic_` view of what was loaded. When a check finds something, show the user the rows (`ViolationKey` tells which) and stop; do not delete rows without asking.

**To see that they work** on a new installation, in a scratch schema: generate a small model, insert one row twice into an anchor table, insert an attribute row for an identity that has no anchor row, and run `IntegrityViolations`. It should name both. The checks join in a way that a `RELY` foreign key cannot remove, but only a test on the account shows that for certain.

In bitemporal and concurrent reliance temporal models, an attribute or tie has a posit table and an annex table, and each has a view (`ic_{name}_Posit`, `ic_{name}_Annex`). The posit and annex keys and the references between them are checked; restatement and overlapping times are not.

## Errors

| Message | Cause |
|---------|-------|
| `XML, line N column M: ...` | the text is not well-formed XML (a pasted model cut short, or something else than the XML code) |
| `The model has no <metadata> element ...` | the XML is not an Anchor model as the modeler saves it; see `model-xml.md` |
| `The temporalization has to be 'uni', 'bi' or 'crt', not '...'` | the second argument |
| `Unknown function` / `Object does not exist` for a function of the generator | not installed in the current schema, or called from a session where it is not visible; qualify the name |
| a SQL error when running the **generated** script | tell the user which statement failed, with the message; that is a defect in the templates and is reported to the Anchor repository, with the model XML if the user can share it |

## Limits

- The input has to be a model the modeler has saved, or one written the same way. It is not a place to describe a domain in prose; for that, design the model (G1, G2 in `SKILL.md`) and then write the XML (`model-xml.md`).
- The generator makes the DDL. Loading data is still done with the patterns in `SKILL.md` (Data Loading Workflow).
- The XML reader is small on purpose, since a JavaScript function in Snowflake has no XML parser: it reads elements, attributes, text and CDATA, and does not read namespaces or entities defined by a DTD, which a model never uses.
- An update of the generator is a newer `anchor_generator.sql`; run it again, it replaces the four objects.
