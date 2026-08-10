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
## 2026-08-08 — Cleaning decisions

**API `description` is truncated at 500 characters.** Across 1,433 raw
records: median, 75th percentile and max are all exactly 500, and 99.8%
end with an ellipsis. 500 characters typically covers only the company
boilerplate, so skill extraction on this field would sample only JDs that
happen to mention tools early. Sponsorship statements sit at the end of a
JD and never appear at all. Full JD text must come from elsewhere.

**`search_keyword` is unusable after deduplication.** A posting matched
by several keywords keeps only the first one after `drop_duplicates`.
Job family must be derived from `title` instead.

**`region` is derived from `location.area[1]`**, which is a UK region,
not a city. Searching "manchester" returns postings across North West
England. Regional grouping is therefore at region level; city-level
claims are not supported by this field.

**68% of salaries are Adzuna estimates** (905 of 1,336). Only 431 are
employer-stated. `salary_is_predicted` is the reliable filter; predicted
values also carry non-integer decimals, but 13 records break that pattern
so the decimal test is indicative only.

**Salary distribution is right-skewed** (mean £70.5k, median £65k, max
£370k, sd £41k). Report medians and quartiles; means are misleading here.
The high tail is genuine — top records are research engineer roles at
Anthropic and Wayve.

**Nine records below £10k were dropped**: six give only an upper bound,
two are day rates (£370, £250), one is a placeholder (£1). Day rates and
placeholders cannot be reliably separated at this size, so all nine were
removed, leaving 422.

**Training providers pollute the employer field.** IT Online Learning,
NEWTO TRAINING LIMITED and IT Career Switch appear as high-volume
"employers" but sell courses rather than hire. Company names also carry
case variants ("Anson Mccade" / "Anson McCade"), inflating the 658 unique
company count. Both to be handled at the next cleaning pass.

## 2026-08-09 — Full JD text is retrievable; collection is rate-limited

**`redirect_url` resolves on adzuna.co.uk itself**, not the employer's
site — no chained redirects to parse. Full JD text sits in the element
with class `adp-body` (~6.5k chars). Extracting the whole page instead
would inflate skill counts: the page footer carries a related-jobs block
whose text would be counted as if it were the JD.

**Term frequency within a JD is not a useful signal.** A senior ML
engineering role mentions "python" once. Skill metrics must be the share
of JDs containing a term, not total occurrences.

**Scraping succeeded on 98 of 150 sampled postings** (47 HTTP 403, 5 HTTP
404). Failures rise with cumulative requests — 77% success in the first
75, 53% in the last 75 — so this is throttling, not a static block.

Success rate varies by keyword: data analyst 49% (23/47) versus data
scientist 76% (44/58). Request order was ruled out as the cause —
keywords are evenly split across both halves of the run. Cause not yet
established. Data analyst postings will be prioritised when topping up
the sample.