# The Anchor Model XML

The input to the generator (`generator.md`). It is the XML that the [Anchor Modeler](https://github.com/Roenbaeck/anchor) saves: open the modeler, build or load the model, and take *Generate > XML code* (or the saved `.xml` file). Prefer that to writing XML by hand. When the user has no model yet, design it with them (G1, G2 in `SKILL.md`) and write the XML from the approved design.

Two files next to the generator are examples:

- `generator/example-model.xml`: a complete model as the modeler saves it (theatre domain: knots, anchors with historized and knotted attributes, a nexus, ties, natural-key identifiers, descriptions).
- `generator/minimal-model.xml`: a small model with only what the generator needs, to start from when writing by hand.

## Structure

```xml
<schema format="0.101.1">
  <metadata .../>                      <!-- settings of the whole model; complete, see below -->
  <description>...</description>       <!-- optional -->
  <knot mnemonic="GEN" descriptor="Gender" identity="tinyint" dataRange="varchar(42)">
    <metadata capsule="public" generator="false"/>
    <description>...</description>
  </knot>
  <anchor mnemonic="AC" descriptor="Actor" identity="int">
    <metadata capsule="public" generator="true"/>
    <attribute mnemonic="NAM" descriptor="Name" timeRange="datetime" dataRange="varchar(42)">
      <metadata capsule="public"/>     <!-- timeRange makes it historized -->
    </attribute>
    <attribute mnemonic="GEN" descriptor="Gender" knotRange="GEN">
      <metadata capsule="public"/>     <!-- knotRange makes it knotted -->
    </attribute>
  </anchor>
  <nexus mnemonic="EV" descriptor="Event" identity="int"> ... </nexus>   <!-- like an anchor, plus <role> elements -->
  <tie timeRange="datetime">           <!-- timeRange makes it historized -->
    <role role="part" type="AC" identifier="true"/>
    <role role="in" type="PR" identifier="true"/>
    <metadata capsule="public"/>
  </tie>
</schema>
```

| Construct | Mnemonic | Naming |
|-----------|----------|--------|
| knot | 3 letters | `{KNT}_{Descriptor}` |
| anchor | 2 letters | `{AN}_{Descriptor}` |
| attribute | 3 letters | `{AN}_{ATR}_{AnchorDescriptor}_{AttributeDescriptor}` |
| tie | none; the name is built from its roles | `{type}_{role}_{type}_{role}...` |

- `dataRange` is a Snowflake data type (`varchar(42)`, `number(19,4)`, `datetime`, `geography`). `identity` is the type of the surrogate id of the construct.
- A role of a tie names a construct by its mnemonic in `type`; a role on a knot takes the knot's mnemonic. `identifier="true"` marks the roles that identify a tie row.
- An attribute that holds the natural key of its anchor (every anchor needs one, see R2 in `SKILL.md`) is marked in the modeler's XML with `<key>` and `<identifier>` elements, as `example-model.xml` shows. `minimal-model.xml` has none and generates, so they are not needed to get the DDL; keep them when the XML comes from the modeler. Keys make the key views of `generator.md` (Natural keys), so they are worth adding by hand too: `<key stop="1" route="1st" of="{AN}" branch="1"/>` in the attribute, and `<identifier route="1st"/>` in the anchor, after its attributes.
- A key of several values, or through roles and ties, is a route of several `<key>` elements with the same `of` and `route`. Each `branch` is a path from the anchor or nexus to one value, and `stop` numbers the elements across the whole route, in the order of the key's values and of each path: an attribute of what the path has reached is a value; a role of a nexus leads from the nexus to its anchor in one stop (`<key>` in the `<role>`); a tie takes two stops in a row, the role the path enters it by and the role it leaves it by; a knotted attribute and a knot role give the value of the knot. `example-model.xml` identifies an event by its date (branch 1), the location of its stage (branch 2: the role `wasHeldAt`, then the location) and the name of its program (branch 3).
- `<metadata capsule="public"/>` on a construct is its schema. `generator="true"` on an anchor, a nexus or a knot gives its id a sequence (`CREATE SEQUENCE {Mnemonic}_{Descriptor}_ID_SEQ` and `default seq.nextval`), which the load patterns draw from.
- A `<description>` on a construct becomes a `COMMENT` on its table and views.
- `<layout>` elements, with `x`, `y` and `fixed`, are positions in the modeler's drawing. The generator ignores them.

## The metadata element

The `<metadata>` directly under `<schema>` holds about fifty settings that decide the whole script: the name of the columns (`metadataPrefix`, `changingSuffix`, `identitySuffix`), the naming convention (`naming`), whether there are metadata columns (`metadataUsage`), equivalence (`equivalence`), checksums, the default for *now* (`now`), the data type of time stamps (`chronon`) and whether the model is `uni`, `bi` or `crt` (`temporalization`).

**It has to be complete.** The generator does not fill in defaults for what is missing. It checks the model first, and a missing setting is an error that lists the settings that are missing, with the value that the modeler would give each (see *What the generator checks*). So:

- Copy the whole `<metadata>` element from `generator/minimal-model.xml` or `generator/example-model.xml`, and change only the settings the user asks for.
- If the generator answers `The model has N problems, so nothing was generated`, correct what it lists and call it again. Nothing from a call that failed is to be run.
- A generator that was installed before this check existed does not check: it gives names such as `GEN_undefined` for a missing setting. Search its script for the text `undefined`; if it is there, the metadata is incomplete, so do not run the script, and offer to update the generator (`generator.md`).
- `databaseTarget` is set to `Snowflake` by the generator, whatever the XML says.
- A temporalization given as the second argument of `ANCHOR_GENERATE` replaces the one in the XML.

## What the generator checks

Before it generates anything, the generator checks the model and stops at the first call that finds a problem, listing all of them with where they are:

- every setting of `<metadata>` is there, flags are `true` or `false`, `naming` is `original` or `improved`, and `temporalization` is `uni`, `bi` or `crt`;
- knots, anchors and nexuses have a mnemonic, descriptor, identity (and, for a knot, a dataRange), and no two have the same mnemonic (an anchor, nexus or knot do not share one either; the attributes of one anchor or nexus do not either);
- an attribute has either a `dataRange` or a `knotRange`, never both and never neither, and a `knotRange` names a knot of the model;
- a role names an anchor, nexus or knot of the model; a nexus has at least one role that is an anchor, and none of its roles is an identifier;
- a tie has at least two roles that are anchors or nexuses (the same one twice is fine), no two of its roles are the same type with the same role name, and no other tie has the same name;
- a `<key>` is of an anchor or nexus of the model, and every `<identifier route="...">` is made up by keys of that anchor or nexus.

It does not check the meaning of the model (that is what `modeling-practice.md` is for), the data types, or that an anchor has an attribute that holds its natural key.

## Settings that users ask about

| Setting | Values | Effect |
|---------|--------|--------|
| `temporalization` | `uni`, `bi`, `crt` | uni-temporal (changing time only), bitemporal (positing and changing time), concurrent reliance temporal (positing, changing and reliability) |
| `naming` | `improved`, `original` | the naming convention of the Anchor Modeler; `original` gives knotted columns no equivalent or checksum names |
| `metadataUsage` | `true`, `false` | `Metadata_` columns on the tables |
| `equivalence` | `true`, `false` | equivalent knots and attributes, with the views that unfold them |
| `checksum` | `true`, `false` | checksum columns on attributes |
| `now` | a Snowflake expression | the time that the now perspectives use; `sysdate()` is UTC |
