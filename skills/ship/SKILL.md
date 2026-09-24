---
name: ship
description: "For client repos; product repos skip specs and branches. Autopilot feature pipeline: brief, research, spec (Max's green light), build, verify, land on a feature branch. Two human touchpoints only, the brief and the spec approval. Auto-detects stack, test and E2E commands, and reviewers in any repo. Use on /ship, 'ship this feature', 'build X end to end'. Never pushes, opens a PR, merges or deploys."
---

# /ship — brief, spec, build, verify, branch

Autopilot feature development. One invocation runs the whole pipeline with exactly
**two** human touchpoints: the **brief** (asked once, up front) and the **spec green
light** (mandatory). Everything else runs unattended and ends with the work
committed to its own branch — **no push, no PR, no merge, no deploy.**

**The ask:** `$ARGUMENTS`

---

## Non-negotiables

1. **Ask once, then run without further questions.** All input is front-loaded into the brief. After
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
   - **e2e:** the repo's `docs/conventions.md` if it names the E2E command; else
     `test/system/` with files → `bin/rails test:system` (plain `bin/rails test`
     skips system tests); `spec/system/` → `bundle exec rspec spec/system`; a
     `playwright.config.*` → its `package.json` script, else `npx playwright test`;
     else `none`, and the spec sets the harness up (see Tests below)
   - **reviewers:** a general code review (the repo's `code-reviewer` agent if
     `.claude/agents/` has one, else a general-purpose subagent briefed to review the
     diff); add any domain reviewer that exists in `.claude/agents/`.
   - **tracker:** if a connected MCP server shares the repo's name, use it; else
     `none`.
   - **profile:** `client` if this is someone else's codebase, else `product`. The
     difference is latitude — `client` means conservative defaults, no opinionated
     restyling, and a higher bar for touching anything outside the brief. If you
     can't tell, it's `client`.
3. **Apply overrides:** if `~/.claude/ship-profiles.json` exists and has this repo's
   key, let its `tracker` / `tests` / `e2e` / `reviewers` / `profile` override the detected
   values. That's all the file does:
   ```json
   {
     "_comment": "OPTIONAL. /ship auto-detects everything; entries here only override that. A repo not listed still works.",
     "my-app": { "profile": "product", "tracker": "linear", "tests": "bin/rails test", "reviewers": ["code-reviewer"] }
   }
   ```
4. State the resolved config in one line (repo · profile · tracker · tests · e2e) and
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

Record the answers as a `## Brief` block at the top of the spec. Then run without further questions until the spec gate.

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
- `## Phases` — functional-first; **every phase ships something an E2E test can
  drive**; no back-to-back plumbing. Each phase: objective, tasks, success
  criteria, files likely affected, recommended agents, its **E2E flows**, and a
  **failure list** if it has code on the isolated list (see below)
- `## Acceptance criteria` — concrete, checkable, must-pass
- `## Test plan` — the E2E flows, each tied to the acceptance criteria it proves
  and the artifact it leaves, then the failure lists with the reason each unit
  needs isolation

### Tests — E2E by default, declared per phase in the spec

> **The rule that overrides everything below: before any fake exists, the feature
> meets its real dependency once.** For anything touching an external API, the
> first verification is one live call — not a stub, not a fixture.
>
> Stubs can't catch a contract the real dependency rejects: from inside the app a valid
> event name and an invalid one are both just strings, and a stub also answers instantly,
> hiding latency the real call has. One live call finds both kinds of failure in minutes.
>
> And never report "tests green / all gates passed" as evidence a feature works
> when it has never met its real dependency. Say plainly that it hasn't.

**No unit tests written after the code.** A test written after its subject tends to
assert whatever the code already does, so it passes and catches nothing.

**E2E is the default test, and usually the only one.** Each phase names the flows its
E2E tests drive the way a user or client would: a browser for UI, HTTP for APIs and
MCP, the real binary for a CLI. One E2E per flow in the acceptance criteria. Every run
leaves an artifact (a trace, screenshots or a result file) at the path the repo's
conventions name, else `tmp/e2e/`, plus the one command that regenerates it. Seeded
data and in-process fakes keep it repeatable, after the live call above has proved
each third party's contract.

If `e2e` resolved to `none`, the spec's first phase sets up the harness (Rails:
Capybara with `capybara-playwright-driver`, a trace saved on every run) and adds its
command and artifact path to the repo's `docs/conventions.md`. If the repo's
`decisions.md` rules out browser tests, follow the repo: drive the flows over HTTP the
way its integration tests do, and say in the final report that there is no browser
artifact.

**Isolated tests need a reason, and they are written before the code.** Only where
E2E can't reach the failure cheaply:

- authn/authz, session handling, anything deciding who may see or do what
- signature/HMAC verification, token validation, CSRF, SSRF guards
- input trust boundaries, injection surfaces, output escaping of third-party data
- money, billing, quotas
- data loss: destructive migrations, deletes, cascade behaviour
- idempotency and replay guards on anything accepting outside traffic
- pure logic with many edge cases, like a parser or a matcher

For these the phase carries a **failure list**: every way the unit can fail. In BUILD
the list becomes tests first, each one runs red, and then the code gets written.
Anything outside this list gets its E2E and nothing else.

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
- A phase with a failure list writes those tests first and runs them red before
  writing the code they cover. E2E tests get written alongside the flow they drive.
- Before writing a fake for a third party, make the one live call that proves its
  contract and record what it returned in the spec. The fake mirrors that response.

---

## Phase 4 — VERIFY (the ladder, unattended)

Run in order. Any red → stop, report what failed, don't proceed:

0. **Does it actually run?** — the gate that matters most and the one most ladders
   are missing entirely. First, confirm every third party the feature touches got
   its live call during BUILD, and make any call a phase skipped. Then the phase's
   E2E tests drive the feature the
   way a user or client would and leave their artifacts; record each artifact's path
   and the command that regenerates it. If a flow cannot be reached without a human
   (a credential only they have, a click only they can make), **stop and say so**
   rather than counting the remaining gates as proof, and mark it
   `# TODO(harden): <flow> has no E2E because <reason>`. Every other rung is a
   self-check that can pass on something that has never worked.
1. **Acceptance criteria** — each item is proved by an E2E assertion or checked by
   hand, and the report says which.
2. **Full suite green** — run the resolved `tests` and `e2e`; fix surgically; re-run
   until green. Gate 0 must have passed first: a green suite over code that has never
   run is not evidence and must never be reported as though it were.
3. **Adversarial review** — a skeptic subagent tries to *break* it (edge cases,
   regressions, security). It defaults to "not done" and must be argued down. **This
   is the real safety net during a first build**, and it does not care whether tests
   exist — it reads the code and attacks it. Each finding it confirms gets a failing
   test that reproduces it first (E2E where the bug is reachable from outside,
   isolated otherwise), then the fix.
4. **Domain reviewers** — always the general code review from Setup; plus the
   profile's reviewers; run a security review if the work touches auth, payments,
   or data; run a design audit if it touches UI.

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

End with: spec path · branch name · verify ladder results (✅/❌ per gate) · each
E2E artifact's path and the command that regenerates it · tracker updates made ·
the push+PR commands. Then stop.

**Also list any flow left without an E2E, and why**, in one short block, ending
with: *"Run `/harden` once it can be reached."* Never present a missing E2E as an
oversight; it was a decision, and naming it is what makes it one.
