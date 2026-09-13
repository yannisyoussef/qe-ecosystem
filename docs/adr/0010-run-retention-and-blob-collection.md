# ADR-0010: Runs carry an explicit expiry, and globally deduplicated blobs are collected only when no run references them

Status: Accepted
Date: 2026-09-13

## Context

The durable platform now holds two resources: the protocol source of
complete validated runs in PostgreSQL (ADR-0008), and their attachment
bytes as immutable content-addressed blobs whose identity is global
across projects (ADR-0009). Both grow and neither shrinks. ADR-0009 also
left two states deliberately: an ingestion that publishes bytes and then
rolls back leaves an unreferenced object, and a writer killed mid-copy
leaves an unpublished temporary file. The architecture has said since the
beginning that every run carries an expiry and that deletion cascades to
blobs, but the schema had nowhere to record one, and global
deduplication means a naive cascade would delete bytes another project
still depends on.

Nothing about deletion can be inferred from the protocol: a run's events
say when it happened, not how long anyone wants to keep it. Nor can it be
inferred from ingestion time without inventing a policy this milestone
has no basis for.

## Decision

Retention is ingestion context, exactly like the project (ADR-0007). A
run archived by the current writer carries an `expiresAt` the caller
supplies, recorded in `qe_run_retention` in the same transaction as the
run itself; an absent or unusable instant archives nothing, and an
instant already past is valid and makes the run eligible at once. The
first expiry recorded stands: ordinary re-ingestion does not move it, and
a content conflict does not touch it. Changing one is a policy operation
that does not exist yet.

A run archived before this milestone has no retention row. It is
retention-unmanaged, not expired: every maintenance pass reports it and
none deletes it, and no expiry is invented for it by the migration.
Re-ingesting such a run from a still available source records the missing
fact without altering the archive, the way ADR-0009 completes missing
blob relations.

Deleting a run deletes its source lines, its run-to-blob relations, and
its retention fact by cascade. It does not cascade into the blob catalog.
A blob is deleted only when no run-to-blob row anywhere in the database
references its hash, counted globally and never per project. The catalog
row goes first, deleted under a final guard in the statement itself:

```sql
DELETE FROM qe_blobs
 WHERE sha256 = $1
   AND NOT EXISTS (SELECT 1 FROM qe_run_blobs WHERE sha256 = qe_blobs.sha256)
```

and the verified object is removed only when that statement deleted a
row. Objects the catalog never knew are found by enumerating the blob
store itself, because no database query can see them. A corrupt object,
an unsafe entry, or one whose size disagrees with the catalog is reported
and left for an operator; when such an object was catalogued, its row is
already gone, so the catalog stops claiming durable content it cannot
produce and the object waits as an uncatalogued orphan.

The order is chosen for what each failure leaves behind. The guard runs
while destructive maintenance holds the exclusive lease, so no ingestion
can create a reference between the check and the deletion:

| What fails | Catalog row | Object | Meaning |
|---|---|---|---|
| The row deletion | remains | remains | Nothing happened; the next pass retries. |
| Nothing | gone | gone | The blob is fully reclaimed. |
| The object removal | gone | remains | An uncatalogued orphan the enumeration collects later. |

The third state is the price of the ordering and it is a safe one: the
row was deleted only because no run referenced the hash, so no reader
loses anything, and the object is exactly the kind of stray the orphan
pass exists for. Removing the object first would make the mirror image
possible instead, which is not safe: a guard that then declined to delete
the row would leave a catalogued hash, reachable by a run, whose bytes
maintenance had already unlinked. Maintenance must never be able to
produce

```
committed run-to-blob relation -> catalog says the blob exists -> bytes already deleted
```

so the ordering that can only fail the other way is the one that stands.

An object the catalog never knew is collected only when it is older than
a cutoff the caller states, as a temporary file is: a blob published a
moment ago by a writer that did not take the maintenance lock is
indistinguishable from one an ingestion abandoned, and the caller is the
only party that knows how old certainly abandoned is. Without a cutoff
the store is not enumerated for orphans at all.

This is the ADR that realises the supersession ADR-0009 recorded: item 3
of ADR-0003 scoped deduplication, and therefore deletion, to a project,
and its consequence deleted a blob once no unexpired run in the same
project referenced it. Both are replaced here by a global, per-hash
reference count.

Ingestion and destructive retention are serialized by one session-level
advisory lock: shared for a mutating ingestion, from before it may
publish bytes until its transaction ends, and exclusive for a destructive
pass. Ingestions still run beside each other; reads take no lock. Run
deletion and blob collection are separate failure domains, and a report
states partial success rather than hiding it. Every operation is bounded,
deterministic, explicitly invoked, and available as a dry run that
mutates nothing. No scheduler exists in the persistence packages.

## Options considered

1. Reference-count blobs per project. Would delete bytes another project
   still references, or force per-project copies and give up the
   deduplication ADR-0009 chose. Rejected.
2. Cascade from the run straight into the blob catalog. Same fault in
   database form: a foreign key cannot express "no other run anywhere".
   Rejected.
3. Delete every file the catalog does not list. Would destroy a blob a
   concurrent ingestion had just published and was about to reference,
   and would treat a foreign or corrupt entry as garbage. Rejected in
   favour of global reference counting under the exclusive lock, with
   verification before every unlink.
4. Derive expiry from `ingested_at` plus a default period. Invents a
   policy nothing asked for and hides it in the storage layer, where a
   later real policy could not correct it. Rejected.
5. Collect any object the catalog does not list, with no age cutoff. A
   blob root shared by two databases, or a database restored from a
   backup, would then destroy live bytes on the first pass. Rejected in
   favour of a caller-stated cutoff, with one blob root per archive
   documented as an operating rule.
6. Let re-ingestion move an established expiry. Makes deletion
   eligibility depend on who last replayed a directory, which is not a
   property of the run. Rejected; an explicit policy operation can
   address it when one exists.
7. One unbounded cleanup pass over the whole installation. Cannot be
   invoked predictably by a future server and cannot be reasoned about
   under load. Rejected in favour of bounded batches with limits and a
   truncation flag.
8. A scheduler inside the persistence package. Puts timing decisions
   where lifecycle semantics belong and makes them untestable before
   deployment exists. Rejected.
9. Collect blobs without coordinating with ingestion. The race is real:
   between deciding a hash is unreferenced and unlinking it, an
   ingestion can publish and reference it. Rejected; the shared and
   exclusive advisory lock is the coordination boundary.

## Consequences

- `persistRunDirectory` requires `expiresAt`. There is no default
  anywhere in the platform, and a caller that does not know one cannot
  archive a run.
- Migration 3 adds the retention table and turns the run's child foreign
  keys into cascades. Migrations 1 and 2 are untouched, so a database
  archived under them keeps its rows and gains identifiable legacy
  records rather than a fabricated expiry.
- Retention is manual until an operations layer calls it. Until then
  expired runs simply remain, which is visible in a preview.
- A catalog row may be deleted while its object survives a failure. The
  object is then an uncatalogued orphan and the enumeration pass collects
  it; nothing a reader can reach is affected, because the row was deleted
  only after the guard proved no run referenced the hash. The opposite
  order, which could leave a referenced catalog row without its bytes, is
  never used.
- The read model and the protocol remain unaware of retention, and
  history and flakiness need no cleaning because they were never stored.
- `asOf` is the caller's authority and is not compared with any clock: a
  pass asked about an instant in the future deletes runs that have not
  reached their expiry. Authorisation for maintenance belongs to the
  operations layer that will invoke it.
- One blob root belongs to one archive. Two databases sharing a root
  would each see the other's objects as uncatalogued, which the age
  cutoff mitigates but does not make safe.

## Deferred

Holds, retention extension, and any policy model that changes deletion
eligibility; per-project or per-tenant retention policy; scheduling;
compaction or re-verification sweeps of the whole store; and S3-compatible
blob maintenance, which a second provider would define for itself.
