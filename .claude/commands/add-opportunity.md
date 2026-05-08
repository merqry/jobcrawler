# /add-opportunity

Create a new opportunity from a job posting URL or pasted text.

## Arguments

```
/add-opportunity [--workdir <path>] <url>
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

2. Ask the user for the **job posting URL** (if not already provided as an argument).

3. Attempt to fetch the URL using the WebFetch tool.
   - If the page returns a login wall, CAPTCHA, or no useful job posting content, tell the user the URL could not be fetched and ask them to either:
     - **Paste the full posting text** directly into the chat, OR
     - **Provide a file path** to a `.txt` file containing the posting text (then read it with the Read tool)

4. From the posting content, extract:
   - **Company name**
   - **Role title**
   - **Full posting text** (preserve as-is)
   - **Key requirements** — 4 to 8 bullet points summarizing the most important qualifications, skills, and experience the role demands

5. Generate a **URL-safe slug** from company + abbreviated role title. Use lowercase, underscores, no special characters. Examples:
   - `microsoft_vp_gtm`
   - `stripe_head_revops`
   - `openai_dir_strategy`

6. Create the opportunity folder and write the opportunity file:
   - Create directory: `$WORKDIR/opportunities/[slug]/`
   - Write file: `$WORKDIR/opportunities/[slug]/opportunity.json`

   File structure:
   ```json
   {
     "id": "opp-NNN",
     "slug": "<slug>",
     "company": "<company>",
     "role_title": "<role title>",
     "posting_url": "<url or empty string>",
     "posting_text": "<full posting text>",
     "key_requirements": ["requirement 1", "requirement 2", ...],
     "contacts": [],
     "threads": [],
     "generated_artifacts": [],
     "stage": "targeting",
     "notes": ""
   }
   ```
   For the `id` field, use `opp-` followed by a 3-digit number. Check existing subdirectories in `$WORKDIR/opportunities/` to determine the next available number.

7. Read `$WORKDIR/index.json`, append an entry for this opportunity, and write it back:
   ```json
   {
     "slug": "<slug>",
     "company": "<company>",
     "role_title": "<role title>",
     "stage": "targeting",
     "created_at": "<ISO 8601 timestamp>"
   }
   ```
   If `index.json` does not exist, create it as an array with this single entry.

8. Confirm to the user:
   - The slug (so they can use it with other skills)
   - The file path
   - A brief summary of the extracted role title and key requirements
