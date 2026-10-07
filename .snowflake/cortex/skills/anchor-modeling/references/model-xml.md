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
- An attribute that holds the natural key of its anchor (every anchor needs one, see R2 in `SKILL.md`) is marked in the modeler's XML with `<key>` and `<identifier>` elements, as `example-model.xml` shows. `minimal-model.xml` has none and generates, so they are not needed to get the DDL; keep them when the XML comes from the modeler.
- `<metadata capsule="public"/>` on a construct is its schema. `generator="true"` on an anchor, a nexus or a knot gives its id a sequence (`CREATE SEQUENCE {Mnemonic}_{Descriptor}_ID_SEQ` and `default seq.nextval`), which the load patterns draw from.
- A `<description>` on a construct becomes a `COMMENT` on its table and views.
- `<layout>` elements, with `x`, `y` and `fixed`, are positions in the modeler's drawing. The generator ignores them.

## The metadata element

The `<metadata>` directly under `<schema>` holds about fifty settings that decide the whole script: the name of the columns (`metadataPrefix`, `changingSuffix`, `identitySuffix`), the naming convention (`naming`), whether there are metadata columns (`metadataUsage`), equivalence (`equivalence`), checksums, the default for *now* (`now`), the data type of time stamps (`chronon`) and whether the model is `uni`, `bi` or `crt` (`temporalization`).

**It has to be complete.** The generator does not fill in defaults for what is missing: a missing setting does not give an error but names such as `GEN_undefined`, and no metadata columns. So:

- Copy the whole `<metadata>` element from `generator/minimal-model.xml` or `generator/example-model.xml`, and change only the settings the user asks for.
- After generating, search the script for the text `undefined`. If it is there, the metadata is incomplete; do not run the script.
- `databaseTarget` is set to `Snowflake` by the generator, whatever the XML says.
- A temporalization given as the second argument of `ANCHOR_GENERATE` replaces the one in the XML.

## Settings that users ask about

| Setting | Values | Effect |
|---------|--------|--------|
| `temporalization` | `uni`, `bi`, `crt` | uni-temporal (changing time only), bitemporal (positing and changing time), concurrent reliance temporal (positing, changing and reliability) |
| `naming` | `improved`, `original` | the naming convention of the Anchor Modeler; `original` gives knotted columns no equivalent or checksum names |
| `metadataUsage` | `true`, `false` | `Metadata_` columns on the tables |
| `equivalence` | `true`, `false` | equivalent knots and attributes, with the views that unfold them |
| `checksum` | `true`, `false` | checksum columns on attributes |
| `now` | a Snowflake expression | the time that the now perspectives use; `sysdate()` is UTC |
