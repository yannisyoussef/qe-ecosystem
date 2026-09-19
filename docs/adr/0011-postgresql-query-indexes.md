# ADR-0011: Cross-run questions are answered from rebuildable PostgreSQL indexes, one run from its own source

Status: Accepted
Date: 2026-09-13
Amended: 2026-09-19 (history order, canonical-source repair, project id contract)

## Context

The durable write side is complete: complete validated runs are archived
as their original protocol source (ADR-0008), their attachment bytes are
immutable content-addressed blobs (ADR-0009), and both have an explicit
lifecycle (ADR-0010). The only complete reporting read model is the
in-memory one from QE-004. It answers the three questions the product
has: what happened in this run, how has this exact historical test
behaved, and was it flaky. It answers them by projecting every run and
assembling one snapshot, which is correct and does not scale: a history
of one test cannot require replaying an installation, and a future
transport cannot be built on a substrate whose cost grows with the
archive.

The tempting shortcuts are all worse than they look. Reimplementing the
projector's rules in SQL would put the definition of a verdict, a final
attempt, a historical identity, or flakiness in two places that must
agree forever. Storing the projected run as a document would freeze
today's interpretation into the only copy of it. Keeping running totals
of flakiness would make a cache that nothing can prove correct.

## Decision

The archived protocol source remains the only semantic truth. PostgreSQL
gains two derived tables, and they are rebuildable indexes rather than a
second write model.

A whole run is answered by replaying that one run: its stored lines
through the validator into `projectRun`, read on a single connection in a
read-only repeatable-read transaction so that the four collections it
spans agree, and taking no maintenance lease. It never consults an index,
so it answers whether or not the project is indexed.

Questions across runs are answered from `qe_run_query_index`, one row per
run holding the listing facts of its projection, and
`qe_history_occurrences`, one row per execution that qualifies as
history. Both are filled by copying facts out of a `ProjectedRun`: the
indexer derives nothing. History occurrences come from a single pure
function in the read-model package, which the in-memory snapshot also
calls, so the two cannot drift apart. The stored `flaky` is the
projector's own boolean; SQL counts those booleans and never decides what
flakiness is.

The history key stays `(project, runner name, historical id)`. History
order is the producer timestamp's position, then run id, then execution
id: a presentation order, not global chronology. The two tie-breakers
are protocol identifiers, which are printable ASCII, so the
`COLLATE "C"` byte order declared on them is exactly code-unit order.

Building a durable index exposed a gap in the in-memory order it was to
reproduce. That comparator subtracted `Date.parse` results, and
`Date.parse` cannot read a leap second, which the protocol's timestamp
schema admits as the last second of a UTC day: such a timestamp got
`NaN`, which is no order at all, and no scalar a database can store.
Reading `23:59:60` as `23:59:59` plus one second instead, as a first
attempt did, collides with the `00:00:00` that follows, so the run ids
decide. QE-008 therefore hardens the read model rather than copying it.
One primitive in the read-model package, `historyInstant`, defines the
position, and the in-memory history, the index writer, and PostgreSQL's
paging all use it:

- an ordinary timestamp is its instant to the millisecond as
  `Date.parse` reads it, so every ordinary history keeps exactly the
  order it had;
- a leap second sorts after every instant of the second before it and
  before the `00:00:00` after it, and by its own milliseconds among
  leap seconds. One `timestamptz` cannot hold that apart from the next
  second, so the index stores a pair: the instant, which for a leap
  second is the last millisecond of `23:59:59`, and a leap place, 0 for
  an ordinary timestamp and 1 plus the millisecond inside a leap second;
- a string the validator refuses has no position and is refused rather
  than mapped to an arbitrary one. None can be archived, so none can
  reach the index.

This is read-model ordering semantics, not wire semantics: the protocol
is unchanged.

The index is keyed by a digest of the runner name and the historical id
rather than by the names: the protocol bounds each at 512 characters,
which in multi-byte text is more than an index key can hold, and the
names are compared exactly beside the digest. Paging is keyset, never
offset.

An index row carries a `QUERY_INDEX_VERSION`, distinct from the protocol
version, the migration version, and the package version, and the content
fingerprint of the source it was derived from. A run is currently indexed
only when both match. A project with any missing or stale run refuses
`listRuns`, history, and flakiness with a structured error rather than
answering a question about part of itself.

Runs archived by the current writer are indexed in their own archive
transaction, so new data is queryable immediately. An already archived
run is different. The content fingerprint deliberately ignores identical
duplicate lines (ADR-0008), so a later, idempotent re-ingestion can
offer the same run in a different physical form with different validator
facts, such as its duplicate count. When such a re-ingestion finds the
run's index missing or stale, it repairs it from the archive's own
stored source, under the run's row lock, and never from what it was
offered:

> An already-present run's query index is always derived from the
> archive's stored raw source, never from an idempotent physical
> representation offered later.

An archive whose own source no longer replays fails the re-ingestion
loudly instead of being indexed from the offered copy.

Runs archived before the indexes existed, and any whose index is missing
or stale, are brought in by an explicit, bounded rebuild that reads the
source, never a stale index, holds the shared maintenance lease so retention cannot
delete a run underneath it, and replaces one run's derived rows in one
transaction under a row lock. Nothing is scheduled. Retention needs to
know nothing about any of this: the derived tables cascade from the run.

## Options considered

1. Replay every archived run for each history request. Correct and
   unusable: the cost of one question grows with the installation.
   Rejected.
2. Serve the stored `validation_summary` as the product read model. It is
   an ingestion-time audit record of validator counts, not a projection,
   and ADR-0008 says so. Rejected.
3. Store the `ProjectedRun` as JSONB and query into it. Freezes today's
   interpretation as the stored artefact, and makes the document the
   thing that would have to be migrated when the projector changes.
   Rejected in favour of explicit columns derived from a replay.
4. Reimplement the projector and the flakiness rule in SQL. Two
   definitions of a verdict and of flakiness, in two languages, that
   nothing can hold together. Rejected outright; if indexing had needed
   it, the answer would have been to refactor the projector boundary.
5. Persist mutable history and flakiness aggregates. A cache updated by
   writes, whose agreement with the occurrences nobody can check.
   Rejected; aggregation over occurrence rows is cheap and provable.
6. Serve partial projects while a backfill is running, behind a flag. A
   history missing the runs nobody indexed yet is wrong in a way the
   caller cannot see. Rejected; incompleteness is an error with counts.
7. Offset pagination. Skips and repeats rows the moment anything is
   archived or deleted between pages. Rejected.
8. Database triggers parsing the raw event JSON into index rows. Puts
   protocol interpretation in the database, where the validator's rules
   are not available. Rejected.
9. Letting a failure to derive an index refuse the archive. Derived
   state would then decide what may be stored, which inverts the whole
   arrangement. Rejected: the derivation is total over everything the
   validator accepts, which is everything that can be archived.
10. Repair an existing run's index from the representation a
   re-ingestion offers, because its fingerprint matches. The fingerprint
   is semantic identity, not physical identity; the index would then
   describe a source that is not the archive. Rejected.
11. Keep one `timestamptz` per occurrence, reading a leap second as the
   next second, and give any unreadable timestamp the earliest instant.
   The first collides with the following second and lets identifiers
   decide chronology; the second invents chronology for strings no
   validator accepts. Rejected for an explicit pair and a refusal.
12. Bound the project id inside the query surface. ADR-0007 defines it as
   opaque and non-empty, and a query-only limit would let the store
   archive a project the queries refuse. Rejected: one contract
   everywhere.
13. Relational tables for sessions, attempts, steps, scope failures, and
   attachments. A relational rewrite of the projected run, and a second
   place its shape is defined. Rejected: one run replays.

## Consequences

- `ts/packages/postgres` gains a query surface beside its archive: run
  lookup by replay, and keyset-paged listings, history, and flakiness.
- Migration 4 creates the two derived tables empty. Backfilling is the
  explicit rebuild operation, not something a migration does.
- A project is completely indexed or it does not answer cross-run
  questions. Operators see exactly what is missing and rebuild it.
- Deleting a run deletes its derived rows by cascade, and the surviving
  project stays complete with no orphaned occurrence.
- Ordering is provably the in-memory ordering, because both come from
  one primitive, the position is stored as it computes it, and the
  collation is declared rather than inherited. Ordinary timestamps keep
  their previous order; the leap second's place is deliberate.
- Nothing derived can refuse an archive: every archived run is
  validator-valid, and every validator-valid timestamp has a position.
  The index is a consequence of the archive and never a condition of it.
- An existing run's index always describes its archived source, whatever
  physical form a later re-ingestion offers.
- The query surface takes the same project id the store archives under.
- There is still no HTTP, no authentication, and no search: display
  names, tags, labels, failure text, time windows, and trends are not
  queryable, and would need evidence from a real interface first.

## Deferred

Transport and authentication; any search or filtering beyond the three
established questions; materialised aggregates, if a measured query need
ever justifies them, and then as explicitly rebuildable state; a second
persistence provider.
