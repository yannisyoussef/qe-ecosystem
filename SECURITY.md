# Security

## Reporting a vulnerability

Use GitHub's private vulnerability reporting on the affected repository
("Report a vulnerability" under the Security tab). Do not open a public
issue for a security problem. Reports are acknowledged within a week and
fixed versions are noted in the repository's release notes.

Before version 1.0 of an artifact, only its latest release receives fixes.

## Invariants

The reporting platform stores content produced by test tools the platform
does not control: HTTP exchanges, screenshots, traces, videos, logs, stack
traces, and environment metadata. The invariants below hold from the first
release and are described in the
[architecture document](docs/architecture.md#9-security-and-privacy-baseline)
[ADR-0003](docs/adr/0003-redaction-and-attachment-handling.md), and
[ADR-0004](docs/adr/0004-opaque-archive-attachments.md):

- Redaction of sensitive headers and secret-shaped values happens in the
  producer before serialization, and again on ingestion. There is no
  unredact path.
- Attachment bytes travel separately from events and are stored under
  server-generated keys. Producer-supplied names and paths are never used
  to locate anything on the server.
- Media types are allowlisted and verified; sizes are capped per attachment
  and per run.
- Producer strings are rendered as text. HTML attachments are never
  rendered in the application origin.
- Archives are never extracted or treated as files to unpack. An opaque
  archive attachment is stored and downloaded unchanged.
- Every stored run has an expiry and a project identifier.
- Secrets reach the platform only through the environment.

A violation of any of these is a security bug, not a missing feature.
