# ADR-0012: An authenticated HTTP transport exposes the archive and its queries, and the credential decides the project

Status: Accepted
Date: 2026-09-19

## Context

Everything up to ADR-0011 is reachable only in-process. A producer in CI
writes a run directory, and something with a database connection has to
archive it. Every durable read (one run replayed, runs listed, a test's
history, its flakiness, an attachment's bytes) is a library call that
takes a project id as an argument. The first network boundary has to add
three things: authentication, the choice of project, and a wire form for
uploads and answers. It must add them without becoming a second
definition of anything: not of the protocol, the verdict, the history
order, flakiness, idempotency, or retention.

The immediate callers are machines: CI jobs and test automation. There is
no user model, no organisation, and no product definition of a project
beyond ADR-0007's partition key.

## Decision

The first transport is an HTTP API, version 1, implemented with Fastify.
Its version is its own. It is independent of the protocol line (0.3),
the database schema version, the query-index version, and the package
version, and none of them moves when it does.

**Authentication is by project-scoped API key.** A key belongs to exactly
one project and carries `runs:write`, `runs:read`, or both; there is no
other scope, no wildcard, and no administrative scope over HTTP. A token
is `qer_k1_<publicId>_<secret>`, with 80 random bits of public id to find
the key and 256 random bits of secret to prove it. Only the SHA-256 of
the secret is stored, because a secret of that entropy needs no slow
password hash; comparison is constant-time. The token is presented as
`Authorization: Bearer` and nowhere else. Unknown, malformed, wrong,
expired, and revoked keys are one indistinguishable `401`; a valid key
without the scope is `403`. A key's project and scopes never change, and
rotation is a new key plus revocation of the old. Keys are issued and
revoked by an operator command against the database, never over HTTP.
No users, organisations, sessions, cookies, OAuth, OIDC, or JWT exist
yet. A later identity layer can map its principals onto the same
project-scoped boundary without touching the protocol.

**The credential decides the project** (ADR-0007, amended): no request
names one, and every store and query call uses the key's.

**Remote ingestion is streamed multipart, not archive extraction.** An
upload carries one `expiresAt` text part, one or more `events` file parts
(each one session file of a run directory), and any number of
`attachment` file parts. The server streams them into a fresh temporary
run directory it names itself: event streams byte for byte in arrival
order, and attachment bytes under the SHA-256 computed as they arrive,
whatever name or type the part claims. It then hands that directory to
the existing store, which validates and archives it exactly as it would
any run directory: complete runs only, idempotent for the same content,
a conflict for different content under one identity, and repair of a
derived index from the archive's own source. The temporary directory is
removed however the request ends. It is never durable or visible
provenance: the store records a logical locator (`http:<request id>`)
and run-relative source file names, and no response, diagnostic, or log
line names a staging path. Every upload is bounded (the body, the number
and size of parts, each attachment within the blob store's own limit)
and streamed without being held in memory; a limit is `413`, and a
refused upload writes nothing.

**Reads keep their established sources.** One run is replayed from its
canonical source and returned as an explicit response shape. That shape
carries no locator, fingerprint, archived line, database identifier, or
storage key, and it never serializes the internal projected run as it
stands. Listings, histories, and flakiness come from the ADR-0011
indexes through opaque keyset cursors. Each cursor is versioned and
bound by a digest to the project and, for a history, to its runner and
historical id, so it cannot be replayed elsewhere. A project whose index
is incomplete refuses cross-run answers with an explicit `503`, and no
request rebuilds anything. A run is addressed by a `runRef`, the
unpadded base64url of its run id, because protocol identifiers may hold
characters a path segment cannot. It is a locator only; the run id is in
every response.

**Attachments are downloaded only through a run-scoped relation.** The
bytes are served only when a run of the key's project references the
hash. They go out as `application/octet-stream` with `nosniff`, whatever
media type the producer declared. A run or hash outside the project is
indistinguishable from one that does not exist.

**Operator actions stay operator actions.** Retention, collection,
migration, index rebuild and verification, and key management have no
route. The server checks that the schema is current and refuses to start
otherwise; it never migrates on start.

**TLS is a deployment boundary.** The server speaks plain HTTP, binds to
loopback unless configured otherwise, trusts no forwarded header, and
expects HTTPS to terminate at the edge in front of it. Connection and
rate limiting belong there as well.

The OpenAPI 3.1 document of API v1 is generated from the route table the
server registers, committed, and checked in CI against generation.

## Options considered

1. A client-supplied project id, in the path, a header, or the body. The
   caller would name the partition it acts on, and authorization would
   become a check that the key may use the project it named. That is one
   mistake away from none at all. Rejected: the credential is the
   project.
2. A project id inside protocol events. Rejected by ADR-0007 for the same
   reasons, and more so over a network: a producer would assert its own
   authority.
3. Unauthenticated ingestion, relying on the network. Rejected; the
   archive is durable, and anything that can write it can fill it.
4. JWT or OIDC now. There is no user or organisation model for claims to
   describe, and machine callers need a revocable secret, not a login.
   Deferred until such a model exists, and then mapped onto this
   boundary.
5. API keys stored reversibly, or as the raw token. A database read would
   then yield working credentials. Rejected for a one-way digest.
6. API keys in the URL or query string. They would end up in proxy logs,
   browser history, and referrers. Rejected; only the Authorization
   header.
7. ZIP or TAR uploads. Path traversal, symbolic links, decompression
   bombs, duplicate entries, permissions, and another parser to harden,
   where the run-directory boundary already exists. Rejected for streamed
   parts that are materialised into exactly that shape.
8. Ingesting a run by naming a path on the server. The API would become a
   file-read primitive. Rejected; every byte arrives in the request.
9. Public maintenance endpoints for retention, collection, rebuild, or
   keys. Destructive and administrative actions would sit behind the
   same credentials as ordinary producers. Rejected.
10. Downloading a blob by hash alone. Anyone who learns a hash, which is
    global across projects (ADR-0009), could read bytes of a project they
    have no key for, or learn that the bytes exist. Rejected; only
    through a referencing run of the key's project.
11. Serializing the internal projected run directly. The wire contract
    would then change with every internal refactor and could leak
    locators. Rejected for explicit response mappers.
12. Offset pagination. It skips and repeats rows when anything is
    archived or deleted between pages. Rejected, as in ADR-0011.
13. Rebuilding an incomplete query index inside the request that found
    it. A read would turn into an unbounded write under the caller's key,
    with the completeness gate as its trigger. Rejected; incompleteness
    stays visible, and rebuild stays an operator action.

## Consequences

- `ts/packages/http-api` holds the transport, a standalone server, and
  the operator command (`migrate`, `schema`, `key create`, `key revoke`).
  PostgreSQL gains the key table and the project id bound in migration 5.
- The archive's semantics are exactly those of a direct call: an HTTP
  upload and a filesystem ingestion of the same run are the same run.
- The source locator of an HTTP-ingested run is logical, and its source
  files are named `events/000001.ndjson` in arrival order. The original
  file names a producer used are not kept.
- Because attachment bytes are staged under the hash they have, bytes
  that do not match their declaration surface as a missing attachment,
  not a hash mismatch.
- Each `events` part must hold one session, as a run-directory file
  must. A stream mixing sessions is refused by the validator's existing
  rule rather than split by the transport.
- A client can page listings and histories without knowing anything of
  their keyset, and cannot page another project with a cursor it holds.
- A server killed outright can leave a staging directory behind. Cleanup
  of a stale staging root is an operator task until evidence asks for
  more.

## Deferred

Users, organisations, and OIDC; any role beyond the two scopes; rate
limiting and quotas, which belong to the deployment edge; Range requests
for attachments; search; stale staging cleanup; and any transport other
than HTTP.
