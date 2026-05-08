# /draft-outreach

Generate a 3-message outreach sequence for a contact at a target opportunity.

## Arguments

```
/draft-outreach [--workdir <path>] [<slug>]
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

2. Ask the user for (if not already provided):
   - **Opportunity slug** (e.g., `microsoft_vp_gtm`). If unsure, read `$WORKDIR/index.json` and show available opportunities.
   - **Contact name** (or ID). If unsure, read `$WORKDIR/inputs/contacts.csv` and show contacts at the relevant company.

3. Read the opportunity file at `$WORKDIR/opportunities/[slug]/opportunity.json`. Extract company, role_title, posting_text, and key_requirements.

4. Read `$WORKDIR/inputs/contacts.csv` and find the matching contact. Note their title, relationship, relationship_type, and notes.

5. Also read `$WORKDIR/inputs/profile.json` to understand the user's background for credibility signals.

6. Generate **three standalone messages**. Each message must stand on its own — a reader should understand the full context without having seen prior messages.

   **Tone:** Formal, professional, befitting an executive leader's communication style. No casual language.

   - **Initial Outreach** (<150 words)
     - For `relationship_type = cold`: Open with a strong hook — a relevant industry insight, shared challenge, or company-specific observation that demonstrates knowledge. Provide value before making any ask.
     - For warm contacts (`mutual`, `alumni`, `prior_colleague`): Leverage the relationship context naturally. Reference the connection without being transactional.
     - Close with a specific, low-friction ask (e.g., 15-minute conversation).

   - **Follow-up 1** (~1 week out, standalone)
     - Different angle from the initial message. Lead with a value-add: a relevant article, data point, industry trend, or perspective that would be useful to the contact regardless of whether they respond.
     - Do NOT reference the previous message ("Following up on my note...").

   - **Follow-up 2** (~2 weeks out, standalone)
     - Graceful close or pivot. Acknowledge their time constraints. Offer an alternative path (e.g., connecting with someone else, sharing something useful with no strings).
     - Do NOT reference previous messages.

7. **Display all three messages directly in the terminal** for easy copy-paste. Format them clearly with headers:

   ```
   ## Initial Outreach

   [message text]

   ---

   ## Follow-up 1 (~1 week)

   [message text]

   ---

   ## Follow-up 2 (~2 weeks)

   [message text]
   ```

8. Also generate the .docx archive copy. Write the JSON payload to a temp file and pipe to the script:
   ```bash
   python3 "${JOBCRAWLER_DIR:-~/code/jobcrawler}/scripts/to_docx.py" < /tmp/outreach_payload.json
   ```

   JSON structure:
   ```json
   {
     "type": "outreach",
     "slug": "<slug>",
     "output_dir": "$WORKDIR/opportunities/[slug]",
     "contact_name": "<contact name>",
     "messages": [
       {"label": "Initial Outreach", "body": "..."},
       {"label": "Follow-up 1 (~1 week)", "body": "..."},
       {"label": "Follow-up 2 (~2 weeks)", "body": "..."}
     ]
   }
   ```

   Substitute the resolved `$WORKDIR` and slug values into `output_dir` before writing the JSON.

9. Update the opportunity file `$WORKDIR/opportunities/[slug]/opportunity.json`:
   - Add or update an entry in the `threads` array:
     ```json
     {
       "contact_id": "<contact id from contacts.csv>",
       "status": "outreach_drafted",
       "last_action_date": "<YYYY-MM-DD>",
       "outreach_file": "<filename from output path>"
     }
     ```
   - Append to `generated_artifacts`:
     ```json
     {
       "type": "outreach",
       "filename": "<filename>",
       "generated_at": "<ISO 8601 timestamp>"
     }
     ```
   Write the updated JSON back.

10. Confirm the .docx output path to the user.
