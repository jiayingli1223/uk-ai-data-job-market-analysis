# Decision Log

## 2026-08-08 — Adzuna `count` is not a job count

**Observation.** Probed all 25 keyword × location combinations. London
returned implausible volumes: 2,316 for "data scientist" vs 772 for
"data analyst" — a 3x gap that does not match the UK market.

**Investigation.**
1. `what_phrase` (exact phrase match) only reduced 2,316 → 2,034,
   so loose word matching was not the main cause.
2. Sampled page 1 of "data scientist" / london: 50/50 titles were
   genuine data scientist roles.
3. Sampled page 40 (records ~1951-2000): 0/50 relevant. Titles included
   Medical Writer, MEP BIM Lead, Electrical Design Engineer,
   Career Coaching Lead.

**Conclusion.** Adzuna ranks by relevance but `count` includes weak
matches where the keyword appears anywhere in the description. Relevance
decays to zero well before the reported total. `count` measures documents
containing the terms, not open positions.

**Decision.** Do not use `count` as a market-size signal anywhere in this
analysis.

## 2026-08-08 — Kept max_pages=5 despite the 250-record truncation

Truncation affects only 3 of 25 combinations, all in London
(data analyst 772, data scientist 2,316, ML engineer 481). Non-London
combinations complete well under the cap, so their counts are exact.

Given the finding above, the truncated tail is largely noise, so raising
`max_pages` would add irrelevant records rather than recover missing
signal. Filtering by job title is the better lever, and it is applied at
the cleaning stage rather than at collection — collection-time filters
cannot be revised without re-spending API quota.

## 2026-08-08 — Rate limit constrains collection design

Free tier: 25 requests/minute, 250/day, 1,000/week, 2,500/month. Hit
HTTP 503 mid-run today after exceeding the daily allowance.

Implications: full re-collection costs roughly 130 requests, so at most
one full run per day. Any scheduled pipeline in v2 must be sized against
the daily cap, not the per-minute one.

## 2026-08-08 — Near-duplicate postings observed

Deduplication by `id` removed ~3% on 2026-08-06, but titles differing
only by whitespace appear in the raw data, e.g.
"Senior Product Data Scientist / Product Data Analyst" and
"Senior Product Data Scientist/ Product Data Analyst". These are likely
the same role posted through different agencies with distinct IDs.

`id` deduplication alone will undercount duplicates and inflate skill
frequencies for the affected employers. To be addressed during cleaning.