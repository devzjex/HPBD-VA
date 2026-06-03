# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

A customizable web-based interactive birthday card. The recipient clicks through a sequence of animated scenes (dark room → empty room → haunted hallway → gift) before reaching the final card. It is configured entirely through environment variables — there is no UI for personalization.

## Commands

```sh
npm run watch    # local dev: build index.html from env + ./local files, then parcel dev server
npm run build    # production: build index.html from remote env data, then parcel production build → ./dist
```

These wrap two stages each (via `npm-run-all`):
- `init-index-local` / `init-index-remote` — run `builder/init.js` to generate `src/index.html` and `src/pic.jpeg`
- `watch:parcel` / `build:parcel` — Parcel bundles `src/index.html`

You can run a single stage directly, e.g. `npm run init-index-local` or `npm run build:parcel`. There is no test suite, linter, or formatter configured.

Local builds require a `.env` file in the repo root (see `example.env`) plus asset files in `./local/`. The built site lands in `./dist/`; serve it with `live-server` or similar.

## Two-stage build architecture

The build has two distinct phases, and understanding the split is key:

**Stage 1 — `builder/` (Node, runs before bundling).** `builder/init.js` is the entry point. It:
1. Reads env vars (`NAME`, `PIC` mandatory; `NICKNAME`, `HBD_MSG`, `SCROLL_MSG`, `OPEN_DATE` optional).
2. Resolves the recipient picture — from `./local/<PIC>` in `--local` mode, or fetched over HTTP in `--remote` mode — and pipes it through `getPic.js` (sharp: resize to 400×400, JPEG q90) to `src/pic.jpeg`.
3. Optionally builds the scrolling-message HTML via `generateMarkup.js`: from a local `.txt` file (`generateMarkupLocal`), or from a [telegra.ph](https://telegra.ph) article fetched via its API (`generateMarkupRemote`, which recursively walks the article's node tree).
4. `genIndex.js` reads `src/template.html`, substitutes the `{{^...}}` placeholders (`{{^NAME}}`, `{{^NICKNAME}}`, `{{^HBD_MSG}}`, `{{^SCROLL_MSG}}`, `{{^READ_TIME}}`), and writes `src/index.html`. The scroll-message read time is computed from word count (~200 wpm) and injected as the `--readTime` CSS variable.

**Stage 2 — Parcel (browser bundle).** Parcel takes the generated `src/index.html` as its entry and bundles `src/js/` and `src/scss/`. The runtime JS reads `process.env.*` (e.g. `OPEN_DATE`, `SCROLL_MSG`) — Parcel inlines these at bundle time, so the same env vars feed both stages.

`src/index.html` and `src/pic.jpeg` are generated artifacts (gitignored). **Never edit them directly — edit `src/template.html` and rebuild.**

## Runtime (browser) flow

- `src/js/index.js` — entry. If `OPEN_DATE` is set, `ext/openDate.js#isBDay()` gates the page: too early → render the `soon` page, too late → the `late` page (both from `pages.js` via `ext/setPage.js`), on the day → run `animate()`. With no `OPEN_DATE`, it always animates.
- `src/js/animation.js` — the whole scene state machine. A single `.btn` element changes its class (`switch` → `door-out` → `door-in` → `gift`) to advance scenes; each click triggers a transition, plays an SFX, and reveals the next scene's text via `readMsg()` (timed reveals, 5s apart). The final `gift` click branches on `process.env.SCROLL_MSG` to show either the card directly or the scrolling message first.
- `src/scss/main.scss` imports partials in order: `_mixins`, `_layout`, `_base`, `_components`, `_animations`.

Note: `src/js/config.js` is not imported by the runtime entry and appears vestigial (it references `BIRTH_DATE`, whereas the active gating uses `OPEN_DATE`).

## Environment variables

Mandatory: `NAME`, `PIC`. Optional: `NICKNAME`, `HBD_MSG`, `SCROLL_MSG`, `OPEN_DATE`. The meaning of `PIC`/`SCROLL_MSG` differs by mode — a local filename in `./local/` for local builds, a URL for remote deploys. Full reference in `docs/variables.md`; feature setup in `docs/customizations.md`.

## Deployment

Remote deploys (Vercel/Netlify) run `npm run build` (= `init-index-remote` + parcel build) and read env vars from the platform. `netlify.toml` publishes `./dist/`. Telegra.ph is used both as an image host and as the source for remote scroll messages.
