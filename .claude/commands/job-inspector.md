---
description: Grade opportunities from the latest /job-search output against the CV. Produces a sorted shortlist with per-job scores, remarks, and CV-tweak suggestions.
argument-hint: [optional: path to a specific search file]
---

# /job-inspector

Read the most recent (or argument-specified) search output and grade each opportunity for fit against the CV.

## Inputs

- Latest `jobs/YYYY-MM-DD-search.md` (or `$ARGUMENTS` if a path is provided).
- CV source files.
- `jobs/preferences.json` for the location/remote hard-filter.

## Scoring rubric

Score every job on four dimensions, then sum (max 30):

1. **Role + tech fit (0-10)**
   - Target roles: Technical Agile Coach, Engineering Manager, Delivery Lead, Senior SWE.
   - Stack overlap with the CV: C#/.NET, TypeScript, React, Python, CI/CD.
   - Bonus for flow metrics, Kanban, OSS-friendly culture.

2. **Agile / lean culture (0-10)**
   - Look for: XP, TDD, continuous delivery, autonomy, Unfix Patterns / Teal, pair/mob mentions.
   - Discount for: command-and-control language, "Scrum master to chase tickets", excessive process theatre, Jira-as-religion.

3. **Location + remote (0-5)**
   - Hard filter: if the listing violates the user's location/remote preference, score 0 and flag in remarks.
   - Otherwise score on flexibility (full remote = 5, hybrid = 3-4, on-site = 2-3 depending on commute distance from prefs).

4. **Seniority + comp signals (0-5)**
   - Senior / staff / lead / manager: 3-5.
   - Mid / junior / unclear: 0-2.
   - +1 bonus if salary range is disclosed (rare in CH).

Show all four component scores in the output so the user can sanity-check.

## Steps

1. **Find the input file**. If `$ARGUMENTS` is a path, use that. Otherwise pick the most recent `jobs/YYYY-MM-DD-search.md` by filename.

2. **Pre-load context**: the CV files (so role/stack fit is grounded) and `jobs/preferences.json` (for the location hard-filter).

3. **For each job in the search file**:
   - If a URL is present, WebFetch the JD to confirm details and pull culture/stack signals. If the fetch fails or returns garbage, score conservatively and note the gap.
   - Score each rubric dimension. Quote a short phrase from the JD for any culture / role score so the rationale is checkable.
   - Write 2-4 sentences of remarks: strongest match, biggest concern, optional CV tweak suggestion (e.g. "lean Lighthouse front in summary" or "add a line on cross-team CI/CD rollout").

4. **Sort by total score descending**. Move location-filtered-out jobs to a separate "Filtered out" section at the bottom (don't drop them silently — the user may want to override).

5. **Write `jobs/YYYY-MM-DD-inspection.md`**. Suggested structure:

   ```markdown
   # Job inspection — YYYY-MM-DD

   Source: <input file path>
   Reviewed: <N> opportunities

   ## Top opportunities

   ### 1. <Title> — <Company> (score: 26/30)
   - **Role+Tech**: 9/10 — <one-line justification>
   - **Culture**: 8/10 — <quote from JD>
   - **Location**: 5/5 — <city, remote policy>
   - **Seniority+Comp**: 4/5 — <signals>
   - **URL**: ...
   - **Remarks**: <2-4 sentences. Strong matches, concerns, CV tweaks.>

   ### 2. ...

   ## Filtered out (location/remote mismatch)

   - <Title> — <Company> — <reason>
   ```

6. **Summarise to chat**: total reviewed, top 3 with scores, suggest `/job-applicator` next.

## Rules

- Be honest. A 5/10 is a real signal, not a politeness floor. The user wants to apply to the right jobs, not feel good about a long list.
- Quote evidence (short phrases from the JD) for any culture/role claim.
- Don't repeat the JD verbatim; extract the signal.
- Apply the same writing-style rules as the CV: no em-dashes as clause separators, no LLM tells.
