# /get-info — Information Filter

Fetch, score, and surface relevant content from Hacker News, LinkedIn, and X/Twitter based on your goals and interests.

## Arguments

`$ARGUMENTS` may include:

- `--source <name>` — fetch from a single source only: `hn`, `linkedin`, `x`. Can be repeated (e.g., `--source hn --source x`).
- `--topic <keyword>` — focus this run on a specific topic, boosting relevance for items matching it.
- `--min-score <n>` — override the minimum score threshold for this run.
- `--max <n>` — override the maximum number of items to display.

If no arguments are provided, fetch from all enabled sources using config defaults.

## Workflow

Execute the following 7 phases in order. Each phase must complete before starting the next.

---

### Phase 1: Setup

1. Read the user's config from `data/config.yml` relative to this plugin's directory.
   - If `data/config.yml` does not exist, tell the user to copy `data/config.example.yml` to `data/config.yml` and customize it, then stop.
2. Read the skill strategy from `skills/info-filter/SKILL.md`.
3. Read the scoring rubric from `skills/info-filter/references/scoring.md`.
4. Parse `$ARGUMENTS` to determine:
   - Which sources to fetch (default: all enabled sources from config)
   - Any topic focus override
   - Any min-score or max-items overrides
5. Initialize an empty in-memory list for collected items (used for dedup across sources).
6. Initialize a source status tracker to record success/failure for each source.

---

### Phase 2: Fetch Hacker News

Skip this phase if HN is not in the active source list or `sources.hacker_news.enabled` is false.

1. Read the HN strategy from `skills/info-filter/references/hacker-news.md`.
2. Follow the fetching workflow documented there:
   - Fetch top story IDs (and best story IDs if configured) via WebFetch
   - Fetch item details for the first N stories
   - Quick filter by title, score, and negative signals
   - Fetch article content for surviving stories (if configured)
   - Score each item using the rubric
3. Add passing items to the in-memory collected list.
4. Record HN status: success (with count of items fetched and passed) or failure (with reason).

---

### Phase 3: Fetch LinkedIn

Skip this phase if LinkedIn is not in the active source list or `sources.linkedin.enabled` is false.

1. Read the LinkedIn strategy from `skills/info-filter/references/linkedin.md`.
2. Check that agent-browser is available by running `npx agent-browser --version`.
   - If not available, record failure: "agent-browser not installed. Run `npm install` then `npx agent-browser install`." and skip to Phase 4.
3. Follow the fetching workflow documented there:
   - Launch agent-browser with `--session-name linkedin`
   - Validate auth (check for login redirect)
   - Scroll feed and capture posts
   - Check specific profiles if configured
   - Parse and normalize posts
   - Quick filter (drop noise: congrats, engagement bait, negative signals)
   - Fetch shared articles for surviving posts
   - Score each item using the rubric
4. Add passing items to the in-memory collected list.
5. Record LinkedIn status: success (with count) or failure (with reason).

---

### Phase 4: Fetch X/Twitter

Skip this phase if X is not in the active source list or `sources.x_twitter.enabled` is false.

1. Read the X strategy from `skills/info-filter/references/x-twitter.md`.
2. Check that agent-browser is available (may already be confirmed from Phase 3).
   - If not available, record failure with install instructions and skip to Phase 5.
3. Follow the fetching workflow documented there:
   - Launch agent-browser with `--session-name x`
   - Validate auth (check for login redirect)
   - Scroll timeline and capture posts
   - Check specific accounts and lists if configured
   - Parse and normalize posts
   - Quick filter (drop noise: empty reposts, engagement farming, negative signals)
   - Fetch embedded articles for surviving posts
   - Score each item using the rubric
4. Add passing items to the in-memory collected list.
5. Record X status: success (with count) or failure (with reason).

---

### Phase 5: Dedup

Remove duplicate content that appeared across multiple sources.

1. Sort the collected items by `final_score` descending.
2. Walk the list in order. For each item, check if substantially the same content has already been kept:
   - Same URL → duplicate
   - Same title or headline with minor wording differences → duplicate
   - Same core topic covered by a higher-scored item → duplicate (keep the better one)
3. When a duplicate is found, keep the version with the higher `final_score`. Discard the other.
4. Track how many duplicates were removed (for the stats report).

---

### Phase 6: Score & Rank

1. Apply any `--topic` boost: if a topic argument was provided, add +1.0 to Topic Relevance for items matching that keyword, then recalculate `final_score`.
2. Re-sort all remaining items by `final_score` descending.
3. Apply the display threshold: drop items below `display.min_score` (or `--min-score` override).
4. Apply the max items limit: keep the top N items per source, where N = `display.max_items_per_source` (or `--max` override).

---

### Phase 7: Present Results

#### If items were found:

1. **Group by topic cluster** (if `display.group_by_topic` is true):
   - Cluster items by their primary matching goal or topic from the user's config.
   - Within each cluster, sort by `final_score` descending.
   - If `group_by_topic` is false, present as a flat list sorted by score.

2. **Display each item** in this format:

   ```
   ### {title}
   **Score: {final_score}** | {source} | {author} | {age}

   {summary — 2-3 sentences}

   **Why relevant:** {1 sentence connecting to user's goals}

   {url}
   ```

3. **Scan stats** — at the end, show a summary:

   ```
   ---
   ## Scan Summary
   - **Sources scanned:** {list of sources attempted}
   - **Items fetched:** {total raw items before filtering}
   - **Items after filtering:** {count after scoring + dedup}
   - **Items displayed:** {final count shown}
   - **Duplicates removed:** {count}
   ```

4. **Source failures** — if any source failed, list them:

   ```
   ### Source Issues
   - {source}: {failure reason + remediation instructions}
   ```

#### If no items were found:

Report that no items passed the scoring threshold. Suggest:
- Lowering `display.min_score` in config
- Broadening topics or goals
- Checking if sources are configured and auth is set up

#### If all sources failed:

List each failure with its remediation instructions. Common fixes:
- `npm install` + `npx agent-browser install` for agent-browser issues
- Session import instructions for auth issues (launch Chrome with `--remote-debugging-port=9222`, log in, then use `npx agent-browser --auto-connect state save` + `npx agent-browser --session-name <name> state load`)
- Check network connectivity for API failures
