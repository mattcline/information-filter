# X/Twitter — Source Strategy

## Overview

X requires authentication for timeline access. Use agent-browser to log in, scroll the timeline, and extract post content. Also check specific accounts and lists from config.

## Prerequisites

1. agent-browser installed locally: `npm install` in the plugin directory
2. Chrome for Testing installed: `npx agent-browser install`
3. X session imported via browser:
   ```bash
   # Quit Chrome entirely (Cmd+Q), then relaunch with remote debugging:
   "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
     --remote-debugging-port=9222 \
     --user-data-dir=/tmp/chrome-debug-profile

   # In that Chrome window, navigate to x.com and log in normally
   # (handle 2FA, CAPTCHAs, etc. as you normally would)

   # In a separate terminal, extract the auth_token cookie via CDP:
   AUTH_TOKEN=$(curl -s http://localhost:9222/json/version | node -e "
     const http=require('http'),net=require('net'),crypto=require('crypto');
     let d='';process.stdin.on('data',c=>d+=c);process.stdin.on('end',()=>{
       const wsUrl=JSON.parse(d).webSocketDebuggerUrl,url=new URL(wsUrl),
         key=crypto.randomBytes(16).toString('base64'),
         sock=net.connect(parseInt(url.port)||9222,url.hostname,()=>{
           sock.write('GET '+url.pathname+' HTTP/1.1\r\nHost:'+url.host+
             '\r\nUpgrade:websocket\r\nConnection:Upgrade\r\nSec-WebSocket-Key:'+
             key+'\r\nSec-WebSocket-Version:13\r\n\r\n');
         });
       let hs=false,buf=Buffer.alloc(0);
       sock.on('data',chunk=>{buf=Buffer.concat([buf,chunk]);
         if(!hs){const i=buf.indexOf('\r\n\r\n');if(i>=0){hs=true;buf=buf.slice(i+4);
           const msg=Buffer.from(JSON.stringify({id:1,method:'Storage.getCookies'})),
             mask=crypto.randomBytes(4),fr=Buffer.alloc(6+msg.length);
           fr[0]=0x81;fr[1]=0x80|msg.length;mask.copy(fr,2);
           for(let j=0;j<msg.length;j++)fr[6+j]=msg[j]^mask[j%4];sock.write(fr);}}
         else{if(buf.length>2){let pl=buf[1]&0x7f,o=2;
           if(pl===126){pl=buf.readUInt16BE(2);o=4;}
           if(buf.length>=o+pl){const r=JSON.parse(buf.slice(o,o+pl).toString());
             if(r.id===1){const c=r.result.cookies.find(x=>x.name==='auth_token');
               if(c)process.stdout.write(c.value);process.exit(0);}}}}});
       setTimeout(()=>process.exit(1),5000);
     });")

   # Close debug Chrome, then set cookie in agent-browser:
   npx agent-browser --session-name x open "about:blank"
   npx agent-browser --session-name x cookies set auth_token "$AUTH_TOKEN" \
     --domain .x.com --path / --httpOnly --secure --sameSite None

   # Verify — should show "Home / X", not a login page:
   npx agent-browser --session-name x open "https://x.com/home"
   ```

   > **Note:** The `state save` / `state load` workflow has a bug where it scopes
   > cookie extraction to the wrong tab, producing empty cookies. The direct
   > `cookies set` approach above bypasses this issue.

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
> "X/Twitter skipped — session expired. Re-import your session using the steps in `skills/info-filter/references/x-twitter.md` under Prerequisites."

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
   - **Tweet permalink URL** — for each tweet article element, find the `<a>` element that contains a `<time>` element (this is the timestamp link that points to the tweet permalink). Extract its `href` attribute and construct the full URL as `https://x.com{href}` (the href is typically in the format `/{handle}/status/{tweet_id}`).
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
  url: {tweet permalink URL (e.g., https://x.com/handle/status/123), or embedded article URL}
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
- Session not configured → skip with message: "X/Twitter skipped — no session configured. See `skills/info-filter/references/x-twitter.md` Prerequisites for import steps."
- Session expired (redirected to login page) → skip with re-import instructions (see Step 2a)
- Timeline fails to load → report and skip
- Individual account or list page fails → skip that account/list, continue with others
- Rate limiting or temporary block → report and skip
