# Quality Engineering ecosystem

Architecture and engineering policy for a set of public projects for
building, running, and observing automated tests. This repository holds no
code. It records the decisions that apply across the ecosystem's
repositories and the policy files those repositories copy.

## Projects

| Project | What it is | Repository | Status |
|---|---|---|---|
| Reporting protocol, adapters, and platform | A runner-agnostic description of test runs, adapters that emit it from JUnit, Playwright, Karate, and others, and a service that stores and displays it | `qe-report` | not started |
| Java QA libraries and BOM | Small test-support libraries for JVM suites and a BOM that pins compatible versions | `qe-java` | not started |
| Reference starters | Copyable projects for Rest Assured, Playwright, Karate, and Selenium, each in that tool's own idiom | `qe-starter-<tool>` | not started |
| TestInbox | Disposable email for automated tests | [testinbox](https://github.com/yannisyoussef/testinbox) | existing |
| KaaS | Karate as a Service | [kaas](https://github.com/yannisyoussef/kaas) | existing |

## Documents

- [Architecture](docs/architecture.md): project kinds, repositories,
  naming, technology baseline, versioning, the reporting protocol
  boundary, security baseline, engineering principles, and what is still
  undecided.
- [Decision records](docs/adr/README.md): the few decisions that needed
  their alternatives written down.
- [Contributing](CONTRIBUTING.md): commit, pull request, and documentation
  conventions used by every repository in the ecosystem.
- [Security](SECURITY.md): how to report a vulnerability and the
  invariants the reporting platform is built on.
