# Summary
![Build and Publish PDFs](https://github.com/huserben/cv/actions/workflows/buildpdf.yml/badge.svg)

This repo contains my CV written in LaTeX using [Awesome-CV](https://github.com/posquit0/Awesome-CV).
It's using GitHub Actions to compile the source files in to pdfs, and then pushing it to a gh-pages branch, which is published as page on GitHub and available to the general public.

This repo also shows how you can use LaTeX to add multi-language support, as the CV is written in English and German, and is using a main file (see *cv_english.tex* and *cv_german.tex*) to set the main language.

The GitHub Action is triggered on any change to the main branch, is tagging the specific version based on the current date, and passes this version to the pdf generation to put the version/timestamp in the generated pdf's.

# Latest Version
## English 🇬🇧
[![Open PDF](https://img.shields.io/badge/Open%20PDF-EC1C24?style=flat&logo=adobeacrobatreader)](https://huserben.github.io/cv/cv_english.pdf)

## German 🇩🇪
[![Open PDF](https://img.shields.io/badge/Open%20PDF-EC1C24?style=flat&logo=adobeacrobatreader)](https://huserben.github.io/cv/cv_german.pdf)

---

## Run locally (Arch Linux + VS Code)

Simple steps to build the PDFs on an Arch-based system using VS Code.

Prerequisites:
- Install TeX Live packages (enough to build these files):

```bash
sudo pacman -S texlive-bin texlive-latexrecommended texlive-latexextra texlive-fontsextra texlive-langgerman
```

- Install VS Code and the LaTeX Workshop extension (`James-Yu.latex-workshop`).

Build (two ways):

1) From VS Code
- Open the repository in VS Code.
- Use LaTeX Workshop build (the workspace provides a `xelatex` recipe that injects the version).

2) From the terminal
- Run (replace the date string with the desired version):

```bash
xelatex -synctex=1 -interaction=nonstopmode "\\def\\version{2025-12-27} \\input{cv_english.tex}"
xelatex -synctex=1 -interaction=nonstopmode "\\def\\version{2025-12-27} \\input{cv_german.tex}"
```

- Run each command twice to clear hyperref / outline warnings.

That's it — the generated PDFs (`cv_english.pdf` and `cv_german.pdf`) will be in the repository root.