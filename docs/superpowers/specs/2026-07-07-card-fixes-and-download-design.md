# Design: Overscroll fix, bigger petal burst, downloadable card (ZIP)

Date: 2026-07-07

## Context

Single-page Japanese-style animated birthday card (`index.html`), no build tools,
fonts loaded from Google Fonts CDN. Two fixes and one new feature requested.

## 1. Fix: white flash on overscroll

**Bug** (see `bugs/1.png`): scrolling past the top or bottom of the page triggers
browser rubber-band overscroll, revealing a white gap above/below the indigo
gradient. Falling petals are visible over this white gap too.

**Cause**: `body` has the indigo gradient background, but `html` has no
background set, so the browser's default white shows through during
overscroll bounce.

**Fix**:
- Set the same gradient (or a matching solid base color) on `html`.
- Add `overscroll-behavior-y: none` to `html`/`body` to disable the rubber-band
  bounce entirely — this is a static card page, no scroll-chaining needed.

## 2. Fix: weak petal burst

**Bug**: clicking the hanko seal spawns 14 petals from a fixed `50vh` position
with a short 1–2s animation — reads as a small flick rather than a bloom, and
if the user has scrolled, the burst doesn't originate from the seal.

**Fix**:
- Compute the hanko's actual on-screen position (`getBoundingClientRect()`) at
  click time and spawn the burst there, so it works at any scroll position.
- Increase count from 14 to ~28, vary size/rotation/duration more per petal,
  extend the animation slightly so the burst is clearly visible.

## 3. Feature: download card as ZIP (PNG + PDF)

Clicking the hanko seal now does both: plays the (boosted) petal burst, and
downloads a ZIP containing a PNG and a PDF snapshot of the card.

**Libraries** (CDN `<script>` tags, same zero-build pattern as the existing
Google Fonts `<link>` tags):
- `html2canvas` — renders the `.card` DOM element to a canvas/PNG.
- `jsPDF` — wraps the PNG into a PDF.
- `JSZip` — bundles PNG + PDF into one `.zip`.

**Capture scope**: `.card` element only (washi paper box: branch art, title,
tategaki text, message, hanko). Excludes the indigo background and falling
petals, which live outside `.card` in the DOM.

**PDF sizing**: custom page size matching the card's captured pixel
dimensions (converted to PDF points), so the card fills the page edge-to-edge
with no margins.

**Flow on seal click**:
1. Fire boosted petal burst immediately (instant visual feedback).
2. Hanko label/seal shows a brief "generating..." state.
3. `html2canvas` captures `.card` → PNG data.
4. `jsPDF` builds a PDF sized to the card, embeds the PNG full-bleed.
5. `JSZip` bundles `donia-birthday-card.png` + `donia-birthday-card.pdf`.
6. Zip blob downloaded via a temporary `<a download>` link.
7. Hanko reverts to normal state.

**Error handling**: none beyond the above. If a CDN script fails to load or
capture throws, the download silently fails (burst still plays). This is a
personal one-page card, not worth building fallback UI for.

## Out of scope

- Server-side rendering/generation.
- Retry/fallback UI for failed downloads.
- Any build tooling — stays a single static `index.html`.
