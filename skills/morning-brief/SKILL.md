---
name: morning-brief
description: Generate a fast, scannable morning tech briefing in the terminal — Hacker News, Reddit, GitHub Trending, YouTube uploads from the last 24h, Product Hunt, trending topics, and news RSS, all fetched in parallel and printed as one compact block of clickable links. Use when the user says "morning brief", "/morning-brief", "what's happening today", "catch me up", or wants a daily tech news scan. Writes no files, keeps no state, and skips any source that fails rather than showing errors.
---

# /morning-brief — the day's tech news, in one screen

Fetch **all** sources below in parallel — maximize parallel tool calls. Output
directly to the terminal. No file writes, no state, no commentary.

---

## Make it yours

Everything in this section is a starting point, not a recommendation. The feeds
below reflect one person's interests — a mix of AI, building, hardware, space and
deliberately-good news. **Edit these four lists and the skill is yours.** Nothing
else in the file needs to change.

### Subreddits, grouped

| Group | Subs |
|---|---|
| **AI/Tech** | `claudeai` `singularity` `technology` `technews` |
| **Building** | `entrepreneur` `saas` `webdev` `web_design` `selfhosted` |
| **Interests** | `apple` `bambulab` `spaceexploration` `science` |
| **Good Vibes** | `goodnews` |

### YouTube channels

| Channel | Feed |
|---|---|
| Fireship | `UCsBjURrPoezykLs9EqgamOA` |
| Two Minute Papers | `UCbfYPyITQ-7l4upoX8nvctg` |
| MKBHD | `UCBJycsmduvYEL83R_U4JriQ` |
| Matt Wolfe | `UChpleBmo18P08aKCIgti38g` |
| Everyday Astronaut | `UC6uKrU_WqJ1R2HMTY3LIx5Q` |
| Made with Layers | `UCb8Rde3uRL1ohROUVg46h1A` |
| My First Million | `UCyaN6mg5u8Cjy2ZI4ikWaug` |
| Greg Isenberg | `UCPjNBjflYl0-HQtUvOx0Ibw` |

To find a channel's ID: open the channel, view source, search for `channelId`.

### News RSS

- MacRumors: `https://feeds.macrumors.com/MacRumors-All`
- Engadget: `https://www.engadget.com/rss.xml`

### Search queries

- Trending topics: `trending technology AI topics today`
- Google Trends: `Google Trends trending searches technology United States today`

---

## Sources to fetch (all in parallel)

### 1. Hacker News (WebFetch)
`https://hn.algolia.com/api/v1/search?tags=front_page&hitsPerPage=5`
Extract title, points, num_comments, objectID for the top 5. The discussion link is
`https://news.ycombinator.com/item?id={objectID}`.

### 2. Reddit (Bash — single command)
Reddit is blocked on WebFetch. Run this ONE Bash command, with your sub list:

```bash
for sub in claudeai singularity technology technews entrepreneur saas webdev web_design selfhosted apple bambulab spaceexploration science goodnews; do curl -s -H "User-Agent: morning-brief/1.0" "https://www.reddit.com/r/${sub}/hot.json?limit=2" 2>/dev/null | python3 -c "import json,sys; d=json.load(sys.stdin); [print(f'r/${sub} — {p[\"data\"][\"title\"][:80]} ({p[\"data\"][\"score\"]} pts) ||| https://reddit.com{p[\"data\"][\"permalink\"]}') for p in d['data']['children'][:2] if not p['data'].get('stickied')]" 2>/dev/null; done
```

If a sub fails silently, skip it.

### 3. GitHub Trending (WebFetch)
`https://github.com/trending` — top 5 repos: owner/name, short description, stars
today. Link is `https://github.com/{owner}/{name}`.

### 4. YouTube (WebFetch, all channels in parallel)
Fetch `https://www.youtube.com/feeds/videos.xml?channel_id={id}` per channel. Only
include videos published in the **last 24 hours**; skip channels with no recent
upload. Extract title, published datetime, and the `<link rel="alternate">` inside
the `<entry>` — that's the `watch?v=` URL.

### 5. Product Hunt (WebFetch)
`https://www.producthunt.com` — top 3 launches: name, tagline, upvotes, product URL.

### 6. Trending topics (WebSearch)
Use the search query above. Top 5 trending tech topics or conversations.

### 7. News RSS (WebFetch, all in parallel)
Top 3 headlines each, with article URLs.

### 8. Google Trends (WebSearch)
Google Trends is JS-heavy — do **not** use WebFetch. Use WebSearch with the query
above. Top 5 trending searches in tech.

---

## Output format

Print this EXACTLY. No intro text, no summary, no commentary. Just the brief.

```
═══════════════════════════════════════════════
  MORNING BRIEF — {today's date, e.g. Apr 10, 2026}
═══════════════════════════════════════════════

▸ HACKER NEWS
  1. [{title}]({hn_discussion_url}) ({points} pts, {comments} comments)
  2. ...

▸ REDDIT
  AI/Tech
    r/{sub} — [{title}]({reddit_permalink}) ({score} pts)
  Building
    r/{sub} — [{title}]({reddit_permalink}) ({score} pts)
  Interests
    r/{sub} — [{title}]({reddit_permalink}) ({score} pts)
  Good Vibes
    r/{sub} — [{title}]({reddit_permalink}) ({score} pts)

▸ GITHUB TRENDING
  1. [{owner/name}]({github_repo_url}) — {description} (⭐ {stars today} today)

▸ YOUTUBE (last 24h)
  {Channel} — ["{title}"]({youtube_watch_url}) ({Xh ago})
  (or: "No new uploads in the last 24h")

▸ PRODUCT HUNT
  1. [{Name}]({product_url}) — {tagline} ({upvotes} upvotes)

▸ TRENDING
  1. {topic} — {brief context}

▸ NEWS
  {Source}
    • [{headline}]({article_url})

═══════════════════════════════════════════════
```

---

## Rules

- ALL items must be clickable markdown links — no plain text titles. `[title](url)`
  everywhere.
- Reddit bash output uses `|||` between title and URL. Parse both parts to build
  the link.
- Show the **top 1** non-stickied post per subreddit (pick the higher scoring of
  the two you fetched).
- For YouTube, calculate hours since publish — skip anything older than 24h.
- Merge the two trending sources into one section, deduplicating similar topics.
- Keep titles SHORT — truncate at ~80 chars.
- **If a source fails, skip it silently.** Never show errors in the brief.
- No trailing summary, no sign-off. End with the bottom border line.
