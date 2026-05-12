---
description: Search Swiss job portals for opportunities matching the CV in this repo. Asks for preferences (with last-run defaults), persists them, and produces a dated markdown list.
---

# /job-search

Search live job listings on Swiss portals matching the CV and the user's preferences. Persist preferences across runs and produce a dated markdown shortlist.

## Inputs

- `jobs/preferences.json` — last-run preferences. If missing, treat every preference as unset.
- `jobs/seeds.txt` (optional) — newline-separated job URLs the user wants treated as anchors (see Channel S below). Lines may include an optional `# note` after the URL. Lines that start with `#` are comments and ignored.
- `jobs/seeds-processed.txt` (auto-managed) — successfully extracted seeds are moved here with a timestamp so they aren't re-fetched every run. Do not hand-edit.
- Command arguments (optional) — when the user invokes `/job-search <url1> [url2 ...]`, each URL is treated as a one-shot seed for this run AND appended to `jobs/seeds.txt` if not already present, so the same URL doesn't have to be retyped next time.
- `jobs/raw/` (optional) — directory where the user drops HTML or text from logged-in browser sessions for the command to parse. See "Ingestion channels" below.
- `jobs/auth/` (optional) — directory for exported session cookies (gitignored via `jobs/`). See "Auth handling" below.
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

5. **Detect Playwright MCP up front.** Scan the available tool list for any `mcp__playwright__*` tool. Two outcomes:
   - **Detected**: log "Playwright MCP detected" in the output's Method notes and use it for any portal that returns 403/404/empty content from WebFetch (LinkedIn, swissdevjobs.ch root, jobup.ch detail pages, jobs.ch detail pages when they 404, etc.). Persistent browser state means once the user has logged in once via the MCP session, detail pages return real content.
   - **Not detected**: surface a visible warning in the chat reply AND at the top of the output file:

     ```
     WARNING: Playwright MCP not detected. Results will be shallower because LinkedIn, jobup.ch detail pages, and several other portals are auth-walled to unauthenticated fetches. Install with:

         claude mcp add playwright npx '@playwright/mcp@latest'

     Then restart Claude Code and re-run /job-search. After install, log in once per portal (visit linkedin.com/login, jobs.ch login, etc. via the MCP browser) and the session persists across runs.
     ```

   Then run all available ingestion channels in the order below. Don't stop at the first one that returns data — combine results across channels. For each channel, note in the output file whether it ran, was skipped, or failed.

   ### Channel S — User-supplied seed URLs (highest priority)

   Two input methods, both supported and run together:

   1. **Persistent file**: read `jobs/seeds.txt` if it exists. Each non-empty, non-comment line is parsed as `<URL> [# optional note]`. The note (if any) is preserved through to the output file. Lines that fail to parse as a URL get logged and skipped, not silently dropped.
   2. **Ad-hoc args**: any URL passed as a `/job-search` argument is treated as a one-shot seed for this run AND appended to `jobs/seeds.txt` (deduped against existing entries) so the same URL doesn't need to be retyped.

   For each seed URL:

   - **Extract the listing**. Prefer Playwright (Channel B tooling) since seeds are often LinkedIn / jobs.ch / Workday URLs that need a logged-in session. WebFetch is the fallback. Capture title, company, location, posted date, schedule, summary, stack, language requirements, salary (if visible), and apply path. If extraction fails (4xx, redirect to login, page returned no usable content), note the failure and move on — do not invent the listing.
   - **Treat each seed as a tier-S result** in the output. They sit above algorithmically-discovered entries and are not subject to the relevance filter (the user already filtered by picking them).
   - **Move on success**: append the seed URL to `jobs/seeds-processed.txt` with the date and a short outcome (`OK`, the resolved title) so it's not re-fetched next run.
   - **Retry on failure**: leave the entry in `jobs/seeds.txt` and append `# FAILED YYYY-MM-DD: <reason>` to the line so the user sees why. Next run will retry.

   After every seed is processed, run a **similar-roles expansion** anchored on the successful seeds:

   - Derive 2-4 anchor tokens per seed: the title head ("Engineering Manager", "Agile Coach", "Tech Lead"), the company name, the strongest stack keyword (e.g. "TypeScript", "Kubernetes", ".NET"), and the location.
   - **Same-company branch**: try to find the company's careers page (look for a `Careers` / `Jobs` link on the seed page itself; common ATS patterns are `boards.greenhouse.io/<co>`, `jobs.lever.co/<co>`, `<co>.wd*.myworkdayjobs.com`, `<co>.bamboohr.com/careers`, `careers.<co>`). If found, pull other open leadership/senior roles at the same company. Cap at 5 same-company sibling roles.
   - **Similar-title branch**: run one targeted Channel B (or D) search using the derived title head + location filter, e.g. `linkedin.com/jobs/search/?keywords="Engineering Manager"&location=Switzerland`. Filter the hits to those NOT already in today's result set and NOT already in `seeds-processed.txt`. Cap at 5 new entries per seed.
   - Annotate each derived entry with `Discovered via seed: <seed URL>` so the user can see the lineage. Do NOT re-process the seed itself as a similar role.

   Cap the total seed-derived entries (seeds + their expansions) at ~15 per run so a single seed can't blow up the file. If a seed produces no useful expansion, that's fine; record `no similar roles found` and move on.

   ### Channel A — Verified RSS / XML feeds (no auth)

   Try these first; they are public and machine-readable.

   - **swissdevjobs.ch**: `https://swissdevjobs.ch/job_feed.xml` — custom XML feed, parseable. Filter items locally by title and location keywords from preferences.
   - **Other portals**: jobs.ch / jobup.ch / jobscout24.ch / stepstone.ch / linkedin.com/jobs **do not expose a native, documented RSS feed for job search** as of the last verification (2026-05). Do **not** invent feed URLs. If the user knows a working feed URL for a portal, store it in `jobs/preferences.json` under `feed_urls` and the command will probe those too.

   When parsing the swissdevjobs feed, capture: title, company, location, URL, posted date if present. Score relevance against preferences before adding to the result set so the output stays focused.

   ### Channel B — MCP browser-automation tools

   Playwright MCP is the supported default (see Step 5 detection above). Use it to:

   - Navigate to portal search pages built from preferences (e.g. `https://www.linkedin.com/jobs/search/?keywords=Engineering+Manager&location=Switzerland`).
   - Extract the visible job cards (title, company, location, posted date, URL).
   - Open the top 5-10 detail pages and pull responsibilities, stack, culture signals.

   If the user has logged in once via the MCP session (visiting linkedin.com/login or jobs.ch login from a tool call), subsequent navigations carry the cookie. Don't pass plaintext passwords; rely on the MCP's persistent browser state.

   Other browser-automation MCPs (`mcp__puppeteer__*`, `mcp__browserbase__*`) work too if present, but treat Playwright as canonical.

   ### Channel C — User-pasted HTML or text (`jobs/raw/`)

   If `jobs/raw/` exists and contains files, read each one and parse it as a job posting (or list page). Accepted formats:

   - `*.html` — full saved page, including search result pages and detail pages.
   - `*.txt` / `*.md` — copy-paste of a job posting from a logged-in browser.

   This is the simplest auth bypass: the user logs in with their own browser, saves or copies the page, drops it into `jobs/raw/`, and the command extracts the data. After processing, leave the files in place (don't delete them) so the inspector can also read them.

   ### Channel D — Unauthenticated WebSearch and WebFetch (fallback)

   Run the original portal searches as in prior versions of this command. Portals to query (in order):

   - jobs.ch — listing pages and detail pages typically reachable
   - swissdevjobs.ch — detail pages reachable, root listing often 403
   - jobup.ch — listing pages reachable, detail pages frequently redirect
   - linkedin.com/jobs — listing pages reachable, detail pages auth-walled
   - indeed.ch — site: queries unreliable
   - jobscout24.ch — listing pages reachable
   - stepstone.ch — search results often empty

   Build queries from the selected titles and location, e.g.:
   - `site:jobs.ch "Technical Agile Coach" Zurich`
   - `site:linkedin.com/jobs "Engineering Manager" Switzerland remote`

   When WebFetch hits a portal that requires JS/auth and returns nothing parseable, note the gap and move on. Do not fabricate listings.

6. **Aggregate and dedupe**. If the same job surfaces on multiple channels, keep the most informative version and note the duplicate sources in a single bullet. Priority order when the same job appears in multiple places: Channel S (user seeds) > Channel C (user-pasted) > Channel A (feed) > Channel B (MCP) > Channel D (unauthenticated web). Seeds beat everything: if a seed URL matches an algorithmic hit, the seed copy survives and the algorithmic source is recorded as a duplicate marker. Similarly, a similar-role hit discovered via seed expansion that also surfaced in Channel A/B/D keeps the expansion lineage in the output, since "found because of seed X" is useful context for the user.

7. **Write output** to `jobs/YYYY-MM-DD-search.md` using today's date. Suggested structure:

   ```markdown
   # Job search — YYYY-MM-DD

   ## Search parameters
   - Location: ...
   - Work mode: ...
   - Schedule: ...
   - Titles: ...

   ## Ingestion channels run
   - Channel S (seeds): N URLs read; M extracted; P failed (kept for retry); Q similar-role expansions surfaced
   - Channel A (feeds): swissdevjobs.ch — N items; <any user-supplied feeds>
   - Channel B (MCP browser tools): <detected tool name> — N detail pages enriched, or "not detected"
   - Channel C (user-pasted): N files from jobs/raw/
   - Channel D (web fallback): N portals searched, N detail pages reached

   ## Seeds (user-anchored)

   ### S1. <Title> — <Company>
   - **Source**: seed (added <date>, note: "<user note if any>") — <URL>
   - **Posted / Schedule / Location / Stack / Summary**: <as above>
   - **Similar roles discovered**: list of derived entries below

   ## Similar roles discovered via seeds

   ### <Title> — <Company>
   - **Discovered via seed**: <seed URL>
   - <standard entry shape>

   ## Results (algorithmic)

   ### 1. <Title> — <Company>
   - **Location**: <city, remote policy>
   - **Source**: <channel + portal name> — <URL>
   - **Posted**: <date if visible>
   - **Summary**: <2-3 sentences of substance>
   - **Stack / keywords**: <extracted>

   ## Seeds that failed extraction (kept in seeds.txt for retry)

   - <URL> — <reason>
   ```

8. **Confirm** to the user: total opportunities found, file path, which channels ran, what was skipped or failed, how many seeds extracted vs. failed, suggest running `/job-inspector` next.

## Auth handling

Three tiers, in order of safety:

1. **No auth needed** (Channels A and D). Default. Always runs.
2. **User-pasted content** (Channel C). User does the login in their own browser, exports content into `jobs/raw/`. No credentials leave the user's machine. Recommended for casual reinforcement of detail-page coverage.
3. **MCP browser automation** (Channel B). User installs a Playwright / Puppeteer / Browserbase MCP server, logs in once via that server's flow (the MCP stores cookies in its own state directory, typically under `~/.cache/` or similar). The command then drives the persistent session. Recommended for sustained use across multiple `/job-search` runs.

What **not** to do:

- Do **not** ask the user to paste passwords, API tokens, or session cookies into chat. They land in transcripts.
- Do **not** read passwords from `jobs/preferences.json` or any file in the repo. The `jobs/auth/` directory may exist for cookie jars used by an MCP fetch tool that supports header-based auth, but the command should never read or echo those files itself; it lets the MCP tool consume them.
- Do **not** invent feed URLs, OAuth flows, or scraping endpoints. If a portal isn't covered by Channels A-D, say so in the output.
- Do **not** silently drop a seed URL that failed to extract. Failed seeds stay in `jobs/seeds.txt` with a `# FAILED YYYY-MM-DD: <reason>` marker so the user can fix or remove them. Move only successful seeds to `seeds-processed.txt`.
- Do **not** treat a seed-derived similar-role hit as if the user had picked it directly. The lineage (`Discovered via seed: <URL>`) stays in the output and in any inspector-side feedback.

## Rules

- Honesty over completeness. If a channel returned nothing, say so in the output. If a feed URL 404'd, say so.
- No emojis. No em-dashes as clause separators (see CLAUDE.md writing-style note).
- Don't crawl beyond what's necessary — aim for ~5-15 strong matches in the algorithmic section, plus however many seed-derived entries the user's seeds.txt and args produced (cap ~15).
- Save preferences even if the user aborts mid-search, so partial progress carries to next run.
- Treat `jobs/raw/` and `jobs/auth/` as user-owned. Read from them, do not write to them, do not delete from them.
- Treat `jobs/seeds.txt` as user-managed: append to it (when args are passed) and rewrite individual lines with FAILED markers, but do not reorder or remove the user's own lines / notes. Successful seeds get cut to `seeds-processed.txt` so the user file shrinks over time without losing provenance.

## Seeds file format

A minimal `jobs/seeds.txt`:

```
# Comments work like in shell. Empty lines are fine.

https://www.linkedin.com/jobs/view/4403619482/ # forwarded by Lukas, says culture is great
https://swissdevjobs.ch/jobs/Some-Company-Tech-Lead
https://boards.greenhouse.io/example/jobs/12345  # FAILED 2026-05-12: 404
```

The third entry would be retried on the next run. If still failing, the user can either fix the URL or delete the line; the command will not.
