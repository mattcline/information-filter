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
- `npx agent-browser auth save linkedin ...` for LinkedIn auth
- `npx agent-browser auth save x ...` for X auth
- Copy `data/config.example.yml` → `data/config.yml` and customize
