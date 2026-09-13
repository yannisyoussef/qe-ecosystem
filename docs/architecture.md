# Quality Engineering ecosystem: architecture

This document records the ecosystem-level decisions that apply to every
repository in the ecosystem. Repository-specific design lives in each
repository. Decisions that needed a durable explanation of alternatives are
in [`adr/`](adr/README.md); everything else is here.

Status: foundation. No protocol, library, or platform code exists yet. The
open items are listed in [section 13](#13-deliberately-undecided).

## 1. What the ecosystem is

A set of public projects for building, running, and observing automated
tests:

- Java libraries for test suites, distributed through Maven Central, with a
  BOM that pins compatible versions.
- Reference starter repositories for Rest Assured, Karate, Playwright
  (TypeScript), and Selenium, each written in that tool's own idiom.
- A runner-agnostic test reporting platform: a protocol that describes test
  runs, adapters that emit it from real runners, and a service that ingests,
  stores, and displays it.
- Two existing products consumed through their public contracts: TestInbox
  (disposable email for tests) and KaaS (Karate as a Service).

The projects interoperate through one direction of data flow:

```
JUnit 5 / TestNG / Cucumber / Karate / Playwright
        |  runner-native extension point
        v
     adapter  (knows the runner and the protocol, nothing else)
        |
        v
  protocol events  (language-neutral schema; Java and TypeScript bindings)
        |
        v
   SDK transport  (file sink first, HTTP sink second; redaction and limits)
        |
        v
 reporting platform  (ingestion -> storage -> read model -> UI)
```

Karate may additionally execute through KaaS. Tests may use TestInbox.
Neither product's internal model appears in the protocol.

What is deliberately not being built at this stage: the protocol schema,
the platform, any adapter, any library, any starter, authentication or
multi-tenancy, a plugin system, and the interactive QA Lab. Each of these
starts only when the milestone before it has produced a working consumer.

## 2. Project kinds and ownership

| Kind | Examples | Owns | Must not contain |
|---|---|---|---|
| Library | protocol SDK, reporter adapters, Rest Assured HTTP capture, BOM | One narrow capability behind a small public API | Project-layout opinions, base test classes, configuration formats, global mutable state |
| Reference starter | `qe-starter-restassured`, `qe-starter-playwright` | Project layout, configuration, CI wiring, example tests, documented patterns | Code that other repositories depend on; shared code with other starters |
| Product | reporting platform, TestInbox, KaaS | Its storage, API, and UI | Knowledge of specific runners (platform); ecosystem-internal types (TestInbox, KaaS) |
| Hub | this repository | Ecosystem architecture, ADRs that span repositories, policy files other repositories copy | Code |

Concerns that stay out of shared code, because sharing them couples suites
that have nothing in common:

- page objects, API clients for the system under test, test data builders
- environment and configuration formats
- assertion helpers and custom matchers
- `BaseTest` and `BasePage` hierarchies
- retry policies, wait strategies
- Karate feature conventions, Playwright fixture composition

These are documented as patterns inside the starters. Any of them becomes a
library only after two starters need the same thing and the shared version
does not erase a native capability.

Coupling rules:

1. Adapters depend on the protocol SDK only. Never on the platform.
2. Platform ingestion, storage, read model, and UI contain no runner-specific
   code path. Runner-specific data travels in labels, raw status, and
   producer metadata, and the UI renders by media type, not by runner.
3. Starters depend on published libraries only, never on other starters or
   on unpublished code.
4. Libraries never depend on starters or on the platform.
5. TestInbox and KaaS are integrated through their public HTTP API and SDK.
   Translation between their contracts and the protocol lives in an adapter
   that belongs to the integration, not in either product's domain model.
6. Reporting attaches at the runner's own extension point. Removing it is a
   configuration change, not a code change. Automation abstractions never
   import reporting types.
7. There is no `common` or `utils` module across repositories. Sharing
   happens by publishing a named library with one purpose.

Runner-native attachment points that adapters will use:

| Runner | Extension point | Notes |
|---|---|---|
| JUnit Platform (Jupiter) | `TestExecutionListener`, registered via `ServiceLoader` | Sees every engine, including Cucumber-JVM and Karate when run through the platform; requires no change to test code |
| TestNG | `ITestListener` / `IReporter` | |
| Cucumber-JVM | `Plugin` on the event bus | Provides step granularity that the JUnit listener cannot see |
| Karate | `RuntimeHook` | Feature, scenario, and step granularity, plus HTTP exchanges |
| Playwright | `Reporter` in `playwright.config` | Attempts, steps, and attachments are native concepts |

Selenium and Rest Assured are not runners. They are enriched by tool-specific
helpers (a Rest Assured `Filter` that captures exchanges, a Selenium helper
that captures a screenshot on failure) that attach data to the current
attempt through the SDK. The JUnit or TestNG adapter carries the lifecycle.

## 3. Repositories

See [ADR-0001](adr/0001-repository-topology.md) for the reasoning.

| Repository | Contents | Created at | Publishes |
|---|---|---|---|
| `qe-ecosystem` | this document, ADRs, policy files | now | nothing |
| `qe-report` | protocol schema and fixtures; Java SDK, adapters, platform; TypeScript SDK and adapters | QE-001 | Maven artifacts, npm packages, one container image |
| `qe-java` | BOM and Java test-support libraries | first starter that needs the BOM | Maven artifacts |
| `qe-starter-restassured` | Rest Assured + JUnit reference project | after the JUnit adapter and BOM exist | nothing (GitHub template repository) |
| `qe-starter-playwright` | Playwright TypeScript reference project | after the Playwright adapter exists | nothing (template) |
| `qe-starter-karate` | Karate reference project, KaaS execution documented | after the Karate adapter exists | nothing (template) |
| `qe-starter-selenium` | Selenium + JUnit reference project | after the Rest Assured starter has proven the JUnit path | nothing (template) |

Architecture-pattern examples are sections and directories inside the
starters, not a repository. Starters are created one at a time and only when
there is something to show; a starter without a working reporting path is
not created.

`qe-report` is a polyglot monorepo with three independent builds:

```
qe-report/
  protocol/        JSON Schema, fixture corpus, protocol documentation (source of truth)
  java/            Gradle build: sdk, adapter modules, platform modules
  ts/              pnpm workspace: protocol types, sdk, adapter packages, web UI when it exists
  compose.yaml     local dependencies for the platform
  docs/
```

The monorepo is split when the protocol reaches 1.0 and a family's release
cadence diverges from the others, or when a third language binding appears.
Not before.

## 4. Naming

Names are temporary technical names. Product naming happens when a product
exists to name. Renaming is free before the first non-prerelease publication
of an artifact and is a major version after it.

| Thing | Scheme | Example |
|---|---|---|
| GitHub repository | `qe-<product>` or `qe-starter-<tool>` | `qe-report`, `qe-starter-karate` |
| Maven group ID | `io.github.yannisyoussef` | decided; verified automatically through GitHub on the Central Portal |
| Maven artifact | `qe-<product>-<module>` | `qe-report-protocol`, `qe-report-sdk`, `qe-report-junit-platform`, `qe-report-karate`, `qe-bom` |
| Java package | `io.github.yannisyoussef.qe.<product>.<module>` | `io.github.yannisyoussef.qe.report.junitplatform` |
| Java internal package | `...<module>.internal` | excluded from compatibility guarantees |
| npm scope and package | `@<scope>/qe-<product>-<module>` | `@<scope>/qe-report-playwright`; scope deferred until the first npm publication, see section 13 |
| Protocol | `qe-report-protocol`, versioned separately from every binding | schema `$id` base URL decided in QE-001 |
| Adapter | `qe-report-<runner>` | one artifact per runner family, named for the extension boundary it observes (`junit-platform`, not `junit5`), never per tool |
| Container image | `ghcr.io/yannisyoussef/qe-<product>` | `ghcr.io/yannisyoussef/qe-report` |
| Milestones | `QE-NNN` | used in issues and pull requests, not in code |

## 5. Technology baseline

Chosen for the smallest stack that a test-suite consumer in 2026 would
accept, not for novelty. Versions below are the state at the time of writing
and are pinned in each repository's version catalog or lockfile, where
Dependabot keeps them current.

### Java

| Concern | Decision | Why |
|---|---|---|
| Build and runtime JDK | Java 25 (LTS) via Gradle toolchains | Current LTS; the platform runs on it |
| Published library bytecode | `--release 17` | Test suites run on 17, 21, and 25. JUnit 6, Spring Boot 4, and the JVM tools in scope all require 17, so 17 is the floor and nothing in a reporter needs a newer API. CI compiles and tests libraries on 17, 21, and 25 |
| Starter toolchain | Java 21 | The LTS a team adopting a template in 2026 is most likely on; switching to 25 is one line |
| Build tool | Gradle 9.1 or newer within the 9.x line, wrapper pinned to the tested release, Kotlin DSL, version catalog, configuration cache on, `FAIL_ON_PROJECT_REPOS` | 9.1 is the first line that runs on Java 25; the wrapper records the exact release and is the only Gradle a contributor needs |
| Test platform | JUnit Platform 6 (Jupiter 6) for the ecosystem's own tests | Current major. The JUnit adapter's supported consumer range is decided and tested in QE-002 |
| Language | Java, not Kotlin, for public libraries | A Kotlin runtime dependency is a cost for Java-only test suites. Kotlin is fine inside the platform if a module benefits |
| Null safety | JSpecify `@NullMarked` on public library packages | Cheap, tool-neutral, and it documents the API contract |
| Static analysis | compiler warnings as errors, Error Prone, Spotless formatting | Catches real bugs at compile time and keeps diffs free of formatting noise. Nothing else until a defect class proves the need |
| Publishing | Maven Central through the Central Portal, `com.vanniktech.maven.publish`, GPG signing, tag-triggered workflow only | OSSRH no longer exists. No snapshots are published; prereleases carry SemVer prerelease identifiers |

### TypeScript

| Concern | Decision | Why |
|---|---|---|
| Node for building and CI | Node 24 (active LTS) | Node 26 becomes LTS in October 2026 and replaces it then |
| Node floor for published packages | `engines.node >= 22` | Node 20 reached end of life in April 2026. Raised deliberately, as a minor with release-note prominence |
| TypeScript | 6.0.x as the initial workspace compiler and tooling baseline; emitted declarations must compile for consumers on TypeScript 5.5 and newer | TypeScript 7 is stable but typescript-eslint does not yet support it officially. Moving to 7 is a compatibility upgrade, done once the lint and build stack supports it or after a spike shows the stack works without unsupported-version overrides. No alias hacks or dual installations without a concrete requirement |
| Package manager | pnpm with a `packageManager` field in workspace repositories; npm in the Playwright starter | Workspaces need pnpm's strictness; a template should not impose a package manager |
| Bundling | `tsdown`, dual ESM and CJS with an `exports` map | Playwright loads reporters through both module systems in the wild |
| Tests, lint, format | Vitest, ESLint flat config with typescript-eslint, Prettier | Standard, small |
| Publishing | npm trusted publishing (OIDC) with provenance, tag-triggered workflow only | No long-lived tokens in CI |

### Platform and infrastructure

| Concern | Decision | Why |
|---|---|---|
| Platform stack | Java 25, Spring Boot 4.x | Same stack as KaaS and TestInbox; the reporting platform is a modular monolith, not services |
| Database | PostgreSQL, one major pinned in Compose, CI, and Testcontainers (16, the major the first run archive is exercised against) | Relational data plus `jsonb` for metadata; both existing products use it |
| Artifact bytes | Behind one storage boundary. First implementation (in place): a local filesystem content-addressed store under server-generated keys derived from the SHA-256, which is the blob identity (ADR-0009). S3-compatible implementation when a non-local deployment exists | Nothing before the first deployment needs object storage; the boundary makes the swap an implementation change |
| Local development | `docker compose up` for PostgreSQL only; the platform runs from Gradle; integration tests use Testcontainers and never a shared database | One command to start; no hidden shared state |
| Container image | Spring Boot buildpacks (`bootBuildImage`), published to GHCR on release | No Dockerfile to maintain; SBOM comes with it |

### CI and supply chain

| Concern | Decision |
|---|---|
| CI | GitHub Actions. `ci.yml` builds, tests, and runs static analysis on every push and pull request. `release.yml` runs only on tags |
| Action pinning | Actions pinned by commit SHA with a version comment; Dependabot updates them |
| Dependencies | Dependabot, weekly, grouped minor and patch updates per ecosystem, low open-PR limit |
| Secret and code scanning | GitHub secret scanning with push protection, gitleaks in CI, CodeQL default setup for Java and JavaScript |
| Release | Tag-driven. Version bumps and changelogs generated from Conventional Commits by release-please once the first artifact is published. Until then, no releases |

## 6. Versioning and compatibility

All published artifacts follow Semantic Versioning 2.0.

Version families in `qe-report`, each with its own tag prefix and cadence:

| Family | Versioned as | Contains |
|---|---|---|
| Protocol | `protocol/vX.Y.Z` | the schema and fixture corpus |
| Java | `java/vX.Y.Z`, one version for all Java artifacts in a release | SDK, adapters |
| TypeScript | `ts/vX.Y.Z`, one version for all packages in a release | protocol types, SDK, adapters |
| Platform | `platform/vX.Y.Z`, container image tag | the service |

Rules:

- Before 1.0 a minor version may break, and the CHANGELOG says so. For the
  protocol, the compatibility unit before 1.0 is `0.minor`: a change that
  would be a major after 1.0 starts a new `0.x` line. 1.0 of an adapter or
  SDK requires 1.0 of the protocol.
- Every event carries the full protocol version (`protocolVersion:
  "0.3.0"`), not only the major. Within a supported major, a consumer
  ignores unknown optional fields, and a producer never requires a new
  field. A new event type may be added in a minor only when an older
  consumer can ignore it without changing the meaning of existing events
  or producing an incorrect projection; if interpreting the new event is
  needed to derive run, attempt, test, or other existing state, adding it
  is a major. An event type the consumer does not support and that is not
  marked safe to ignore produces a visible unsupported-protocol or
  unsupported-event error, never a silently incomplete read model. The
  platform accepts every minor of each supported major and supports the
  current and previous major. An adapter emits exactly one protocol major.
  The mechanics (how an event declares that it is safe to ignore, and how
  the error surfaces) are decided in QE-001.
- Each adapter declares the runner version range it supports and CI tests
  that range. A runner major that keeps the extension point compatible
  extends the range in an adapter minor; one that breaks it is an adapter
  major.
- The BOM has its own version. A BOM minor updates pins; a BOM major drops a
  Java baseline or crosses a major of a pinned library. Consumers import it
  with `platform()`.
- Public API is what is not in an `internal` package (Java) or not in the
  `exports` map (TypeScript). Removal in 1.x requires deprecation for at
  least one minor and a CHANGELOG entry.
- Prereleases use `-alpha.N` and `-rc.N` identifiers, on Maven Central and
  on the npm `next` dist-tag. Snapshots are never published.
- Starters are templates, not dependencies. They carry no version; the
  README states the BOM and adapter versions in use, and consumers diff
  against the template when they want updates.
- TestInbox and KaaS are pinned by their own published versions, in the BOM,
  only once a library or starter consumes them.

## 7. Reporting protocol boundary

The protocol is designed in QE-001. This section fixes what QE-001 must
satisfy so that the schema is not accidentally shaped by the first runner.
See [ADR-0002](adr/0002-reporting-protocol-boundary.md),
[ADR-0005](adr/0005-scope-failures-are-protocol-events.md), and
[ADR-0006](adr/0006-session-outcomes-are-protocol-facts.md). The current
compatibility line is stated by the protocol documentation in `qe-report`.

The protocol must represent runs from JUnit 5, Rest Assured under JUnit,
Karate, Playwright, Selenium under JUnit or TestNG, Cucumber, and TestNG
without a runner-specific branch anywhere past the adapter. It must remain
representable for Cypress, pytest, Jest, Vitest, and Robot Framework, which
is checked by keeping worked mappings for two of them in the protocol
documentation without implementing adapters.

Minimum canonical concepts:

| Concept | Meaning | Constraint |
|---|---|---|
| Run | One logical execution of a test process or pipeline | Identity generated by the producer, so events can be written before any server exists and multiple processes can contribute to one run |
| Session | One producer process contributing to a run | Parallel workers, forked JVMs, and sharded CI jobs each open a session against a shared run id |
| Test path | Where a test sits in the runner's own hierarchy | A list of typed, named segments, not a fixed suite/class/method shape; Karate feature/scenario, Playwright project/file/describe, JUnit engine/class/method all fit |
| Test case | A display name, tags, a source location when known, and two identities: an execution identity that correlates lifecycle events within a run, and a historical identity that links the same logical test across runs | Adapters own the derivation rules; the resulting formats are canonical. Runner-native identifiers such as JUnit `UniqueId` are kept as metadata, not assumed to be the historical identity. An adapter can state that a stable historical identity cannot be guaranteed, as with dynamically generated tests. Variants (Playwright projects, parameterized rows, outline examples) are part of identity |
| Attempt | One execution of a test case within a run | Retries are additional attempts, never overwrites; the run-level outcome of a test case is derived, and "passed after failure" stays visible |
| Status | Canonical set (at least passed, failed, skipped, and an aborted or infrastructure outcome) plus the runner's raw status preserved | The canonical set is small and the raw status is never lost |
| Step | A named, timed, nested unit inside an attempt | Optional; Karate, Cucumber, and Playwright supply them, JUnit does not |
| Attachment | Metadata for a screenshot, video, trace, HTTP exchange, log, or text, scoped to an attempt or step | Bytes never travel inside events; see section 9 |
| Failure | Message, type, structured or raw stack trace, expected and actual when available, cause chain | Redacted by the producer before serialization |
| Scope failure | A failure of a non-test node of the runner hierarchy (class, suite, file, module) | Recorded as its own event with the scope's path; it fails the run without changing any attempt's verdict (ADR-0005) |
| Session outcome | The aggregate outcome a runner reports for its own invocation (passed, failed, inconclusive), with its raw word and any invocation-level failures | Optional on the event that closes the session; a runner without one emits nothing and the verdict is derived from attempts and scope failures (ADR-0006) |
| Output root and run directory | The directory an adapter is configured with is an output root; each run lives in `runs/<run directory>` below it, named from the run id (readable stem plus a short hash) | One physical run directory holds one logical run; the name is a locator and the `runId` inside the events stays authoritative; validation and ingestion address one run directory |
| Project | The partition a platform ingests a run into: an opaque key supplied by whoever ingests, never carried by events or derived from them | The run key is `(project, run id)` and the history key `(project, runner name, historical id)`; adapters and the protocol know nothing of it (ADR-0007) |
| Durable run archive | Complete, validator-valid runs stored in PostgreSQL as their original protocol JSON lines under `(project, run id)`, with a content fingerprint and ingestion metadata | The raw lines are the only durable truth; projections, histories, flakiness, and the blob catalog are rebuilt through the validator and projector; same content is idempotent, different content under one identity is refused (ADR-0008) |
| Durable attachment bytes | The bytes an attachment event names, stored once under their full SHA-256 in an immutable content-addressed blob store, with a global catalog row and a per-run relation in PostgreSQL | Blob identity is the hash alone; the same bytes across runs and projects are one object; a run commits only after its blobs are durable, and full verification re-reads the bytes; the run archive, not the blob store, records who referenced what (ADR-0009) |
| Run expiry and retention | An absolute instant supplied at ingestion, stored beside the run, and enforced only by explicit bounded maintenance that deletes expired runs and reclaims bytes no run anywhere references | Expiry is ingestion context like the project, never derived from events or ingestion time, and never moved by re-ingestion; run deletion cascades source lines, blob relations, and the expiry itself, never the global blob catalog; ingestion and destructive maintenance are serialized by one shared/exclusive advisory lock; nothing is scheduled (ADR-0010) |
| Tags and labels | Free tags (JUnit `@Tag`, Playwright `@tag`, Cucumber tags) and key-value labels with a small reserved key set | |
| Environment, executor, source, producer | System under test, CI context, VCS state, and the adapter and runner versions | Run-level, with an attempt-level override only where a real runner needs it |

Lifecycle and event concepts QE-001 must investigate and decide:

- events as the canonical unit, with a newline-delimited file form so an
  offline capture and a live stream are the same format
- event identity, per-session sequence numbers, idempotent ingestion,
  tolerance of reordering within a run, rejection of duplicate completion
- run completion: explicit finish versus inactivity timeout for producers
  that crash, and how partial runs are represented
- producer clocks: timestamps with offsets and durations carried separately
- expected failures, fixme, assumptions, and skip reasons across runners
- size limits per event, per attachment, and per run, with visible
  truncation markers for text
- what belongs to the envelope (protocol version, producer identity slot
  that later authentication can use) versus the payload
- the schema technology: JSON Schema 2020-12 as the single source of truth,
  camelCase JSON, integers for durations in milliseconds, strings for ids,
  no constructs that only map naturally to one language. Java and
  TypeScript bindings must both round-trip the same fixture corpus in CI.
  Cross-language equivalence is semantic: outputs validate against the
  same schema, parse to equivalent protocol values, keep the ordering the
  protocol requires, and carry identical attachment hashes. Byte-identical
  serialization is not required unless QE-001 adopts a documented
  canonical JSON form.

The detailed schema is not frozen here.

## 8. Reporter architecture

| Layer | Knows | Must not know |
|---|---|---|
| Runner | nothing about the ecosystem | |
| Adapter | the runner's extension API and the protocol model | HTTP, configuration formats beyond what the runner provides, the platform |
| Enricher (Rest Assured filter, Selenium screenshot helper) | its tool and the SDK's current-attempt context | the runner lifecycle |
| Protocol model | the schema | runners, transport |
| SDK transport | the protocol, file and HTTP sinks, batching, bounded transport retry, redaction, limits | runners, the platform's storage |
| Ingestion | the protocol, validation, idempotency, limits, producer identity | runners |
| Storage | events, blobs, retention | runners, UI |
| Read model and query | projections: run summary, test history, flakiness | runners |
| UI | the read model and media types | runners |

The current-attempt context used by enrichers is the one place a scoped,
implicit context exists (thread-scoped in Java, async-local in Node). It is
documented, explicit in the SDK API, and never global configuration.

Reporting failure never fails a test and never crashes a run. The SDK
reports its own problems once, visibly, and continues. A failed test is
never retried by reporting infrastructure; transport retries are bounded
and apply only to delivering events.

KaaS produces its own execution result and artifact manifest contracts. A
KaaS integration translates those into protocol events in an adapter owned
by the integration, or runs the Karate adapter inside the KaaS engine. That
choice is made when the Karate adapter exists.

## 9. Security and privacy baseline

These requirements apply from the first line of protocol, SDK, and platform
code. See [ADR-0003](adr/0003-redaction-and-attachment-handling.md) and
[ADR-0004](adr/0004-opaque-archive-attachments.md).

Data handled: HTTP requests and responses, screenshots, browser traces,
videos, console output, environment metadata, stack traces, logs, test data,
and credentials that test tools emit by accident. All of it is untrusted.

1. Redaction happens in the producer before serialization. The SDK redacts
   a built-in set of sensitive headers (authorization, cookie, set-cookie,
   proxy-authorization, API-key style headers) and secret-shaped values in
   text (bearer tokens, JWTs, cloud access keys, `password=` style pairs),
   with a way to add patterns. Environment variables are captured only from
   an explicit allowlist. The platform applies the same text redaction on
   ingestion as a second layer and never has an unredact path.
2. Capture is minimal by default. Enrichers attach request and response
   bodies on failure only; capturing everything is opt-in.
3. Attachment bytes travel separately from events, with a SHA-256 content
   hash for integrity and deduplication, a declared media type, and a
   size. Ingestion enforces a media-type allowlist, validates declared
   types by sniffing, caps size per attachment and per run, and stores
   under server-generated keys. Producer-supplied names and paths are
   display strings only and are never used to locate anything on the
   server. Storage-level deduplication is global: the same bytes are one
   blob whatever project references them (ADR-0009). Nothing may
   therefore expose whether a blob exists by hash alone without a run
   reference the caller is allowed to see, and retention must
   reference-count blobs across projects before deleting any.
4. Archives are never extracted and never accepted as transport containers.
   An opaque archive attachment (a Playwright trace, for example) is stored
   and downloaded unchanged under the same hash, size, and media-type
   rules as any binary attachment, and is download-only until an isolated
   viewer is decided separately (ADR-0004).
5. Rendering treats every producer string as text. Stack traces and logs are
   escaped, ANSI sequences are stripped, and lengths are capped. HTML
   attachments are never rendered in the application origin; they are
   downloaded, or later served from a separate sandboxed origin with a
   strict Content Security Policy.
6. Responses for attachments carry `X-Content-Type-Options: nosniff`,
   `Content-Disposition: attachment` for anything that is not an image, and
   never echo `text/html`.
7. Request bodies, batch sizes, events per run, and attachments per run are
   capped. Text over the limit is truncated with a marker; binary over the
   limit is rejected with a reason.
8. Every run ingested into the durable archive carries an explicit expiry
   supplied by whoever ingests it. Deleting a run cascades to its source
   and its blob references; the bytes themselves are reclaimed by the same
   explicit maintenance pass once no run anywhere references them, never
   by a foreign-key cascade. The invariant is enforced by the writer from
   ADR-0010 (migration 3); runs archived before it are identifiable
   retention-unmanaged records, never deleted by guesswork, and gain the
   fact only when re-ingested. A retention job is invoked explicitly;
   none runs by itself yet.
9. Every stored record carries a project identifier from day one so that
   authentication and tenancy are additive. Until authenticated producer
   identity and authorization exist, `projectId` is a partitioning
   attribute only and must not be treated as an authorization boundary.
   Authentication itself is deferred.
10. Secrets reach the platform only through the environment. Repositories
    contain `.env.example` with names and no values.

## 10. Engineering principles

1. Native idioms first. Karate, Playwright, JUnit, and TestNG keep their own
   shape; the ecosystem adapts to them, not the reverse.
2. Share contracts, not classes. Cross-language and cross-repository
   agreement is a schema or an interface with a fixture corpus, never a
   shared implementation.
3. Small public surface. Everything not explicitly public is internal and
   free to change. An API is added when a second heterogeneous consumer
   needs it.
4. Deterministic where practical. Clocks and id generators are injected at
   boundaries; ordering rules are explicit; the same input produces the same
   stored state.
5. Everything is bounded. Sizes, counts, durations, queue depths, and
   retries have limits, and each limit has a test that reaches it.
6. Failures are data. A failing test is recorded, never retried by
   infrastructure to make it pass. Reporting problems are reported once and
   loudly, and never change a test result or abort a run.
7. No hidden state. No static registries or global mutable configuration;
   the documented current-attempt context is the single scoped exception.
8. Errors say what happened, where, and what to do next.
9. Secure by default, capture by opt-in.
10. Compatibility is a tested promise. Supported runner, JDK, Node, and
    protocol ranges are CI matrices, not README claims.
11. Tests sit at boundaries: fixture round-trips for bindings, the real
    runner for adapters, Testcontainers for the platform, and the shared
    fixture corpus as the contract test between SDK and ingestion.
12. The platform is observable: structured logs and health endpoints from
    the first slice.
13. Documentation is for users and maintainers. A decision is recorded once,
    in the place a reader would look for it.

## 11. Documentation and repository policy

The full policy, including what is never committed, is in
[`CONTRIBUTING.md`](../CONTRIBUTING.md). In short: README, architecture
overview, a small number of ADRs, CONTRIBUTING, SECURITY, a generated
CHANGELOG per published family, and CI configuration are public. Plans,
prompts, agent configuration, reviews, handoffs, and drafts are local and
ignored. New repositories copy the policy files from this one.

## 12. Direction

Milestones are named, not dated. Each one ends with something that runs.

1. QE-001: protocol draft, fixture corpus, Java and TypeScript bindings, a
   file sink, and a validator.
2. QE-002: JUnit Platform adapter writing protocol files from real runs.
3. QE-003: Playwright reporter doing the same, as the second heterogeneous
   producer.
4. QE-004: platform vertical slice that ingests those files, stores them, and
   shows runs and tests.
5. QE-005: HTTP sink and attachment upload.
6. QE-006: BOM, Rest Assured HTTP capture, and the Rest Assured starter as
   the first full consumer.
7. Then, in the order demand shows: Playwright starter, Karate adapter and
   starter with KaaS execution, Selenium starter, TestNG and Cucumber
   adapters, TestInbox usage in a starter, and the QA Lab.

## 13. Deliberately undecided

| Decision | Why deferred | Decide by |
|---|---|---|
| npm scope | Must be a scope the owner controls; confirmed at publication time. The license (Apache-2.0) and the Maven group ID (`io.github.yannisyoussef`) are decided | First npm publication |
| Protocol schema `$id` base URL | Follows the naming decision | QE-001 |
| Protocol detail: identity derivation, status set, envelope | QE-001 | QE-001 |
| JUnit adapter supported consumer range | Needs the adapter to exist to test it | QE-002 |
| Web UI framework and location in `qe-report` | No UI until QE-004; the read model comes first | QE-004 |
| Authentication and tenancy | Only the `projectId` partitioning attribute is fixed now; it is not an authorization boundary | After the first non-local deployment |
| Object storage backend | Filesystem behind a boundary until deployment | First non-local deployment |
| KaaS integration shape | Adapter inside KaaS versus translation of its result contract | Karate adapter milestone |
| Maven-build variants of the Java starters | Gradle first; demand decides | After the first two starters |
