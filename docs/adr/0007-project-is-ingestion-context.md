# ADR-0007: The project is ingestion context, and the run key is (project, run id)

Status: Accepted
Date: 2026-09-12

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
