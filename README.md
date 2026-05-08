# JobCrawler

An executive job search orchestration system for Director through C-suite candidates. JobCrawler is built for high-stakes, relationship-driven searches where the difference between landing the role and being overlooked is preparation, timing, and network activation — not just applying.

## Vision

Most job search tools are passive trackers. JobCrawler is an execution system.

At the executive level, roles are rarely won through job boards. They're won through relationships, narrative precision, and coordinated outreach across multiple stakeholders. The process is slow, non-linear, and high-context — and most candidates manage it in a fragmented patchwork of spreadsheets, email drafts, and calendar reminders.

JobCrawler changes the workflow:

- **Company-first, not posting-first** — Build a target list and monitor for role emergence rather than reacting to what's posted
- **Evidence-based narrative** — Your resume and outreach are grounded in metrics and decisions, not adjectives
- **Multi-threaded outreach** — Every opportunity has a stakeholder map and a threading plan, not a single "apply and hope"
- **Artifact generation** — Tailored resumes, outreach sequences, and interview prep are generated from your achievement bank, not rewritten from scratch each time
- **Pipeline clarity** — One view of where every opportunity stands and what the next action is

The goal is to compress the search timeline and improve conversion by making the hard parts systematic — not easier, but consistent.

## How It Works

JobCrawler is a set of Claude Code commands that operate on two locations:

1. **This repo** (`jobcrawler/`) — the command definitions, scripts, and configuration templates
2. **Your personal work folder** (`PERSONAL_WORKDIR`) — where all your data and generated artifacts live

### Why a separate personal folder?

Your resume artifacts, job postings, contact notes, and generated documents are personal data. Keeping them outside the repo means:

- **Sync without friction** — Point `PERSONAL_WORKDIR` to a cloud-synced folder (OneDrive, Dropbox, iCloud Drive) and your artifacts are automatically backed up and accessible across devices
- **Clean separation** — The repo stays generic and shareable; your data stays private
- **No accidental commits** — Your work history, compensation targets, and contact lists never touch version control

A typical setup looks like:

```
~/code/jobcrawler/          # This repo — commands and scripts
~/OneDrive/job-search/      # PERSONAL_WORKDIR — your data and artifacts
  inputs/
    profile.json            # Your background, target titles, comp targets
    workhistory.csv         # Employment history
    achievements.csv        # Achievement bank with metrics and themes
    contacts.csv            # Contact list with relationship context
    companies.csv           # Target company list
  opportunities/
    stripe_vp_strategy/     # One folder per opportunity
      opportunity.json
      resume_2026-05-01.docx
      outreach_alex_chen.docx
  index.json                # Pipeline index across all opportunities
```

## Setup

### Prerequisites

- [Claude Code](https://claude.ai/code) — required to run the commands
- Python 3.8+ — required for document generation
- `python-docx` — install with `pip install python-docx`

### Installation

1. **Clone the repo:**
   ```bash
   git clone https://github.com/yourusername/jobcrawler.git
   cd jobcrawler
   ```

2. **Set up Python dependencies:**
   ```bash
   python3 -m venv .venv
   source .venv/bin/activate
   pip install python-docx
   ```

3. **Create your personal work folder:**
   ```bash
   mkdir -p ~/job-search/inputs
   ```
   Or point it at a cloud-synced location:
   ```bash
   mkdir -p ~/OneDrive/job-search/inputs
   ```

4. **Configure Claude Code settings:**

   Copy the settings template:
   ```bash
   cp .claude/settings.local.json.template .claude/settings.local.json
   ```

   Edit `.claude/settings.local.json` and set your paths:
   ```json
   {
     "env": {
       "PERSONAL_WORKDIR": "/path/to/your/job-search/folder",
       "JOBCRAWLER_DIR": "/path/to/jobcrawler"
     }
   }
   ```

   `PERSONAL_WORKDIR` should point to wherever you want artifacts saved — ideally a cloud-synced folder. `JOBCRAWLER_DIR` should point to where you cloned this repo.

5. **Populate your inputs:**

   JobCrawler reads from a set of CSV and JSON files in `$PERSONAL_WORKDIR/inputs/`. See [Input File Reference](#input-file-reference) below for the expected format of each file.

   At minimum, you need:
   - `profile.json` — your name, contact info, title targets, and compensation range
   - `workhistory.csv` — your employment history
   - `achievements.csv` — your achievement bank (the core of everything)

---

## Commands

Commands are invoked from within Claude Code by typing `/command-name`. Each command reads your input files, generates the appropriate artifact, and writes output to your `PERSONAL_WORKDIR`.

### `/add-opportunity`

Adds a new job opportunity to your pipeline from a URL or pasted text.

```
/add-opportunity [url]
```

**What it does:**
1. Fetches the job posting (or accepts pasted text if the URL is behind a login)
2. Extracts the company, role title, and key requirements
3. Creates an opportunity folder under `$PERSONAL_WORKDIR/opportunities/[slug]/`
4. Adds the opportunity to your pipeline index

**When to use it:** Any time you want to track a role — whether you found it on a job board, a career page, or heard about it through a contact.

---

### `/scan-roles`

Scans job boards and company career pages for roles matching your target profile. Scores, deduplicates, and adds matches to your pipeline.

```
/scan-roles [--titles "VP Strategy,Director Ops"] [--min-comp 250000] [--companies all|tier1|<name>]
```

**What it does:**
1. Reads your target titles and company list from your input files
2. Searches LinkedIn, Indeed, Glassdoor, Lever, and Greenhouse
3. Scores each result on title fit, seniority alignment, theme overlap, and company fit
4. Shows results ranked A–D and asks which to add
5. Logs the scan to prevent re-scanning too frequently

**When to use it:** Periodically (weekly or when you want to expand your pipeline) to surface new roles at your target companies without manually checking each career page.

---

### `/generate-resume`

Generates a tailored executive resume for a specific opportunity.

```
/generate-resume [slug]
```

**What it does:**
1. Reads the opportunity's key requirements
2. Selects achievements from your bank that match the role's themes
3. Rewrites subheadings and emphasis to align with the posting's language
4. Checks for gaps (key requirements with no matching evidence) and flags them
5. Generates a formatted `.docx` resume saved to the opportunity folder

**When to use it:** Once you've decided to pursue an opportunity seriously. Run it when you have a specific posting to tailor against — not as a general resume refresh.

---

### `/draft-outreach`

Generates a 3-message outreach sequence for a contact at a target company.

```
/draft-outreach [slug]
```

**What it does:**
1. Reads the opportunity and contact context
2. Generates three standalone messages: initial outreach, follow-up 1 (~1 week), follow-up 2 (~2 weeks)
3. Each message is self-contained — no "following up on my previous note"
4. Tone is calibrated for executive-level communication (formal, value-first, low-friction ask)
5. Saves a `.docx` archive copy to the opportunity folder

**When to use it:** Before reaching out to any stakeholder at a target company. Works for cold outreach, warm introductions, and alumni connections — the tone adapts to the relationship type.

---

### `/update-opportunity`

Updates an opportunity's stage, notes, or contacts.

```
/update-opportunity [slug]
```

**What it does:**
Updates the stage (`targeting` → `applied` → `screening` → `interviewing` → `offer` → `closed`), appends notes, or adds a contact to the opportunity record.

**When to use it:** After any meaningful interaction — a response to outreach, a screening call, moving to interviews, receiving an offer.

---

## Input File Reference

All input files live in `$PERSONAL_WORKDIR/inputs/`.

### `profile.json`
Your core profile used across all commands.
```json
{
  "name": "Your Name",
  "email": "you@example.com",
  "phone": "+1 555 000 0000",
  "linkedin_url": "https://linkedin.com/in/yourhandle",
  "title_targets": ["VP Strategy and Operations", "Senior Director Strategy"],
  "compensation_targets": {
    "min": 250000,
    "target": 300000,
    "max": 400000,
    "currency": "USD"
  },
  "core_strengths": ["Operating model design", "Revenue acceleration", "Cross-functional leadership"]
}
```

### `workhistory.csv`
One row per role. Columns: `job_id`, `company`, `title`, `location`, `start_date`, `end_date`, `scope`, `is_early_career`

### `achievements.csv`
The core of the system. One row per achievement. Columns: `id`, `job_id`, `Co`, `subheading`, `title`, `narrative`, `themes`, `confidence`

- `themes` — semicolon-separated tags (e.g., `revenue growth;GTM strategy;team leadership`)
- `confidence` — `high`, `medium`, or `low` (affects selection priority for resumes)
- `narrative` — the full achievement bullet as it appears on the resume

### `contacts.csv`
One row per contact. Columns: `id`, `name`, `company`, `title`, `relationship`, `relationship_type`, `notes`

- `relationship_type` — `cold`, `mutual`, `alumni`, or `prior_colleague` (affects outreach tone)

### `companies.csv`
Your target company list. Columns: `company`, `tier`, `priority_score`, `careers_url`, `comp_override`, `ats_platform`

---

## License

See LICENSE for details.
