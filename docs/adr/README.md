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
| [0003](0003-redaction-and-attachment-handling.md) | Producer-side redaction and out-of-band attachments | Accepted (item 5 clarified by 0004) |
| [0004](0004-opaque-archive-attachments.md) | Opaque archive attachments are stored, never extracted | Accepted |
| [0005](0005-scope-failures-are-protocol-events.md) | Non-test hierarchy failures are first-class protocol events and do not alter child test verdicts | Accepted |
