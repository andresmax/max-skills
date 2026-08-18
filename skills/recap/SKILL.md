---
name: recap
description: End-of-session summary in plain English — what got built, exactly how to test it by hand, what has to be set up before it works anywhere but this laptop, and a copy-paste block for the team. Use when the user says "recap", "/recap", "what did we just do", "summarize this session", "what do I need to test", "give me something to send the team", or asks for a wrap-up at the end of a long build or after finishing a spec. Also use when returning to a branch cold and needing the context back fast. Never a code review, never a diff read back.
---

# /recap — what we built, and what you should go click

You've been heads-down for hours. The session left behind a wall of completed
steps, green tests, and merged branches. None of that tells you what to open in a
browser, or what to tell the person who asked for it.

Answer four things, in this order:

1. **What is now possible** that wasn't before, in customer words.
2. **How to test it by hand** — exact URL, exact login, exact click, expected result.
3. **What has to be set up** before it works anywhere but this laptop.
4. **A block they can paste to a human** without editing it.

They can read the diff. Do not read them the diff.

`$ARGUMENTS` may narrow it: a spec or PRD number (`/recap PRD-049`), an issue key
(`/recap ENG-273`), or an audience (`/recap for the designer`). With no arguments,
recap this session's work.

## Gather the evidence

Read-only. Change nothing, run nothing but git and file reads. **Never run the test
suite** — a green suite is not what this is for, and it burns the minutes this is
meant to save.

```bash
git rev-parse --show-toplevel && git branch --show-current
git status --short
git log --oneline "$BASE"..HEAD
git diff --stat "$(git merge-base HEAD "$BASE")"
```

`$BASE` is `origin/HEAD` if it resolves, else `main`, else `master`, else
`develop`. Work is often **uncommitted** when this gets run, so `git status
--short` matters as much as the log.

Sources of meaning, best first:

1. **This session's conversation.** Usually the strongest source, and free. What
   was asked for, what changed mid-flight, what got rejected.
2. **The spec** — `docs/prds/`, `docs/specs/`, or whatever the repo uses. Carries
   intent, scope, and acceptance criteria; the acceptance criteria are the raw
   material for the test list.
3. **The issue**, if a key appears in the branch or commits. Fetch it only if the
   spec and conversation leave the intent unclear.
4. **User-facing surfaces in the diff**: views, copy, routes, emails, seeds,
   flags, pricing.

Tests, config and internal cleanup tell you the shape of the work, not its value.
Never build the summary out of them.

Large diff (2k+ lines): read the spec and the user-facing surfaces, let the file
list carry the rest. Delegate that reading to a subagent if it's cheaper. This runs
at the end of a long day — it stays fast.

## The three sentences

Every sentence must survive being read out loud to someone who has never seen the
code.

A sentence fails if it contains a file name, class, table, endpoint, library,
framework, line count, or the words *refactor*, *implement*, *wire up*, *scaffold*,
*abstraction*. Rewrite it as something a person can now do.

For each cluster of change ask: **who can now do what they could not do before?**

If the honest answer is "nobody yet", say that plainly. Plumbing sessions are real
and normal. Inventing customer value for a foundation phase is the worst failure
here, worse than boring, because people make shipping calls off this.

Stay proportional. "This is a polish pass on the cards so the board stops looking
half-finished" beats a fake epic. This gets trusted precisely because it doesn't
oversell.

If part of the work resists translation, say which part you couldn't place. Never
fill the gap with something plausible.

## Test it — the part they actually opened this for

The centre of the recap. They want to close the laptop knowing what to click.

**Prioritise by what the tests cannot prove.** A green suite already covers the
logic. What it never covers is the real browser, the real third-party round trip,
the real email landing, the real payment, the real upload. Those go first.
Something that ran only in CI has not run.

Order the list by **what is most likely broken**, not by feature-tour order.

Each step names four things:

- **Where** — the full URL, and which environment it's live on right now.
- **Who** — the exact account. Pull real credentials from the repo's demo-accounts
  file if it has one. **Never invent a login.** If none exists for the path, say
  which kind of account is needed.
- **What to click** — the actual path through the UI, not "navigate to the feature".
- **What should happen** — the observable result. Where a bug was fixed this
  session, name what should *not* happen, because that's the real assertion.

Cap it at seven steps. Fewer is better. If the change is invisible by hand — a
background job, a migration, a mailer — say how to observe it instead: the admin
page that reflects it, the row count that should have moved, the inbox it lands in.

Add a **watch for** line only when this change plausibly broke something nearby
they'd never think to check. Direct evidence only. Speculative regressions train
people to skim.

If a spec was in play, note in one line anything in its scope that visibly did not
get built. One line, not an audit.

## Before it works

What stands between here and the thing running somewhere real. Deploys rarely break
on code. They break on a value that only ever existed on this laptop, or a task
nobody remembered to run.

Two scans catch most of it:

```bash
git diff --name-only "$(git merge-base HEAD "$BASE")" \
  | grep -Ei "migrat|\.rake$|seed|schedule|cron|procfile|\.env|docker-compose|deploy"

git diff "$(git merge-base HEAD "$BASE")" | grep -E "^\+" \
  | grep -E "ENV\[|ENV\.fetch|process\.env|os\.environ|credentials\."
```

| Signal | What to say |
| --- | --- |
| New env var or credential reference | The exact name, and where it gets set. If code reads it but nothing added it to `.env.example` or the secret store, say so. That's the one that bites. |
| New migration | Run migrations. If one backfills rows rather than changing structure, its own line — it's slow and it touches real data. |
| New seed data | Flag it. Where deploy scripts run seeds, a seed bug fails the deploy of good code. |
| New task, script, or command | The exact command, and whether it's once after deploy or on a schedule. |
| New job or queue | The queue or schedule that must exist in production, not just in the config file. |
| New webhook, vendor client, API key, bucket, DNS record | The setup that happens outside the repo. Name the vendor and the exact thing to add. |
| Feature flag or constant gate | The flag to flip, and who sees it before it's flipped. |
| Code reading a column added this session | Deploy order: migrate first, or the first request after deploy 500s. |

Split **before deploy** and **after deploy** when both apply. Only list items with
direct evidence. Nothing to do gets one line: "Nothing to set up."

Note where the work currently sits — local, staging, or prod — because that's three
different answers and nobody remembers which.

## For the team

A block they can paste into chat without editing. Default audience is the
**non-technical stakeholder** — the founder, the client, the person who asked for
it. That's the harder constraint, and an engineer reads it fine anyway. Naming an
audience switches it: an engineer gets branch names and issue keys, a designer gets
the screens and what changed visually.

**Order by size of the work, never by who's reading.** The block leads with the
largest thing built and works down. A small item the reader personally asked for
does not get to jump the queue just because it's addressed to them, and it never
becomes the headline — that misrepresents the session as a handful of favours and
buries the real build. Read the spec's brief for the split: it names the main scope
first and the add-ons second, and that order carries into the block. If a
teammate's request genuinely was the main event, it leads because it's biggest, not
because they asked.

Same rule for whose work it is. Credit the piece to whoever owns it — a designer's
layout, a founder's product call — inside the item, not by reordering the block
around them.

Rules for the block:

- Self-contained. No "as discussed", no bare ticket keys, nothing that only makes
  sense to someone who was in the session.
- Leads with what they can now do, then the link, then the login if they need one.
- Names anything they own. Copy someone has to write, a payment product someone has
  to create, a design decision someone is blocked on.
- Never promises a date nobody said.

Skip the block entirely when the session was internal plumbing with nothing a
teammate can see or act on. Say so in a line instead of manufacturing an update.

## Output

```
**<repo> / <branch>**  ·  <spec or feature name>  ·  <where it lives now>

<Sentence 1: what's now possible, as the user experiences it.>
<Sentence 2: why it matters, the problem it kills.>
<Sentence 3: where it leaves us, what's next or still missing.>

**Test it**
1. <URL> as <account> → <click path> → <what you should see>
2. ...

Watch for: <only when there's a real adjacent risk>

**Before it works**
- Before deploy: <exact action, exact name>
- After deploy: <exact action, exact name>

**For the team**
> <paste block>
```

Drop any section that's genuinely empty. Never pad one to fill the shape.

## Voice

Plain words, no hype, proportional to the work. This is leaving the building, so
write it like it will be forwarded — because it will.

## What this is not

- **Not a save command.** Writes nothing anywhere. No files, no notes, no memory.
- **Not a test-writing pass.** Doesn't write tests or note missing ones.
- **Not a code review.** No quality flags, no suggested improvements, no bug hunt.
  The setup list is operational, which is a different thing from whether the code
  is good.
- **Not release notes.** Written for the person who did the work, even though they
  may well paste it.

If there are no changes and no session work to summarize, say so in one line and
stop. Do not pad.
