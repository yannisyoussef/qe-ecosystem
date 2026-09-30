# ADR-0014: qe-report v1 is deployed as a single instance with explicit operations

Status: Accepted
Date: 2026-09-29

## Context

The application path is complete: an adapter writes a run directory, a
producer uploads it over HTTPS (ADR-0013), an authenticated API v1 accepts
it (ADR-0012), the validator decides whether it is a run, PostgreSQL keeps
its protocol source (ADR-0008), attachment bytes live in a
content-addressed store (ADR-0009), retention deletes what has expired
(ADR-0010), and rebuildable indexes answer questions across runs
(ADR-0011).

None of that says how the thing runs, who may restart it, what happens to
a request when it stops, or whether anyone can get the data back after a
disk is lost. A system that cannot be deployed and recovered is not
finished, and a recovery procedure nobody has executed is a description
rather than a capability.

The question is also what *not* to build. Kubernetes, replication,
failover, object storage, and a human identity model are each a milestone
of their own, and none of them is needed to run this service for a team.

## Decision

The reference deployment is one API instance, one PostgreSQL 16, one POSIX
attachment store, and one reverse proxy, described by a Compose stack and
a Dockerfile in the repository. It is the arrangement this project tests;
it is not a claim about scale.

It is not highly available and does not claim to be. One instance, one
database, no replica, no failover: a restart is a short outage, and a
backup is a planned one. Everything below is written for that deployment,
and none of it should be read as a statement about a larger one.

TLS terminates at a standard reverse proxy, which also owns the public
port, connection and request limits, the largest body it will pass, and
its timeouts. The application speaks HTTP on the private deployment
network and terminates no TLS itself. Only the proxy publishes a port:
PostgreSQL and the application are unreachable from outside, on any host
that does not route the container subnet, which is a property of the host
and is stated as such rather than claimed as a guarantee. The deployment
network is split so that the proxy can reach the application and not the
database, and the half carrying SQL has no route off the host at all. The
application runs as a dedicated non-root user with a read-only root
filesystem, no added capabilities, and writable space only for its data
roots and a temporary directory that cannot hold an executable.

The proxy's limits are not authentication and are never mistaken for it;
they count the client address the proxy can see, which behind NAT is one
address for many clients. The proxy resolves the application's name per
request rather than once at start-up, because an upgrade replaces the
application's container and an edge holding the old address would answer
502 until somebody reloaded it.

PostgreSQL and the attachment store are the durable state. Staging is
ephemeral material for one request, and exists after a crash only so that
it can be inspected and removed; it is never backed up.

Every operation that changes or destroys state is something a person runs.
Migrations are explicit, and the application refuses to serve a schema it
does not recognise rather than migrating itself. Retention and blob
collection, query-index rebuilds, and staging cleanup are operator
commands, and staging cleanup additionally requires the instance that owns
the root to be stopped, because nothing can distinguish an abandoned
request from one still being written. Nothing in the deployment schedules
any of it.

Backups are quiesced: the API stops, the database is dumped with standard
PostgreSQL tooling, the attachment store is copied as opaque files in its
own layout, checksums and a manifest are written, and the API starts
again. Ingestion publishes attachment bytes before committing the rows
that reference them, so with no writer running the two halves describe the
same moment.

Stopping the API does not stop an operator, so a backup also holds the
exclusive maintenance lock of ADR-0010, in one session, across the dump and
the copy, and proves afterwards that the same session held it throughout. A
retention pass between the two halves could otherwise delete bytes the dump
still references, and the result would pass every check and restore an
archive pointing at objects that are not there. A quiesce that is only a
procedure is not one.

A backup directory is a trust boundary and is documented as one: its
checksums sit beside the files they describe, which detects bit rot and
truncation rather than an adversary, and `pg_restore` executes whatever the
dump contains. The manifest is inside the same envelope, and the digests it
records are cross-checked against the verified ones on the way in.

Restore is destructive, requires saying so, and replaces both halves; it is
rehearsed in CI, where the persistent state is genuinely destroyed before
the backup is restored and the same key, runs, history, flakiness,
attachment bytes, retention facts, and index agreement are proven again. It
also restores the credential table as it was, so a key revoked since the
backup authenticates again, and re-reviewing keys is part of the procedure
rather than a footnote.

API keys remain machine credentials, issued and revoked by the operator
command, listed as metadata that cannot include a secret. This milestone
adds no human identity, no session, and no browser surface.

## Options considered

1. Migrate automatically when the application starts. A restart would
   then be a schema change, and a rollback would be a data-loss event.
   Rejected: migration is an operator step, and the application refuses a
   schema it does not know.
2. Publish PostgreSQL's port for convenience. The database holds every
   run and every key hash; the convenience is not worth the exposure.
   Rejected.
3. Terminate TLS in the application. Certificate lifecycle, ciphers, and
   limits would become application concerns, and the one thing a bearer
   credential needs would depend on a release. Rejected in favour of a
   standard proxy.
4. Accept API keys over plaintext HTTP in production. A bearer token on
   the wire is the credential. Rejected; plaintext is refused except to
   this machine, for development.
5. Run the application as root, or with a writable root filesystem.
   Neither is needed: it writes to two data roots and a temporary
   directory. Rejected.
6. Collect garbage, rebuild indexes, or clean staging on a timer inside
   the application. Destructive work on a schedule nobody asked for, with
   no way to see what it would do first. Rejected: every one is an
   explicit, bounded, previewable command.
7. Delete stale staging when the process starts. The state that looks
   abandoned may belong to a live request elsewhere, and start-up is
   exactly when an operator is not watching. Rejected.
8. Back up staging along with the rest. It is request material, not
   archive. Rejected.
9. Back up only the database, or only the attachment store. Either alone
   restores an archive that references bytes it does not have, or bytes
   nothing references. Rejected: a backup is both, taken together.
10. Invent a database export format. Standard `pg_dump` and `pg_restore`
    are understood, supported, and verifiable. Rejected.
11. Claim zero-downtime backups. Nothing here has shown a consistent
    snapshot while writers run. Rejected: the reference is quiesced and
    says so.
12. Run several API replicas. Shared staging and store semantics across
    instances are unproven, and the cleanup rules depend on knowing who
    owns a root. Rejected until that is established.
13. Kubernetes, Helm, or Terraform before the single node is proven. They
    describe a deployment; they do not make one correct. Rejected.
14. Add OIDC or a user model as part of deployment. Authorisation for
    people is a design question, not an operational one. Rejected.
15. Introduce metrics and tracing because this milestone is operational.
    Structured logs, health, readiness, and container state answer the
    questions a single instance raises. Deferred until a real deployment
    shows otherwise.

## Consequences

- The deployment lives in `deploy/reference` and is built from the
  repository. Base images are pinned by digest with their human-readable
  version recorded beside them, and nothing is pushed to a registry.
- The database credential may be a file, which is how a container holds a
  secret. Exactly one form may be set, and a failure to read it says so
  without printing the value or naming the file.
- Shutdown is a bounded state machine: the first signal drains and exits
  cleanly, a drain that outlasts its grace exits non-zero, and a second
  signal stops at once. Nothing new is served once the drain begins. The
  container's stop grace is deliberately longer than the application's own,
  and the application's grace is not long enough for every request it will
  accept: an upload still running when it expires is abandoned, and is
  described that way rather than as allowed to finish.
- The one previously known gap, a request directory left behind by a
  killed process, is now closed operationally rather than tolerated.
- A restore that fails at any step does not start the application, and a
  backup that fails at any step starts it again, because a failed backup
  must not also be an outage.
- Every destructive operator command states its own bound, and a bound that
  means its own opposite is refused: a cutoff of zero is not a cutoff.
- No schema migration was needed for any of this.

## Deferred

Registry publication and release artefacts are now decided in ADR-0015, which
this paragraph deferred them to.

Still deferred: metrics and tracing; a
scheduler for the maintenance commands; object storage for attachment
bytes; PostgreSQL replication, failover, and horizontal application
scaling; automatic certificate issuance; and any model of human identity.
