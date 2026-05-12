---
description: Search Swiss job portals for opportunities matching the CV in this repo. Asks for preferences (with last-run defaults), persists them, and produces a dated markdown list.
---

# /job-search

Search live job listings on Swiss portals matching the CV and the user's preferences. Persist preferences across runs and produce a dated markdown shortlist.

## Inputs

- `jobs/preferences.json` — last-run preferences. If missing, treat every preference as unset.
- CV source files (`summary.tex`, `skills.tex`, `experience.tex`, `certifications.tex`) — used as context to suggest titles/stack, **not** quoted back at the user.

## Steps

1. **Read the CV briefly** so suggested titles and stack keywords are grounded. Read, don't echo.

2. **Read `jobs/preferences.json`** if it exists. Use values to pre-fill suggestions.

3. **Ask the user via AskUserQuestion** for preferences. For each, show the previous value if any and let them override or accept:
   - **Location**: Zurich / Bern / Basel / Zug / Lausanne / Switzerland-wide / specify city.
   - **Work mode**: remote / hybrid / on-site / any.
   - **Schedule**: full-time / part-time / either.
   - **Job titles**: pre-populate from CV-derived suggestions (Technical Agile Coach, Engineering Manager, Delivery Lead, Senior Software Engineer, Engineering Lead). Multi-select; allow custom additions.

4. **Persist** the updated values to `jobs/preferences.json`. Create the `jobs/` directory if needed; it should already be gitignored.

5. **Search Swiss job portals** using WebSearch and WebFetch. Portals to query (in order):
   - jobs.ch
   - jobup.ch
   - swissdevjobs.ch
   - linkedin.com/jobs
   - indeed.ch
   - jobscout24.ch
   - stepstone.ch

   Build queries from the selected titles and location, e.g.:
   - `site:jobs.ch "Technical Agile Coach" Zurich`
   - `site:linkedin.com/jobs "Engineering Manager" Switzerland remote`

   When WebFetch hits a portal that requires JS/auth and returns nothing parseable, note the gap and move on. Do not fabricate listings.

6. **Aggregate and dedupe**. If the same job surfaces on multiple portals, keep the most informative one and note the duplicates in a single bullet.

7. **Write output** to `jobs/YYYY-MM-DD-search.md` using today's date. Suggested structure:

   ```markdown
   # Job search — YYYY-MM-DD

   ## Search parameters
   - Location: ...
   - Work mode: ...
   - Schedule: ...
   - Titles: ...

   ## Results

   ### 1. <Title> — <Company>
   - **Location**: <city, remote policy>
   - **Source**: <portal name> — <URL>
   - **Posted**: <date if visible>
   - **Summary**: <2-3 sentences of substance>
   - **Stack / keywords**: <extracted>
   ```

8. **Confirm** to the user: total opportunities found, file path, suggest running `/job-inspector` next.

## Rules

- Honesty over completeness. If portal X returned nothing, say so in the output.
- No emojis. No em-dashes as clause separators (see CLAUDE.md writing-style note).
- Don't crawl beyond what's necessary — aim for ~5-15 strong matches, not 50 weak ones.
- Save preferences even if the user aborts mid-search, so partial progress carries to next run.
