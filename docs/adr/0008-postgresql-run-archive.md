# ADR-0008: PostgreSQL archives complete validated runs as their original protocol source

Status: Accepted
Date: 2026-09-12

## Context

With two real producers, a validator that is safe as a trust boundary,
and a read model (QE-004) that projects validated runs deterministically,
the platform needs its first durable store (QE-005). The read model gets
its input from the validator's decoded events, a representation made for
today's projector: it drops unknown optional properties and unknown but
ignorable event types, both of which protocol line 0.3 deliberately
permits so that producers can move ahead of consumers. A store built on
that representation, or on projected runs, would erase forward-compatible
data the moment it was written, and a store with its own tables for
histories or flakiness would be a second write model whose agreement with
the projector nobody could prove. Incomplete runs are viewable but
mutable by nature: the same identity may gain content when the runner
finishes, and updating archived state is incremental ingestion, which is
not this milestone.

## Decision

PostgreSQL is the first durable run store, accessed through the `pg`
driver with explicit SQL and a small versioned, append-only migration
list. The database identity of a run is `(projectId, runId)`, enforced by
the primary key; the project is caller-supplied ingestion context
(ADR-0007), there is no projects table, and the same run id may exist in
two projects. Only a run the validator reports valid and complete is an
archive candidate; `closed` is not required, so forked runs without
`run.finished` are archived, and an incomplete run writes nothing.

The durable semantic source is the original validator-accepted JSON
line, stored verbatim minus its terminator, one row per line in a
deterministic storage order (session id by code unit, session sequence,
event id, source occurrence), with its envelope facts, a digest of its
canonical form, and its disposition: accepted, ignored, or duplicate.
Unknown optional properties, unknown ignorable events, prototype-named
keys, and identical duplicate lines are all retained. Decoded events,
projected runs, histories, flakiness, and the blob catalog are rebuilt
from that source through the current validator and the read model's
projector and are never written. The physical run directory is stored as
provenance only. Attachment bytes are not stored in this milestone; their
events are, with hash and size, and the run row records that the bytes
were verified at ingestion.

A run's content fingerprint is the SHA-256 of the sorted, de-duplicated
canonical digests of its accepted and ignored lines. Re-ingesting the
same content under the same identity is idempotent (`already_present`,
first provenance kept), whatever its property order, whitespace,
duplicate-line count, or directory; different content under one identity
is refused (`RUN_CONFLICT`) and never overwrites or merges. One run is
one transaction; concurrency is settled by the constraints, not by
process locks.

## Options considered

1. Persist only the projected run. Rebuilding a run under a changed
   projector would be impossible, and unknown fields would be lost.
   Rejected.
2. Persist only the decoded accepted events. Cleaner to query, but
   forward-compatible data and ignorable events would be erased
   silently. Rejected.
3. Persist histories and flakiness as tables. A second write model
   whose consistency with the projector would have to be maintained by
   hand; rejected until replay through persistence is proven and a query
   need is measured, and then only as rebuildable indexes.
4. Mutable upserts for incomplete runs. Turns the archive into an
   incremental store before its semantics exist. Rejected for this
   milestone.
5. An ORM or a generic persistence interface first. One provider and two
   tables do not justify it, and it would hide the SQL the constraints
   depend on. Rejected.
6. Attachment bytes as `bytea` beside the events. Large binary rows in
   the relational store, for bytes that are content-addressed and belong
   in blob storage. Rejected.
7. The original JSON lines in PostgreSQL behind `(projectId, runId)`,
   complete runs only, projections derived (chosen).

## Consequences

- The store, `ts/packages/postgres`, has two tables (runs, source lines)
  plus the migration record; `raw_line` is authoritative and the other
  columns are copies for indexing.
- A future decoder can replay every archived run and understand more of
  it than today's does; the stored validator summary is an audit record
  compared on replay, never a source of outcomes.
- Runs gain an out-of-band `ingestion_sequence`, a durable ordering
  primitive for later queries; the read model's producer-clock history
  order is unchanged.
- Integration tests run against PostgreSQL 16 in Testcontainers.

## Deferred

Durable attachment bytes, incremental ingestion of incomplete runs,
retention and deletion, relational or materialised projection indexes,
and any transport that fills the store.
