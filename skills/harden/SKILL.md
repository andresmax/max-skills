---
name: harden
description: "Add the E2E test a shipped feature is missing, with its artifact and the command that regenerates it. Scope comes from TODO(harden) markers, /prune's coverage holes and recent features with no E2E. Derives correctness from the spec and proves each test bites by breaking its subject. Use on /harden, 'add the E2E', 'backfill the tests'. Not for unit tests on code that already exists, or for code that never met its real dependency."
---

# /harden — add the E2E a feature shipped without

**The ask (optional):** `$ARGUMENTS` — a path, a feature name, or nothing. With no
argument, harden everything marked or recently shipped without an E2E.

---

## Why this exists

The test policy is E2E first. A feature is verified by driving it the way a user or
client would, and every run leaves an artifact plus the one command that regenerates
it. `/ship` writes that E2E during the build. `/harden` covers what got past it: a
flow `/ship` marked `TODO(harden)` because it couldn't be reached at the time,
features built before the policy, and the coverage holes `/prune` proves by mutation.

`/harden` never writes unit tests for code that already exists. A test written after
its subject tends to assert whatever the code already does, so it passes and catches
nothing. Isolated tests belong to the next change to that code, written as its failure
list before the change.

**The precondition: the feature must have actually run**, against its real
dependencies, with a human having used it. Stubs can't catch a contract the real
dependency rejects: from inside the app a valid event name and an invalid one are both
just strings, and a stub also answers instantly, hiding latency the real call has. If
the feature has not met its real dependency, the answer is "go make it work, then come
back". Say that and stop.

If the code is still in flux, say so and stop. An E2E written against an undecided
design gets rewritten before it catches anything.

When you write the tests, prefer **behaviour observed to break** over behaviour
imagined. A bug that actually happened is worth ten hypotheticals.

---

## Phase 0 — Scope

Work out what to harden, in this order:

1. **Explicit argument** — a path, file, class, or feature name. That's the scope.
2. **`# TODO(harden):` markers** — `grep -rn "TODO(harden)"` across the repo. Each
   marker names a flow without an E2E and why. These are the primary work-list.
3. **Coverage holes from `/prune`** — mutations nothing caught. Each one becomes an
   E2E flow that would have caught it.
4. **Recently shipped, no E2E** — `git log --since="14 days ago" --name-only
   --pretty=format:` for changed app files, cross-referenced against what the E2E
   suite drives.

Then **confirm the scope in one line** and proceed. Do not ask more than once.

If nothing is found, say so plainly, don't invent work, and stop.

---

## Phase 1 — Find the harness, then establish what "correct" is

**The harness.** The repo's `docs/conventions.md` says what E2E means there, the
command, and where artifacts land. If it says nothing, look for `test/system/`,
`spec/system/` or a `playwright.config.*`. If there is no harness, setting one up is
the first job (Rails: Capybara with `capybara-playwright-driver`, a trace saved on
every run), and adding its command and artifact path to `docs/conventions.md` is part
of that job. If the repo's `decisions.md` rules out browser tests, follow it: drive the
flow over HTTP the way the repo's integration tests do, and say in the report that
there is no browser artifact.

**What correct is.** Do not infer intent from the implementation. A test derived from
the code it tests proves only that the code does what it does, and it locks in bugs as
specification. Read, in this order:

1. **The spec** (`docs/prds/`, `docs/specs/`, the issue). Acceptance criteria and use
   cases are the contract.
2. **Class-level comments.** Good codebases explain why a thing is shaped the way it
   is, usually citing a real incident. Those paragraphs are executable intent.
3. **Git history for the feature.** Commit messages say what was decided and what was
   rejected.
4. **The `# TODO(harden)` markers themselves.** They name the specific gaps.

Where implementation and stated intent **disagree**, that is a finding, not a test to
write. Stop and report it. It's either a bug or stale documentation, and a human
decides which.

---

## Phase 2 — Write the E2E

Per flow in scope, drive it from outside and cover, in this order:

1. **The stated contract** — every acceptance criterion the flow touches.
2. **The refusals** — what a user without the right role, state or input must not be
   able to do, tried through the UI, or through a crafted request where that's how
   someone would try it.
3. **Error and absence paths a user can reach** — the third party returned garbage or
   a 500, the record was deleted, the field is empty.
4. **Regressions for anything an adversarial pass found** during the original build.
   Check the spec and commit messages for confirmed findings and assert they stay
   fixed.

Every run leaves its artifact (a trace, screenshots or a result file) at the path the
repo names, else `tmp/e2e/`, and the test file or `docs/conventions.md` gives the one
command that regenerates it. Seeded data and in-process fakes keep it repeatable. A
third party gets a fake only after one live call has proved its contract.

**Tests that must not be written:**
- Any unit test on code that already exists.
- Any test asserting a value the test itself just set.
- `assert_nothing_raised` and its equivalents as a stand-in for coverage.
- Snapshot or golden tests of copy or markup unless the exact string is genuinely the
  contract.

**Every test gets a name that states the behaviour**, not the method. `"refuses a
second owner"` over `"test claim!"`.

Where a test encodes something hard-won (an incident, a trap, a decision), **put the
why in a comment above it**. A test with a reason attached survives a refactor; one
without gets deleted by the next person who finds it inconvenient.

---

## Phase 3 — Prove the tests bite

A test that passes against broken code is worse than no test, because its green tick
says the code works when it doesn't.

For each **non-trivial** test written, temporarily break the code it covers, confirm
the test fails, and restore. Report the failure message you saw. Do this in a scratch
edit you revert, and never commit a broken state.

If a test cannot be made to fail by breaking its subject, it is not testing anything.
Delete it or rewrite it.

---

## Phase 4 — Clear the markers, verify, report

- **Remove each `# TODO(harden):` marker you have actually satisfied.** Leave any you
  haven't and say why.
- Run the full suite, the E2E suite, the linter, and the security scanner. All green.
- **Do NOT commit and do NOT push** unless asked. Leave the tree dirty for review.

**Report:**
- scope hardened, and the marker list before/after
- each E2E written, with its artifact path and the command that regenerates it
- for each significant test, the failure message you saw when you broke its subject
- **any place implementation and stated intent disagreed**, flagged and not silently
  resolved
- anything still without an E2E, and why

---

## Rules

- **Surgical.** `/harden` writes E2E tests and the harness they need. It does not
  refactor, rename, or improve the code it is testing. If it finds a bug, it reports
  it, and fixes it only on an explicit ask, starting with a failing test that
  reproduces it.
- Where an E2E suite already exists, extend it in the house style rather than starting
  a parallel one.
- Never fake the thing under test. Fake at the boundary: the third-party client, the
  clock, the queue.

### Known traps, Rails 8 + Minitest

These cost real hours and none of them fail loudly:

- **Minitest 6 has no `Object#stub`** — `minitest/mock` is unavailable.
- **Never stub `Rails.application.credentials`.** It answers via `method_missing`,
  so the stub silently returns nil, the block never runs, and the test passes having
  asserted nothing.
- **Never name an ivar `@app` in an `ActionDispatch::IntegrationTest`.** Rack builds
  its session from it and every route helper raises a bare `NameError`.
- **Never name a test helper `run`** — it shadows Minitest and hangs the whole suite.
- **Stop a Playwright trace in `before_teardown`, never in a `teardown` block.** Rails
  takes its failure screenshot in `before_teardown`, and that screenshot ends the
  tracing session, so a `teardown` hook finds tracing already gone on exactly the runs
  worth tracing.
- **`capybara-playwright-driver` does not bundle Playwright.** Pin `playwright-core`
  in `package.json` to the gem's `COMPATIBLE_PLAYWRIGHT_VERSION`, and move the two
  together.
