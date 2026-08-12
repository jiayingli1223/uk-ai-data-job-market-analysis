# UK AI/Data Job Market Analysis

Skill demand and visa sponsorship across UK data roles, built from the
Adzuna job board API and the Home Office register of licensed sponsors.

The interesting part of this project is not the numbers — it is what the
data would not support. Several planned analyses were dropped after the
underlying fields turned out to mean something other than their names
suggest. Those decisions are recorded in
[`docs/decisions.md`](docs/decisions.md).

## Data

| Source | What | Size |
|---|---|---|
| Adzuna API | Job postings, 5 keywords × 5 UK regions | 1,336 after dedup |
| Adzuna listing pages | Full JD text, scraped | 214 |
| Home Office | Register of Licensed Sponsors, Skilled Worker route | 122,767 |

The API returns a `description` field truncated at 500 characters, which
is not enough for skill extraction — full text had to be scraped
separately, and scraping is rate-limited, so the JD-level analyses run on
a 214-posting sample rather than all 1,336.

## Pipeline

Notebooks run in order. Each writes to `data/` and the next reads from it.

| Notebook | Does | Re-run cost |
|---|---|---|
| `01_fetch.ipynb` | Adzuna API collection | High — 250 requests/day cap |
| `02_clean.ipynb` | Dedup, unpack nested fields, filter predicted salaries | Low |
| `03_fetch_jd.ipynb` | Scrape full JD text | High — throttled |
| `04_skills.ipynb` | Skill matching, seniority classification | Low |
| `05_sponsorship.ipynb` | Classify visa statements in JD text | Low |
| `06_sponsors.ipynb` | Match employers to the sponsor register | Low |

Collection and analysis are split deliberately: the analysis notebooks can
be re-run freely, the collection ones cannot.

## Reproducing

1. Get an Adzuna API key at https://developer.adzuna.com/ and set
   `APP_ID` / `APP_KEY` at the top of `01_fetch.ipynb`.
2. Download the Skilled Worker register from
   https://www.gov.uk/government/publications/register-of-licensed-sponsors-workers
   and save it as `data/external/sponsors.csv`.
3. Run the notebooks in numerical order.

`data/` is gitignored. The register is republished frequently; results
here use the 2026-08-12 edition.

## Key findings

<!-- TODO 8/16 -->

## Limitations

**Adzuna's `count` is not a job count.** It reports documents containing
the search terms. Relevance decays to nothing well before the reported
total — page 40 of a 2,316-result query returned zero relevant titles.

**JD sample is not fully random.** Scraping succeeds on roughly 60% of
attempts and the failure rate climbs with cumulative requests, so the
214 scraped postings under-represent whatever was requested late in a run.

**Skill matching is keyword-based.** Word boundaries and per-term
disambiguation fix the obvious failures, but the method cannot read
context — a skill listed as desirable counts the same as one listed as
essential.

**Sponsor matching is a floor, not a census.** 60% of employers do not
match the register, and an unknown share of those are false negatives
caused by marketing names that differ from registered ones.

## Repository

```
data/
  raw/          # API output and scraped JD text (gitignored)
  processed/    # cleaned, analysis-ready (gitignored)
  external/     # Home Office register (gitignored)
docs/
  decisions.md  # why each analytical choice was made
notebooks/
```