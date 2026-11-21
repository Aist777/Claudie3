# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

A single-file, dependency-free IT PMO Kanban demo: `index.html` holds all markup, CSS and JavaScript. There is no build step, no npm, and no external resources. Board state is in memory only, so a refresh reseeds the demo data.

## Commands

- Run locally: open `index.html` in a browser. No server needed.
- Syntax-check the script: extract the `<script>` block and run `node --check` on it.

## Deployment

`.github/workflows/deploy-pages.yml` publishes `index.html` to GitHub Pages on every push to `main`. Pages must be set to source "GitHub Actions" in repo settings. Keep `index.html` at the repo root, because the workflow copies only that file.

## Constraints to preserve

- No frameworks, bundlers or CDN scripts.
- No storage APIs (localStorage, sessionStorage, IndexedDB, cookies).
- The FormSubmit endpoint is the only network call; it is set by `FORMSUBMIT_ENDPOINT` at the top of the script.
