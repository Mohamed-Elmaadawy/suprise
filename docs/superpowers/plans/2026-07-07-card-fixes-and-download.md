# Card Fixes & Download Feature Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix the overscroll white-flash bug, make the hanko petal burst bigger/originate from the seal, and add a "download card as ZIP (PNG+PDF)" feature triggered by clicking the hanko seal.

**Architecture:** Single static `index.html`, no build step. All changes are inline CSS/JS edits to this one file. New download feature pulls three libraries (`html2canvas`, `jsPDF`, `JSZip`) from CDN via `<script>` tags, same zero-build pattern as the existing Google Fonts `<link>` tags.

**Tech Stack:** Vanilla HTML/CSS/JS. CDN libs: `html2canvas@1.4.1`, `jspdf@2.5.1` (UMD build), `jszip@3.10.1`.

## Global Constraints

- No build tools, no package.json — stays a single `index.html` file.
- New third-party code loaded via CDN `<script>` tags only (cdnjs), matching the existing Google Fonts CDN pattern.
- Capture scope for download is the `.card` element only — not the indigo background or falling petals.
- PDF page size must exactly match the card's rendered pixel dimensions (full-bleed, no margins).
- Downloaded zip contains exactly two files: `donia-birthday-card.png` and `donia-birthday-card.pdf`.
- Spec: `docs/superpowers/specs/2026-07-07-card-fixes-and-download-design.md`

---

### Task 1: Fix overscroll white flash

**Files:**
- Modify: `index.html` (inside the `<style>` block)

**Interfaces:**
- Produces: no new JS symbols. Purely visual/CSS fix, verified in-browser.

- [ ] **Step 1: Add `html` background + disable overscroll bounce**

Find this exact block in `index.html`:

```css
  * { margin: 0; padding: 0; box-sizing: border-box; }

  body {
    min-height: 100vh;
    background: linear-gradient(160deg, var(--indigo-deep) 0%, var(--indigo) 55%, #46557a 100%);
```

Replace it with:

```css
  * { margin: 0; padding: 0; box-sizing: border-box; }

  html {
    min-height: 100%;
    background: linear-gradient(160deg, var(--indigo-deep) 0%, var(--indigo) 55%, #46557a 100%);
  }

  html, body {
    overscroll-behavior-y: none;
  }

  body {
    min-height: 100vh;
    background: linear-gradient(160deg, var(--indigo-deep) 0%, var(--indigo) 55%, #46557a 100%);
```

**Step 1 explanation:** `html` previously had no background, so browser overscroll rubber-banding revealed white. Giving `html` the same gradient closes that gap; `overscroll-behavior-y: none` stops the bounce itself since this static card page has no need for scroll-chaining.

- [ ] **Step 2: Verify in browser with Playwright**

Run:
```
mcp__playwright__browser_navigate to file:///<absolute-path-to>/index.html
mcp__playwright__browser_resize to width=800 height=600
mcp__playwright__browser_evaluate: () => getComputedStyle(document.documentElement).overscrollBehaviorY
```
Expected: returns `"none"` (not `"auto"`).

Then:
```
mcp__playwright__browser_evaluate: () => getComputedStyle(document.documentElement).backgroundImage
```
Expected: contains `"linear-gradient"` (not `"none"`).

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "fix: close overscroll white gap and disable scroll bounce"
```

---

### Task 2: Boost the hanko petal burst

**Files:**
- Modify: `index.html` (inside the `<script>` block, the `burst()` function)

**Interfaces:**
- Consumes: `.petal` CSS class and its `fall`/`sway` keyframe animations (unchanged, defined in Task 1's untouched CSS).
- Produces: `burst()` function (same name/signature as before — no args, no return value) — Task 4 will call it from `onSealClick()`.

- [ ] **Step 1: Replace the `burst()` function**

Find this exact block in `index.html`:

```javascript
  // petal burst when the seal is tapped
  function burst() {
    for (let i = 0; i < 14; i++) {
      const p = document.createElement('div');
      p.className = 'petal';
      p.style.left = (45 + Math.random() * 10) + 'vw';
      p.style.top = '50vh';
      p.style.animationDuration = (2 + Math.random() * 2) + 's, 1s';
      p.style.animationDelay = '0s, 0s';
      document.body.appendChild(p);
      setTimeout(() => p.remove(), 4000);
    }
  }
```

Replace it with:

```javascript
  // petal burst when the seal is tapped, originating from the hanko itself
  function burst() {
    const hanko = document.querySelector('.hanko');
    const rect = hanko.getBoundingClientRect();
    const originLeftVw = (rect.left + rect.width / 2) / window.innerWidth * 100;
    const originTopPx = rect.top + rect.height / 2;
    for (let i = 0; i < 28; i++) {
      const p = document.createElement('div');
      p.className = 'petal';
      p.style.left = (originLeftVw - 6 + Math.random() * 12) + 'vw';
      p.style.top = originTopPx + 'px';
      const dur = 1.6 + Math.random() * 2.2;
      p.style.animationDuration = dur + 's, 1s';
      p.style.animationDelay = '0s, 0s';
      const s = 8 + Math.random() * 16;
      p.style.width = s + 'px';
      p.style.height = s + 'px';
      document.body.appendChild(p);
      setTimeout(() => p.remove(), (dur + 1) * 1000);
    }
  }
```

**Step 1 explanation:** origin now comes from the hanko's live `getBoundingClientRect()` so the burst starts at the seal regardless of scroll position; count goes 14→28, each petal gets randomized size (8–24px) and duration (1.6–3.8s) for a fuller "bloom" look; cleanup timeout matches each petal's own duration instead of a flat 4s.

- [ ] **Step 2: Verify in browser with Playwright**

```
mcp__playwright__browser_navigate to file:///<absolute-path-to>/index.html
mcp__playwright__browser_click on the hanko seal (element with text "祝")
mcp__playwright__browser_evaluate: () => document.querySelectorAll('.petal').length
```
Expected: `18 + 28 = 46` immediately after click (18 ambient + 28 burst), dropping back toward 18 a few seconds later.

```
mcp__playwright__browser_take_screenshot
```
Expected: visible cluster of petals around the hanko, not at a fixed page-center position unrelated to scroll.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: make petal burst originate from hanko and bloom bigger"
```

---

### Task 3: Add CDN libraries for capture/PDF/zip

**Files:**
- Modify: `index.html` (add `<script>` tags before the existing inline `<script>` block, just before `</body>`)

**Interfaces:**
- Produces: global `html2canvas` function, `window.jspdf.jsPDF` constructor, global `JSZip` constructor — all consumed by Task 4's `downloadCard()`.

- [ ] **Step 1: Add the three CDN script tags**

Find this exact line in `index.html`:

```html
<script>
  // falling petals
```

Replace it with:

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/html2canvas/1.4.1/html2canvas.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/jszip/3.10.1/jszip.min.js"></script>
<script>
  // falling petals
```

- [ ] **Step 2: Verify libraries load with Playwright**

```
mcp__playwright__browser_navigate to file:///<absolute-path-to>/index.html
mcp__playwright__browser_console_messages
```
Expected: no 404/network-error messages for the three CDN scripts.

```
mcp__playwright__browser_evaluate: () => [typeof html2canvas, typeof window.jspdf?.jsPDF, typeof JSZip]
```
Expected: `["function", "function", "function"]`.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "chore: add html2canvas, jsPDF, JSZip via CDN"
```

---

### Task 4: Download card as ZIP (PNG + PDF) on seal click

**Files:**
- Modify: `index.html` (add `downloadCard()` and `onSealClick()` to the inline `<script>` block; change the hanko `onclick` attribute)

**Interfaces:**
- Consumes: `burst()` from Task 2 (no args, no return), `html2canvas`/`window.jspdf.jsPDF`/`JSZip` globals from Task 3.
- Produces: `downloadCard()` (async, no args, no return — side effect: triggers a file download), `onSealClick()` (no args, no return — calls `burst()` then `downloadCard()`).

- [ ] **Step 1: Change the hanko's onclick and add the new functions**

Find this exact line in `index.html`:

```html
  <div class="hanko" title="心" onclick="burst()">祝</div>
```

Replace it with:

```html
  <div class="hanko" title="心" onclick="onSealClick()">祝</div>
```

Find this exact block in `index.html` (the end of the `<script>` block, right before `</script>`):

```javascript
      document.body.appendChild(p);
      setTimeout(() => p.remove(), (dur + 1) * 1000);
    }
  }
</script>
```

Replace it with:

```javascript
      document.body.appendChild(p);
      setTimeout(() => p.remove(), (dur + 1) * 1000);
    }
  }

  // capture the card and download PNG + PDF bundled in a ZIP
  async function downloadCard() {
    const hanko = document.querySelector('.hanko');
    const card = document.querySelector('.card');
    const originalText = hanko.textContent;
    hanko.textContent = '…';
    hanko.style.pointerEvents = 'none';
    try {
      const canvas = await html2canvas(card, { scale: 2, backgroundColor: null });
      const pngBlob = await new Promise(resolve => canvas.toBlob(resolve, 'image/png'));

      const { jsPDF } = window.jspdf;
      const pageWidth = card.offsetWidth;
      const pageHeight = card.offsetHeight;
      const doc = new jsPDF({ unit: 'px', format: [pageWidth, pageHeight] });
      const imgData = canvas.toDataURL('image/png');
      doc.addImage(imgData, 'PNG', 0, 0, pageWidth, pageHeight);
      const pdfBlob = doc.output('blob');

      const zip = new JSZip();
      zip.file('donia-birthday-card.png', pngBlob);
      zip.file('donia-birthday-card.pdf', pdfBlob);
      const zipBlob = await zip.generateAsync({ type: 'blob' });

      const url = URL.createObjectURL(zipBlob);
      const a = document.createElement('a');
      a.href = url;
      a.download = 'donia-birthday-card.zip';
      document.body.appendChild(a);
      a.click();
      a.remove();
      URL.revokeObjectURL(url);
    } catch (err) {
      console.error('Card download failed:', err);
    } finally {
      hanko.textContent = originalText;
      hanko.style.pointerEvents = '';
    }
  }

  function onSealClick() {
    burst();
    downloadCard();
  }
</script>
```

**Step 1 explanation:** `html2canvas` captures `.card` at 2x scale for a crisp PNG; the PDF page is sized in `px` to the card's actual `offsetWidth`/`offsetHeight` so the image fills it edge-to-edge; `JSZip` bundles both into one file; the hanko shows `…` and ignores clicks while generating, reverting in `finally` whether it succeeds or fails. Any failure is swallowed to a `console.error` per the spec's "no fallback UI" decision — the burst still plays either way since `burst()` runs first and independently.

- [ ] **Step 2: Verify the generation pipeline with Playwright (without exercising the OS download dialog)**

```
mcp__playwright__browser_navigate to file:///<absolute-path-to>/index.html
mcp__playwright__browser_evaluate:
  async () => {
    const card = document.querySelector('.card');
    const canvas = await html2canvas(card, { scale: 2, backgroundColor: null });
    const pngBlob = await new Promise(r => canvas.toBlob(r, 'image/png'));
    const { jsPDF } = window.jspdf;
    const doc = new jsPDF({ unit: 'px', format: [card.offsetWidth, card.offsetHeight] });
    doc.addImage(canvas.toDataURL('image/png'), 'PNG', 0, 0, card.offsetWidth, card.offsetHeight);
    const pdfBlob = doc.output('blob');
    const zip = new JSZip();
    zip.file('donia-birthday-card.png', pngBlob);
    zip.file('donia-birthday-card.pdf', pdfBlob);
    const zipBlob = await zip.generateAsync({ type: 'blob' });
    return { pngSize: pngBlob.size, pdfSize: pdfBlob.size, zipSize: zipBlob.size };
  }
```
Expected: all three sizes `> 0`.

- [ ] **Step 3: Verify the real click path with Playwright**

```
mcp__playwright__browser_click on the hanko seal (element with text "祝")
mcp__playwright__browser_console_messages
```
Expected: no "Card download failed" error logged.

```
mcp__playwright__browser_evaluate: () => document.querySelector('.hanko').textContent
```
Expected (after waiting ~1s for generation to finish): back to `"祝"`, not stuck on `"…"`.

**Manual check (not automatable headlessly):** open `index.html` in a real browser, click the hanko, confirm `donia-birthday-card.zip` lands in the downloads folder and contains a valid PNG and PDF of the card.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: download card as PNG+PDF zip when hanko seal is clicked"
```

---

## Final Manual Verification

- [ ] Open `index.html` in a real browser window sized small enough to require scrolling.
- [ ] Scroll past the top and bottom — confirm no white flash, no bounce.
- [ ] Click the hanko — confirm bigger petal burst originates from the seal.
- [ ] Confirm `donia-birthday-card.zip` downloads and unzips to a valid PNG + PDF matching the card.
