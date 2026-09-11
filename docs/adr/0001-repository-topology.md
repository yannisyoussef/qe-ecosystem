# ADR-0001: Repository topology

Status: Accepted
Date: 2026-09-11

## Context

The ecosystem will contain a reporting protocol with Java and TypeScript
bindings, reporter adapters in both languages, a reporting platform, a Java
BOM and libraries, and several reference starter projects. Each of these
could be its own repository. The protocol must be versioned independently of
its bindings, adapters must be published as separate artifacts, and starters
must be copyable as templates. At the same time nothing exists yet, the
protocol will change while the first adapters are written, and every extra
repository costs CI, release wiring, and cross-repository version
coordination before there is anything to coordinate.

## Decision

Start with two repositories and add starters one at a time:

- `qe-ecosystem`: this hub. Cross-repository architecture, ADRs that span
  repositories, and the policy files that other repositories copy. No code.
- `qe-report`: a polyglot monorepo holding the protocol (schema and fixture
  corpus, the source of truth), the Java build (SDK, adapters, platform),
  and the TypeScript workspace (protocol types, SDK, adapters, web UI when
  it exists). Three independent builds, four independently versioned
  families (protocol, Java, TypeScript, platform) with tag prefixes.
- `qe-java`: a Gradle multi-module repository for the BOM and Java
  test-support libraries, created when the first starter needs the BOM.
- `qe-starter-<tool>`: one GitHub template repository per starter, created
  only when its adapter exists and it has a working reporting path.

Architecture-pattern examples live inside starters, not in a repository.

The `qe-report` monorepo is split when the protocol reaches 1.0 and a
family's release cadence diverges, or when a third language binding
appears.

## Options considered

1. One repository per concept from the start (eight or more repositories).
   Cleanest ownership, but cross-repository version pinning during the
   period when the protocol changes weekly would turn every schema change
   into three pull requests and a release. Rejected for now.
2. One monorepo for everything, including starters and the BOM. Starters
   stop being templates when they are subdirectories: GitHub's "use this
   template" does not apply, and a Node starter and a Gradle starter share
   nothing but a root. The BOM's audience is every Java test suite, not
   users of the platform. Rejected.
3. Protocol and adapters with the platform, starters separate, BOM separate
   (chosen). The things that change together while the protocol is proven
   stay together; the things that are copied or imported by strangers stay
   separate.
4. Java adapters in `qe-java` next to the BOM, with `qe-report` publishing
   only the SDK. This puts adapters one published-version hop away from the
   protocol they exercise during the period when that feedback loop matters
   most. Rejected until the protocol is stable; it can be revisited at the
   split.

## Consequences

- Playwright's reporter and the Java platform sit in one repository. CI
  runs three builds; a change to `protocol/` must pass both bindings'
  fixture round-trips in the same pull request, which is the point.
- Four version families in one repository require per-family tag prefixes
  and a release tool that understands them (release-please manifests).
- The BOM cannot exist before the first adapter is published, because it
  would pin nothing of the ecosystem's own.
- Starters are created late. Until the Rest Assured starter exists there is
  no end-to-end demonstration, which is accepted in exchange for not
  maintaining empty repositories.

## Deferred

- When to split `qe-report`, beyond the stated triggers.
- Whether a `.github` repository for account-wide default community files is
  worth adding once there are more than three repositories.
