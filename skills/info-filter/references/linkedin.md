# LinkedIn — Source Strategy

## Overview

LinkedIn requires authentication. Use agent-browser to log in, scroll the feed, and extract post content. No public API is available for feed access.

## Prerequisites

1. agent-browser installed locally: `npm install` in the plugin directory
2. Chrome for Testing installed: `npx agent-browser install`
3. LinkedIn session imported via browser:
   ```bash
   # Quit Chrome entirely (Cmd+Q), then relaunch with remote debugging:
   "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
     --remote-debugging-port=9222 \
     --user-data-dir=/tmp/chrome-debug-profile

   # In that Chrome window, navigate to linkedin.com and log in normally
   # (handle 2FA, CAPTCHAs, etc. as you normally would)

   # In a separate terminal, extract the li_at cookie via CDP:
   LI_AT=$(curl -s http://localhost:9222/json/version | node -e "
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
             if(r.id===1){const c=r.result.cookies.find(x=>x.name==='li_at');
               if(c)process.stdout.write(c.value);process.exit(0);}}}}});
       setTimeout(()=>process.exit(1),5000);
     });")

   # Close debug Chrome, then set cookie in agent-browser:
   npx agent-browser --session-name linkedin open "about:blank"
   npx agent-browser --session-name linkedin cookies set li_at "$LI_AT" \
     --domain .linkedin.com --path / --httpOnly --secure --sameSite None

   # Verify — should show "Feed | LinkedIn", not a login page:
   npx agent-browser --session-name linkedin open "https://www.linkedin.com/feed/"
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

If this fails, skip LinkedIn with the message:
> "LinkedIn skipped — agent-browser not installed. Run `npm install` in the plugin directory, then `npx agent-browser install`."

### Step 2: Launch Browser with Session

```bash
npx agent-browser --session-name linkedin open "https://www.linkedin.com/feed/"
```

The `--session-name linkedin` flag restores saved cookies/state and auto-saves them back after each run.

### Step 2a: Validate Auth

After the page loads, check the current URL. If it contains `/login`, `/uas/login`, or `/authwall`, the session has expired. Skip LinkedIn with the message:
> "LinkedIn skipped — session expired. Re-import your session using the steps in `skills/info-filter/references/linkedin.md` under Prerequisites."

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
   - **Post permalink URL** — for each post, extract the permalink by finding the `<a>` element whose `href` contains `/feed/update/` within the post's article element. Construct the full URL as `https://www.linkedin.com{href}`. If no permalink `<a>` is found, try extracting the `data-urn` attribute from the post's container element and construct the URL as `https://www.linkedin.com/feed/update/{urn}/`.
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
  url: {shared article URL, or LinkedIn post permalink URL}
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
- Session not configured → skip with message: "LinkedIn skipped — no session configured. See `skills/info-filter/references/linkedin.md` Prerequisites for import steps."
- Session expired (redirected to login page) → skip with re-import instructions (see Step 2a)
- Feed fails to load → report and skip
- Individual profile page fails → skip that profile, continue with others
