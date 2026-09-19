# ADR-0007: The project is ingestion context, and the run key is (project, run id)

Status: Accepted
Date: 2026-09-12
Amended: 2026-09-19 (the transport and the project id contract; see Amendment)

## Context

The first read model (QE-004) consumes protocol line 0.3 output from two
runner families and must keep runs of different products, repositories,
or teams apart. The protocol identifies a run by `runId` alone, and run
ids are chosen by adapters or by their users: a generated UUID, or a
configured `build-123` that two unrelated builds may both use. Histories
are keyed by `(runner.name, historicalId)`, which two unrelated products
can also share when they use the same runner and the same test names. A
platform therefore needs a partition above the run, and it has to come
from somewhere.

## Decision

The partition is a `projectId` supplied by whoever ingests a run
directory. It is ingestion context, not a protocol field: no event carries
it, no adapter knows it, and it is never derived from labels,
environment, runner metadata, source repository, or the run directory's
name. It is an opaque, non-empty string chosen by the caller of the read
model and, later, by the transport that receives runs. The canonical run
key is `(projectId, runId)`; the same run id in two projects is two runs.
The history key is `(projectId, runner.name, historicalId)`, the
protocol's collision domain partitioned by project; the producer's name
stays out of it so that replacing an adapter keeps a test's history. Run
directory names remain locators: a run id found in two directories of
one project is a conflict that ingests neither, never a merge.

## Options considered

1. A `projectId` field on `session.started`. Adapters would have to be
   configured with a platform concept they cannot verify, every run
   would carry a claim the platform must not trust, and moving a run
   between projects would mean rewriting events. Rejected.
2. Deriving the project from `source.repository` or a label. Both are
   optional, producer-controlled, and legitimately shared by several
   products in one repository. Rejected.
3. Deriving it from the output root or the run directory name. Names are
   locators chosen by a filesystem layout, not identities. Rejected.
4. No partition: `runId` alone. Two unrelated builds with the same
   configured id would merge, and histories of unrelated products would
   mix. Rejected.
5. Caller-supplied ingestion context (chosen).

## Consequences

- Persistence (QE-005) keys runs on `(projectId, runId)` with a uniqueness
  constraint that reproduces the read model's duplicate-run conflict, and
  keys history on `(projectId, runner.name, historicalId)`.
- A future transport carries the project out of band (the credential or
  endpoint that accepts the upload), never inside the events, which is
  also where authentication and authorization will attach.
- Adapters and the protocol are unchanged; a run directory can be
  ingested into any project, or into several.

## Deferred

Whether a project may be renamed or merged, and what a project is to a
user (a repository, a product, a team), are product decisions for the
platform, not for the read model.

## Amendment (2026-09-19): transport and the project id contract

The transport this ADR deferred now exists (ADR-0012), and it settles
where the project comes from and what a project id may be. Nothing above
changes: the project is still ingestion context, and still never a
protocol field.

**The project comes from the authenticated principal.** An HTTP request
acts on the project its API key was issued for, and on nothing else. No
path, query parameter, header other than the credential, multipart part,
request body, or protocol event names a project; a `projectId` member in
an event is ordinary forward-compatible protocol data with no authority.
The project is never derived from `source.repository`, labels, the run
id, session metadata, an upload's filename, or anything else the request
carries. The protocol has no project field and gains none.

**One project id contract, system-wide.** A project id is opaque,
well-formed Unicode, non-empty, without U+0000, and at most 512 bytes of
UTF-8. It is neither trimmed nor normalised: canonically equivalent
spellings are different projects unless their code points are
identical. The limit is counted in bytes, not characters. It is the same
limit in the read model, the archive, the query layer, the credential
store, and the transport, enforced by one shared check and, durably, by
a constraint on the canonical archive table that every other table's
key refers to.

The bound was set before remote callers could create project ids through
credentials, and for a physical reason. Every durable table is keyed by
the project id inside compound B-tree keys, and PostgreSQL bounds an
index entry at about 2.7 kB. "Opaque non-empty string" was therefore
never unlimited in practice. An explicit 512-byte bound keeps every such
key far below the ceiling with room for the rest of the key, and makes
the limit a stated contract instead of an accident of whichever index
happened to overflow first. A limit applied only by the query layer was
considered and rejected: the store could then archive a project the
queries refuse to read.

An existing archive holding a longer project id is not rewritten,
truncated, or hashed. The migration that adds the bound refuses and
reports it, and an operator reconciles it.

