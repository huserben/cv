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

## Release flow

`.github/workflows/buildpdf.yml` runs on push to `main`: it creates a daily date-prefixed tag (`fregante/daily-version-action`), injects it as `\version`, compiles both PDFs with `xelatex`, and publishes them to the `gh-pages` branch (which serves `huserben.github.io/cv/cv_{english,german}.pdf`). PDFs are not committed to `main` — they're build artifacts produced per push.
