# /generate-resume

Generate a tailored executive resume for a specific opportunity.

## Arguments

```
/generate-resume [--workdir <path>] [<slug>]
```

- `--workdir <path>` — optional override for the reswork directory

## Instructions

1. **Resolve the work directory.**
   - If `--workdir <path>` was passed as an argument, use that path.
   - Otherwise run:
     ```bash
     echo "${PERSONAL_WORKDIR:-$HOME/reswork}"
     ```
   Use the printed path as `$WORKDIR` for all file operations below.

2. Ask the user for the **opportunity slug** (e.g., `microsoft_vp_gtm`) if not already provided. If they are unsure, read `$WORKDIR/index.json` and show the available opportunities.

3. Read the opportunity file at `$WORKDIR/opportunities/[slug]/opportunity.json`. Extract `key_requirements` and `posting_text`.

4. Read all input data files:
   - `$WORKDIR/inputs/profile.json`
   - `$WORKDIR/inputs/workhistory.csv`
   - `$WORKDIR/inputs/achievements.csv`
   - `$WORKDIR/inputs/eduhistory.csv`
   - `$WORKDIR/inputs/extras.csv`

5. **Select and filter achievements.** Target a full 2-page resume — if the selection would leave the resume under ~1.8 pages, add more bullets. Filter using these rules:

   | Rule | Detail |
   |---|---|
   | Selection priority | Semantic match between achievement `themes` and posting `key_requirements` |
   | Current/recent role | 5–7 bullets |
   | Mid-career roles | 3–5 bullets per role |
   | Early career (`is_early_career=true`) | 1–2 bullets only, collapsed format |
   | Confidence ranking | Prefer high > medium > low; use low only if no better match |
   | Subheadings | Create/rename subheadings to align with the posting's language. Subheadings ARE printed on the resume for structure. **Maximum 3 subheadings per job** — if more themes exist, consolidate into broader groupings |
   | Narratives | Pre-polished but you MAY rewrite to better tune emphasis for the target role |

6. **Gap check:** Before generating, compare the posting's key requirements against matched achievements. If any key requirement has ZERO matching achievements, flag it to the user and ask whether to proceed or adjust.

7. Generate the resume with these sections:

   - **Header** — Name, phone, email, LinkedIn from profile.json. Name centered 14pt bold. Contact line centered 9pt with " | " separators.

   - **Summary** — 2–3 sentences. Target 50–70 words (renders to ~4–5 lines). Lead with scope and scale, then execution approach, then differentiator. Evidence-grounded — no adjectives without metrics.

   - **Core Strengths** — Derive 5–8 strengths directly from the selected achievements and posting requirements. Do not copy profile.json strengths verbatim; use them as a reference pool and source of phrasing, but synthesize strengths that specifically reflect the evidence in the selected achievements and the posting's language. Joined with " | " separators.

   - **Professional Experience** — For each non-early-career job in workhistory.csv (in chronological order, most recent first):
     - Company — Title line with Location | Dates right-aligned
     - Scope line in italic (if present in workhistory.csv)
     - Selected achievements grouped under thematic subheadings (italic bold)
     - Each bullet is a complete achievement narrative

   - **Early Career** — Jobs flagged `is_early_career=true`. Abbreviated: Company — Title + Location | Dates, then 1–2 bullet lines only.

   - **Education** — From eduhistory.csv. School bold, degree + location + date.

   - **Thought Leadership** — From extras.csv. Include only if relevant to the posting. Pass the `url` field through to the JSON payload so hyperlinks are rendered in the output.

8. Build the JSON payload and pipe it to the formatting script:

   The JSON must follow this structure:
   ```json
   {
     "type": "resume",
     "slug": "<slug>",
     "output_dir": "$WORKDIR/opportunities/[slug]",
     "header": {"name": "...", "phone": "...", "email": "...", "linkedin_url": "..."},
     "summary": "...",
     "core_strengths": ["...", "..."],
     "experience": [
       {
         "company": "...",
         "title": "...",
         "location": "...",
         "dates": "...",
         "scope": "...",
         "sections": [
           {"subheading": "...", "bullets": ["...", "..."]}
         ]
       }
     ],
     "early_career": [
       {"company": "...", "title": "...", "location": "...", "dates": "...", "bullets": ["..."]}
     ],
     "education": [
       {"school": "...", "degree": "...", "location": "...", "graddate": "..."}
     ],
     "extras": [
       {"title": "...", "url": "...", "description": "...", "date": "..."}
     ]
   }
   ```

   IMPORTANT: Write the JSON to a temporary file and pipe it, rather than using inline echo, to avoid shell escaping issues with quotes in the content. Substitute the resolved `$WORKDIR` and slug values into `output_dir` before writing the JSON.

   ```bash
   python3 "${JOBCRAWLER_DIR:-~/code/jobcrawler}/scripts/to_docx.py" < /tmp/resume_payload.json
   ```

9. Read the output path printed by the script and confirm it to the user.

10. Update the opportunity file: read `$WORKDIR/opportunities/[slug]/opportunity.json`, append to the `generated_artifacts` array:
    ```json
    {
      "type": "resume",
      "filename": "<filename from output path>",
      "generated_at": "<ISO 8601 timestamp>"
    }
    ```
    Write the updated JSON back.
