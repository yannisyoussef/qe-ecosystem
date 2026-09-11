# ADR-0003: Producer-side redaction and out-of-band attachments

Status: Accepted
Date: 2026-09-11

## Context

Test tooling emits credentials by accident: authorization headers in
Rest Assured logs, cookies in Playwright traces, tokens in stack traces,
whole environments in failure dumps. The platform will store screenshots,
videos, traces, HTTP exchanges, and logs, all produced by code the platform
does not control. Two questions must be settled before any event or upload
exists: where redaction happens, and whether attachment bytes ride inside
events.

## Decision

1. Redaction is the producer's job first. The SDK redacts before
   serialization: a built-in set of sensitive headers, secret-shaped values
   in text (bearer tokens, JWTs, cloud access keys, password-style pairs),
   and an extensible pattern list. Environment variables are captured only
   from an explicit allowlist. Enrichers attach request and response bodies
   on failure only unless the user opts in to more.
2. The platform applies the same text redaction on ingestion as a second
   layer, for producers that are old, misconfigured, or not ours. There is
   no unredact path and redaction is not reversible.
3. Attachment bytes never travel inside events. An event carries metadata
   (name, media type, size, SHA-256 content hash, scope); bytes are
   uploaded separately, referenced by content hash, and stored under
   server-controlled keys. Producer-supplied names and paths are display
   strings. The hash is integrity and deduplication metadata; any
   deduplication visible in the logical model is scoped to a project.
   Cross-project storage-level deduplication is not required and is
   introduced only if it cannot act as an existence oracle across projects
   or leak retention between them. Until authenticated producer identity
   and authorization exist, `projectId` partitions data and is not an
   authorization boundary.
4. Ingestion enforces a media-type allowlist, sniffs declared types, and
   caps size per attachment and per run. Binary over the limit is rejected
   with a reason; text over the limit is truncated with a marker.
5. No archives in protocol version 1.
6. Every producer string is untrusted text when rendered. HTML attachments
   are never rendered in the application origin.

## Options considered

1. Platform-only redaction. Simpler SDKs, but secrets would cross the
   network and land in ingestion logs and raw event storage before
   redaction, and binary artifacts cannot be redacted after the fact.
   Rejected.
2. Producer-only redaction. Leaves the platform exposed to any producer
   that skips it. Rejected; producer first, platform second.
3. Inline base64 attachments in events. One upload path and a single file
   per run, but event size becomes unbounded, screenshots and videos would
   bloat event storage, and streaming would stall on large payloads.
   Rejected.
4. Out-of-band content-addressed attachments (chosen). Deduplicates
   identical screenshots across retries, keeps events small, and lets the
   file sink write a sidecar directory that the upload step walks.

## Consequences

- The file sink format has two parts from QE-001: an event file and an
  attachment directory keyed by hash.
- The Rest Assured enricher and the Playwright adapter both need the
  redaction API in the SDK, so redaction is designed in QE-001 with the
  protocol, not added later.
- Attachment upload is a separate endpoint with its own limits (QE-005).
- Deduplication by hash means retention deletes a blob only when no
  unexpired run in the same project references it.

## Deferred

- The default pattern list and its update policy (QE-001).
- A sandboxed origin for rendering HTML attachments (after the first UI).
- Per-producer rate limits (with authentication).
