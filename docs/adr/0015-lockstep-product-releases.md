# ADR-0015: qe-report v1 uses lockstep product releases over independent protocol, API, and storage compatibility lines

Status: Accepted
Date: 2026-09-30

## Context

qe-report is functionally complete: a protocol, two producer SDKs, two runner
adapters, a validator, a durable archive with content-addressed attachments,
retention, rebuildable query indexes, an authenticated HTTP API, a producer
uploader, and a reference deployment with rehearsed backup and restore.

Turning that into something a stranger can install raises a question the code
has so far avoided: what does a version number mean here? There are already
four numbers in the system that answer different questions, and none of them is
a product version:

```
protocol compatibility line   0.3    what events mean
HTTP API                      v1     what the service accepts and answers
database schema               5      what the server expects of PostgreSQL
API key format                qer_k1 what a machine credential looks like
```

A first stable release adds a fifth, and the risk is that it swallows the other
four. Calling the protocol 1.0 because the product reached 1.0.0 would destroy
information: a consumer of the protocol needs to know when *the protocol*
changed, not when the packages were released.

The second question is what to publish. There are ten TypeScript packages, four
of which exist only to make the server work, and a service that is better
distributed as an image than as a pile of implementation packages.

## Decision

The product version is `1.0.0` and follows SemVer. It governs the published
packages and artifacts, and nothing else.

The other version domains stay independent, and are not renamed, aligned, or
bumped because the product was released. A change to the protocol compatibility
line is its own architectural decision, recorded as its own ADR. A new HTTP API
version is its own contract decision. The database schema version is a
deployment concern that no producer ever sees. `COMPATIBILITY.md` in qe-report
states all of this for consumers; `release/release.json` records it in a form CI
checks against the code, so the claim and the implementation cannot drift.

Five npm packages are public:

```
qe-report-protocol
qe-report-sdk
qe-report-validator
qe-report-http-client
qe-report-playwright
```

Three Maven artifacts are public, under `io.github.yannisyoussef`:

```
qe-report-protocol
qe-report-sdk
qe-report-junit-platform
```

`qe-report-read-model`, `qe-report-blob-fs`, `qe-report-postgres`,
`qe-report-http-api` and `qe-report-equivalence` remain internal. They are not
published and carry no compatibility promise. The service they make is
distributed as a container image, `ghcr.io/yannisyoussef/qe-report`, because what
an operator wants is a thing that runs, not a set of implementation packages to
assemble.

For the 1.x line every public artifact carries the same product version and is
released together. This is a convenience, not a property of the design: nothing
in the code requires it, and it will be revisited if independent release pressure
appears. Internal packages may change between releases in ways that would be
breaking if they were public, as long as the container and the public packages
stay compatible.

`develop` stays integration and `master` stays the released line. There is no
permanent release branch. Promotion is a pull request a person merges, and
publication is driven by a tag on `master` alone: the release workflow verifies
that the tagged commit is `master`'s release commit and refuses to publish from
anywhere else.

Registries are immutable. Once npm or Maven Central has accepted a version, that
version is what it means; a defect is fixed in the next one. Publication across
three registries is not one transaction, so a partial release is resumed from the
same tag rather than retagged or renumbered, and every step checks what already
exists before adding to it. The GitHub Release is created last, only after all
three registries are verified, because it is an index of things that exist rather
than an announcement of intent.

## Options considered

1. Publish every internal npm package. It would turn four implementation
   details into public contracts nobody asked for, and every refactor into a
   breaking change. Rejected: the server is the product, not its parts.
2. Make the protocol version equal the product version. A consumer could no
   longer tell when the protocol changed, which is the only thing the protocol
   version exists to say. Rejected.
3. Make the HTTP API version equal the product version. Same loss, and it would
   imply a new API contract on every major product release. Rejected.
4. Make the database schema version part of the external compatibility story.
   No producer can see it and no consumer can act on it. Rejected: it is a
   deployment fact.
5. Publish from `develop`. The released line would then be whatever integration
   happened to hold. Rejected: a release comes from `master`.
6. Allow a release from an arbitrary `workflow_dispatch` commit. It would make
   the tag decorative and the provenance meaningless. Rejected: dispatch takes a
   tag, and that tag is validated exactly as a pushed one is.
7. Keep a long-lived release branch. A third branch to keep in sync, for a
   two-branch model that already works. Rejected.
8. Replace an already-published `1.0.0` when a defect is found. Registries do
   not allow it, consumers may already have it, and a version that changed
   meaning is worse than a version with a known bug. Rejected: the fix is
   `1.0.1`.
9. Publish `latest` as the only container tag, or at all in the first release.
   `latest` makes a deployment's version a question of when it pulled.
   Rejected for v1; exact versions and two moving aliases, with digest pinning
   recommended.
10. Give Java and TypeScript unrelated first stable versions. Every
    compatibility conversation would begin by translating between them.
    Rejected for 1.x.
11. Distribute the HTTP server primarily as a public npm package. It would make
    its internal dependencies public, and it is not how anyone wants to run a
    service with a database and an attachment store. Rejected in favour of the
    image.
12. Publish during pull-request CI, so the release path is exercised.
    Irreversible side effects on every branch. Rejected: CI rehearses
    everything except the network calls, and asserts that it cannot make them.
13. Create the GitHub Release first, as a place to attach artifacts. It would
    announce a release that might not exist. Rejected: it is last.
14. Multi-architecture images for symmetry. Only `linux/amd64` is built and
    rehearsed. Rejected: claiming an architecture nothing has run is a defect,
    not a feature.

## Consequences

- One committed contract, `release/release.json`, is the single source of the
  product version and the compatibility lines. CI fails if it disagrees with the
  code, including if a package's `private` flag changes without the contract.
- Publication readiness is provable without publishing. The rehearsal packs,
  audits, clean-installs, stages, signs with an ephemeral key, builds the image
  and generates the release metadata, using no credential, and asserts that it
  cannot publish.
- Account state is not provable from a repository. Namespace ownership on Maven
  Central and npm's publishing configuration are owner preconditions, recorded as
  such rather than assumed.
- The release workflow fails closed. A missing credential stops the release; it
  never skips a registry and reports success.
- A recorded API baseline for both languages makes an accidental break visible
  before it is published rather than after.

## Deferred

Independent per-artifact versioning; a second container architecture; an object
storage backend; a human identity model; anything that would require API v2 or a
new protocol compatibility line.
