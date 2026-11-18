# Identity and keys

Load this when a user asks how to identify things, wants hashed keys (a Data Vault habit), has natural keys that change or are reused, or asks how loads map source rows to identities (R2, G2, L1). The advice is the author's (*Sources* at the end).

## The rules

1. **An identity is a meaningless surrogate integer**, preferably allocated from a sequence where available. A natural or source key is a way to find the identity, not the identity itself. A static identifier attribute is the simplest pattern when one stable value suffices; a key can also be a composite route through attributes and roles or be resolved by an upstream identity service. Never expose a surrogate as a business key, interpret its value, or reuse one.
2. **Do not hash a natural key to make an identity.** Four costs, to say out loud when the user asks for it:
   - **A key that can change makes a new identity.** A hash keeps every trace of its input, so a corrected or renamed key gives another hash, which is another thing. The repair (a same-as tie and duplicated rows) has to be allowed for in every query afterwards.
   - **Size.** The author's example is a tie over three things: three hashes of 128 bits make 384 bits per row, three sequence integers of 32, 32 and 1 bits make 65 (about six times less). Every key, index and join pays it, and so does every row of every tie. As text, ten million hashes need about 1.2 billion characters against under 70 million for sequence numbers.
   - **CPU and risk.** The safer the hash, the more it costs to compute; a collision may be unacceptable, and cannot be ruled out.
   - **Nothing is gained**: the natural key is already in its attribute, and the generator's key views (`key_`, `pkey_`) find an identity from it.

   What the user probably wants from a hash is a **cheap comparison of long values**: that is the `checksum` option (`model-xml.md`: the `checksum` setting, and the `checksum` flag of a knot or an attribute), which adds a hash column used for comparing and for uniqueness. It is not the identity.
3. **Historize the facts that make a lookup key change.** The same identity can have several keys over time. Scope the match by source or other namespace when keys are not globally unique. If a key can be reused, match at the relevant changing time, not against any version or just the latest. Where supported, a historized route has a point-in-time key (`pkey_`); for many reference times, resolve that route over the relevant history. Stop on an ambiguous match. The generated `ic_key_` view is not a guarantee of uniqueness across all historical slices.
4. **A key is a repeatable lookup, not a prescription for model structure.** The model describes the domain, and key routes traverse it for automation. A **tie identifier role**, by contrast, says which participants identify a relationship instance and determines its cardinality. A key route or key view does not by itself enforce uniqueness.
5. **Assign the identity of a thing centrally**, in one place (the anchor's sequence), not in each source or application. Two sources that mean the same thing are told apart or joined through their keys, not by giving them each an identity.

## Late or early

| | What it means | When |
|---|---|---|
| **Late** | the natural key is stored in the warehouse (the identifier attribute) and the identity is generated from it: new keys get new identities, known keys find theirs | the default, and what the load patterns in `SKILL.md` and `ddl-patterns.md` do: it can be rebuilt, queried and checked from the stored keys |
| **Early** | the connection from source key to identity is made in the integration process and not stored | only when the user has an identity service that already does it; ask for it, and keep the same one for every load |

## Distribution

Snowflake spreads tables over micro-partitions itself and has no distribution key to choose, so the author's ideas about co-locating a parent and its children on one node (one sequence per node) do not apply here. Identities come from the generated sequence (`<name>_ID_SEQ`), never from `IDENTITY` (`SKILL.md`, *Sequences, Not IDENTITY*).

## Sources

*On the hashing of keys* (`https://www.anchormodeling.com/on-the-hashing-of-keys/`), *Identification, identity, and key*, *Visualization of Keys*, *Ensemble Modeling*, the 2007 cheat sheet (identity management, early and late), *Co-locating in an MPP-RDBMS*.
