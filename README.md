# max-skills

Skills I actually use, every day, on real client work and my own products.

Nothing here is a demo. Each one exists because I kept doing the same thing by
hand and got tired of it, or because I shipped something broken and wanted the
lesson written down where an agent would read it before repeating my mistake.

Built for [Claude Code](https://claude.com/claude-code), but they're just markdown
with a bit of frontmatter — most of the content transfers to any agent that reads
instructions from a file.

## The skills

| Skill | What it does |
|---|---|
| **[`docs`](skills/docs)** | Makes a repo explain itself in about a second. A thin `AGENTS.md` router — hard-capped at 120 lines — plus a `docs/` library it points into, with `CLAUDE.md` as a symlink so every toolchain reads one file. Kills the "read my notes app for context" dependency that breaks the moment someone else clones your repo. |
| **[`ship`](skills/ship)** | Autopilot feature pipeline: brief → research → spec → *your green light* → build → verify → land on a branch. Two human touchpoints, everything else unattended. Never pushes. |
| **[`harden`](skills/harden)** | Writes the test suite you deliberately deferred, now that the design is settled. Proves each test bites by breaking its subject and watching it fail. Refuses to run on code that has never met its real dependency. |
| **[`recap`](skills/recap)** | End-of-session summary in plain English: what's now possible, exactly what to click to test it, what has to be set up before it works anywhere but your laptop, and a block you can paste to a non-technical teammate. |
| **[`morning-brief`](skills/morning-brief)** | One screen of the day's tech news — HN, Reddit, GitHub Trending, YouTube, Product Hunt, RSS — fetched in parallel, printed as clickable links. The feed lists are yours to edit. |

## The two ideas worth stealing even if you skip the skills

**A context file is a router, not an encyclopedia.** The file that loads every
session should answer *"what will I get wrong if nobody tells me?"* and then point
at everything else with a trigger attached — "read this **when** you're deploying".
A table of contents makes an agent read everything. A table of triggers makes it
read one thing. `docs` enforces this with a hard byte budget, because the discipline
doesn't survive without one.

**Don't write the test suite until the thing has actually run.** Tests written
before first contact with a real dependency test your assumptions, then report that
back as confidence. I shipped 489 green tests over an integration that had never
made a single real request; it failed on the first one, and no stub could have
caught it. `ship` splits test depth in two — boundary code gets tested immediately,
product semantics get one smoke test and a `TODO(harden)` marker — and `harden` is
the other half of the bargain.

## Install

Clone anywhere, then symlink the skills you want into `~/.claude/skills/`:

```bash
git clone https://github.com/andresmax/max-skills.git ~/Code/max-skills
ln -s ~/Code/max-skills/skills/docs   ~/.claude/skills/docs
ln -s ~/Code/max-skills/skills/harden ~/.claude/skills/harden
```

Or copy the folder in if you'd rather edit freely:

```bash
cp -r ~/Code/max-skills/skills/recap ~/.claude/skills/
```

Either way, `/docs`, `/harden`, `/recap`, `/ship` and `/morning-brief` become
available in your next session. Skills also auto-trigger from their `description`,
so you often don't need to type the slash command at all.

Project-scoped instead of global? Same thing, into `.claude/skills/` inside the
repo.

## Configuration

Only `ship` reads an optional config, and it works fine without one.
`~/.claude/ship-profiles.json` overrides auto-detection for specific repos:

```json
{
  "my-app": {
    "profile": "product",
    "tracker": "linear",
    "tests": "bin/rails test",
    "reviewers": ["code-reviewer"]
  }
}
```

A repo that isn't listed still works — everything is detected from the repo itself.

`morning-brief` has its feeds inline, in one clearly-marked block at the top. Swap
in your own subreddits and channels; nothing else in the file needs to change.

## A note on what's here

These lean Rails and Hotwire in places, because that's what I build. The
*mechanisms* are stack-agnostic and the skills detect what they're looking at —
`ship` reads your `package.json` scripts or your `go.mod` the same way it reads a
`Gemfile`. Where something is genuinely Rails-specific it's marked as such
(`harden` ends with a list of Minitest traps that have each cost me an afternoon).

Corrections and better ideas welcome. If a skill is wrong about your stack, that's
a bug.

## License

MIT. Take them, fork them, make them yours.
