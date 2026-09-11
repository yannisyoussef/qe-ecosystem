# ADR-0006: Runner-level outcomes are first-class session facts and the run verdict stays derived

Status: Accepted
Date: 2026-09-11

## Context

The second real adapter (the Playwright reporter) reproduced four cases the
protocol's line 0.2 could not represent. Playwright's `onEnd` reports an
authoritative status for the whole invocation, `passed`, `failed`,
`timedout`, or `interrupted`, and the invocation exits accordingly, while
no attempt and no hierarchy scope explains it: a global setup exception
(no test ran), a global teardown exception (every test passed), a global
timeout (the running test never finished), an interruption (the running
test ended without a verdict). Under line 0.2 each of these derived as a
passed run. A runner policy that fails an invocation for a flaky test has
the same shape: every attempt is accounted for, the last one passed as
expected, and the runner still failed. JUnit's launcher, pytest's session
hooks, Cypress plugin errors, and Karate configuration errors are the
analogues; none of them belongs to a test or to a node of the hierarchy.

## Decision

The aggregate outcome of a runner invocation is a fact of its own, carried
by the event that already closes the session. `session.finished` gains
three optional fields: a canonical `status` from a dedicated set
(`passed`, `failed`, `inconclusive`), the runner's own word in
`rawStatus`, and `failures` for errors of the invocation itself, in the
ordinary failure shape. A producer whose runner exposes an authoritative
aggregate outcome emits `status`; one whose runner does not (the JUnit
Platform adapter, whose forked JVM knows nothing of the build's verdict)
emits the empty payload, and consumers derive the outcome from attempt and
scope facts alone. `rawStatus` requires `status` and never enters
derivation, `failures` require `status`, a failed session may carry none,
and a passed session carries none.

Three layers stay distinct and none replaces another: an attempt's outcome
(one test execution), a scope failure (a real node of the hierarchy), and
the session outcome (the invocation as the runner saw it). The run verdict
remains derived and never persisted, now from four values with a fixed
precedence: `incomplete` while any session, attempt, or step is
structurally open; otherwise `failed` on any unexpected final test
outcome, any scope failure, or any failed session; otherwise
`inconclusive` on any inconclusive final attempt or any inconclusive
session; otherwise `passed`. A session's `passed` never erases an
unexpected attempt or a scope failure, and a properly closed interrupted
run is inconclusive, not incomplete.

Because a line 0.2 consumer that ignored the new fields would derive a
passed run from a failed one, this is a new compatibility line, 0.3. Line
0.2 was never published and lives in the repository history only; there is
no parallel runtime support, migration tooling, or dual schema.

## Options considered

1. Session failures only. A timeout, an interruption, or a policy has no
   exception to attach, so a failed invocation could still read as passed.
   Rejected as the sole mechanism; kept as the optional `failures` list.
2. The raw runner status only. A consumer would need to know that
   Playwright's `timedout` means failed and `interrupted` means
   inconclusive, and every future runner word would extend that knowledge.
   Rejected; kept as `rawStatus` for display.
3. A synthesised attempt for the invocation. A test that does not exist,
   distorting counts and history. Rejected.
4. A synthesised scope. The invocation is not a node of the hierarchy and
   has no truthful path; a spec file that fails to load is a real scope
   and is already a `scope.failed`. Rejected.
5. A persisted run status on `run.finished`. Only a coordinator may emit
   that event, most runs have none, and a stored status would compete with
   the facts it summarises. Rejected.
6. Treating an interrupted completion as structurally incomplete. The
   lifecycle did finish correctly; conflating "the producer crashed" with
   "the runner stopped on purpose" would hide which one happened.
   Rejected; `inconclusive` is the verdict for the latter.
7. Optional outcome fields on the closing session event (chosen).

## Consequences

- Read models derive a four-valued verdict from attempts, scope failures,
  and session outcomes; no run status field exists anywhere.
- Adapters for runners with an aggregate outcome map it to the canonical
  status; the Playwright reporter maps `timedout` to `failed` with the raw
  word kept, and `interrupted` to `inconclusive`.
- The JUnit Platform adapter is unchanged in semantics and moves to line
  0.3 through the SDK.
- Line 0.3 is the current line; bindings, validator, SDKs, fixtures, and
  documentation moved to it in one step.
