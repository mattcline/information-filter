# Hacker News — Source Strategy

## Overview

Hacker News uses a free, public Firebase API. No authentication, no agent-browser, no rate limits to worry about. Use WebFetch for all requests.

## API Endpoints

Base URL: `https://hacker-news.firebaseio.com/v0`

| Endpoint | Returns |
|----------|---------|
| `/topstories.json` | Array of up to 500 item IDs, ordered by current rank |
| `/beststories.json` | Array of up to 500 item IDs, ordered by best algorithm |
| `/item/{id}.json` | Single item (story, comment, job, poll) |

## Fetching Workflow

### Step 1: Get Story IDs

Fetch top stories (always) and best stories (if `sources.hacker_news.include_best` is true):

```
WebFetch https://hacker-news.firebaseio.com/v0/topstories.json
WebFetch https://hacker-news.firebaseio.com/v0/beststories.json
```

Take the first N IDs from each list, where N = `sources.hacker_news.top_stories_count` from config. Merge and deduplicate the two lists.

### Step 2: Fetch Story Details

For each story ID, fetch the item:

```
WebFetch https://hacker-news.firebaseio.com/v0/item/{id}.json
```

Each item returns JSON:
```json
{
  "id": 12345,
  "type": "story",
  "by": "username",
  "time": 1710000000,
  "title": "Show HN: Something Interesting",
  "url": "https://example.com/article",
  "score": 150,
  "descendants": 87
}
```

Extract from each story:
- `title` — headline text
- `url` — link to article (may be absent for Ask HN / Show HN text posts)
- `by` — author username
- `score` — upvote count
- `descendants` — comment count
- `time` — Unix timestamp (convert to relative time for display)

### Step 3: Quick Filter

Before fetching full articles, do a fast pass using title + metadata only:

1. **Auto-reject** any story whose title matches a `negative_signals` keyword.
2. **Drop** stories with `score` below `sources.hacker_news.min_score`.
3. **Preliminary score** using title alone against goals/topics — drop anything that scores below 3.0 on Topic Relevance (saves fetching articles that clearly won't make the cut).

### Step 4: Fetch Article Content

If `sources.hacker_news.fetch_articles` is true, fetch the linked URL for each surviving story:

```
WebFetch {story.url}
```

Use the article content to refine the Topic Relevance and Actionability scores. Some articles won't be fetchable (paywalls, PDFs, etc.) — that's fine, score based on title and HN metadata alone.

For Ask HN and Show HN posts without a URL, the content is in the `text` field of the item itself.

### Step 5: Score

Apply the full 4-dimension rubric from `skills/info-filter/references/scoring.md`.

HN-specific scoring notes:
- **Source Quality:** Check `by` against `trusted_sources.hacker_news_users`. Known users get a boost.
- **Timeliness:** Convert `time` (Unix) to age. HN top stories are usually <24h old, so most will score 7+.
- **Actionability:** Show HN posts often score high (tool to try). Ask HN discussions may score lower unless directly relevant.
- **Boost:** Apply +0.5 if `descendants` > 100 (indicates substantive discussion worth reading).

## Output Format

For each HN item that passes scoring, produce:

```
- title: {title}
  url: {url or HN comments link}
  author: {by}
  source: Hacker News
  hn_score: {score}
  comments: {descendants}
  age: {relative time}
  summary: {2-3 sentence summary from article content or title}
  relevance_scores:
    topic: {score}
    quality: {score}
    timeliness: {score}
    actionability: {score}
  final_score: {weighted score with modifiers}
  why_relevant: {1 sentence explaining connection to user's goals}
```

## Error Handling

- Firebase API down → skip HN entirely, report "Hacker News API unavailable"
- Individual story fetch fails → skip that story, continue with others
- Article URL not fetchable → score based on title/metadata only, note "article content unavailable"
- Empty response from top/best stories → report and skip
