# Time

Load this when the user talks about history, dates, "as of", "at the time", corrections or time zones (G1, G2, R1), when you plan where the timestamps of a load come from (L1), and when a question asks what was true or known at some time (Q2). The practices are the author's, from his writing on time in databases (*Sources* at the end). Figures from them were measured on SQL Server, on one data set: they show the kind of effect to look for, not what the user's database will do.

## Timelines and where each goes

| Kind | Means | Where it lives | Example |
|---|---|---|---|
| **Appearance time** (changing time; also valid, effective or application time) | since when a value or relationship holds in the domain | `ChangedAt` of a historized attribute or tie | the new price is in effect from Friday |
| **Assertion time** (positing time; also called transaction time) | when someone assessed or asserted a statement | `PositedAt` on the assertion in bi and crt | the manager announced the Friday price on Monday |
| **Recording time** (load time) | when a particular system stored the information | recording metadata, for example via `Metadata_` and the batch log | the warehouse loaded the announcement on Tuesday |
| **Happening time** | when a domain event occurred or is scheduled | an ordinary time-valued attribute, static or historized; not its `ChangedAt` | a performance's date, a person's birth date, an order's delivery date |
| **Structuring time** (schema versioning time) | when the model had a particular structure or set of constraints | a model version and deployment history, if needed; not a timestamp on a value | price discounts were first modeled in October |

- **A column that holds a date is not automatically one of them.** An order date, a birth date and a delivery date are happening times: model each as an attribute. Do not put them in `ChangedAt`, and do not ask the user to "choose a temporal type" for them. If the order date itself gets corrected, that is a change of the attribute's value, and the attribute may be historized like any other.
- **Changing time is one column, the start of an interval.** The next row for the same identity closes the previous one, so the interval is open until another row arrives. Questions about where intervals begin and end need more than one row; the usual question, "what was valid at time T", needs only the latest row on or before T (the point-in-time perspective).
- **Keep the timelines apart.** Appearance and assertion times are orthogonal: someone can assert a future change or correct a past one. In bi and crt each axis has its own column. If one timestamp is used as a proxy for another, label the proxy and its limitations; do not treat them as equivalent.
- **Recording time may be several times**: data that passed through systems was recorded once in each. Metadata can hold an array of them (source system, who entered it, when it was loaded); `metadata.md` has the batch side.

## How to represent a time

1. **UTC, ISO 8601, a total order.** Store times that order things in UTC. Never local time (daylight saving makes an hour happen twice, so two different moments get the same value), and never a time of day alone. Convert at the edge: say which zone a source uses and convert when loading; convert again for display. In Snowflake use `SYSDATE()` for *now*: it is UTC, whatever the session's `TIMEZONE` (`CURRENT_TIMESTAMP()` cast to NTZ is the session's local time, and an interactive session and a task can differ).
2. **Precision finer than the shortest gap between two events of one identity.** A coarse type can make two events one key (SQL Server `datetime` rounds to about 3 ms; Snowflake's `TIMESTAMP_NTZ(9)` does not). Where the source is coarser (only a date), a smaller type is cheaper in every key and join: the author's example is `date` at 3 bytes against `datetime2(7)` at 8. So ask what granularity the events really have, and set the type of `timeRange` (and of `chronon`, `positingRange`) to match, for example `TIMESTAMP_NTZ(9)` as generated or `DATE` for day-level history; do not take the default blindly.
3. **Distinguish unknown, ended, and retracted.** An unknown value has no stored row. A domain fact that ends is a change in the domain: record the end, for example with `Inactive` from its changing time, and keep the earlier history. A retraction withdraws a claim on the assertion timeline; it does not say when the fact ended in the world. Physically deleting uni history is not an ending either. If the model needs a beginning or end of time, agree on what each means; do not overload NULL.
4. **Choose the least temporalization that answers the questions.** Use uni when changing time is enough, bi when you also need past assertions from a single viewpoint, and crt when you need separately attributed or weighted assertions. Recording or publication history is a separate requirement. Snowflake has no generated triggers to enforce the assertion rules; validate them before publishing.

## Where the times of a load come from

Ask, for every timestamp column of a source: what does it mean (a change, a load, an extract), at what granularity (second, day), in which zone, and can the row be corrected later. A source rarely says.

- **The source gives change times:** use them as `ChangedAt`, including for late arrivals. Plan for late versions explicitly (`SKILL.md`, L2); a latest-only snapshot recipe must report an unsupported late arrival, not silently lose it. Recording time still tells when the rows arrived.
- **The source gives none (snapshots only):** if a proxy is acceptable, use a stable observation timestamp, such as the original extract or first load time, as `ChangedAt`. Keep the actual recording time separately; do not recompute the proxy on retry. Say how much the proxy can lag the actual change and size the observation interval to the tolerated error. If a proxy is not acceptable, report that the changing time is unknown.
- **The ABA problem.** A snapshot cannot tell *A, then B, then A again* from *A all the time* when both happen between two loads: it sees only the ends. If the user needs those changes, ask for change timestamps, delta files or an audit log from the source. Nothing in the model can recover them.
- **A value that disappears** might mean an incomplete extract, a fact ending, or a claim being withdrawn. Ask what the source guarantees before taking any action. Absence from a snapshot alone is not evidence for a timestamp, a retraction, or a delete.
- **A row that repeats a key and a time** may be a correction, the next step of a sequence, or another thing: ask which, and for the granularity of the timestamp, before choosing the key.

## Corrections, retractions and reliability (bi and crt)

The author's talks on bitemporal and concurrent-reliance-temporal data (*Back to the Moment*, *Posits and Assertions*) add two ideas to the three kinds above. Use them to explain, and to check loads.

- **A posit and an assertion are different.** A posit states a value (or relationship) and its changing time, if historized. An assertion says when a positor assessed that posit and with what reliability. Reuse the posit when recording a new assessment of the same version; do not change it in place.
- **A statement is an assertion**: who says it (the *positor*), how much they trust it (*reliability*, from -1, certainly not so, through 0, nothing is claimed, to 1, certainly so), and when. In bi there is one implicit positor and the reliability is only *shown* (1) or *retracted* (0). In crt each positor has their own. Several positors may disagree about the same thing without conflict.
- **Never update or delete to correct.** A **retraction** is the same statement asserted again, later, at reliability 0. A **correction** is a retraction of the old value plus an assertion of the new one: the author's golf deck loads the pair (old at 0, new at 1) at the same posit moment, and restating something that is already asserted changes nothing. Both versions stay queryable: "what was true", and "what did we believe on that day".
- **One reliability per positor and posit moment.** Two different reliabilities for the same posit by the same positor at the same time contradict each other. Snowflake has no triggers to reject them, so a load must not produce them, and the integrity checks of the generated model look for them.
- **Uncertain identity: keep the candidates.** When it is not known whether two records are the same thing, do not merge or guess. Tie the record to each candidate with a reliability (early: 0.7 to one and 0.3 to the other as the information arrives; late: 0.5 to each when the data cannot tell), and correct later by asserting 1 and 0. The reliabilities of the ties still add up to the number of real relationships, so a count weights by reliability, not by rows. If nothing in the data can decide, ask the source or a person.
- **Unknown versus a value of "Unknown".** Where the business acts on not knowing (an order whose delivery date is open), give the knot an explicit value for it; where nobody does, leave the row out. The author's 2007 rule: an attribute has a row only when the value is known.

## Ending a value versus deleting history in uni

A historized attribute keeps what it was; a source that stops sending a value says nothing about when it ended. Record a known ending as a domain fact, not by deleting history. A null in a load is not a delete. If a user explicitly requests physical removal, treat it as a separate approved operation and record what was removed in the batch log as policy permits. For the history of corrections, use bi or crt.

## Asking what was in effect

- **What was in effect then?** Use a changing-time cutoff (`p` for a chosen time, `n` for now where available); `l` has no changing-time cutoff and may return future-effective data. The perspective resolves versions, not whether an identity existed then. Model and filter on the relevant existence or eligibility facts for counts.
- **What did someone believe then?** Use both changing and assertion cutoffs (the `t` perspective where available; inspect its signature) and choose the relevant positor in crt. **What had this warehouse received or published then?** That is recording or publication time. Reproducing a past report may also need the original inputs and metric/model definitions; bi or crt alone does not guarantee it.
- A lookup of the row in effect is needed twice in bi: the latest assertion, then the latest appearance among those. The lookup is a window function or a `QUALIFY ROW_NUMBER()` in Snowflake, which the generated perspectives already do. For as-of joins over many reference points see `performance.md`.

## Sources

Posts at https://www.anchormodeling.com/ by Lars Rönnbäck: *Time in Databases* (`/time-in-databases/`), *Three Concepts of Time in Anchor Modeling* (2009 note, `/three-concepts-of-time-in-anchor-modeling/`), *Time is both one and many*, *Temporal Complexity*, *Information in Effect and Performance*, *The ABA Problem*, *Poor Man's Auditability*, *A Lack of Context*.
