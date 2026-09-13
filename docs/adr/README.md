# Architecture decision records

ADRs are written only for decisions a future maintainer would reasonably
ask "why was it done this way?" about, and where the answer depends on
alternatives that were considered. Everything else is stated in
[`../architecture.md`](../architecture.md).

Format: Status, Date, Context, Decision, Options considered, Consequences,
Deferred. An ADR is never edited to change its meaning; it is superseded by
a new one that links back.

| # | Title | Status |
|---|---|---|
| [0001](0001-repository-topology.md) | Repository topology | Accepted |
| [0002](0002-reporting-protocol-boundary.md) | Reporting protocol is language-neutral and event-based; runner knowledge stops at the adapter | Accepted |
| [0003](0003-redaction-and-attachment-handling.md) | Producer-side redaction and out-of-band attachments | Accepted (item 5 clarified by 0004; item 3 deduplication and its retention consequence superseded by 0009) |
| [0004](0004-opaque-archive-attachments.md) | Opaque archive attachments are stored, never extracted | Accepted |
| [0005](0005-scope-failures-are-protocol-events.md) | Non-test hierarchy failures are first-class protocol events and do not alter child test verdicts | Accepted |
| [0006](0006-session-outcomes-are-protocol-facts.md) | Runner-level outcomes are first-class session facts and the run verdict stays derived | Accepted |
| [0007](0007-project-is-ingestion-context.md) | The project is ingestion context, and the run key is (project, run id) | Accepted |
| [0008](0008-postgresql-run-archive.md) | PostgreSQL archives complete validated runs as their original protocol source | Accepted (attachment bytes made durable by 0009) |
| [0009](0009-durable-attachment-blobs.md) | Attachment bytes are durable content-addressed blobs on a local filesystem | Accepted (expiry and collection added by 0010) |
| [0010](0010-run-retention-and-blob-collection.md) | Runs carry an explicit expiry, and globally deduplicated blobs are collected only when no run references them | Accepted |
| [0011](0011-postgresql-query-indexes.md) | Cross-run questions are answered from rebuildable PostgreSQL indexes, one run from its own source | Accepted |
