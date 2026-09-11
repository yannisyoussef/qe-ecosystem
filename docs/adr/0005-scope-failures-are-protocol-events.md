# ADR-0005: Non-test hierarchy failures are first-class protocol events and do not alter child test verdicts

Status: Accepted
Date: 2026-09-11

## Context

The first real adapter (JUnit Platform) reproduced a case the protocol's
line 0.1 could not represent: a class whose tests all ran and kept their
own verdicts, and whose `@AfterAll` then threw. JUnit reports the class
container as failed and the build fails; no test carries the failure.
pytest module- and class-scoped teardown errors and Cypress suite `after`
hook failures have the same shape. Line 0.1 knew attempts, steps inside
attempts, and sessions; a failure that belongs to none of them had nowhere
to go, and an adapter could only print it.

## Decision

A failure of a non-test node of the runner hierarchy, a scope, is a
protocol event of its own, `scope.failed`, carrying the scope's path in
the same typed segments a test path uses, the failure list with its phase,
and optional display name, raw status, and location. It contributes to the
execution's failed outcome and never rewrites a child attempt's verdict:
`testA = passed`, `testB = passed`, one teardown scope failure, execution
failed. It is session-scoped, may occur before or after the child attempts,
and has no identity of its own beyond the event id and no history.

Because a consumer that ignored the event would derive a passed run from
a failed one, it is not an ignorable addition. Under ADR-0002 such a change
is a major; before 1.0 the compatibility unit is `0.minor`, so the major
bump is applied as a new line, 0.2. Line 0.1 was never published, so it
lives in the repository history only and nothing is maintained for it.

## Options considered

1. Attribute the failure to a child test (Playwright's behaviour for
   `afterAll`). False for JUnit and pytest, where the tests demonstrably
   finished with their own verdicts. Rejected as the canonical model;
   adapters for runners that themselves attribute this way need no event.
2. Synthesise a test attempt named after the container. Distorts test
   counts and history with a test that does not exist. Rejected.
3. Print the failure and record nothing. The run reads as green while the
   build failed. Rejected; this was the interim state.
4. A container lifecycle (`scope.started`, `scope.finished`, statuses,
   ids, history). Far more than the evidence needs. Rejected.
5. One event for the observed failure (chosen).

## Consequences

- Read models derive a run's verdict from unexpected test outcomes or scope
  failures; no persisted run status field is added.
- Attachments stay attached to attempts; container-level report entries
  remain an adapter limitation.
- Line 0.2 is the current line; bindings, validator, SDKs, fixtures, and
  documentation moved to it in one step.

## Deferred

- Whether a scope failure in `setup` should coexist with synthesised
  failed attempts for prevented tests (the JUnit `@BeforeAll` mapping stays
  as decided in QE-002 until evidence shows a distinct need).
- Attachments on scopes.
