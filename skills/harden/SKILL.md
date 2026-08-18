---
name: harden
description: Write the test suite that was deliberately deferred during the first build, now that the feature has been used and confirmed to behave right. Finds TODO(harden) markers and thinly-covered recent changes, establishes what "correct" means from the spec rather than from the implementation, writes the suite, then proves each test bites by breaking its subject and watching it fail. Use when the user says "harden", "/harden", "write the real tests now", "add proper test coverage", or comes back to a feature after using it. Refuses to run on code that is still in flux or that has never met its real dependency — hardening an undecided design just recreates the problem.
---

# /harden — write the tests you deliberately deferred

**The ask (optional):** `$ARGUMENTS` — a path, a feature name, or nothing. With no
argument, harden everything marked or recently changed.

---

## Why this exists

Test depth splits in two.

**Boundary** code — auth, signatures, SSRF, injection, money, data loss — is fully
tested the moment it's written, because it fails open and doesn't churn.

**Provisional** code — product semantics, states, copy, the shape of an API — gets
one smoke test and a `# TODO(harden):` marker, because writing an exhaustive suite
for behaviour still being *decided* is waste: the design pivots, the tests get
deleted having caught nothing, and worse, they slow the pivot down.

`/harden` is the other half of that bargain. Run it once you have actually used the
thing and confirmed it's right. **The design is now settled, so the tests are now
worth writing.**

If this is run on code that is still in flux, say so and stop. Hardening an
undecided design just recreates the problem.

**And the harder precondition: the feature must have actually run.** Not "the smoke
test passes" — *run*, against its real dependencies, with a human having used it.
`/harden` is where the real suite gets written, so it is also where the waste is
largest if the thing doesn't work yet.

A real case, and the reason this paragraph exists: a project shipped **489 tests,
~1,700 assertions, all green**, over a third-party integration that had never made
a single request to that third party. First real contact failed instantly — the
app manifest asked for event types the vendor does not allow you to subscribe to —
and no stubbed test could have caught it, because from inside the app a real event
name and a made-up one are both just strings. The same stubs answered instantly, so
a discovery pass that took **22 seconds** in production looked free. One live call
would have found both in five minutes, before 3,800 lines and four deploys.

If the feature has not met its real dependency, the answer is not "harden it" — it
is "go make it work, then come back". Say that and stop.

When you do write the suite, prefer tests that encode **behaviour observed to
break** over behaviour imagined. A bug that actually happened is worth ten
hypotheticals, and it cannot be a test that merely agrees with the implementation.

---

## Phase 0 — Scope

Work out what to harden, in this order:

1. **Explicit argument** — a path, file, class, or feature name. That's the scope.
2. **`# TODO(harden):` markers** — `grep -rn "TODO(harden)"` across the repo. Each
   marker names what was deliberately left untested and why. These are the primary
   work-list.
3. **Recently changed, thinly covered** — `git log --since="14 days ago"
   --name-only --pretty=format:` for changed app files, cross-referenced against
   what has tests. Anything substantial with only a smoke test is a candidate.

Then **confirm the scope in one line** and proceed. Do not ask more than once.

If nothing is found: say so plainly, don't invent work, stop.

---

## Phase 1 — Establish what "correct" is

This is the step that makes `/harden` different from "write more tests".

**Do not infer intent from the implementation.** A test derived from the code it
tests proves only that the code does what it does — it locks in bugs as
specification. Instead, read, in this order:

1. **The spec** (`docs/prds/`, `docs/specs/`, the issue) — acceptance criteria and
   use cases are the contract.
2. **Class-level comments** — good codebases explain *why* a thing is shaped the
   way it is, usually citing a real incident. Those paragraphs are executable
   intent.
3. **Git history for the feature** — commit messages say what was decided and what
   was rejected.
4. **The `# TODO(harden)` markers themselves** — they name the specific gaps.

Where implementation and stated intent **disagree**, that is a finding, not a test
to write. Stop and report it. It's either a bug or stale documentation, and a human
decides which.

---

## Phase 2 — Write the suite

Per unit of scope, cover in this order:

1. **The stated contract** — every acceptance criterion that touches this code.
2. **State transitions** — every legal one, and every illegal one refused. State
   machines are where semantic bugs hide.
3. **Error and absence paths** — the network is down, the field is nil, the record
   was deleted, the third party returned garbage, the response was a 500 rather
   than an exception. Absence-of-data paths are the most common gap in a
   provisional suite.
4. **Boundaries and off-by-ones** — empty, one, many, the cap, one past the cap.
5. **Regression tests for anything an adversarial pass found** during the original
   build. Check the spec and commit messages for confirmed findings and assert they
   stay fixed.

**Tests that must not be written:**
- Any test asserting a value the test itself just set.
- `assert_nothing_raised` and its equivalents as a stand-in for coverage.
- Tests that mirror the implementation line by line — they break on every refactor
  and catch nothing.
- Snapshot or golden tests of copy or markup unless the exact string is genuinely
  the contract.

**Every test gets a name that states the behaviour**, not the method. `"refuses a
second owner"` over `"test claim!"`.

Where a test encodes something hard-won — an incident, a trap, a decision — **put
the why in a comment above it**. A test with a reason attached survives a refactor;
one without gets deleted by the next person who finds it inconvenient.

---

## Phase 3 — Prove the tests bite

A test that passes against broken code is worse than no test — it's a false
negative wearing a green tick.

For each **non-trivial** test written: temporarily break the code it covers, confirm
the test fails, restore. Report the failure message you saw. Do this in a scratch
edit you revert — never commit a broken state.

If a test cannot be made to fail by breaking its subject, it is not testing
anything. Delete it or rewrite it.

---

## Phase 4 — Clear the markers, verify, report

- **Remove each `# TODO(harden):` marker you have actually satisfied.** Leave any
  you haven't and say why.
- Run the full suite, the linter, and the security scanner. All green.
- **Do NOT commit and do NOT push** unless asked. Leave the tree dirty for review.

**Report:**
- scope hardened, and the marker list before/after
- test count delta
- for each significant test: the failure message you saw when you broke its subject
- **any place implementation and stated intent disagreed** — flagged, not silently
  resolved
- anything still deliberately untested, and why

---

## Rules

- **Surgical.** `/harden` writes tests. It does not refactor, rename, or improve the
  code it is testing. If it finds a bug, it reports it — and only fixes it on an
  explicit ask.
- Where a test suite already exists for a file, extend it in the house style rather
  than starting a parallel one.
- Never stub the thing under test. Stub at the boundary — the HTTP call, the clock,
  the queue.

### Known traps, Rails 8 + Minitest

These cost real hours and none of them fail loudly:

- **Minitest 6 has no `Object#stub`** — `minitest/mock` is unavailable.
- **Never stub `Rails.application.credentials`.** It answers via `method_missing`,
  so the stub silently returns nil, the block never runs, and the test passes having
  asserted nothing.
- **Never name an ivar `@app` in an `ActionDispatch::IntegrationTest`.** Rack builds
  its session from it and every route helper raises a bare `NameError`.
- **Never name a test helper `run`** — it shadows Minitest and hangs the whole suite.
