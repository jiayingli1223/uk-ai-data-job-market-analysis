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
## 2026-08-10 — Skill matching requires more than word boundaries

Naive substring matching is unusable: "r" matched 100% of JDs, "excel"
58%, "scala" 39%, "rag" 49% — all as fragments of other words
(excellent, scalable, average). Word boundaries fixed most of it.

Four terms needed custom patterns beyond `\b`:
- `r` still matched "R&D" (& is a valid boundary) — excluded explicitly
- `excel` still matched the verb ("you'll excel at…") — excluded
- `git` missed GitHub and GitLab, which do imply the skill — included
- `sql` missed PostgreSQL, MySQL, NoSQL variants — included

Manual review of 8 hits per term after correction found no false
positives. Metric is share of JDs containing a term, never occurrence
count: a senior ML role mentions "python" once.

**"Manager" is not a seniority marker in UK job titles.** It appears in
functional roles (Product Manager, Programme Manager) as often as in
managerial ones, so it was removed from the lead pattern.

**A zero in the sample is not a zero in the market.** No title in the
183-JD sample contained "junior", which I initially read as a market
characteristic. Checking all 1,336 titles found 15 (1.1%) — the sample
simply missed them. Any claim about a category must be checked against
the full title data before it is stated.

**Entry-level roles are ~9% of all postings** (graduate 42, trainee 43,
junior 15, placement 15, entry 2 of 1,336). The random JD sample
contains only 13 graduate-level roles, where one posting moves a
percentage by 7.7 points. The 2026-08-14 decision gate on AWS
prevalence at graduate level cannot be answered at this sample size;
entry-level postings will be scraped specifically.

**Scraping throttle carries across days.** The 2026-08-10 run started at
44% success rather than the 77% seen at the start of the previous day's
run, having made ~300 requests the day before.
## 2026-08-11 — Sponsorship status is mostly absent from JD text

**87% of JDs say nothing about visa status** (187 of 214). Explicit
refusals: 14 (6.5%). Explicit offers: 3 (1.4%). Ambiguous mentions: 10.

This answers the original question in an unexpected way. The plan was to
measure what share of postings refuse sponsorship and compare across job
families. That comparison is not possible — any subgroup has single-digit
counts. The usable finding is the absence itself: **sponsorship status
cannot be determined from a job posting before applying**, so filtering a
job pipeline on JD text is not viable. An external source is required.

**Entry-level roles refuse more often.** Explicit refusal by seniority:
graduate 7/41 (17.1%), senior 2/41 (4.9%), lead 1/25 (4.0%). Both
graduate and senior groups have n=41, so the comparison is on equal
footing. No graduate posting explicitly offered sponsorship. This is
consistent with the Skilled Worker salary floor sitting above typical
graduate pay. Absolute counts are small — direction is credible, magnitude
is not.

**Regex classification hit its ceiling at ~70% precision.** Manual review
of all 20 classified records found four recurring failure modes:
1. `unable to offer` vs `not able to offer` — different word order
2. Contractions without apostrophes (`isnt`, `dont`)
3. `sponsor` in non-visa senses: sponsoring industry registrations,
   funding women's university degrees, a role titled "Sponsorship Analyst"
4. Words inserted mid-phrase (`must have uk right to work`,
   `only accept applicants for this role who have a right to work`)

Modes 1–3 were fixed. Mode 4 was not: catching it needs loose separators
that raise false positives, and each new batch brings new variants. Recall
is therefore below precision — some refusals remain inside the 187
`not_mentioned` records, so 6.5% is a floor, not an estimate.

**Training providers were excluded before scraping.** Of 111 unscraped
entry-level postings, 64 (58%) came from four course sellers advertising
as employers: ITOL Recruit, IT Online Learning, NEWTO TRAINING LIMITED,
IT Career Switch. Their text describes what a student would learn, not
what a job requires, and would have distorted graduate-level skill
frequencies. Scraping the remaining 47 yielded 31 JDs (66%).
## 2026-08-12 — External sponsor register recovers what JD text cannot

Home Office Register of Licensed Sponsors (142,701 rows, 2026-08-12
edition), filtered to the Skilled Worker route (122,767 rows), joined
against employer names from the postings.

Exact string matching hit 21 of 658 companies (3.2%). After
normalisation — lowercase, strip `T/A` suffixes, `&`→`and`, drop
punctuation and generic tokens (Ltd, Limited, UK, Group, Services,
Solutions) — 263 of 658 matched (40.0%).

**Normalisation trades false negatives for false positives.** Stripping
generic tokens collapses distinct companies onto the same key: "Wise"
(the fintech) matched "UK Wise Group Ltd", an unrelated organisation.
Matches were therefore graded rather than treated as binary. A match is
high-confidence when the normalised name is a distinctive token
(convatec, gohenry, moniepoint); low when it is a common English word
(wise, beyond, trace, salt) that will inevitably collide across 120k
organisations.

By posting: 27.3% high, 6.7% medium, 5.7% low, 60.3% unmatched.
Excluding agencies, **344 postings (25.8%) can be verified as
direct employers holding a Skilled Worker licence before applying** —
against 1.4% derivable from JD text alone.

**Agencies match at 12.3% versus 43.4% for direct employers, which is
backwards.** Recruitment agencies employ consultants and are more likely
to hold a licence, not less. The gap is a measurement artefact: agency
listings use marketing names carrying business descriptions ("Harnham -
Data & Analytics Recruitment", "Trace | Expert Accountancy & Finance
Recruitment") that normalisation cannot reduce to a registered name.
Agencies therefore drop out of the matched set for the wrong reason.
This happens to help — an agency's licence says nothing about whether
the end employer sponsors — but it is luck, not design. Improving name
normalisation would pull agencies into the matched set and change what
the 25.8% figure means.

**Unmatched does not mean cannot sponsor.** The 806 unmatched postings
carry an unknown share of false negatives. 344 is a floor on verifiable
opportunities, not a count of viable ones.
## 2026-08-12 — Note for cleanup

Output filenames hard-code a date, and the date appears in several cells
per notebook. Re-running `04_skills` against newer input overwrote the
previous day's output because the save cell still carried the old date.
No data was lost — raw JD text is retained — but the pattern is fragile.
Replace with a single `RUN_DATE` constant, or drop dates from filenames
entirely, during the 8/17 cleanup.
## 2026-08-14 — Salary gaps between job families are a seniority artefact

Job family assigned from `title` by ordered pattern matching, same
approach as seniority. Order encodes precedence: ML engineer before
software engineer so "Machine Learning Software Engineer" lands in ML;
specific analyst types before a generic `analyst` catch-all.

**Teaching roles and course adverts were excluded outright** (44
postings): maths teacher listings pulled in by keyword overlap, and
training providers advertising "Placement Programme No Experience
Required" as if they were vacancies.

Of 396 stated salaries, only four families clear n=38: data_analyst 87,
data_scientist 86, other_analyst 66, ml_engineer 38. Software engineering
(13), AI engineering (16) and data engineering (9) are reported nowhere —
too few to support any statistic.

**Headline medians suggest a 2.3x spread** — data analyst £46.0k, other
analyst £57.9k, data scientist £83.3k, ML engineer £106.3k.

**Splitting by seniority reverses it.** At graduate level the families
converge: data analyst £40.0k (n=25), data scientist £35.0k (n=13), other
analyst £34.0k (n=9). The separation appears only at senior level (£70.0k
/ £95.0k / £65.0k). The headline spread mostly reflects family
composition — 25 of 87 data analyst salaries are graduate-level against 2
of 38 for ML engineering — not a premium attached to the family itself.
ML engineering has too few graduate salaries (n=2) to include in the
comparison, so the claim covers analyst and scientist families only.

Range midpoints are used rather than `salary_min`: employers advertising a
band typically pay inside it, so the lower bound understates.

**Entry-level and senior postings ask for different tool categories.**
Comparing graduate and senior postings (n=41 each), only three of 29
skills above the 5% threshold are more common at graduate level, and all
three are BI tools: Power BI (29.3% vs 4.9%), Excel (26.8% vs 9.8%),
Tableau (17.1% vs 4.9%). Every other skill leans senior, led by Python
(+29.3pp), Azure and Git (+22.0pp each). Git appears in zero graduate
postings.

**Skill co-occurrence was dropped.** With 214 JDs, the expected count for
a pair like Python × AWS is around 18, and splitting by seniority puts
every cell in single digits. Nothing at that size would survive being
written down.

**Chart titles describe, findings go in prose.** A first draft titled a
chart "The gap between job families opens at senior level, not entry
level" — a claim resting on cells of n=9 to n=25, and one the chart could
not support for ML engineering at all, where the graduate bar is absent.
Titles state what is plotted and carry n; the caveats live in text that
has room for them.
## 2026-08-14 (later) — Salary figures restricted to London

The headline medians above are inflated by sample composition, not by the
job families themselves. For data scientist roles, `salary_min` alone has
a median of £76.5k — high for the UK — and the reason is visible in the
breakdown: 73% of those postings are London, and senior plus lead account
for 42% of them against 15% graduate.

The skew is structural. London returns far more postings per keyword than
any other region (2,316 versus 160 for "data scientist" in Manchester),
and the 250-record page cap only ever binds in London, so a fixed number
of keywords across five regions cannot produce a nationally balanced
sample.

London share also varies by family — 84% for ML engineering against 61%
for the other-analyst group — so pooling regions distorts between-family
comparisons too, not just absolute levels.

**All salary figures are therefore restricted to London** (237 of 396
stated salaries). Medians at graduate and senior level are unchanged by
this restriction — data analyst £40k/£70k, data scientist £35k/£95k,
other analyst £34k/£65k — so the earlier finding stands; only the stated
scope changes. Sample sizes shrink (data scientist graduate n=13 → 9),
which does not alter how much weight those cells can carry: direction
credible, magnitude not.

Salary claims in the README will say London, not the UK.
## 2026-08-15 — Sponsor status against salary: attempted, not concluded

The question was whether postings from licensed sponsors differ from
unmatched ones. Raw medians differ by £22.5k (£87.5k, n=56, versus
£65.0k, n=310), but two confounders account for it and neither can be
controlled at this sample size.

Agencies make up 37.7% of unmatched postings against 17.9% of
high-confidence ones — a direct consequence of the name-matching
limitation recorded on 8/12, since agency listings carry marketing names
that do not reduce to registered ones. Restricting to direct employers
narrows the gap to £85k versus £65k but does not close it.

Job family composition then explains most of what remains: 45.7% of
high-confidence direct-employer salaries are data scientist roles against
16.6% of unmatched ones, and the medium-confidence group is 37% ML
engineering, which is why its median (£114k) sits above the
high-confidence group's. Controlling for family as well leaves 21 versus
32 records in the only cell large enough to look at.

No conclusion is drawn. The comparison is recorded because the confounding
structure is itself informative: which employers match the register is
not independent of what kind of employer they are.

## 2026-08-15 — Skill comparison is not affected by the regional skew

The salary figures needed restricting to London because London
## 2026-08-16 — Credentials in notebooks

API credentials live in a notebook cell and are cleared by hand before
each commit. That held for nine days but not from the start: a key
reached the history on 8/8 and survives in the diff of that commit, even
though every current file is clean.

The key was revoked rather than the history rewritten. Rewriting would
change every commit hash, and the commit history is part of what this
repository is meant to show; a free-tier key is not worth that trade.

Two fixes before v2, in order of what they actually solve: move
credentials to a gitignored `.env` read via `os.environ`, so they are
never in a cell to begin with; and install `nbstripout` so notebook
outputs are stripped at commit time rather than by memory. The second
alone would not have prevented this — the key was in cell source, not
output.