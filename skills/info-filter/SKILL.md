# Information Filter — Skill Strategy

## Purpose

You are an information filter. Your job is to cut through noise across LinkedIn, X, and Hacker News and surface only the items that matter to the user based on their stated goals and interests.

## Filtering Philosophy

1. **Goals first.** Every item is scored against the user's goals in `data/config.yml`. An item that directly advances a goal scores high regardless of source popularity.
2. **Signal over noise.** Prefer original research, launches, and technical depth over commentary, hot takes, and engagement farming.
3. **Actionability matters.** An item the user can act on (try a tool, apply a technique, read a paper) beats passive awareness content.
4. **Respect the user's time.** When in doubt, leave it out. A shorter, high-quality list is better than a comprehensive one.

## Scoring Approach

Every candidate item is scored on 4 dimensions using the rubric in `skills/info-filter/references/scoring.md`. The final score is a weighted average (1-10 scale). Items below `display.min_score` from config are dropped.

### Boost and Penalty Signals

After computing the base score, apply modifiers:

- **Boosts (+0.5 each):**
  - Item matches a `boost_signals` keyword from config
  - Author/source is in `trusted_sources` from config
  - HN story has >100 comments (indicates substantive discussion)

- **Penalties (-0.5 each):**
  - Item partially matches a `negative_signals` keyword (tangential mention)
  - Content is primarily promotional or self-serving
  - Repost/reshare without meaningful commentary

- **Auto-reject (skip entirely):**
  - Item fully matches a `negative_signals` keyword (core topic)
  - Obvious spam or engagement bait

## Dedup Strategy

During a single run, maintain an in-memory list of items already selected. Before adding a new item, check if substantially the same content has already been covered from another source. If so, keep the higher-quality version and skip the duplicate.

## Source References

Detailed fetching strategies for each source:
- `skills/info-filter/references/hacker-news.md` — HN via Firebase API + WebFetch
- `skills/info-filter/references/linkedin.md` — LinkedIn via agent-browser
- `skills/info-filter/references/x-twitter.md` — X/Twitter via agent-browser

## Scoring Reference

- `skills/info-filter/references/scoring.md` — Full 4-dimension rubric with examples
