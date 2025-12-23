# Sourish Chandra Resume (LaTeX)

This repository contains the LaTeX source code for the resume of Sourish Chandra.

## Additional Documentation

This repository also includes comprehensive API documentation for DECIMER.ai:

- **[DECIMER API Key Findings](DECIMER_API_Key_Findings.md)** - Quick summary of key findings (START HERE!)
- **[DECIMER API Documentation](DECIMER_API_Documentation.md)** - Complete guide to DECIMER.ai API endpoints for file upload and SMILES generation
- **[DECIMER API Quick Reference](DECIMER_API_Quick_Reference.md)** - Quick reference guide with diagrams and examples

## Description

The resume is formatted using the `article` class with a two-column layout created using the `multicol` package. It features customized section headings, compact lists, and colored clickable hyperlinks.

## Requirements

- A LaTeX distribution such as TeX Live, MiKTeX, or MacTeX
- Packages used:
  - geometry
  - multicol
  - titlesec
  - enumitem
  - xcolor
  - hyperref

These are usually included by default in modern LaTeX installations.

## How to Compile

Compile the document using `pdflatex` or any LaTeX editor of your choice:

```bash
pdflatex resume.tex
