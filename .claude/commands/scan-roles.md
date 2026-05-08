# /scan-roles

Scan job boards and company career pages for roles matching the user's target titles, compensation floor, and positioning profile. Score, deduplicate, and optionally add matches to the pipeline.

## Arguments

```
/scan-roles [--workdir <path>] [--titles "VP Strategy,Director Ops"] [--min-comp 250000] [--companies all|tier1|<name>] [--boards all|linkedin,indeed,glassdoor]
```

- `--workdir <path>` — optional override for the reswork directory
- `--titles` — comma-separated title patterns (overrides profile defaults)
- `--min-comp` — minimum total compensation floor (overrides profile default)
- `--companies` — `all`, `tier1` (tier<=2 or priority_score>=7), or a specific company name
- `--boards` — `all` or comma-separated list from: `linkedin`, `indeed`, `glassdoor`, `lever`, `greenhouse`

## Instructions

### Step 1 — Resolve the work directory

- If `--workdir <path>` was passed, use that path.
- Otherwise run:
  ```bash
  echo "${PERSONAL_WORKDIR:-$HOME/reswork}"
  ```
  Use the printed path as `$WORKDIR` for all file operations below.

### Step 2 — Read configuration and defaults

Read these files to assemble scan parameters:

| File | What to extract |
|---|---|
| `$WORKDIR/inputs/profile.json` | `compensation_targets.min` (salary floor), `title_targets` (default title patterns) |
| `$WORKDIR/inputs/companies.csv` | Target company list — columns include `company`, `tier`, `priority_score`, `careers_url`, `comp_override`, `ats_platform` |
| `$WORKDIR/inputs/workhistory.csv` | Recent job titles as fallback title patterns (if `title_targets` is missing) |
| `$WORKDIR/scan_config.json` | Persistent scan preferences and thresholds (see defaults below) |

**`scan_config.json` defaults** (create this file on first run if it does not exist):
```json
{
  "default_board_scope": ["linkedin", "indeed", "glassdoor", "lever", "greenhouse"],
  "default_company_filter": "tier<=2 OR priority_score>=7",
  "scan_cooldown_hours": 24,
  "max_websearch_calls": 10,
  "max_webfetch_calls": 5,
  "min_match_grade_to_display": "C",
  "duplicate_title_similarity_threshold": 0.7,
  "duplicate_date_window_days": 14
}
```

Apply CLI overrides in this priority order (highest wins):
1. CLI arguments (`--titles`, `--min-comp`, `--companies`, `--boards`)
2. `scan_config.json` values
3. `profile.json` values
4. `workhistory.csv` fallback (titles only)

**Cooldown check:** Read `$WORKDIR/scan_history.json` (if it exists). If the most recent entry's `scan_date` is within `scan_cooldown_hours` of now, warn the user and ask whether to proceed.

If `profile.json` is missing `compensation_targets` or `title_targets`, tell the user these fields are needed and show the expected format:
```json
{
  "compensation_targets": { "min": 250000, "target": 300000, "max": 400000, "currency": "USD" },
  "title_targets": ["VP Strategy and Operations", "Senior Director Strategy", "Chief Strategy Officer"]
}
```
Ask the user to update `profile.json` before continuing, or accept inline values for this scan only.

### Step 3 — Build target profile for match scoring

Assemble an in-memory target profile used for scoring in Step 9:

1. **Title patterns** — from Step 2 resolution
2. **Theme clusters** — read `$WORKDIR/inputs/achievements.csv`, collect all unique values from the `themes` column (semicolon-separated within each cell), deduplicate, and group into clusters by semantic similarity
3. **Requirement fingerprints** — read `$WORKDIR/index.json`, find opportunities that have resume artifacts in `generated_artifacts`. For each, read `opportunities/[slug]/opportunity.json` and extract `key_requirements`. Collect all unique requirements across opportunities.
4. **Compensation floor** — resolved from: CLI `--min-comp` > per-company `comp_override` in `companies.csv` > `profile.json` `compensation_targets.min`

### Step 4 — Search job boards

Search for matching roles. Respect `max_websearch_calls` (default 10) and `max_webfetch_calls` (default 5) from `scan_config.json`. Track call counts and stop when limits are reached.

**Company filtering:**
- `--companies all` → use all companies in `companies.csv`
- `--companies tier1` → filter to `tier<=2 OR priority_score>=7`
- `--companies <name>` → single company match

**Job board searches** — use WebSearch with `site:` queries. Use snippet data only. Do NOT WebFetch job board URLs (login walls).

Example queries:
```
site:linkedin.com/jobs ("VP Strategy" OR "Director Strategy and Operations")
site:indeed.com/viewjob ("VP Strategy" OR "Director Business Operations")
site:glassdoor.com/job ("VP Strategy" OR "Director Strategy")
```

**Company career page searches** — for each target company with a `careers_url` or known `ats_platform`, search their career pages:
```
site:jobs.lever.co/companyname "strategy" OR "operations"
site:boards.greenhouse.io/companyname "VP" OR "Director"
site:careers.company.com "VP" OR "Director" strategy
```

Batch 3–4 companies into a single WebSearch query where possible to conserve rate limits.

**WebFetch rules** (count against `max_webfetch_calls`):

| Source | WebFetch? | Reason |
|---|---|---|
| LinkedIn Jobs | **Never** | Login wall |
| Indeed | **Never** | Partial login wall |
| Glassdoor | **Never** | Login wall |
| Lever (`jobs.lever.co`) | **Yes — always** | Public, reliable |
| Greenhouse (`boards.greenhouse.io`) | **Yes — always** | Public, reliable |
| Company career pages | **Attempt** | Fall back to snippet if fetch fails |

### Step 5 — Parse results

For each found role, extract into a structured record:

| Field | Source |
|---|---|
| `source` | Board name or career page domain |
| `company` | Company name |
| `role_title` | Job title |
| `location` | Location if visible |
| `salary_range` | `{min, max, currency}` if visible in snippet or posting |
| `posting_url` | URL to the posting |
| `snippet` | Search result snippet text |
| `posting_text` | Full text (only from successful WebFetch) |
| `posting_date` | Date if visible |

### Step 6 — Filter by compensation floor

This is a **hard filter**, not a scoring dimension.

For each result:
- If the role's stated salary range **maximum** is below the applicable compensation floor (per-company `comp_override` from `companies.csv`, or global floor from Step 2), **exclude it entirely**.
- If **no salary is listed**, **keep the role** — do not penalize missing data. Most executive postings omit compensation.

Track the count of excluded roles to report in the summary.

### Step 7 — Deduplicate across sources

If the same role appears on multiple sources (e.g., LinkedIn + company career page):
- Merge into one entry
- Prefer the career page URL over the board URL
- Prefer full posting text over snippet
- Note all sources in the record

### Step 8 — Duplicate detection against existing pipeline

Read `$WORKDIR/index.json` and for each existing opportunity read `opportunities/[slug]/opportunity.json` to get `company`, `role_title`, and `posting_url`.

Apply these three rules **in order** to each scan result:

**Rule 1 — URL match:** If `posting_url` matches an existing opportunity's `posting_url` → mark as `DUPLICATE_EXACT`. Hide from results, count in summary.

**Rule 2 — Company + title fuzzy:** Same company (case-insensitive) + normalized title Jaccard similarity >= threshold (default 0.7) → mark as `DUPLICATE_LIKELY`. Show in results with a flag.

**Rule 3 — Company + title + date proximity:** Same company + title Jaccard similarity >= 0.5 + posting dates within `duplicate_date_window_days` (default 14) → mark as `DUPLICATE_POSSIBLE`. Show with flag.

Otherwise → mark as `NEW`.

**Title normalization for Jaccard:**
1. Lowercase
2. Strip prefixes: senior, sr, lead, principal
3. Replace synonyms: vp → vice president, dir → director, ops → operations, strat → strategy, biz → business
4. Tokenize on whitespace and punctuation
5. Remove stop words: of, the, and, for, &, in, at
6. Compute Jaccard similarity on resulting token sets: `|A ∩ B| / |A ∪ B|`

### Step 9 — Score matches

Score each `NEW` and `DUPLICATE_LIKELY` / `DUPLICATE_POSSIBLE` result on 4 dimensions:

| Dimension | Weight | 1 (Low) | 5 (High) |
|---|---|---|---|
| **Title Fit** | 30% | No keyword overlap with title targets | Exact or near-exact match to a title target |
| **Seniority Alignment** | 20% | 2+ levels away from user's trajectory | Same level as recent roles or natural next step |
| **Theme Overlap** | 35% | No overlap with achievement themes or prior resume requirements | 3+ theme clusters match |
| **Company Strategic Fit** | 15% | Company not in `companies.csv` | Company in `companies.csv` tier 1, high priority score |

**Overall score** = weighted average of the four dimensions, normalized to 0–100%.

**Letter grade:**
- **A** — 80–100%
- **B** — 60–79%
- **C** — 40–59%
- **D** — below 40%

Compensation is NOT a scoring dimension — it is a hard filter handled in Step 6.

### Step 10 — Display results and prompt for action

**Display format:**

Show results grouped by grade (A first, then B, then C). For each result:

```
[#N] [GRADE] Role Title — Company
     Location | Salary: $XXXk–$XXXk (or "Not listed")
     Source: linkedin, lever | Posted: YYYY-MM-DD
     [DUPLICATE_LIKELY — similar to existing: company_role_slug]
     Match: "Strong theme overlap with GTM and revenue operations experience.
             Title is an exact match to target list."
     Key requirements: requirement1, requirement2, requirement3
```

For **D-grade** results, show only a count: `Plus N other roles below C grade (not shown).`

**After displaying results, prompt:**
> Which roles would you like to add? Enter numbers (e.g., 1,2,4), "all-A", or "none".

**For each selected role:**

1. If full `posting_text` is available (from WebFetch), create the opportunity directly.
2. If only `snippet` is available, attempt WebFetch on the `posting_url`:
   - If it succeeds, use the full text.
   - If it fails (login wall), ask the user to paste the posting text.
3. Create the opportunity folder and file:
   - Directory: `$WORKDIR/opportunities/[slug]/`
   - File: `$WORKDIR/opportunities/[slug]/opportunity.json`
   - Use the same schema as `/add-opportunity` plus these scan-specific fields:
     ```json
     {
       "source": "scan-roles",
       "scan_date": "<ISO 8601>",
       "match_score": 85,
       "match_grade": "A",
       "salary_range": {"min": 250000, "max": 350000, "currency": "USD"},
       "posting_text_partial": false
     }
     ```
   - Set `posting_text_partial: true` if only snippet text was captured.
4. Append to `$WORKDIR/index.json` (same format as `/add-opportunity`).

Generate slugs using the same convention as `/add-opportunity`: lowercase, underscores, company + abbreviated role. Examples: `stripe_vp_strategy_bizops`, `google_dir_operations`.

For `id`, check existing subdirectories in `$WORKDIR/opportunities/` to determine the next available `opp-NNN` number.

### Step 11 — Write scan log

Append an entry to `$WORKDIR/scan_history.json` (create the file as an array if it does not exist):

```json
{
  "scan_date": "<ISO 8601>",
  "title_patterns": ["VP Strategy", "Director Strategy and Operations"],
  "sources_searched": ["linkedin", "indeed", "stripe.careers"],
  "results_found": 14,
  "results_after_dedup": 10,
  "duplicates_exact": 2,
  "duplicates_likely": 1,
  "duplicates_possible": 1,
  "comp_filtered": 3,
  "results_added": 3,
  "added_slugs": ["stripe_vp_strategy_bizops", "google_dir_operations"]
}
```

### Final summary

After all operations complete, display a summary:

```
Scan complete.
  Searched: 8 sources (linkedin, indeed, glassdoor, 5 career pages)
  Found: 14 results → 10 after cross-source dedup
  Filtered: 3 below comp floor
  Duplicates: 2 exact (hidden), 1 likely, 1 possible
  Scored: 4 results (2A, 1B, 1C)
  Added: 3 opportunities
```
