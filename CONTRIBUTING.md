# Contributing

These conventions apply to every repository in the ecosystem. Repositories
copy this file or link to it.

## Branches and merging

- Two long-lived branches: `develop` is the integration branch and the
  default target for pull requests; `master` holds what has been released
  or, for documentation repositories, what is current.
- Work happens on short-lived branches off `develop`, named `feat/<topic>`,
  `fix/<topic>`, `docs/<topic>`, and so on, and is merged back into
  `develop` through a pull request.
- `develop` is merged into `master` when a release is cut, or for
  documentation repositories when a change is complete. `master` only ever
  receives merges from `develop`, or from a `fix/` branch for an urgent
  correction that is merged back into `develop` immediately.
- Pull requests are squash-merged by default, so the pull request title
  becomes the commit message. Rebase-merge is used when each commit in the
  branch stands on its own.
- Nothing is pushed directly to `develop` or `master` once a repository
  has CI.

## Commits

Conventional Commits, kept short:

```
<type>(<optional scope>): <imperative summary, lower case, no period, 72 chars max>

<optional body: what changed and why, when the summary is not enough>

<optional footer: BREAKING CHANGE: ..., Refs: QE-004>
```

Types: `feat`, `fix`, `refactor`, `perf`, `test`, `docs`, `build`, `ci`,
`chore`. The scope is a module or package name when it helps
(`feat(junit5): ...`). A breaking change uses `!` after the type or a
`BREAKING CHANGE:` footer.

A commit is one logical change. A refactor and the feature that needed it
are two commits. Documentation for a decision travels in the same pull
request as the change it explains.

Examples of the intended register:

```
feat: add run ingestion contract
fix: reject duplicate completion events
refactor: separate artifact metadata from storage
test: cover concurrent event ingestion
docs: document protocol compatibility
```

Enforcement is on the pull request title, which is what lands on `develop`.
Release tooling derives version bumps and changelog entries from these
messages, so the type must be right; the scope is optional.

## Pull requests

A pull request explains a change to a reviewer, not to a project manager.
The template asks for:

- What changed.
- Why, including the alternative not taken when there was one.
- How it was verified: the command, the test, or the manual check.
- Compatibility: public API, protocol, schema, or configuration changes,
  and any follow-up this creates.

Most descriptions fit in twenty lines. No status reports, no phase names,
no checklists of every file touched.

Keep pull requests reviewable: one concern, small enough to read in one
sitting. A large mechanical change (formatting, a rename) is its own pull
request.

## Documentation

What is committed, per repository:

| File | Purpose | Rule |
|---|---|---|
| `README.md` | What the project is, how to use it, current status, where to read more | Describes the product, not its development history. Short for libraries |
| `docs/architecture.md` | How the repository is structured and why, at the level a new maintainer needs | One document; grows only when the structure does |
| `docs/adr/NNNN-title.md` | A decision whose alternatives a future maintainer would ask about | Only when the "why" is not obvious from the code and architecture document. Superseded, never rewritten |
| `CONTRIBUTING.md` | This file, or a link to it | |
| `SECURITY.md` | Reporting channel and the invariants the project is built on | |
| `CHANGELOG.md` | Generated from commits for each published version family | Not hand-maintained |
| `LICENSE` | | |
| `.github/` | CI, release, Dependabot, pull request template | |

Direction is a short list in the README or architecture document, without
dates. There is no separate roadmap file.

Wording is factual. Words like "robust", "comprehensive", "enterprise-grade",
"seamless", and "production-ready" are not used unless the sentence
explains what concretely makes them true.

## What is never committed

Working material for the people and tools writing the code is not part of
the product and stays out of public history:

- plans, task breakdowns, and backlogs
- prompts, agent instructions, and agent configuration
- review notes, handoff documents, status reports, and completion reports
- brainstorming, drafts, and scratch files

Each repository has a `local/` directory for this material, ignored by
`.gitignore`, together with the usual agent configuration directories. The
`.gitignore` in this repository is the reference. Do not force-add ignored
files. If something in `local/` turns out to matter to users or maintainers,
it is rewritten into the README, the architecture document, or an ADR.

## Dependencies

- Versions are pinned in a Gradle version catalog or an npm lockfile.
  Dependabot proposes grouped weekly updates.
- Adding a dependency to a published library needs a sentence in the pull
  request saying why it is worth the consumer's classpath.
- A dependency that changes a documented decision updates the architecture
  document or adds an ADR in the same pull request.

## Tests

- Libraries: unit tests plus tests against the real runner or tool they
  integrate with, across the JDK, Node, and runner versions the README says
  are supported.
- Platform: unit tests plus Testcontainers-backed integration tests; no
  test depends on a shared database.
- Protocol: every binding round-trips the fixture corpus.
- CI verifies that tests ran, not only that the build exited zero.
- A failing test is fixed or deleted, never retried into passing.
