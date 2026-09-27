---
description: Walk through application for a specific opportunity from /job-inspector output. Creates a branch, drafts cover letter + interview prep, and guides any CV tweaks specific to the role.
argument-hint: [optional: opportunity number or company slug]
---

# /job-applicator

Pick one opportunity from the latest inspection, set up a dedicated branch and workspace, draft application materials, and guide CV tweaks specific to the role.

## Inputs

- Latest `jobs/YYYY-MM-DD-inspection.md` (or use `$ARGUMENTS` to disambiguate).
- CV source files.

## Steps

1. **Find the latest inspection file**. Show the top 5 entries via AskUserQuestion and let the user pick, OR honour `$ARGUMENTS` if it identifies an opportunity (number, company name, or slug).

2. **Generate a slug** from the selection: `{company-slug}-{role-keyword}-{YYYYMMDD}`. Lowercase, hyphens, ASCII-only.

3. **Create a git branch** `apply/{slug}` off `main`. Confirm with the user before creating. Use Bash. Don't switch back without asking.

4. **Create the application workspace** at `jobs/applications/{slug}/`:
   - `notes.md` — fields: job URL, company, role, contact (if known), source portal, status (`drafting` -> `applied` -> `interview` -> `outcome`), date applied, key links, decision rationale. Keep it scannable.
   - `cover-letter.md` — tailored draft. See style notes below.
   - `interview-prep.md` — likely questions, talking points, questions to ask back.

5. **Read the inspector's remarks** for this opportunity. If CV tweaks were suggested, walk through them one at a time:
   - Show the proposed change.
   - Get user approval.
   - Apply the edit.
   - Move to the next suggestion.

6. **Build the PDFs** (`xelatex` twice for each language; see CLAUDE.md). Confirm both built cleanly before claiming the change is done.

7. **Summarise**: branch name, files created, what changed in the CV, what to do next (apply via the portal URL). Do **not** commit automatically; ask the user if they want to commit and what message.

## Cover-letter style

- 3-5 short paragraphs. The CV's voice.
- Lead with concrete evidence (Lighthouse, the 350k LOC migration, the CI/CD shift) tied to the job's needs. Adjectives last, not first.
- One specific thing you'd want to discuss in an interview — turns the close into a hook.
- No em-dashes as clause separators. No "I am writing to express my interest in" boilerplate.

## Interview-prep style

- 5-10 likely interview questions for this role / company type.
- For each, a 2-3 sentence talking point grounded in CV evidence (mention the specific example you'd reach for).
- 3-5 questions for the interviewer that signal preparation: how is success measured in this role at 6 / 12 months? What's the team's release cadence? How is engineering quality measured? etc.

## Rules

- Scope: only change the CV to the extent the role warrants. A bug-fix doesn't need a refactor; a coaching-leaning JD doesn't need a senior-SWE rewrite.
- Always preview before editing. Always rebuild PDFs before declaring done.
- Branch isolation: never apply role-specific tweaks to `main`. The `apply/{slug}` branch is the working surface.
- Status updates: when the user reports back ("applied", "got an interview", "rejected"), update `notes.md` status field. Don't auto-archive.
