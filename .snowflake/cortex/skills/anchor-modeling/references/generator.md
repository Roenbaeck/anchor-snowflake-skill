# The Anchor Generator

A set of Snowflake objects that turns an Anchor model, given as the XML that the [Anchor Modeler](https://github.com/Roenbaeck/anchor) saves, into the complete DDL script for it: the same script as the modeler's *Generate SQL*, for uni-, bi- and concurrent-reliance-temporal models, with equivalence, checksums, descriptions and all the perspectives. It runs inside Snowflake; nothing is installed outside it.

Use it when the user has, or can make, the model XML. Without it, generate the DDL by hand from `ddl-patterns.md`.

## Install

The script is `generator/anchor_generator.sql`, next to `SKILL.md`. It is built from the Anchor repository (`tools/build-snowflake-generator.ps1`), and its first lines say from which commit. It is not edited by hand.

Run it once, in the database and schema where the generator should live (it creates its objects in the current schema, so pick one, for example a schema called `ANCHOR_TOOLS`):

```sql
CREATE SCHEMA IF NOT EXISTS ANCHOR_TOOLS;
USE SCHEMA ANCHOR_TOOLS;
```

Then run the whole file (in a Snowsight workspace: open the file, *Run all*). The role needs `CREATE FUNCTION` and `CREATE TABLE` on the schema. The script is about 450 KB in seven statements; the largest is about 140 KB, which is within Snowflake's limit of 1 MB for the text of a statement.

It creates four objects:

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
7. `COMMENT`s from the descriptions of the model

Tables are `CREATE TABLE IF NOT EXISTS`, so running the script again on a database that has the model adds what is new and changes nothing else. Views and functions are `CREATE OR REPLACE ... COPY GRANTS`, so grants on them survive. Every table has a `CLUSTER BY`, every key is declared `RELY`, and the default for *now* is `sysdate()` (UTC).

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
- **Nexus ids.** The generator gives a nexus whose `generator` flag is true an `IDENTITY(1,1)` id, not a sequence. The nexus load pattern in `SKILL.md` draws the ids from a sequence and inserts them explicitly, which `IDENTITY` does not accept. Do not apply that pattern to a nexus that the generator made; tell the user, who can change the template or the table. Anchors and knots whose `generator` flag is true get sequences and work with the load patterns.
- The XML reader is small on purpose, since a JavaScript function in Snowflake has no XML parser: it reads elements, attributes, text and CDATA, and does not read namespaces or entities defined by a DTD, which a model never uses.
- An update of the generator is a newer `anchor_generator.sql`; run it again, it replaces the four objects.
