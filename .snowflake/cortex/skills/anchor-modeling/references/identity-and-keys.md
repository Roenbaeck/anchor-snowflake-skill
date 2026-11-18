# Identity and keys

Load this when a user asks how to identify things, wants hashed keys (a Data Vault habit), has natural keys that change or are reused, or asks how loads map source rows to identities (R2, G2, L1). The advice is the author's (*Sources* at the end).

## The rules

1. **An identity is a meaningless integer from a sequence.** The anchor holds the identity of a thing, technical by nature; the natural key is **data about the thing**, kept in an attribute (the identifier attribute), not the identity. Never expose a surrogate as a business key, interpret its value, or reuse one.
2. **Do not hash a natural key to make an identity.** Four costs, to say out loud when the user asks for it:
   - **A key that can change makes a new identity.** A hash keeps every trace of its input, so a corrected or renamed key gives another hash, which is another thing. The repair (a same-as tie and duplicated rows) has to be allowed for in every query afterwards.
   - **Size.** The author's example is a tie over three things: three hashes of 128 bits make 384 bits per row, three sequence integers of 32, 32 and 1 bits make 65 (about six times less). Every key, index and join pays it, and so does every row of every tie. As text, ten million hashes need about 1.2 billion characters against under 70 million for sequence numbers.
   - **CPU and risk.** The safer the hash, the more it costs to compute; a collision may be unacceptable, and cannot be ruled out.
   - **Nothing is gained**: the natural key is already in its attribute, and the generator's key views (`key_`, `pkey_`) find an identity from it.

   What the user probably wants from a hash is a **cheap comparison of long values**: that is the `checksum` option (`model-xml.md`: the `checksum` setting, and the `checksum` flag of a knot or an attribute), which adds a hash column used for comparing and for uniqueness. It is not the identity.
3. **A natural key that changes is a historized identifier attribute.** One identity carries several identifiers over time, without duplication. A load maps source rows to identities through **any version** of the identifier, and asks whether a key may be reused by another thing later (if it may, the time is part of the match, and `ic_key_` views report keys that two identities share). For a route that passes something historized the generator makes a point-in-time key (`pkey_`), since the key itself changes.
4. **A key is a repeatable way to identify something, and it never shapes the model.** It is whatever lookup, always repeated the same way against well-defined data, finds the same thing; it can be replaced, is not the only way, and is not necessarily permanent or independent of time. So do not let a key decide what is an anchor, an attribute, a tie or a time frame: the model describes the domain, and the keys are routes through it for automation. In the generator they are informational routes (`naturalKeyAttributes`, key views); the database does not enforce them.
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
