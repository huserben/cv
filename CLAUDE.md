# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build commands

Local build (run each command twice to resolve hyperref/outline warnings):

```bash
xelatex -synctex=1 -interaction=nonstopmode "\def\version{LOCAL} \input{cv_english.tex}"
xelatex -synctex=1 -interaction=nonstopmode "\def\version{LOCAL} \input{cv_german.tex}"
```

The `\version` macro is mandatory — `cv.tex` references it for the footer hyperlink. CI passes a date tag; the VS Code LaTeX Workshop recipe in `.vscode/settings.json` passes `LOCAL`.

Arch prerequisites: `sudo pacman -S texlive-bin texlive-latexrecommended texlive-latexextra texlive-fontsextra texlive-langgerman`.

## Architecture

This is a LaTeX CV built on the [Awesome-CV](https://github.com/posquit0/Awesome-CV) document class (`awesome-cv.cls` is vendored). The build is structured so a single content source produces two localized PDFs:

- `cv_english.tex` and `cv_german.tex` are the entry points. Each one only sets `\documentclass`, loads `babel` with its language, and `\input{cv.tex}`.
- `cv.tex` is the shared spine: it sets layout/colors, defines `\versionlink` (which branches on `\iflanguage{german}`), and `\input`s every section file in display order: `personalinfo` → `summary` → `skills` → `education` → `communitycontributions` → `\pagebreak` → `certifications` → `experience`.
- Section `.tex` files contain bilingual content using `\iflanguage{german}{...}{...}` inline rather than separate per-language section files. When editing content, add both languages in the same place.

Because both PDFs share `cv.tex`, structural changes (section order, page breaks, header/footer) belong in `cv.tex`; the language entry points should stay as thin wrappers.

## Writing style

When editing CV prose, do not use em-dashes or en-dashes as clause separators (in LaTeX: `---` or `--`, or the literal `—` / `–` characters). They read as an LLM tell. Rephrase as two sentences, use a comma, or use a colon. Hyphens inside compound modifiers (`AI-augmented`, `data-informed`, `hands-on`, `Software-Engineering-Hintergrund`) are normal grammar and stay.

## Job-hunt commands

Three project-scoped slash commands live in `.claude/commands/`. They work as a pipeline:

- **`/job-search`** — asks the user for preferences (location, work mode, schedule, target titles), persists them to `jobs/preferences.json`, and ingests opportunities through five channels: (S) user-supplied seed URLs from `jobs/seeds.txt` or `/job-search <url>` args, each seed gets full extraction plus a similar-roles expansion (same-company siblings + similar-title search); (A) verified XML feeds (currently `swissdevjobs.ch/job_feed.xml`); (B) any browser-automation MCP installed (Playwright, Puppeteer, Browserbase); (C) user-pasted HTML or text dropped into `jobs/raw/`; (D) WebSearch/WebFetch fallback across jobs.ch, jobup.ch, LinkedIn, indeed.ch, jobscout24.ch, stepstone.ch. Writes a dated shortlist to `jobs/YYYY-MM-DD-search.md`. Successfully extracted seeds are cut from `seeds.txt` to `seeds-processed.txt` with a date stamp; failed seeds get a `# FAILED YYYY-MM-DD: <reason>` marker and stay for the next run. Preferences and optional auth artifacts (`jobs/auth/` for MCP cookie jars) are loaded on the next run; no credentials are ever read from chat or repo files.
- **`/job-inspector`** — reads the latest search file (or a path arg) and walks each opportunity one at a time. Scores on four dimensions (role+tech fit, agile/lean culture, location/remote, seniority+comp; max 30), surfaces any matching prior feedback, then asks for an Apply/Maybe/Skip verdict with optional comment. Appends every verdict to `jobs/feedback.jsonl` (append-only history) and regenerates `jobs/feedback-summary.md` with company-level and keyword-level patterns. Writes the bucketed shortlist to `jobs/YYYY-MM-DD-inspection.md`.
- **`/job-applicator`** — pick one opportunity from the inspection, creates an `apply/{company-role-date}` branch, drops `notes.md` / `cover-letter.md` / `interview-prep.md` into `jobs/applications/{slug}/`, walks the user through any CV tweaks suggested by the inspector, and rebuilds the PDFs. Does not auto-commit.

The `jobs/` directory is gitignored — preferences, search results, inspections, and per-application notes stay local. Application-specific CV edits live on the `apply/{slug}` branch, not `main`.

## Release flow

`.github/workflows/buildpdf.yml` runs on push to `main`: it creates a daily date-prefixed tag (`fregante/daily-version-action`), injects it as `\version`, compiles both PDFs with `xelatex`, and publishes them to the `gh-pages` branch (which serves `huserben.github.io/cv/cv_{english,german}.pdf`). PDFs are not committed to `main` — they're build artifacts produced per push.
