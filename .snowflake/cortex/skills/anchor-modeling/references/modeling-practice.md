# Modeling Practice: Getting the Model Right

A model is a statement of what the business means. The DDL is the easy part; the risk is a model that is valid, loads, and says something the business did not mean. That happens when a question of meaning is settled by a guess that nobody saw. Load this file at R1, R2, G1, G2 and E2 and follow it.

## Principles

1. **Settle no question of meaning silently.** Every modeling decision is one of four things, and you can say which: *stated* by the user (quote them), *found* in the data (name the query), *asked*, or *assumed* (listed, see Assumptions). Never a fifth.
2. **Talk about the domain, not about Anchor Modeling.** The person answering may not know the vocabulary. Do not use *anchor*, *knot*, *tie*, *nexus*, *historized*, *identifier* or *attribute* in a question or in what you show for confirmation, unless the user used the word first. Use their words and an example from their domain.
3. **Ask what is costly to reverse; assume the rest and say so.** Too many questions is a cost too. See *Which decisions matter*.
4. **Ask the data before you ask the person.** A query answers *what is*; only a person can answer *what it means* and *what may happen*. See *Asking the data*.
5. **Confirm with sentences and examples, not diagrams.** People can judge a sentence about their own work, and a concrete case, far better than a table of constructs. See *Presenting the model*.
6. **Keep the confirmed meaning in the model.** Put it in the `<description>` of each construct, in the user's words; it becomes the `COMMENT` on the table, and the next reader sees what was agreed.

## Which decisions matter

| Decision | Cost of being wrong | So |
|---|---|---|
| What is a thing with an identity, and what is only a property of one | High: tables are restructured and reloaded | Ask, unless the user or the data leaves no doubt |
| How to tell two of a thing apart (its natural key), and whether that is unique and stable | High: loads map source rows to identities through it; a wrong key merges or splits things | Ask, and check it against the data |
| How many of one thing can relate to how many of another, at the same time (the key of a relationship) | High: tightening later loses nothing, loosening needs a migration | Ask with an example |
| Whether a relationship is a happening with its own details and date | High | Ask |
| Whether the model must tell *what was true* from *what we knew* or from *who said it* (uni, bi or crt) | High: it changes every table | Ask in plain words, default uni |
| Which properties change and need their history kept | Medium: history that was not kept cannot be recovered; adding it later is a migration | Ask for the ones that are not obvious; assume for the rest |
| A short list of values or a free property | Medium | Decide from the data, confirm |
| Mnemonics, descriptors, schema names, data types | Low | Assume, show them |

## Questions in plain language

Ask these in the user's words. Each has a guess of your own in it, so that a plain yes is enough, and an example from their domain, so that the question cannot be misread. Batch them: at most five in a round, numbered, the costliest first, and say that "yes to all" is a fine answer.

| To decide | Ask like this | The answer settles |
|---|---|---|
| A thing | "Is an *order* something you can point to and give a number, one single order that other things refer to? Could the same order turn up twice?" | thing or property |
| Its key | "How do you tell two *actors* apart in real life? Can two share that, or can it be reused or corrected later?" | the identifying property, and whether it is unique and stable |
| Change | "If an *actor's name* changes, say after a marriage, do you need to know the old name? Would anyone ask what it was last March?" | whether the property keeps its history |
| Relationship, how many | "Can one *program* be played on more than one *stage* at the same time? Can one stage have more than one program at the same time?" | the key of the relationship |
| Relationship, the same pair twice | "Can the same *actor* be in the same *program* twice, perhaps with different ratings?" | whether a relationship may repeat, and what makes it distinct |
| Relationship over time | "Does the link itself begin and end, like an actor leaving a program? Do you need to know who it linked before?" | whether the relationship keeps its history |
| A happening | "Is a *performance* something that happened at a time, with details of its own and several parties? Is it ever corrected afterwards?" | thing, relationship, or happening |
| A fixed list | "Is *gender* always one of a short list? Who may add a value? Does a value have a longer text besides its code?" | list of values, or a thing with a code and a description |
| Missing values | "Is a *phone number* always known? If it is not, is the *customer* still a valid customer?" | whether absence is normal (it is, in this model: no row) |
| Mistakes and endings | "When a value was wrong, do you overwrite it or keep the wrong one marked as wrong? When something ends, does it disappear?" | corrections and retractions, deletability |
| What was true or what we knew | "Do you ever need to reproduce a report exactly as it looked on a past date, although corrections have come in since? Do sources or people disagree about the same fact, so that you want to keep each version?" | bitemporal, concurrent-reliance-temporal, or uni (neither: the default) |

## Presenting the model

Show the user, in this order. Show the construct table last, as the technical record, and only if they want it.

1. **Sentences**, one per thing, property and relationship, in the user's words:
   - "Each actor is identified by an actor code. Two actors never share a code."
   - "An actor has one name at a time. When it changes we keep the old names."
   - "An actor has one gender, one of: Female, Male, Other."
   - "An actor can be in many programs, and a program can have many actors. For each actor in a program there is one rating."
   - "A performance is held on one date, at one stage, and plays one program."
2. **Examples and a test that must fail.** Give a few made-up rows for each group of sentences, and for each relationship the case the model does not allow: "So Alicia can be in *Hamlet* with the rating *good*. She cannot have both *good* and *bad* in *Hamlet* at the same time. Is that right?" The case that fails is what the key of the relationship says, and it is where wrong models hide.
3. **Assumptions**, each with the alternative and what it would change, in plain words:
   - "An actor's name can change and we keep the old ones. (You said names are sometimes corrected. If not, we only keep the latest.)"
4. **Open questions**, at most five, each with your proposed answer.
5. **The construct table.**

The user confirms the sentences, the examples, the assumptions and the questions. Do not go on to generate on a bare "looks fine" to a table.

Put each confirmed sentence, shortened, in the `<description>` of its construct.

## Asking the data

In R1 and wherever the user can point at data, test your guesses with queries, in aggregates and not in a ten-row sample, and report what you found:

- **A key:** `SELECT COUNT(*), COUNT(DISTINCT key), COUNT_IF(key IS NULL)`. Duplicates or nulls in a key that was meant to be unique are the finding to report.
- **How many:** `SELECT a, COUNT(DISTINCT b) FROM t GROUP BY a HAVING COUNT(DISTINCT b) > 1`, in both directions, to see whether a relationship is one-to-many or many-to-many.
- **A list of values:** `COUNT(DISTINCT col)` against the rows, and how often new values appear. A small, stable set is a list; an open one is not.
- **Change:** more than one row per key with a date, or an audit or history table, shows that something changes. A single current snapshot shows nothing about change: ask.
- **Missing values:** the share of nulls for each column.

Report evidence in a sentence: "In 12,431 rows `order_no` is unique and never empty, so I would identify an order by it." Report surprises as questions. The data shows what is now; it cannot show whether a relationship may become many-to-many, or whether history will be needed. Ask about those.

## Check before you present

Go through this list on your own model, then fix a point or put it in the assumptions.

**Things and keys**
- Every thing has a property that holds its natural key, and you know whether it is unique and stable.
- Nothing is a thing in its own right that has no identity and that nothing refers to: that is a property.
- No property is hiding a thing: a "property" with details of its own, or one that many things share (a delivery address), is probably a thing.

**Properties**
- Everything the user said can change keeps its history; keys and birth-like facts are static.
- No property holds a list (several values in one, or repeating columns): make it a relationship.
- A list of values is only a short, closed, stable list with one value column; a code that has a description is a thing (see `constructs.md`, Knot).
- Each property sits on what it describes (the price on the ticket, not on the performance).

**Relationships**
- Each relationship reads as a sentence with its roles, and the roles that make up its key match the answers about how many.
- A happening with details of its own has a date and a key of its own.
- No property holds another thing's key.

**Time**
- The temporalization was asked about or defaulted to uni, and the assumptions say which.

**Names and meaning**
- Every construct has a description in the user's words.
- The names are the domain's words, not the source's (`tbl_`, abbreviations, system codes), and the mnemonics are unique.
- The model is small enough to read. More than about fifteen constructs: present it one subject area at a time.

## When the user cannot answer

Say so, use the cautious default, and record it as an assumption "to be confirmed by *whoever knows*". The cautious default is the one that loses nothing if it turns out wrong:

- **How many:** allow the larger number (many-to-many). Tightening later loses nothing; loosening needs a migration.
- **Change:** keep the history. History that was not kept cannot be recovered.
- **Time:** uni-temporal.
- **A key:** do not invent one from a row number or a surrogate. A thing without a natural key cannot be loaded twice without duplicating it; ask who knows how to tell two apart.

If the user says "just generate it", do, but put the assumption list at the top of what you present. Assumptions are never left out.
