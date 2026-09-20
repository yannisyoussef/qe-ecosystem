# ADR-0013: Producers upload completed run directories; HTTP is not an event sink

Status: Accepted
Date: 2026-09-20

## Context

ADR-0012 gave qe-report an authenticated HTTP boundary that takes a whole
run as a multipart upload. Nothing on the producer side used it: a runner
wrote a run directory with the SDK's file sink, and something with a
database connection had to archive it. The obvious-looking next step is
an HTTP sink beside the file sink, sending each event as it happens.

It does not work, for a reason that is about the domain rather than the
transport. The service archives a complete, immutable run, and idempotency
is defined for a whole run. A process writing events cannot say when the
run is over. JUnit Platform runs one test plan per Gradle or Surefire
fork, so a fork is a session and never knows whether another fork will
open one. A Playwright shard is a separate process against a shared run
id. Anything that streamed events to the service would have to decide,
event by event, that nothing more is coming, and would archive a run that
is still growing.

Producers also need delivery to survive an unreliable network without
duplicating runs, and a CI job that loses the answer to a request must be
able to ask again.

## Decision

The producer keeps writing locally first. `FileSink` stays the producer's
write boundary, and the completed run directory is the producer-side
durable spool: it is what a delivery reads, what a retry re-reads, and
what remains when a delivery fails. HTTP delivery is a separate, explicit
step over a finished directory.

The first client is TypeScript, `qe-report-http-client`, with a library
and a `qe-report-upload` command. It uploads a run directory whatever
produced it: the Playwright reporter, the JUnit Platform adapter, either
SDK, or an adapter that does not exist yet. There is no runner-specific
upload protocol and no Java HTTP client; a Java build uploads its
directory with the same command, which is what proves the format, not the
language, is the boundary.

The project still comes from the credential (ADR-0007, ADR-0012). The
client takes an API key and no project.

One upload plan is built before the first attempt: which files, in which
order, how large each one is, and the exact expiry to send. Every attempt
re-opens exactly those files and checks that each is still the same file,
by type, size, modification time, device, and inode, before and after its
bytes are streamed. A run directory that changes ends the upload rather
than mixing two versions of a run into one archive. Attachments are
uploaded only under the hash their name claims, verified against their
bytes: an uploader may not repair a producer's directory into a different,
valid remote run. Nothing is followed through a symbolic link, and nothing
is buffered whole.

Retries exist because `POST /v1/runs` is idempotent for the same run. A
producer that loses the answer to an attempt the service already archived
retries and is told `already_present`; the archive holds one run. Only
deliveries are retried, with a bounded number of attempts, bounded
backoff, and a bounded honouring of `Retry-After`. A refusal is never
retried: a conflict, an invalid or incomplete run, a credential, or a
limit means the same thing however often it is asked. Redirects are not
followed, because the request carries a bearer token.

Automatic upload by a runner is allowed only where one process can prove
it owns the whole run. The Playwright reporter generates a run id when
none is configured, and then it is the only session of that run and the
one that writes `run.finished`: in that case, and only after the sink is
closed, it uploads. With a configured or shared run id, sharded or not, it
uploads nothing and says that the coordinator should, because another
shard may still be writing. The JUnit adapter never uploads at all. An
upload failure is a reporting problem and never changes a test outcome;
the directory stays on disk for a later, explicit upload.

There is no queue, daemon, watcher, or background delivery, and nothing
deletes the producer's output. The service's API v1 is unchanged.

## Options considered

1. An HTTP `ReportSink` sending each event. It would have to decide when a
   run is finished from inside one session, which no fork or shard can
   know, and would turn every network failure into lost events rather
   than a retryable delivery. Rejected: the local directory is the spool.
2. One upload per session worker. Each fork or shard would archive its own
   part, and the first complete-looking snapshot would make the archive
   refuse whatever arrived next. Rejected.
3. Automatic upload from the JUnit listener. Same reason, in a runner
   whose forks routinely share a configured run id. Rejected: the build
   coordinator uploads after the build.
4. Every Playwright shard uploading the shared run. Rejected for the same
   reason, and proven against rather than documented against.
5. Deleting the local output after a successful upload. It is still the
   evidence, the diagnostic material, and the source of a later upload.
   Rejected.
6. Retrying from the directory as it is at each attempt. A run that
   changed between attempts would be delivered as a mixture of two runs.
   Rejected for a fixed plan and per-attempt verification.
7. A client-supplied project id. Rejected by ADR-0012: the credential
   decides the project.
8. An API key as a command-line argument. It would sit in shell history
   and in the process list of every user on the machine. Rejected: the
   key comes from the environment.
9. Following redirects. A redirect can point at another origin, and the
   request carries a bearer token. Rejected: a 3xx is a terminal answer.
10. A Java HTTP client for symmetry. There is no evidence yet that a Java
    application needs in-process delivery rather than coordinator
    delivery, and the generic uploader already proves the format is
    language-neutral. Deferred until such evidence exists.
11. Parsing the event stream in the client, to check it or to leave out
    attachments nothing references. The service owns protocol semantics,
    and a second opinion in the producer would drift from it. Rejected:
    the client reads the filesystem, not the protocol.

## Consequences

- `ts/packages/http-client` holds the client and the command;
  `qe-report-playwright` gains an optional, off-by-default upload that
  uses it rather than repeating it.
- A CI pipeline that wants an upload failure to fail the build runs the
  command as its own step; the reporter never fails tests for it.
- A run whose delivery was interrupted can be uploaded later from
  anywhere that can read the directory, including another machine.
- Because the uploader checks that attachments hash to their names, a
  producer directory that is corrupt is refused locally instead of
  becoming a different remote run.
- The archive's semantics are untouched: an upload and a direct
  filesystem ingestion of the same run are the same run, and canonical
  source repair (ADR-0011) still derives from what was archived first.

## Deferred

A native Java client, for in-process delivery when there is evidence for
it; an offline queue or daemon, if producers turn out to need delivery
without a coordinator; a read client for the query API; and anything
about deployment, which is where rate limiting and TLS live.
