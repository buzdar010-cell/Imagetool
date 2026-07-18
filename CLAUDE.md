# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file, client-side web app (`index.html`) that bulk-generates images from a list of text prompts using the Together AI API (Flux models). There is no backend, no build step, and no package manager — everything (HTML, CSS, JS) lives in one file. The only external dependency is JSZip, loaded from a CDN (`<script src="https://cdnjs.cloudflare.com/ajax/libs/jszip/3.10.1/jszip.min.js">`) for zipping images before download.

## Running / testing

There is no build, lint, or test tooling in this repo. To work on it:
- Open `index.html` directly in a browser, or serve it locally, e.g. `python3 -m http.server` from the repo root and visit `http://localhost:8000`.
- Manual verification is the only form of testing: paste a Together AI API key, enter a prompt or two, and click "Generate images" to confirm the flow works end-to-end in the browser.
- Since all logic is inline JS in one `<script>` tag, use the browser devtools console to catch runtime errors when testing changes.

## Architecture

Everything is in `index.html`: inline `<style>` for the dark-themed UI, inline `<script>` for all logic. There's no module system — all functions and DOM references are top-level and share global state (`results`, `stopFlag`, `uploadedFileName`).

Core flow, in `startBatch()`:
1. Prompts come from either a pasted textarea or an uploaded `.txt`/`.csv` file (one prompt per line).
2. Each line is parsed by `parseTimestamp()` for an optional leading `[MM:SS]` tag (used as the output filename, e.g. `01_02.png`); the tag is stripped before sending the prompt to the API.
3. Prompts are processed **sequentially** (not in parallel) with a configurable delay between requests (`delayInput`, default 2500ms) to avoid Together AI rate limits — this is important, Flux 2 Pro in particular returns 429s if calls come too fast.
4. Each prompt goes through `generateOne()`, which POSTs to `https://api.together.xyz/v1/images/generations` and converts the base64 response into a blob URL via `base64ToBlobUrl()` (falls back to fetching `item.url` if no base64 is returned).
5. UI state per-prompt is tracked in the global `results` array (`queued` → `generating` → `done`/`error`), and `renderResults()` re-renders the whole results list from that array on every state change (no diffing/virtual DOM — simple full re-render).
6. Individual images can be retried (`retryOne()`) or edited in place before regeneration; completed images can be downloaded individually or all together as a zip (`saveAll()`, using JSZip, with a two-phase progress bar: fetching blobs 0–70%, compressing 70–100%).

Key implementation details worth knowing before editing:
- `aspectToDimensions()` maps aspect-ratio labels (e.g. `"16:9"`) to explicit pixel dimensions required by the API — there's no dynamic computation, just a lookup table.
- The `steps` parameter is deliberately omitted for `FLUX.2-pro` in `generateOne()` because that model rejects it outright; other models get `steps: 4` (schnell) or `28` (others). If adding new models, check whether they accept `steps`.
- The Together AI API key is persisted in `localStorage` (`together_api_key`) and auto-filled on load — it never leaves the browser except in the `Authorization` header sent directly to Together AI's API.
- `getZipFileName()` derives the downloaded zip's name from the uploaded prompts file name (e.g. `image_prompts.txt` → `images.zip`).

## Conventions

- No frameworks, no build tooling, no dependencies beyond the JSZip CDN script — keep changes to plain HTML/CSS/JS in `index.html` consistent with this.
- Errors surfaced to the user are prefixed by category (e.g. `MISSING KEY:`, `NETWORK BLOCK:`, `API ERROR {status}:`, `EMPTY RESPONSE:`) — follow this pattern when adding new error paths so failures are easy to distinguish in the UI.
