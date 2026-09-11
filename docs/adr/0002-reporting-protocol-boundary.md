# ADR-0002: Reporting protocol is language-neutral and event-based; runner knowledge stops at the adapter

Status: Accepted
Date: 2026-09-11

## Context

The reporting platform must accept results from JUnit 5, TestNG, Cucumber,
Karate, Playwright, and Selenium-based suites, and later from Cypress,
pytest, Jest, Vitest, and Robot Framework. The first adapters are in Java
and TypeScript. Playwright reports attempts, steps, and attachments as
native concepts; JUnit reports a tree of test identifiers with start and
finish events and no steps; Karate reports features, scenarios, steps, and
HTTP exchanges; KaaS emits a complete result document after execution;
CI jobs may want to upload a file produced offline.

The risk is a protocol shaped by whichever runner is implemented first,
usually the Java one, that the TypeScript adapter then has to bend to.

## Decision

1. The protocol is a JSON Schema (draft 2020-12) with a fixture corpus.
   The schema is the source of truth. The Java and TypeScript bindings are
   derived from or checked against it, and both must round-trip the same
   fixtures in CI. No binding is authoritative.
2. Events are the canonical unit. A run is a sequence of events (run and
   session lifecycle, test case attempt lifecycle, steps, attachments,
   failures). A newline-delimited file of events is the same format as a
   live stream, so an offline capture, a CI upload, and a streaming adapter
   are one ingestion path.
3. The first SDK transport is a file sink. The HTTP sink comes after two
   heterogeneous adapters have produced files and the platform has ingested
   them. This proves the protocol before storage or transport constrain it.
4. Runner knowledge stops at the adapter. The protocol, SDK transport,
   ingestion, storage, read model, and UI contain no runner-specific code
   path. Runner-specific information travels as raw status, labels, test
   path segment kinds, and producer metadata. The UI renders attachments by
   media type.
5. Test location in the runner's hierarchy is a typed path of segments, not
   a fixed suite/class/method structure. A test case carries an execution
   identity, which correlates lifecycle events within a run, and a
   historical identity, which links the same logical test across runs.
   Both are canonical strings whose derivation rules are per adapter.
   Runner-native identifiers are kept as metadata, and an adapter may
   declare that a stable historical identity cannot be guaranteed.
6. Retries are attempts. Nothing in the protocol overwrites an earlier
   attempt.
7. JSON conventions are chosen so that both bindings are natural: camelCase
   keys, string ids, integer millisecond durations, ISO-8601 timestamps
   with offsets, no polymorphism that needs a class hierarchy to decode.

## Options considered

1. Java classes as the source of truth with generated JSON, and TypeScript
   types generated from that. Fast for the first adapter and shaped by
   Java's type system; Playwright's attempt and step model would be fitted
   afterwards. Rejected.
2. Adopt an existing format (JUnit XML, Allure results, CTRF). JUnit XML
   has no attempts, steps, or attachments and is a snapshot. Allure's
   result files are per-test snapshots tied to Allure's label vocabulary
   and to its report. CTRF is a JSON snapshot without a streaming form,
   sessions, or a path model. Each is a candidate for an import adapter
   later; none is the canonical model. Rejected as the protocol, kept as
   future inputs.
3. Snapshot document as canonical, with events as an optional streaming
   extension. Simpler to store, but a crashed producer leaves nothing, and
   parallel workers must merge documents before upload. Rejected; a
   snapshot is a batch of events.
4. Events as canonical (chosen).

## Consequences

- QE-001 delivers a schema, fixtures, two bindings, a file sink, and a
  validator, with no server. The first two adapters write files. The
  platform's first job is ingesting files that came from real runs.
- Ingestion must be idempotent and tolerate reordering within a run,
  because files and streams share the path.
- Every adapter carries an identity-derivation responsibility that must be
  documented and tested per runner, for both execution and historical
  identity. JUnit `UniqueId` is runner-native metadata; which of its
  components form a stable historical identity is investigated in QE-002.
- Both bindings are checked against the fixture corpus for semantic
  equivalence (same schema, equivalent parsed values, required ordering,
  identical attachment hashes), not byte equality. Deterministic byte
  serialization would be a separate protocol decision with documented
  canonicalization rules.
- Every event carries the full protocol version. A new event type is a
  minor only when an older consumer can ignore it without misreading
  existing state; otherwise it is a major, and an unsupported event that
  is not marked ignorable is a visible error, not a silent gap.
- Existing formats can be imported by adapters that translate them into
  events, including KaaS's result contract.
- The read model must project a run summary from events; the platform never
  trusts a producer-supplied summary.

## Deferred

- Exact event types, envelope fields, canonical status set, reserved label
  keys, and limits (QE-001).
- Whether bindings are generated or hand-written and checked (QE-001).
- The HTTP transport's batching and acknowledgement semantics (QE-005).
