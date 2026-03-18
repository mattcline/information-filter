# X/Twitter — Source Strategy

## Overview

X requires authentication for timeline access. Use agent-browser to log in, scroll the timeline, and extract post content. Also check specific accounts and lists from config.

## Prerequisites

1. agent-browser installed locally: `npm install` in the plugin directory
2. Chrome for Testing installed: `npx agent-browser install`
3. X session imported via browser:
   ```bash
   # Close all Chrome windows, then relaunch with remote debugging:
   "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --remote-debugging-port=9222

   # In that Chrome window, navigate to x.com and log in normally
   # (handle 2FA, CAPTCHAs, etc. as you normally would)

   # Save state from Chrome to a temp file:
   npx agent-browser --auto-connect state save /tmp/x-auth.json

   # Load into a named session (auto-persists across runs):
   npx agent-browser --session-name x state load /tmp/x-auth.json

   # Clean up temp file and close the debug Chrome window:
   rm /tmp/x-auth.json
   ```

## Fetching Workflow

### Step 1: Check Prerequisites

Before launching agent-browser, verify it's available:

```bash
npx agent-browser --version
```

If this fails, skip X with the message:
> "X/Twitter skipped — agent-browser not installed. Run `npm install` in the plugin directory, then `npx agent-browser install`."

### Step 2: Launch Browser with Session

```bash
npx agent-browser --session-name x open "https://x.com/home"
```

The `--session-name x` flag restores saved cookies/state and auto-saves them back after each run.

### Step 2a: Validate Auth

After the page loads, check the current URL. If it contains `/login` or `/i/flow/login`, the session has expired. Skip X with the message:
> "X/Twitter skipped — session expired. Re-import your session by launching Chrome with `--remote-debugging-port=9222`, logging in to x.com, then running `npx agent-browser --auto-connect state save /tmp/x-auth.json` followed by `npx agent-browser --session-name x state load /tmp/x-auth.json`."

### Step 3: Scroll and Capture Timeline

Use agent-browser steps to scroll the "For You" or "Following" timeline and capture content. Collect posts from approximately 30 scroll iterations (configurable via `sources.x_twitter.timeline_scroll_count`).

Agent-browser steps:

1. **Wait for timeline to load** — wait for the main timeline container to be visible.
2. **Switch to "Following" tab** if available — this gives a chronological feed with less algorithmic noise.
3. **Scroll loop** — repeat `timeline_scroll_count` times:
   - Scroll down by one viewport height
   - Wait 1-2 seconds for new posts to load
   - Take a snapshot of currently visible posts
4. **Extract post data** — from each snapshot, extract:
   - Author display name and @handle
   - Post text content (full text, not truncated)
   - Any embedded URLs or linked articles
   - Media indicators (image, video, link card)
   - Engagement counts (likes, reposts, replies, views) if visible
   - Post timestamp or relative age
   - Whether it's a repost (and original author if so)
   - Whether it's a quote post (and quoted content)

### Step 4: Check Specific Accounts

If `sources.x_twitter.accounts` is non-empty, visit each account's profile:

```
https://x.com/{handle}
```

For each account:
1. Navigate to the profile page
2. Scroll 2-3 times to load recent posts
3. Extract posts using the same format as timeline posts
4. Skip replies — only capture original posts and reposts

### Step 5: Check Lists

If `sources.x_twitter.lists` is non-empty, visit each list:

```
https://x.com/i/lists/{list_id}
```

For each list:
1. Navigate to the list timeline
2. Scroll 3-5 times
3. Extract posts using the same format

### Step 6: Parse and Normalize

From the raw captured data, produce a normalized list. For each post:

```
- title: {first sentence or line of post text}
  url: {link to the post on X, or embedded article URL}
  author: {display name} (@{handle})
  source: X/Twitter
  engagement: {likes/reposts/replies/views}
  age: {relative time}
  content: {full post text, up to 500 chars}
  embedded_urls: [{any URLs found in the post}]
  is_repost: {true/false}
  is_quote: {true/false}
  quoted_content: {quoted post text if applicable}
```

### Step 7: Quick Filter

Before full scoring:

1. **Auto-reject** posts matching `negative_signals` keywords.
2. **Drop** posts that are:
   - Pure reposts without commentary (unless the original is from a trusted source)
   - Engagement farming ("Like if you agree", ratio bait, ragebait)
   - Promotional threads selling courses/newsletters with no substantive content
   - Single-emoji or reaction-only posts
3. **Preliminary score** on Topic Relevance using post text — drop anything below 3.0.

### Step 8: Fetch Embedded Articles

If a post contains an external URL (not another X link), attempt to fetch it with WebFetch to get richer content for scoring. Skip if the URL is not fetchable.

### Step 9: Score

Apply the full 4-dimension rubric from `skills/info-filter/references/scoring.md`.

X-specific scoring notes:
- **Source Quality:** Check @handle against `trusted_sources.x_accounts`. Practitioners sharing firsthand experience or original work score higher than commentary accounts. Verified status alone does not affect score.
- **Timeliness:** Parse the timestamp. X posts are often very recent; favor posts from the last few hours.
- **Actionability:** Threads with technical detail, tool announcements, or linked resources score high. Hot takes and opinion-only posts score low unless they provide genuine insight.
- **Topic Relevance:** Score the actual content, not the engagement metrics. A 5-like post from a domain expert directly addressing a user goal beats a 50K-like viral post that's tangentially related.
- **Repost penalty:** Apply -0.5 to reposts without added commentary. Quote posts with substantive commentary are scored normally.

## Output Format

For each X item that passes scoring:

```
- title: {title}
  url: {post URL or embedded article URL}
  author: {display name} (@{handle})
  source: X/Twitter
  age: {relative time}
  summary: {2-3 sentence summary of the post/thread content}
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
- Session not configured → skip with message: "X/Twitter skipped — no session configured. Import your session by launching Chrome with `--remote-debugging-port=9222`, logging in to x.com, then running `npx agent-browser --auto-connect state save /tmp/x-auth.json` followed by `npx agent-browser --session-name x state load /tmp/x-auth.json`."
- Session expired (redirected to login page) → skip with re-import instructions (see Step 2a)
- Timeline fails to load → report and skip
- Individual account or list page fails → skip that account/list, continue with others
- Rate limiting or temporary block → report and skip
