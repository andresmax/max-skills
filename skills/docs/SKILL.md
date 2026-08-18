---
name: docs
description: Write or refresh a repo's self-contained agent context — a thin AGENTS.md router (hard-capped at 120 lines) plus a docs/ library it points into, with CLAUDE.md as a symlink so every toolchain reads one file. Also keeps the two files that go stale on their own current — CHANGELOG.md derived from merges on the default branch, backlog.md collected from TODO(harden) markers and unresolved spec questions. Four modes — bare "/docs" inits or refreshes, "/docs check" audits read-only and writes nothing, "/docs changelog" catches up just the changelog, "/docs migrate <path>" lifts execution context out of an external notes file into the repo. Use when the user says "/docs", "run docs", "set up agent context", "make this repo self-contained", "the AGENTS.md is stale", "update the changelog", "why is this session so slow to load", or starts a new repo that needs context. The point is that a teammate, another machine, or a cold agent can work the repo correctly with nothing but the repo — so it never writes an outside path into a repo, never puts a credential value in a file, never invents a roadmap, and never commits or pushes.
---

# /docs — make the repo explain itself, cheaply

A repo should tell an agent everything it needs to build correctly, and it should
do it in about a second. Most repos fail at both ends at once.

**They're coupled to something the reader doesn't have.** A context file that opens
with *"read my notes app / the wiki / that Google Doc for current status"* is a
repo that doesn't work on a new laptop and doesn't work for a teammate. It also
isn't cheap: a project-notes file that has accumulated for a year is routinely
100KB+ — call it 25k tokens, paid every session, mostly meeting notes and
invoices, to reach maybe half a file of actual conventions.

**And where `AGENTS.md` and `CLAUDE.md` both exist, they have almost always
rotted.** They get forked once and never linked, so they drift silently: one names
a dependency version the other contradicts, one points at a path that no longer
exists. The failure is quiet, because whichever tool you're using reads only its
own file and looks fine.

So this skill does three things: **collapse the two files into one**, **cut the
outside dependency**, and **hold a hard budget** so the file that loads every
session stays small enough that nobody notices it.

`$ARGUMENTS`: `check` | `changelog` | `migrate <path>` | *(empty)*. Empty means
init-or-refresh — work out which by looking. Anything else, say what you're
assuming and proceed.

---

## The shape it produces

```
AGENTS.md               ← the router. ≤120 lines. The only always-loaded file.
CLAUDE.md -> AGENTS.md  ← symlink, so every tool reads one file
docs/
├── architecture.md     ← the mental model, and invariants WITH their reasoning
├── conventions.md      ← code style, test policy, naming, the traps
├── design.md           ← design system, or a route to an existing root DESIGN.md
├── deploy.md           ← hosts, envs, runbook, and where the secrets live
├── decisions.md        ← ADR log, newest first, a paragraph each
├── backlog.md          ← what's next, or "the issue tracker is the backlog"
├── CHANGELOG.md        ← what shipped
└── prds/               ← specs, if the project works that way
```

**The router answers "what will I get wrong if nobody tells me?"** The library
answers "how does X work?" Anything only needed *while doing a specific thing* is
library, not router. When in doubt it goes down, not up.

### The routing table is the whole trick

Every row carries a trigger, so an agent knows when **not** to open a file. A table
of contents makes it read everything; a table of triggers makes it read one.

```markdown
## Where things live

| Read this | When |
|---|---|
| `docs/architecture.md` | Before changing matching, jobs, or the data model |
| `docs/design.md` | Before touching any view, partial, CSS, or JS controller |
| `docs/deploy.md` | Deploying, adding an env var, or chasing a prod-only failure |
| `docs/decisions.md` | A rule looks arbitrary, or you're about to relitigate one |
| `docs/CHANGELOG.md` | Asking what shipped recently, or when something changed |
| `docs/backlog.md` | Picking up work, or wondering what's deliberately unfinished |
| `docs/prds/` | Implementing a spec — the spec is a complete instruction |
```

---

## The budget — measured, not hoped for

| Rule | Limit |
|---|---|
| `AGENTS.md` | **120 lines / ~6KB** (≈1.5k tokens, bytes ÷ 4). Anthropic's own guidance is "under 200 lines"; this is stricter on purpose |
| Any file behind a routing row | ~500 lines; longer ones get an archive split |
| Every `docs/` file | opens with one `> Purpose:` line, so `head -3` decides |
| Outside paths in `AGENTS.md` | **zero** |

Over budget is a **failure state**, not a warning. The fix is always to move
content down into `docs/` and add a routing row — never to thin it into vagueness.
A router that says "follow the existing patterns" costs tokens and teaches nothing.

The usual offender is `CHANGELOG.md`. Any changelog that has been appended to for
a year is six figures of bytes and has never been read end to end. Cap the live
file at ~15 entries and roll the rest into `docs/changelog-archive.md`.

---

## Written to be shared

Assume a teammate, a client engineer, or a stranger reads this. Same rule for
every repo, no per-repo profile:

- **No references to anything outside the repo.** Not a path into a notes app,
  not "see the project file", nothing that only resolves on one machine.
- **No business context.** No pricing, no deal history, no invoicing, no
  headcount, no internal company framing.
- **The map, not the keys.** `docs/deploy.md` says
  `object storage credentials → 1Password: <vault> > <item>`. It never says the
  value. If you find a live credential already committed in a context file, stop
  and say so; don't quietly relocate it.

---

## Phase 0 — Survey (read-only, every mode)

```bash
git rev-parse --show-toplevel
ls -la AGENTS.md CLAUDE.md 2>/dev/null          # -la shows a symlink as a symlink
wc -lc AGENTS.md CLAUDE.md 2>/dev/null
ls -la docs/ bin/ 2>/dev/null
ls Gemfile package.json pyproject.toml go.mod compose.yaml Dockerfile .env.example 2>/dev/null
git log --oneline -20
```

Also look for a root `DESIGN.md`, `SPEC.md`, design-tool sidecars, and any
`.claude/` agents, skills or settings.

**Never open these.** Glob and `stat` them if you need to know they exist:

| Path | Why |
|---|---|
| `.claude/worktrees/`, `.git/` | thousands of files, zero context value |
| `docs/CHANGELOG.md` | routinely 100KB+ — `head -60` if you must |
| Vendored deps, build output, lockfiles | `node_modules/`, `vendor/`, `dist/`, `target/` |
| Anything over ~200KB in `docs/` | it's a data dump somebody parked there |
| Deploy artifacts that contain frozen copies of context files | they look authored and aren't |

A quick way to find the last two before you read anything:

```bash
find docs -type f -size +200k -exec ls -lh {} \;
```

---

## Phase 1 — Reconcile the two files

Skip if only one exists.

Diff `AGENTS.md` against `CLAUDE.md` and take the **union**. On conflict, prefer
the value you can **verify against the repo right now** over the one with the
newer mtime. A stale fork is often *newer* than the facts it gets wrong — someone
ran a find-and-replace over it last month without checking whether the paths still
resolved. Check the claim, don't date it.

Every line you drop gets named in the report. Silent loss is the one unrecoverable
failure here — these files hold decisions nobody wrote down twice.

---

## Phase 2 — Derive, don't invent

Read the answer out of the repo wherever the repo knows it:

| Fact | Source |
|---|---|
| Stack + versions | `Gemfile.lock`, `package.json`, `pyproject.toml`, `go.mod` |
| Commands | `bin/`, `Makefile`, `Procfile.dev`, `package.json` scripts |
| Deploy | `bin/deploy*`, `compose.yaml`, `.github/workflows/`, host config |
| Env vars | `.env.example`, plus a grep for `ENV[`/`process.env`/`os.environ` |
| Design system | root `DESIGN.md`, the stylesheet's token block |
| Tracker | issue-tracker config, MCP server names in `.claude/settings*.json` |

**Invariants are the valuable part and they are not derivable.** An agent can read
that a method exists; it cannot read that it is *the* seam every state change must
go through and a second write path is forbidden. Harvest those from existing prose
and carry the reasoning with the rule. A rule with its why survives a refactor; a
bare rule gets "improved" away by the next person who finds it inconvenient.

What genuinely can't be inferred — a host, a gotcha, why a decision went the way it
did — goes into **one** `AskUserQuestion` call, four questions maximum. Then go
quiet and write.

---

## Phase 2b — Refresh the two files that go stale on their own

`architecture.md`, `conventions.md` and `design.md` change when someone decides
something. **`CHANGELOG.md` and `backlog.md` go out of date on their own, every
week, without anyone touching them.** So every run refreshes them from evidence.

Both are derived. Neither is invented. If the evidence isn't there, the file
doesn't get written — see the empty-file rule below.

### `docs/CHANGELOG.md` — what actually shipped

**Only merged work counts.** Read the default branch, not the branch you're
standing on. A feature sitting on `build/thing` has not shipped, and writing it
into a changelog is a claim that will be wrong for as long as that branch lives.

```bash
DEF=$(git symbolic-ref --short refs/remotes/origin/HEAD 2>/dev/null | sed 's|origin/||')
DEF=${DEF:-main}
LAST=$(grep -m1 -oE '\(([0-9a-f]{7,40})\)' docs/CHANGELOG.md 2>/dev/null | tr -d '()')
git log --first-parent --no-merges ${LAST:+$LAST..}"origin/$DEF" \
  --pretty=format:'%h%x09%ad%x09%s' --date=short
```

`--first-parent` keeps it to what landed on the branch rather than every commit
inside every merged feature. The last recorded sha is the watermark — everything
since it is new, and nothing before it gets rewritten.

**First run on an existing repo: do not backfill the whole history.** A project
that has been going a while has hundreds of commits, and turning all of them into
a changelog produces a file that breaks the size cap on creation and that nobody
will ever read. Cover the **most recent ~15 dated groups** and stop. Then say
plainly what you skipped:

> Changelog started at 2026-08-13 (~15 entries). 140 earlier commits are in git
> history and were not backfilled.

Silently truncating is the failure — a changelog that starts mid-project without
saying so reads as "nothing happened before this", which is worse than not having
one. The line above costs nothing and makes the file honest.

Then, per entry:

- **One line per user-visible change**, not per commit. Five commits that built one
  feature are one line. Two unrelated fixes are two lines.
- **Written for a person, not a reviewer.** What someone can now do, or what
  stopped being broken. No file names, no class names, no framework names.
- **Group by date**, newest first, and keep the sha in parentheses so the next run
  knows where to resume.
- **Internal-only work a user cannot perceive** — refactors, test coverage, CI
  config — is left out entirely rather than dressed up.

```markdown
> Purpose: what shipped, newest first. Merged work only.

## 2026-08-18
- Actors can set the hours they're available, and matching respects them (45709e6)
- Dragging a card into an occupied column stops failing (a1b2c3d)
```

**Respect what a human wrote.** If an entry already exists for a sha, leave it
exactly as it is — a person's sentence about their own work beats a derived one.
Only append what has no entry yet.

**Then apply the size cap** (below): keep ~15 dated entries live, roll older ones
into `docs/changelog-archive.md` verbatim, newest-first.

### `docs/backlog.md` — what's known to be left

Collected, never invented. **Do not write a roadmap.** Guessing what a project
should do next and putting it in the repo is the worst failure this skill has,
because the next agent will read it as a decision somebody made.

Three sources, all verifiable:

```bash
grep -rn "TODO(harden)\|TODO:\|FIXME" --include='*.rb' --include='*.js' --include='*.ts' --include='*.py' . | grep -v node_modules
grep -rln "## Open Questions" docs/prds/ 2>/dev/null
```

| Source | Becomes |
|---|---|
| `TODO(harden)` markers | "Deferred tests" — the file, and what it says is untested |
| `TODO:` / `FIXME` in app code | "Known gaps" — file:line and the text |
| Unresolved `## Open Questions` in specs | "Undecided" — the question and which spec it's in |
| A connected issue tracker | **one line pointing at it**, never a mirrored copy — a stale mirror of a live tracker is worse than no file |

If the repo's real backlog is a tracker, `backlog.md` is four lines: where the
tracker is, what the deferred-test list is, and nothing else. That's a complete
and honest file.

---

## Phase 3 — Write

1. `AGENTS.md`, inside budget, ending with `Last verified: YYYY-MM-DD`.
2. The `docs/` files **that have real content**. Never scaffold an empty one. An
   empty file is a lie the routing table tells, and half the repos that have tried
   this have a `docs/` tree of empty directories to prove it. This applies to the
   two derived files as much as the rest: **no merged commits since the watermark
   means no changelog write, and zero markers with no tracker means no
   `backlog.md`** — and in that case the routing row doesn't get added either.
3. The symlink, from inside the repo root:
   ```bash
   ln -sfn AGENTS.md CLAUDE.md && ls -la CLAUDE.md
   ```
   Relative target, never absolute — the repo gets cloned elsewhere.

   **This costs nothing and there is no double-load.** Claude Code reads
   `CLAUDE.md` and *not* `AGENTS.md` (per its docs: "Claude Code reads
   `CLAUDE.md`, not `AGENTS.md`"), and `ln -s AGENTS.md CLAUDE.md` is the
   documented way to serve both toolchains from one file. One file on disk, one
   copy in context.

   On **Windows** a symlink needs Administrator or Developer Mode, so there
   `CLAUDE.md` is a one-line file containing `@AGENTS.md` instead — the import
   loads it at session start. Same result, no elevation.
4. If `.gitignore` excludes `CLAUDE.md`, say so. A symlink nobody clones is a
   symlink that doesn't exist for the team.
5. Verify with `/context` in a fresh session — `CLAUDE.md` must appear under
   **Memory files**, once.

Pre-existing docs usually need only the `> Purpose:` header prepended — leave their
content alone. **Don't prepend with `printf '%s\n\n%s' "$hdr" "$(cat $f)"`**:
command substitution strips the trailing newline, so the diff shows a deletion and
the file loses its final line ending. Use a temp file, then check the result is a
**pure insertion**:

```bash
git diff -U0 "$f" | grep -c '^-[^-]'    # must be 0
```

### Design system: freeze it, don't point at it

If the design system lives in another repo on your machine, **copy the current
block into `docs/design.md` with its version header** rather than routing to that
path. A path only you can resolve breaks the repo for exactly the teammate this is
for. A frozen copy with a version stamp is honest — it says what the app was built
against, and a mismatch is visible.

If the repo already has a root `DESIGN.md` or a design-tool sidecar, **leave them
exactly where they are** and route to them. Moving a file another tool reads by
name is out of scope.

---

## Phase 4 — Report

- Line count and byte count against 120 / 6KB.
- What moved down into `docs/`, and what each routing row now points at.
- **Everything dropped in reconciliation**, listed.
- **Changelog entries added** (how many, and the sha range), or why none were —
  "no merges since `<sha>`" is a fine answer and a useful one.
- **Backlog deltas** — markers that appeared or got cleared since last run.
- Anything you couldn't answer, named as an open question rather than guessed.

**Never commit. Never push.** Leave the tree dirty for review.

**One sequencing rule.** The default run deletes the outside pointer, but `migrate`
is opt-in — so if the old `CLAUDE.md` referenced an external notes file, the report
**must** end with: *"execution context still lives outside the repo — run
`/docs migrate <path>`."* Without that line, a default run orphans the knowledge:
pointer gone, content not yet in the repo, and nothing on screen says so.

---

## Mode: `check` — audit, write nothing

Read-only. Report and stop. No edits, no symlink, no `docs/` creation.

| Check | Fails when |
|---|---|
| Budget | `AGENTS.md` over 120 lines or 6KB |
| Freshness | `Last verified` older than 60 days |
| Changelog behind | merges on the default branch newer than the last recorded sha |
| Backlog behind | `TODO(harden)` markers in the code that `backlog.md` doesn't list |
| Dead rows | a routing row points at a file that doesn't exist |
| Drift | the file names a version the manifest contradicts |
| Outside coupling | any absolute path outside the repo, or a reference to a personal notes system |
| Symlink | `CLAUDE.md` exists and is not a symlink to `AGENTS.md` |
| Secrets | a credential value, not a location, in any context file |
| Unbounded | a file behind a routing row over ~500 lines |
| Empty | a routing row pointing at a file with no content below its `> Purpose:` |

Order findings worst-first. Offer the fix; don't apply it.

---

## Mode: `changelog` — catch the changelog up, nothing else

Runs Phase 2b's changelog half and stops. No `AGENTS.md` rewrite, no re-derivation,
no reconcile. Use it when the changelog has fallen behind and the rest of the
context is fine — a full run costs far more and changes files you didn't ask about.

Also the right target for a commit-time hook that wants the changelog current
without paying for the whole skill.

**One rule it does not relax: merged work only.** If the current branch isn't the
default branch, say what would be recorded once it merges and write nothing. A
changelog that lists unmerged work is worse than one that's behind, because behind
is obvious and wrong is not.

---

## Mode: `migrate <path>` — lift execution context out of an external notes file

For the very common case where a project's real knowledge has accumulated in a
notes app, a wiki, or a long-lived doc, and the repo has been pointing at it.

**Takes the source as an argument** — `/docs migrate ~/notes/projects/thing.md`.
It reads that file and **never writes to it.** If no path is given, ask for one;
don't guess, and don't go hunting through a home directory.

**1. Classify every section.** Long project notes are rarely sectioned by
audience — the split runs bullet by bullet inside one big list.

| Goes to the repo | Stays in the notes |
|---|---|
| Stack, architecture, data model | Contacts, relationships, timeline |
| Deploy, hosts, env, CI | Invoicing, rates, agreements |
| Design system rules, tokens | Meeting notes |
| Test policy, conventions, traps | Deal history, referrals, legal |
| Technical decisions + their why | Anything flagged open with a lawyer |
| Ship log → `docs/CHANGELOG.md` | Anything whose value is that it's *private* |

**2. The trap that will catch you.** The same word means different things on
adjacent lines. *Product* pricing — what customers pay, which tiers exist — is
execution spec and belongs with the code. *Engagement* pricing — what you bill the
client, equity, invoicing — is private. They sit two bullets apart under the same
heading and go opposite directions. Read each one; don't pattern-match on a
keyword.

**3. Check the notes copy isn't the stale one.** Where the repo also documents
something, the repo is usually current and the notes are months behind — notes get
written once at decision time and never revisited, while the repo gets corrected
every time it's wrong. Verify, then take the winner. Don't assume the longer file
is the better one.

**4. Show the split and wait.** Then write **only the repo side**.

**5. Trimming the source is a separate proposal and a separate approval.** Never
delete the user's notes. Surface duplication you trip over; don't go fix it.

---

## Rules

- **Derive before you ask.** The repo knows its own stack, commands and env vars.
- **Carry the why with the rule.** A convention without its reason gets deleted.
- **Verify over recency** when two files disagree.
- **Move content down, never thin it.** Over budget means the router is doing the
  library's job, not that the content is unnecessary.
- **Never scaffold empty files.**
- **Never commit, never push, never deploy.**
- **Never write to the user's notes** — `migrate` reads and proposes; that's all.
- **Never put a credential value anywhere.** Location only.

## What this is not

- **Not a note-taking command.** It writes nothing to any external notes system,
  ever. It only ever reads one, and only in `migrate`.
- **Not a session summary.** It doesn't recap what you built or tell you what to
  click.
- **Not a design-system sync.** It freezes the current design block into
  `docs/design.md` and routes to it; keeping that system up to date is a different
  job.
- **Not a linter.** It flags duplication it trips over; it doesn't go hunting.
- **Not a doc generator.** It writes what's true and verifiable. An invented
  convention is worse than a missing one — an agent will follow it.
