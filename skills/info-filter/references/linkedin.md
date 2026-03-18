# LinkedIn — Source Strategy

## Overview

LinkedIn requires authentication. Use agent-browser to log in, scroll the feed, and extract post content. No public API is available for feed access.

## Prerequisites

1. agent-browser installed locally: `npm install` in the plugin directory
2. Chrome for Testing installed: `npx agent-browser install`
3. LinkedIn credentials saved: `npx agent-browser auth save linkedin --url https://www.linkedin.com --username <email> --password <password>`

## Fetching Workflow

### Step 1: Check Prerequisites

Before launching agent-browser, verify it's available:

```bash
npx agent-browser --version
```

If this fails, skip LinkedIn with the message:
> "LinkedIn skipped — agent-browser not installed. Run `npm install` in the plugin directory, then `npx agent-browser install`."

### Step 2: Launch Browser and Authenticate

```bash
npx agent-browser run --auth linkedin --url "https://www.linkedin.com/feed/" --steps '<steps>'
```

The `--auth linkedin` flag loads saved credentials from the agent-browser vault.

### Step 3: Scroll and Capture Feed

Use agent-browser steps to scroll the feed and capture content. The goal is to collect raw post data from approximately 30 scroll iterations (configurable via `sources.linkedin.feed_scroll_count`).

Agent-browser steps:

1. **Wait for feed to load** — wait for the main feed container to be visible.
2. **Scroll loop** — repeat `feed_scroll_count` times:
   - Scroll down by one viewport height
   - Wait 1-2 seconds for new posts to load
   - Take a snapshot of currently visible posts
3. **Extract post data** — from each snapshot, extract:
   - Author name and headline
   - Post text content
   - Any shared article URL and title
   - Engagement counts (likes, comments) if visible
   - Approximate post age (e.g., "2h", "1d")

### Step 4: Check Specific Profiles

If `sources.linkedin.profiles` is non-empty, visit each profile's recent activity:

```
https://www.linkedin.com/in/{profile}/recent-activity/all/
```

For each profile:
1. Navigate to the activity page
2. Scroll 2-3 times to load recent posts
3. Extract posts using the same format as feed posts

### Step 5: Parse and Normalize

From the raw captured data, produce a normalized list. For each post:

```
- title: {first line of post or shared article title}
  url: {shared article URL, or LinkedIn post URL if available}
  author: {author name}
  author_headline: {author headline/title}
  source: LinkedIn
  engagement: {likes/comments counts}
  age: {relative time from post}
  content: {full post text, truncated to 500 chars}
  shared_url: {URL of any shared article, if present}
```

### Step 6: Quick Filter

Before full scoring:

1. **Auto-reject** posts matching `negative_signals` keywords.
2. **Drop** posts that are purely congratulatory ("Congrats!", "Welcome aboard"), job change announcements with no substance, or engagement bait ("Agree?", "Thoughts?") with no original content.
3. **Preliminary score** on Topic Relevance using post text — drop anything below 3.0.

### Step 7: Fetch Shared Articles

If a post shares an external URL, attempt to fetch it with WebFetch to get richer content for scoring. Skip if the URL is not fetchable.

### Step 8: Score

Apply the full 4-dimension rubric from `skills/info-filter/references/scoring.md`.

LinkedIn-specific scoring notes:
- **Source Quality:** Check author against `trusted_sources.linkedin_authors`. Industry practitioners sharing original insights score higher than influencer-style content.
- **Timeliness:** Parse the relative time string ("2h" = high, "1w" = medium, "1mo" = low).
- **Actionability:** Posts sharing tools, frameworks, case studies, or technical approaches score high. Motivational or thought-leadership posts without specifics score low.
- **Topic Relevance:** Weight the post's actual substance, not engagement metrics. A low-engagement post directly addressing a user's goal beats a viral post that's only tangentially related.

## Output Format

For each LinkedIn item that passes scoring:

```
- title: {title}
  url: {post or shared article URL}
  author: {author name} ({author headline})
  source: LinkedIn
  age: {relative time}
  summary: {2-3 sentence summary of the post content}
  relevance_scores:
    topic: {score}
    quality: {score}
    timeliness: {score}
    actionability: {score}
  final_score: {weighted score with modifiers}
  why_relevant: {1 sentence explaining connection to user's goals}
```

## Error Handling

- agent-browser not installed → skip with install instructions (see Step 1)
- Auth not configured → skip with message: "LinkedIn skipped — no auth configured. Run `npx agent-browser auth save linkedin --url https://www.linkedin.com --username <email> --password <password>`"
- Login fails / CAPTCHA → report: "LinkedIn login failed (possible CAPTCHA or credential issue). Skipping."
- Feed fails to load → report and skip
- Individual profile page fails → skip that profile, continue with others
