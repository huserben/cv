# Summary

This repo contains my CV written in LaTeX using [Awesome-CV](https://github.com/posquit0/Awesome-CV).
It's using GitHub Actions to compile the source files in to pdfs, and then pushing it to a gh-pages branch, which is published as page on GitHub and available to the general public.

This repo also shows how you can use LaTeX to add multi-language support, as the CV is written in English and German, and is using a main file (see *cv_english.tex* and *cv_german.tex*) to set the main language.

The GitHub Action is triggered on any change to the main branch, is tagging the specific version based on the current date, and passes this version to the pdf generation to put the version/timestamp in the generated pdf's.

# Latest Version
## English 🇬🇧
[![Open PDF](https://img.shields.io/badge/Open%20PDF-EC1C24?style=flat&logo=adobeacrobatreader)](https://huserben.github.io/cv/cv_english.pdf)

## German 🇩🇪
[![Open PDF](https://img.shields.io/badge/Open%20PDF-EC1C24?style=flat&logo=adobeacrobatreader)](https://huserben.github.io/cv/cv_german.pdf)