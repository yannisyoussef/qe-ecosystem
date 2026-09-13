# ADR-0009: Attachment bytes are durable content-addressed blobs on a local filesystem

Status: Accepted
Date: 2026-09-13

## Context

The run archive (ADR-0008) makes the protocol source of a complete
validated run durable in PostgreSQL, so a run directory can be replayed
and projected from the database alone. Its attachment events are part of
that source, with the hash, size, name, and media type the producer
declared, but the bytes they name were left where the run directory kept
them: after ingestion the directory was still needed for anything that
opened an attachment, and replay could not re-verify bytes at all. The
architecture already fixes the first shape of artifact storage: a local
filesystem behind one storage boundary, under server-generated keys,
with the SHA-256 kept as identity and metadata, and an S3-compatible
implementation only when a non-local deployment exists. The validator
already has a posture for reading untrusted files (no link following,
regular files only, descriptor re-checked, streamed hashing), and the
read model already treats the full SHA-256 as the identity of stored
bytes across every run of a snapshot.

## Decision

Attachment bytes become immutable content-addressed blobs. The blob
identity is the protocol's full lower-case SHA-256 and nothing else:
names, media types, run ids, project ids, session ids, and source paths
never reach the filesystem, and the same bytes referenced by two runs or
two projects are one object. The first and only provider is a local
filesystem store (`ts/packages/blob-fs`) under an operator-configured
root, keyed `sha256/ab/cd/<sha256>`, with the key derived from the
validated hash alone. A blob is materialised by re-reading the source
securely, copying it into an exclusively created temporary file under
the root, hashing and counting during the copy, comparing both with the
declaration, and publishing with an exclusive hard link, so that nothing
unverified is ever visible at a final path and an existing object is
never replaced. An existing object is re-verified in full before it is
reused; a wrong one is an error that no ingestion repairs. Concurrent
writers are settled by the filesystem's exclusive link, not by a process
lock.

PostgreSQL stores blob metadata and relations, never bytes: a global
catalog `qe_blobs` (hash, size, provider key, catalog time) and a
rebuildable storage-integrity relation `qe_run_blobs` naming the
distinct blobs each archived run requires. The protocol source stays
the semantic truth: multiplicity, names, media types, attempts, and
steps are read from the raw lines only, and the read model's blob
catalog stays derived and storage-independent. A hash has one size; a
catalog row or a declaration that disagrees is a consistency error.

Blobs are materialised before the run transaction, and the transaction
records the run only after every required blob is durable. The two
resources cannot share one transaction, so the invariant is
one-directional: a committed run never references a blob that was not
materialised, while a rollback after publication may leave an
unreferenced blob, which is immutable, safe, and not deleted in the
failure path. The run row's ingestion-time claim about source
attachments is renamed `source_attachments_verified` by migration 2 and
no longer stands for durable bytes; no caller-supplied boolean creates a
blob record, and durable integrity is established only by re-reading
the store (`verifyStoredRunBlobs`). Migration 1 is not edited. Runs
archived before migration 2 are recognisable as archives with attachment
events and no relations, and re-ingesting one from a still available
directory completes its bytes and relations automatically without
touching its source. Reading a blob takes the run as part of the
address (`openBlob(projectId, runId, sha256)`), so that a caller learns
whether a hash exists only through a run it may see.

This supersedes item 3 of ADR-0003 where it scoped deduplication to a
project, and the retention consequence there that a blob is deleted once
no unexpired run in the same project references it: deduplication is
global, and retention must count references across projects.

## Options considered

1. Bytes in PostgreSQL as `bytea` beside the run. Large binary rows in
   the relational store for content that is addressed by hash and
   deduplicated globally; already rejected in ADR-0008. Rejected.
2. Bytes inside the source-line rows. Mixes opaque binary content into
   the semantic archive and breaks the fingerprint's meaning. Rejected.
3. Paths derived from attachment names or run ids. Producer strings as
   filesystem components, no deduplication, and an injection surface
   the protocol was designed to avoid. Rejected.
4. Overwriting an existing object when a writer has "better" bytes.
   Would let one ingestion alter content another run already references;
   immutability under the hash is the whole guarantee. Rejected.
5. S3-compatible storage (MinIO in tests) now. The architecture defers
   it until a non-local deployment exists; a local store behind the
   same boundary is enough and adds no service to CI. Rejected for this
   milestone.
6. Deleting a published blob when the run transaction fails. A
   concurrent ingestion may reference the same hash; the orphan is
   harmless and belongs to retention. Rejected.
7. Immutable content-addressed blobs on a local filesystem, metadata
   and relations in PostgreSQL, materialised before the run transaction
   (chosen).

## Consequences

- A run directory can be deleted after successful ingestion; replay,
  projection, the blob catalog, and full byte verification work from
  PostgreSQL and the blob root.
- Deduplication is global across projects. Whether a blob exists is
  therefore not a per-project fact; the store's own read takes a run as
  part of the address, a future transport must keep that, and retention
  must reference-count blobs across projects before it deletes any. Item
  3 of the security posture in the architecture notes is updated to say
  so.
- Unreferenced blobs may accumulate from rolled-back transactions and
  from conflicting re-ingestions until a retention milestone collects
  them.
- The blob root is an application storage boundary: its contents are
  re-verified when read, and mutation by a hostile operator is out of
  scope.
- The blob-fs and validator packages each carry the same small safe
  opening primitive rather than sharing an exported filesystem API; its
  behaviour is pinned by tests in both.

## Deferred

S3-compatible storage, garbage collection and retention of blobs and of
temporary files left by a killed process, reference counting across
projects, a provider or root identifier beside the storage key once a
second root or provider exists, any transport that fills the store, and
incremental ingestion.
