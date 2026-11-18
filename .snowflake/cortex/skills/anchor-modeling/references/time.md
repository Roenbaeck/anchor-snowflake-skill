# Time

Load this when the user talks about history, dates, "as of", "at the time", corrections or time zones (G1, G2, R1), when you plan where the timestamps of a load come from (L1), and when a question asks what was true or known at some time (Q2). The practices are the author's, from his writing on time in databases (*Sources* at the end). Figures from them were measured on SQL Server, on one data set: they show the kind of effect to look for, not what the user's database will do.

## Three kinds of time, and where each goes

| Kind | Means | Where it lives | Example |
|---|---|---|---|
| **Changing time** (what others call valid, effective or application time) | since when a value or relationship holds in the world | `ChangedAt` of a historized attribute or tie | the name has been *Ann Lee* since 2020-03-01 |
| **Happening time** | when an event took place | an **ordinary attribute of the event**, static or historized; never `ChangedAt` | a performance's date, a person's birth date, an order's delivery date |
| **Recording time** (transaction or positing time) | when the database learned it | metadata (the `Metadata_` reference; in bi and crt the posit's `PositedAt`) | loaded by job 17 on 2026-11-18 at 03:00 |

- **A column that holds a date is not automatically one of them.** An order date, a birth date and a delivery date are happening times: model each as an attribute. Do not put them in `ChangedAt`, and do not ask the user to "choose a temporal type" for them. If the order date itself gets corrected, that is a change of the attribute's value, and the attribute may be historized like any other.
- **Changing time is one column, the start of an interval.** The next row for the same identity closes the previous one, so the interval is open until another row arrives. Questions about where intervals begin and end need more than one row; the usual question, "what was valid at time T", needs only the latest row on or before T (the point-in-time perspective).
- **Keep the timelines apart.** Appearance time (changing) and assertion time (positing) are orthogonal: either can change without the other. Never store both in one column, and never let one stand in for the other. bi and crt give each its own column; do not merge them in views or loads.
- **Recording time may be several times**: data that passed through systems was recorded once in each. Metadata can hold an array of them (source system, who entered it, when it was loaded); `metadata.md` has the batch side.

## How to represent a time

1. **UTC, ISO 8601, a total order.** Store times that order things in UTC. Never local time (daylight saving makes an hour happen twice, so two different moments get the same value), and never a time of day alone. Convert at the edge: say which zone a source uses and convert when loading; convert again for display. In Snowflake use `SYSDATE()` for *now*: it is UTC, whatever the session's `TIMEZONE` (`CURRENT_TIMESTAMP()` cast to NTZ is the session's local time, and an interactive session and a task can differ).
2. **Precision finer than the shortest gap between two events of one identity.** A coarse type can make two events one key (SQL Server `datetime` rounds to about 3 ms; Snowflake's `TIMESTAMP_NTZ(9)` does not). Where the source is coarser (only a date), a smaller type is cheaper in every key and join: the author's example is `date` at 3 bytes against `datetime2(7)` at 8. So ask what granularity the events really have, and set the type of `timeRange` (and of `chronon`, `positingRange`) to match, for example `TIMESTAMP_NTZ(9)` as generated or `DATE` for day-level history; do not take the default blindly.
3. **Special points are not all NULL.** There is no NULL in an Anchor model. *Unknown* is the absence of a row. *Ended* is a retraction in bi and crt, and in uni a deletion made by the load (below), or a value in a knot that means "none" when the user wants the ending kept as information. A time that is not known is not invented: say so, and keep the load time as the recording time only. If the model needs a beginning or end of time, give each its own agreed value (the generator's `EOT` in crt is one) and never use NULL for them.
4. **Choose the least temporality that answers the questions.** The author counts the cases a design has to consider at about 4 without history, 18 with versions (uni), 162 with corrections (bi) and 810 with several asserters (crt). History is cheap; corrections are the big jump. So: uni by default; bi only if reports must reproduce a past state *after* corrections arrived; crt only if sources or people disagree and each version is to be kept. The generated triggers hold the rules, so the user does not see the complexity: another reason not to hand-write them.

## Where the times of a load come from

Ask, for every timestamp column of a source: what does it mean (a change, a load, an extract), at what granularity (second, day), in which zone, and can the row be corrected later. A source rarely says.

- **The source gives change times:** use them as `ChangedAt`. Rows that arrive late or out of order are loaded with their own change time anyway: changing time puts the changes in their order, recording time records the order they arrived in. This is what makes loads idempotent and replayable.
- **The source gives none (snapshots only):** `ChangedAt` is the load time, a *proxy*. Say what error that brings: a change is placed as late as the interval between loads. The author's rule of thumb is that a proxy for assertion time may be off by about an hour, and one for appearance time by days. Never present a proxy as the real time; size the load interval to the error that the user accepts.
- **The ABA problem.** A snapshot cannot tell *A, then B, then A again* from *A all the time* when both happen between two loads: it sees only the ends. If the user needs those changes, ask for change timestamps, delta files or an audit log from the source. Nothing in the model can recover them.
- **A value that disappears** from the source is probably a deletion, not an unknown: model it as a retraction (bi, crt) or a deletion made by the load (uni), after asking.
- **A row that repeats a key and a time** may be a correction, the next step of a sequence, or another thing: ask which, and for the granularity of the timestamp, before choosing the key.

## Deleting a value in uni

A historized attribute keeps what it was; a source that stops sending a value says nothing about when it ended. Snowflake has no triggers on tables, so a model's latest view cannot take a deletion as an update, and a null in a load is not a deletion. When the user needs a value to end: in bi and crt, load a retraction (an annex row with reliability 0); in uni, the load deletes the rows (it moves them to a deletion table of its own if the user wants an audit of deletions, with the time of the deletion) and says so in the batch log. Ask first whether a value that disappears is a deletion or an unknown (see above), and do not delete without asking. For the history of corrections, use bi or crt.

## Asking what was in effect

- "What was true then" is changing time; "what did we know then" is recording time. If the user asks the second kind, the model needs bi or crt (`modeling-practice.md`, *Questions in plain language*).
- A lookup of the row in effect is needed twice in bi: the latest assertion, then the latest appearance among those. The lookup is a window function or a `QUALIFY ROW_NUMBER()` in Snowflake, which the generated perspectives already do. For as-of joins over many reference points see `performance.md`.

## Sources

Posts at https://www.anchormodeling.com/ by Lars Rönnbäck: *Time in Databases* (`/time-in-databases/`), *Three Concepts of Time in Anchor Modeling* (2009 note, `/three-concepts-of-time-in-anchor-modeling/`), *Time is both one and many*, *Temporal Complexity*, *Information in Effect and Performance*, *The ABA Problem*, *Poor Man's Auditability*, *A Lack of Context*.
