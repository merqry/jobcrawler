# /update-opportunity

Update an existing opportunity's stage, notes, or contacts.

## Arguments

```
/update-opportunity [--workdir <path>] [<slug>]
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

2. Ask the user for the **opportunity slug** (if not already provided as an argument). If unsure, read `$WORKDIR/index.json` and show available opportunities with their current stages.

3. Ask what they want to update:
   - **stage** — e.g., `targeting`, `applied`, `screening`, `interviewing`, `offer`, `closed`
   - **notes** — free-text notes about the opportunity
   - **contacts** — add a contact ID to the contacts array

4. Read `$WORKDIR/opportunities/[slug]/opportunity.json`.

5. Apply the requested changes:
   - For **stage**: update the `stage` field
   - For **notes**: replace the `notes` field with the new text (or append if the user says "add" rather than "replace")
   - For **contacts**: append the contact ID to the `contacts` array (avoid duplicates)

6. Write the updated JSON back to `$WORKDIR/opportunities/[slug]/opportunity.json`.

7. If the **stage** was changed, also update `$WORKDIR/index.json`:
   - Read the file
   - Find the entry with the matching slug
   - Update its `stage` field
   - Write it back

8. Confirm the changes to the user, showing the old and new values.
