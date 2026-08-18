---
name: ship
description: Autopilot feature pipeline — brief, research, spec (your green light), build, verify, land on a branch. One invocation runs the whole thing with exactly two human touchpoints, the brief up front and the spec approval before any code. Auto-detects the repo's stack, test command and reviewers; works in any repo. Use when the user says "/ship", "ship this feature", "build me X end to end", "run the pipeline", or hands over a feature to be built unattended. Stops at a feature branch and stops there — never pushes, never opens a PR, never merges, never deploys.
---

# /ship — brief, spec, build, verify, branch

Autopilot feature development. One invocation runs the whole pipeline with exactly
**two** human touchpoints: the **brief** (asked once, up front) and the **spec green
light** (mandatory). Everything else runs unattended and ends with the work
committed to its own branch — **no push, no PR, no merge, no deploy.**

**The ask:** `$ARGUMENTS`

---

## Non-negotiables

1. **Ask once, then go quiet.** All input is front-loaded into the brief. After
   that, the only stop is the spec green light.
2. **The spec needs approval before any code.** Always. No exceptions, any repo.
3. **Verify-gated.** Nothing advances past a red gate. The pipeline saves typing,
   not thinking.
4. **Surgical.** Every changed line traces to the brief. No adjacent refactors,
   restyles, or "while I was here" edits.
5. **Lands on a branch and stops.** The human pushes. Never run `git push`, open a
   PR, merge, or deploy.

---

## Setup: detect repo + config

`/ship` runs in **any** repo. Auto-detection comes first; a config file is an
optional *override* layer, never a gate. A repo not being in that file is normal —
never treat it as an error or "unknown profile."

1. **Repo root:** `git rev-parse --show-toplevel`; basename = repo key.
2. **Auto-detect the stack (always):**
   - `Gemfile` + `test/` → `tests = bin/rails test`; `Gemfile` + `spec/` →
     `bundle exec rspec`
   - `package.json` → read its `scripts.test` and use that (ignore the placeholder
     "no test specified")
   - `pyproject.toml` / `pytest.ini` → `pytest`; `go.mod` → `go test ./...`
   - none of the above → ask for the test command in the brief
   - **reviewers:** always a general `code-reviewer`; add any domain reviewer that
     exists in the repo's `.claude/agents/`.
   - **tracker:** if a connected MCP server shares the repo's name, use it; else
     `none`.
   - **profile:** `client` if this is someone else's codebase, else `product`. The
     difference is latitude — `client` means conservative defaults, no opinionated
     restyling, and a higher bar for touching anything outside the brief. If you
     can't tell, it's `client`.
3. **Apply overrides:** if `~/.claude/ship-profiles.json` exists and has this repo's
   key, let its `tracker` / `tests` / `reviewers` / `profile` override the detected
   values. That's all the file does:
   ```json
   {
     "_comment": "OPTIONAL. /ship auto-detects everything; entries here only override that. A repo not listed still works.",
     "my-app": { "profile": "product", "tracker": "linear", "tests": "bin/rails test", "reviewers": ["code-reviewer"] }
   }
   ```
4. State the resolved config in one line (repo · profile · tracker · tests) and
   proceed. No "this repo isn't registered" friction.

---

## Phase 0 — BRIEF (the only up-front ask)

Use **one** `AskUserQuestion` call with up to 4 questions. Do not ask anything else
until the spec gate.

- **Scope** — what's explicitly in, what's explicitly out
- **Decisions to own** — anything the user wants to decide themselves (pricing, a
  UX call, a data rule). Everything else is yours to decide from research.
- **Autonomy** — `Plan only` (stop after the spec) or `Build` (run through to the
  branch after approval)
- **Constraints** — perf, deadline, "don't touch X", or the test command if the
  repo was unknown

Record the answers as a `## Brief` block at the top of the spec. Then go quiet.

---

## Phase 1 — RESEARCH (unattended)

Fan out parallel subagents; do NOT write the spec until these return:

- **Explore agent** — existing patterns, conventions, naming, where this plugs in
- **Regression scan** — what could break (affected flows, N+1s, shared partials or
  components)
- **Library docs** — current documentation for any library or API involved. Never
  rely on training data for library specifics.
- **Web search** — best practices and known gotchas for the specific approach
- **Design-system read** — the repo's `DESIGN.md` / design tokens, so "fits the
  product" is concrete rather than vibes

Only interrupt for a genuine fork the brief didn't cover.

---

## Phase 2 — SPEC → ⛔ GREEN LIGHT GATE

Write **one self-contained spec** to `docs/prds/PRD-XXX-<name>.md` in the target
repo (or wherever the repo already keeps specs — match its convention). Determine
`XXX` by scanning the existing folder and incrementing; create it if absent.

The spec must contain:

- `## Brief` (from Phase 0)
- `## Research & Context` (Phase 1 findings — the research lives *here*, not in a
  separate file)
- `## Problem & Use Cases` — all use cases considered, including edge cases
- `## UI/UX` — interaction, states, empty/error/loading, components used
- `## Data & Migration` — schema changes, migration, **rollback path**
- `## Fits-the-product checklist` — reuses existing components, follows naming, no
  duplicated logic, regression notes
- `## Phases` — functional-first; **every phase ships something testable**; no
  back-to-back plumbing. Each phase: objective, tasks, success criteria, files
  likely affected, recommended agents, and **test depth: `boundary` or
  `provisional`** (see below)
- `## Acceptance criteria` — concrete, checkable, must-pass
- `## Test plan` — split into **Boundary (now)** and **Deferred**

### Test depth — declare it per phase, in the spec

> **The rule that overrides everything below: a feature gets no test suite until
> its happy path has run for real, once.**
>
> Tests written before the thing has ever run do not test the product. They test
> that your assumptions are internally consistent, and then report that back as
> confidence. A stub agrees with whatever you already believe.
>
> This is not theoretical. A project shipped **489 tests, ~1,700 assertions, all
> green**, over a third-party integration that had never made one request to that
> third party. First real contact failed instantly — the app manifest asked for
> event types the vendor does not allow you to subscribe to — and no test could
> have caught it, because from inside the app a real event name and a made-up one
> are both just strings. The same stubs answered instantly, so a discovery pass
> that took **22 seconds** in production looked free. One live call would have
> found both in five minutes, before 3,800 lines and four deploys.
>
> So: **build it, run it against the real dependency, then test.** For anything
> touching an external API, the first verification is one live call — not a stub,
> not a fixture. Until that has happened, `boundary` does not license a suite
> either; it licenses *one* test once the path works.
>
> And never report "tests green / all gates passed" as evidence a feature works
> when it has never met its real dependency. Say plainly that it hasn't.

Writing an exhaustive suite for behaviour still being *decided* is waste: the design
pivots and the tests get deleted, having caught nothing. Worse, they slow the pivot
down. So every phase declares one of two depths.

**`boundary` — full tests immediately, no exceptions.** Anything that fails *open*,
is expensive to get wrong, and does not churn when the product changes its mind:

- authn/authz, session handling, anything deciding who may see or do what
- signature/HMAC verification, token validation, CSRF, SSRF guards
- input trust boundaries, injection surfaces, output escaping of third-party data
- money, billing, quotas
- data loss: destructive migrations, deletes, cascade behaviour
- idempotency and replay guards on anything accepting outside traffic

**`provisional` — one happy-path smoke test, and stop.** Product semantics genuinely
still being decided: what states exist, what a card says, what a job does when it
can't decide, copy, layout, the shape of a model's API. For these, write:

- one test proving the thing runs and the happy path works
- a `# TODO(harden): <what is deliberately untested and why>` marker at the top of
  the relevant file(s)

and **do not enumerate branches, edge cases or error paths yet.**

Default to `provisional` for anything in the first build of a new feature. Promote
to `boundary` only for the list above. When unsure, it's `provisional`.

**Then STOP.** Present a tight summary + the spec path and call `AskUserQuestion`:

- **Approve** → continue to BUILD
- **Revise** → take the notes, update the spec, ask again
- **Plan only / stop** → leave the spec, end the run

If autonomy was `Plan only`, stop here regardless.

---

## Phase 3 — BUILD (unattended)

Execute the spec phase by phase.

- Create the feature branch first: `build/<name>` (or `feat/<ticket>-<name>` if
  there's a tracker ticket). Work only on this branch.
- Per phase, dispatch the recommended agents. Use **worktree isolation** for any
  agents doing parallel file edits that could collide.
- **Surgical fence:** agents may only change what the phase requires. No
  opportunistic refactors.
- Each phase must hit its success criteria before the next begins. Functional-first.

---

## Phase 4 — VERIFY (the ladder, unattended)

Run in order. Any red → stop, report what failed, don't proceed:

0. **Does it actually run?** — the gate that matters most and the one most ladders
   are missing entirely. Exercise the feature's happy path against the **real**
   dependency: a live call to the third-party API, the actual page rendered in a
   browser, the real binary invoked. Not a stub, not a fixture, not a test. If it
   cannot be reached without a human (a credential only they have, a click only
   they can make), **stop and say so** rather than counting the remaining gates as
   proof — every other rung is a self-check that can pass on something that has
   never worked.
1. **Acceptance criteria** — self-check each item from the spec.
2. **Test suite green** — run the resolved `tests`; fix surgically; re-run until
   green. Green means *the tests that exist* pass; a `provisional` phase is not a
   red gate for having thin coverage — that is the point of it. Gate 0 must have
   passed first: a green suite over code that has never run is not evidence and
   must never be reported as though it were.
3. **Adversarial review** — a skeptic subagent tries to *break* it (edge cases,
   regressions, security). It defaults to "not done" and must be argued down. **This
   is the real safety net during a first build**, and it does not care whether tests
   exist — it reads the code and attacks it. Findings it confirms get a regression
   test regardless of the phase's declared depth: a bug that was real once is no
   longer a semantic still being decided.
4. **Domain reviewers** — always `code-reviewer`; plus the profile's reviewers; run
   a security review if the work touches auth, payments, or data; run a design audit
   if it touches UI.

CI is intentionally **not** in this ladder — CI needs a push, and push is manual.

---

## Phase 5 — HANDOFF (lands on a branch, stops)

- Commit the work to the feature branch with a clear message.
- **Do not push. Do not open a PR. Do not merge.** Full stop.
- Output: branch name, files changed, the verify report, and the exact commands to
  push + open a PR when the human decides.

---

## Phase 6 — LOG (close the loop, unattended)

- **Tracker:** if the resolved `tracker` ≠ none, update the ticket to "in review /
  branch ready" with a short non-technical "what was built / decisions made"
  comment. Create a ticket only if the brief implied one.
- **Repo:** record the branch in the spec's metadata block (`branch: build/<name>`).
  That's the whole log — repo-local, so it survives a machine that has none of your
  other tooling.
- **Do NOT** stamp the spec `implemented:` or touch `CHANGELOG.md` — those are the
  human's calls when they actually merge. A landed branch is not a shipped feature.

---

## Final report

End with: spec path · branch name · verify ladder results (✅/❌ per gate) ·
tracker updates made · the push+PR commands. Then stop.

**Also list the `provisional` phases and what they deliberately left untested**, in
one short block, ending with: *"Run `/harden` once you've used this and it behaves
right."* Never present thin coverage as an oversight — it was a decision, and naming
it is what makes it one.
