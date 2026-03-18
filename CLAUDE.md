# Information Filter — Claude Code Plugin

A Claude Code plugin that surfaces relevant information from LinkedIn, X, and Hacker News based on user-defined goals and interests.

## How It Works

- Invoked via `/get-info`
- Fetches content from HN (Firebase API via WebFetch), LinkedIn and X (via agent-browser)
- Scores items on a 4-dimension rubric (Topic Relevance, Source Quality, Timeliness, Actionability)
- Presents results grouped by topic cluster

## Key Paths

- `data/config.yml` — user preferences (gitignored, copy from `data/config.example.yml`)
- `skills/info-filter/SKILL.md` — filtering philosophy and scoring approach
- `skills/info-filter/references/` — per-source strategies and scoring rubric
- `commands/get-info.md` — main workflow (7 phases)

## Prerequisites

- `npm install` to get agent-browser locally
- `npx agent-browser install` to download Chrome for Testing
- Import LinkedIn session: launch Chrome with `--remote-debugging-port=9222 --user-data-dir=/tmp/chrome-debug-profile`, log in to linkedin.com, then run `npx agent-browser --auto-connect state save /tmp/linkedin-auth.json` followed by `npx agent-browser --session-name linkedin state load /tmp/linkedin-auth.json` (see `skills/info-filter/references/linkedin.md` for details)
- Import X session: same flow with x.com, using `--session-name x` (see `skills/info-filter/references/x-twitter.md` for details)
- Copy `data/config.example.yml` → `data/config.yml` and customize
