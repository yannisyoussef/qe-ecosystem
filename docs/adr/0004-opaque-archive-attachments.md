# ADR-0004: Opaque archive attachments are stored, never extracted

Status: Accepted
Date: 2026-09-11

## Context

ADR-0003 and the security baseline say "no archives in protocol version 1".
Playwright produces its trace as a zip file, and a reporting platform that
cannot keep a trace loses the most useful artifact of a failed browser
test. QE-001 showed that the wording conflated two different things: an
archive as a transport container that a server would have to unpack, and
an archive as an attachment whose bytes are stored and downloaded
unchanged. Only the first carries the zip-slip, decompression-bomb, and
path-traversal risks the prohibition was written for.

## Decision

The prohibition concerns extraction and container ingestion, not opaque
storage:

1. Protocol events never inline archive contents.
2. No server-side archive extraction is introduced, and no archive is ever
   trusted as a set of files or directories.
3. An opaque archive attachment may be stored as bytes when the applicable
   ingestion media-type policy allows it, exactly like any other binary
   attachment: uploaded separately from events, SHA-256 and size verified,
   stored under a server-generated key, capped in size.
4. Opaque archives are download-only unless a future isolated viewer
   explicitly handles them; such a viewer is a separate decision.

This clarifies ADR-0003 item 5 and the corresponding line of the security
baseline; it does not change any other part of ADR-0003.

## Options considered

1. Keep "no archives" literally and drop Playwright traces. Loses the
   artifact the protocol most needs to keep for browser tests, for a risk
   that only exists when archives are opened. Rejected.
2. Accept archives and extract them into browsable attachments. Reintroduces
   every extraction risk and a large attack surface for a marginal viewing
   convenience. Rejected.
3. Store archives as opaque bytes, never extract (chosen).

## Consequences

- The protocol fixtures may carry `application/zip` attachments; the SDKs
  treat them as binary and do not redact them.
- The platform's media-type policy, when it exists, decides which archive
  types are accepted; acceptance never implies inspection.
- Any future viewer for archive content must be isolated (separate origin,
  no extraction on the platform's own host) and gets its own decision.

## Deferred

- The ingestion media-type policy (QE-004).
- Any isolated viewer for traces or other archives.
