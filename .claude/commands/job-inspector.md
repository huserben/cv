---
description: Grade opportunities from the latest /job-search output against the CV, step-by-step with user verdicts. Persists feedback so future runs learn from prior decisions.
argument-hint: [optional: path to a specific search file]
---

# /job-inspector

Walk through each opportunity from the latest search file one at a time. Show the score and rationale, then ask the user whether to apply / maybe / skip, and capture an optional comment. Persist every verdict to `jobs/feedback.jsonl` so future inspections can surface patterns ("you've thumbed down 3 SAFe-heavy roles", "Roche has 2 prior positive verdicts").

## Inputs

- Latest `jobs/YYYY-MM-DD-search.md` (or `$ARGUMENTS` if a path is provided).
- CV source files.
- `jobs/preferences.json` for the location/remote hard-filter.
- `jobs/feedback.jsonl` (append-only history; created on first run).
- `jobs/feedback-summary.md` (regenerated each run; human-readable view of the history).

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

Show all four component scores so the user can sanity-check.

## Steps

1. **Find the input file**. If `$ARGUMENTS` is a path, use that. Otherwise pick the most recent `jobs/YYYY-MM-DD-search.md` by filename.

2. **Pre-load context**:
   - CV files (so role/stack fit is grounded).
   - `jobs/preferences.json` (for the location hard-filter).
   - `jobs/feedback.jsonl` if it exists. Parse each line as JSON. Build three lookups:
     - **Company-level**: for each company, list of `{verdict, note, date}`.
     - **Keyword-level**: count of verdicts per stack/culture keyword (e.g. "SAFe", "flow metrics", "Symfony", "Roche-style pharma").
     - **Pattern signals**: derived (e.g. "user has skipped 3 of 3 part-time roles" → flag part-time as negative).

3. **Detect Playwright MCP**. If `mcp__playwright__*` tools are present, use them to refresh detail-page content for any opportunity whose search-file summary is thin (especially recruiter listings and LinkedIn-sourced entries). If not detected, score conservatively on those entries and surface the same install warning as `/job-search`.

4. **Iterate opportunities one at a time, in their order from the search file**. For each:

   a. **Refresh detail** (only if score will swing on it): WebFetch or Playwright the URL. Cap at one detail fetch per opportunity to keep the loop snappy.

   b. **Score** all four dimensions. Quote a short phrase from the JD for any culture / role score so the rationale is checkable.

   c. **Surface prior feedback context** if any matches:
      - Same company: list prior verdicts and notes verbatim.
      - Overlapping keywords: e.g. "SAFe Coach" matches a prior negative on "SAFe" → flag.
      - Pattern hits: e.g. part-time when the user has skipped all prior part-time entries.

   d. **Print to chat** the per-opportunity card:

      ```
      [N/Total] <Title> — <Company> (score: X/30)
      Role+Tech: X/10 — <justification>
      Culture: X/10 — "<quote>"
      Location: X/5 — <city, remote>
      Seniority+Comp: X/5 — <signals>
      URL: <url>
      Prior feedback: <none | summary of matches>
      Remarks: <2-4 sentences. Strong matches, concerns, CV tweaks.>
      ```

   e. **Ask the user via AskUserQuestion**. Single question per opportunity, 4 options:

      - **Apply** (Recommended if score >= 22 AND no negative pattern hit; otherwise omit "Recommended")
      - **Maybe** — interested, not now, or needs more info
      - **Skip** — not a fit
      - **Other** — user adds a freeform reason

      Set `header` to "Verdict" and phrase the question as "How do you want to handle this one?"

   f. **Capture comment**. If the user picks Apply / Maybe / Skip with no annotation, treat the comment as empty. If they pick Other, treat their freeform text as the comment AND infer the verdict from the text (apply / maybe / skip). If unclear, ask a follow-up.

   g. **Append to `jobs/feedback.jsonl`** one JSON object per line:

      ```json
      {"date":"YYYY-MM-DD","company":"<name>","title":"<title>","url":"<url>","score":<total>,"components":{"role_tech":X,"culture":X,"location":X,"seniority":X},"verdict":"apply|maybe|skip","note":"<user comment or empty>","keywords":["<stack/culture keyword>","..."],"source_channel":"<feed|mcp|raw|web>","inspection_file":"jobs/YYYY-MM-DD-inspection.md"}
      ```

      The keywords list is what the inspector extracted (3-7 tokens). It's what the next run's pattern lookup will key off.

5. **After the loop, write `jobs/YYYY-MM-DD-inspection.md`** with verdicts inline:

   ```markdown
   # Job inspection — YYYY-MM-DD

   Source: <input file path>
   Reviewed: <N> opportunities
   Verdicts: apply=<n> maybe=<n> skip=<n>

   ## To apply

   ### 1. <Title> — <Company> (score: 26/30, verdict: apply)
   - **Role+Tech**: 9/10 — <one-line justification>
   - **Culture**: 8/10 — <quote from JD>
   - **Location**: 5/5 — <city, remote policy>
   - **Seniority+Comp**: 4/5 — <signals>
   - **URL**: ...
   - **Prior feedback**: <if any>
   - **User note**: "<user's comment, if any>"
   - **Remarks**: <2-4 sentences>

   ## Maybe

   ### N. ...

   ## Skipped

   ### N. ... (one-line per skip is fine here; full detail in jobs/feedback.jsonl)

   ## Filtered out (location/remote mismatch)

   - <Title> — <Company> — <reason>
   ```

6. **Regenerate `jobs/feedback-summary.md`** from the full `feedback.jsonl`. Keep it short and scannable. Suggested structure:

   ```markdown
   # Feedback summary — last updated YYYY-MM-DD

   Total verdicts: <N>  (apply=<n> maybe=<n> skip=<n>)

   ## Companies seen more than once
   - <Company>: apply=<n> maybe=<n> skip=<n> — latest note: "<...>"

   ## Keywords leaning positive (apply rate > 60%, n >= 3)
   - <keyword> (apply n/n)

   ## Keywords leaning negative (skip rate > 60%, n >= 3)
   - <keyword> (skip n/n) — latest note: "<...>"

   ## Patterns
   - <e.g. "all 3 part-time roles skipped">
   - <e.g. "all 2 Geneva-only roles filtered or skipped">
   ```

7. **Summarise to chat**: total reviewed, verdict counts, top apply with score, suggest `/job-applicator` next.

## Rules

- Be honest. A 5/10 is a real signal, not a politeness floor. The user wants to apply to the right jobs, not feel good about a long list.
- Quote evidence (short phrases from the JD) for any culture/role claim.
- Don't repeat the JD verbatim; extract the signal.
- Apply the same writing-style rules as the CV: no em-dashes as clause separators, no LLM tells.
- **Never** re-ask the user about an opportunity they've already verdicted in this session. If the loop is interrupted, the next `/job-inspector` run picks up from the first un-verdicted entry.
- **Never** edit or delete entries in `feedback.jsonl`. It's append-only history. If a verdict needs revising, append a new entry with a `revises_date` field pointing to the prior one.
- Treat `feedback-summary.md` as fully regenerable from `feedback.jsonl`; never hand-edit it.
