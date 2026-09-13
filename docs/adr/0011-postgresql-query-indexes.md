# ADR-0011: Cross-run questions are answered from rebuildable PostgreSQL indexes, one run from its own source

Status: Accepted
Date: 2026-09-13

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

The history key stays `(project, runner name, historical id)`, and the
history order stays the in-memory order: producer instant, then run id,
then execution id. The instant is stored as the in-memory comparator
reads it, to the millisecond, computed before it is stored rather than
parsed by the database; the two tie-breakers are protocol identifiers,
which are printable ASCII, so the `COLLATE "C"` byte order declared on
them is exactly code-unit order. Both models read a producer's clock
through one shared function, so a clock that no obvious parse turns into
an instant, such as a leap second, cannot order differently in the two.
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
transaction, so new data is queryable immediately. Everything else is
brought in by an explicit, bounded rebuild that reads the source, never a
stale index, holds the shared maintenance lease so retention cannot
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
   arrangement. Rejected: the derivation is total.
10. Relational tables for sessions, attempts, steps, scope failures, and
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
- Ordering is provably the in-memory ordering, because the instant is
  stored as read and the collation is declared rather than inherited.
- Nothing derived can refuse an archive. A run the indexer cannot read a
  clock from is still stored, still replayable, and still indexed; the
  index is a consequence of the archive and never a condition of it.
- There is still no HTTP, no authentication, and no search: display
  names, tags, labels, failure text, time windows, and trends are not
  queryable, and would need evidence from a real interface first.

## Deferred

Transport and authentication; any search or filtering beyond the three
established questions; materialised aggregates, if a measured query need
ever justifies them, and then as explicitly rebuildable state; a second
persistence provider.
