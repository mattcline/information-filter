# Scoring Rubric

Score every candidate item on these 4 dimensions (1-10 scale), then compute the weighted average.

## Dimensions

### 1. Topic Relevance (40%)

How closely does this item relate to the user's goals and topics?

| Score | Criteria |
|-------|----------|
| 9-10 | Directly addresses a stated goal or primary topic. The user would specifically seek this out. |
| 7-8 | Strongly related to a primary topic. Clear connection to user's interests. |
| 5-6 | Tangential or superficial connection. Related to a secondary topic. |
| 3-4 | Loosely related. Requires a stretch to connect to user's interests. |
| 1-2 | Off-topic. No meaningful connection to stated goals or topics. |

### 2. Source Quality (20%)

How credible and substantive is this content?

| Score | Criteria |
|-------|----------|
| 9-10 | Trusted source, original research, primary data, or recognized expert. In `trusted_sources` list. |
| 7-8 | Credible author with domain expertise. Well-sourced claims. |
| 5-6 | Unknown source but substantive content. Makes verifiable claims. |
| 3-4 | Thin content, unsourced claims, or opinion without evidence. |
| 1-2 | Spam, clickbait, engagement farming, or promotional fluff. |

### 3. Timeliness (20%)

How fresh is this information?

| Score | Criteria |
|-------|----------|
| 9-10 | Published in the last few hours. Breaking or just-announced. |
| 7-8 | Published today or yesterday. Still developing. |
| 5-6 | Published this week. Recent and relevant. |
| 3-4 | Published this month. Still useful but not urgent. |
| 1-2 | Older than a month. Only score high if it's a timeless resource surfacing for the first time. |

### 4. Actionability (20%)

Can the user do something with this information?

| Score | Criteria |
|-------|----------|
| 9-10 | Specific tool to try, technique to apply, paper to read, or decision to inform. Clear next step. |
| 7-8 | Provides a framework, approach, or resource the user could adopt. |
| 5-6 | Awareness-level information. Good to know but no immediate action. |
| 3-4 | Vague or abstract. Would need significant effort to extract value. |
| 1-2 | Pure noise. No actionable content. |

## Final Score Calculation

```
final_score = (topic_relevance * 0.4) + (source_quality * 0.2) + (timeliness * 0.2) + (actionability * 0.2)
```

Then apply modifiers:

- Add +0.5 for each applicable boost signal
- Subtract -0.5 for each applicable penalty signal
- Cap final score at 10.0

## Score Interpretation

| Score | Meaning |
|-------|---------|
| 8.0+ | Must-see. Highly relevant and actionable. |
| 6.0-7.9 | Worth reading. Relevant to user's interests. |
| 5.0-5.9 | Borderline. Include only if few higher-scored items exist. |
| <5.0 | Drop. Below default `min_score` threshold. |
