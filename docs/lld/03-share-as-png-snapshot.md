# LLD 3: Share as PNG snapshot

## Scope

Add a "Save a snapshot" affordance to the live site (`src/index.html`) that renders the
visitor's current life-in-weeks view to a downloadable PNG via `<canvas>`, for sharing.

**In scope:**
- A quiet outlined "Save a snapshot" button beneath the headline stats.
- A share-sheet modal (blur backdrop, layout toggle, live canvas preview, download + optional native share).
- A single canvas renderer driving three layouts from one shared branded frame and a `FORMATS` size table:
  - **Both** (hero stat band + full grid, 1080×1350, 4:5) — **the default**.
  - **Full grid** (1080×1350, 4:5).
  - **Summary card** (big % + decade band, 1080×1080, 1:1).
- Download PNG (all platforms) and native Share (progressive enhancement where `navigator.canShare({files})` is supported).
- A toast confirmation on save.
- A **static, non-personalized** `og-image.png` + OG/Twitter meta tags for link unfurls.

**Explicitly NOT in scope:**
- **No dynamic / per-visitor OG image.** That would require server-side rendering and would
  leak the birthday off-device — it violates the client-side-only privacy invariant. We ship
  only one static branded `og-image.png`. This scope guard is intentional and confirmed here.
- **No milestones in the PNG now.** Issue #2 (labeled milestones) ships next and its dots must
  later appear in the snapshot. The grid-drawing routine is structured so milestone marks can be
  layered in later (see Approach → Forward compatibility) but milestones are NOT implemented here.
- No backend, storage, analytics, or any network transmission of the birthday/derived state.
- No new fonts, no build step, no dependencies.

## Approach

**Client-side only (hard invariant).** The PNG is produced entirely in the browser: draw to an
in-memory/off-screen `<canvas>`, then `canvas.toBlob()` → object URL → `<a download>` (and
optionally `navigator.share({files})`). Nothing about the birthday or derived stats leaves the
device. This is consistent with `src/index.html`, which already keeps all state in the URL hash.

**Reuse the existing render state, don't recompute.** `src/index.html`'s `render()` already
computes `lived`, `remaining`, `pct`, `lifespan`, and `totalWeeks`. Refactor `render()` to store
the last-computed values in a module-level `state` object; the snapshot renderer reads that object
so the PNG always matches exactly what the visitor is looking at. The share button is disabled
until a valid render has produced state (i.e. a birthday is entered).

**One frame, three layouts, one size table.** All three layouts call the same helpers for the
shared branded frame — `drawFrame` (background + dot-grid texture), `drawTitle` ("your life in
*weeks*"), `drawFooter` (wordmark `weeks.danbing.app` + a stat line). Per-layout size and preview
aspect ratio come from a single `FORMATS` table (see Interfaces). Adding or removing a layout is a
table entry plus one `renderX(W,H)` function — layouts stay cheap. This matches the reference
mockup (`design-mockups/share-as-png-snapshot.html`, live at
`https://harennon.github.io/weeks/share-as-png-snapshot.html`), which already implements this exact
structure with a working canvas renderer.

**Default = "both", but keep the toggle.** Owner-approved direction: default the layout to
**"both"** (hero stat band + full grid). The reference mockup defaults to `"full"`; the production
code MUST default to `"both"` and set the "Both" toggle button's `aria-pressed="true"` initially.
All three options remain in the toggle — do not drop it.

**Fonts before first paint.** Canvas text does not wait for web fonts; if we paint before
DM Sans / Libre Baskerville load, the export falls back to system fonts and looks off-brand.
`await document.fonts.ready` (or `document.fonts.ready.then(...)`) before the first snapshot paint,
and re-render if the sheet is already open when fonts resolve. The button may open the sheet before
fonts are ready; render a placeholder/skeleton and repaint on `fonts.ready`.

**High-DPI note.** Export dimensions are fixed device pixels (e.g. 1080×1350), so we set
`canvas.width`/`canvas.height` to the format size and draw in those coordinates — no
`devicePixelRatio` scaling needed. The on-screen preview is scaled down with CSS
(`canvas { width:100%; height:100% }`); the exported bitmap is always full-resolution.

**Forward compatibility (milestones, issue #2).** Keep `drawLifeGrid(left,right,top,bottom)` as the
single grid-drawing routine and have it iterate weeks through a per-week color/decoration decision
so a later change can layer milestone marks (ring, label, or accent dot) on top of a week without
rewriting layout math. Concretely: factor the per-dot fill decision into a small
`weekStyle(i)` → `{ fill }` step today; issue #2 can extend that return to include an optional
milestone overlay and add a post-pass draw. Do NOT implement milestones now.

## Interfaces / Types

All code lives inline in `src/index.html` (vanilla JS, no modules), matching the current file.

**Shared render state (populated by the refactored `render()`):**

```js
// Module-level; the single source of truth the snapshot reads.
const state = {
  birthday: "",        // "YYYY-MM-DD" (used only for the download filename, never transmitted)
  lifespanYears: 90,
  weeksPerYear: 52,
  totalWeeks: 4680,
  lived: 0,            // clamped to totalWeeks
  remaining: 0,
  pct: 0,              // number, one decimal (e.g. 38.6)
  hasData: false,      // false until a valid birthday has rendered
};
```

**Format table (single source for size + preview shape):**

```js
const FORMATS = {
  both:    { w: 1080, h: 1350, ratio: "portrait", label: "Both" },        // DEFAULT
  full:    { w: 1080, h: 1350, ratio: "portrait", label: "Full grid" },
  summary: { w: 1080, h: 1080, ratio: "square",   label: "Summary card" },
};
let format = "both"; // default
```

**Canvas palette (mirror the CSS tokens exactly):**

```js
const COLORS = {
  bg: "#1a1a1f", text: "#f0f0f0", muted: "rgba(255,255,255,0.45)",
  gold: "#c9a84c", lived: "#c9a84c", current: "#f5f0e8",
  future: "rgba(255,255,255,0.12)",
};
```

**Function signatures (specifications, not implementations):**

```js
function captureState(): void          // called at end of render(); fills `state`
function weekStyle(i: number): {fill}  // per-dot color decision; extension point for #2
function drawFrame(W, H): void         // bg + dot-grid texture
function drawTitle(W, baseline): void  // "your life in weeks" (gold italic "weeks")
function drawFooter(W, H): void        // "weeks.danbing.app" + "{pct}% elapsed · {years}-year life"
function drawLifeGrid(left, right, top, bottom): void  // full 52×N grid fit to rect
function renderBoth(W, H): void        // hero stat band + full grid
function renderFull(W, H): void        // full grid + one-line stat caption
function renderSummary(W, H): void     // big % + "of my life, elapsed" + decade band
function renderSnapshot(): void        // sets canvas.w/h from FORMATS[format], dispatches
function openSheet(): void / closeSheet(): void
function setFormat(f): void            // updates aria-pressed on all 3 buttons, re-renders
function showToast(msg): void
```

**New DOM (inside `src/index.html`):**
- Action row under `.stats`: `<button id="open-share" class="btn btn-share" aria-haspopup="dialog">Save a snapshot</button>` (disabled until `state.hasData`).
- Overlay: `<div id="overlay" class="overlay" role="dialog" aria-modal="true" aria-labelledby="sheet-title">` containing the sheet:
  - `<h2 id="sheet-title">`, close button (`aria-label="Close"`).
  - Format toggle: `role="group"`, three `<button aria-pressed>` (`fmt-both` pressed by default).
  - `.preview` wrapper (`data-ratio`) holding `<canvas id="snap">`.
  - Actions: `#download` (primary), `#native-share` (secondary, `hidden` unless supported), privacy hint line.
- Toast: `<div id="toast" class="toast">`.
- CSS: port the mockup's `.btn/.btn-share`, `.overlay/.sheet`, `.format-toggle`, `.preview`, `.sheet-actions`, `.toast` rules into the site's `<style>` block, reusing existing `:root` tokens.

**New static asset + `<head>` tags:**
- `src/og-image.png` — static 1200×630 branded card ("your life in weeks", dot-grid, wordmark). Not personalized.
- Add to `<head>`: `og:title`, `og:description`, `og:image` (absolute `https://weeks.danbing.app/og-image.png`), `og:url`, `og:type=website`, `twitter:card=summary_large_image`, `twitter:image`.

## Frontend Design

Incorporates the approved frontend architecture decision: **ship BOTH as the default, but keep the
layout toggle.**

- **Default layout = "both".** When the sheet opens, the canvas renders the "both" layout (hero stat
  band on top, full grid below, 1080×1350). The "Both" toggle button starts with
  `aria-pressed="true"`. This deviates intentionally from the mockup (which defaults to "full").
- **Toggle stays.** All three options remain: **Full grid** (1080×1350), **Summary card**
  (1080×1080), **Both** (1080×1350, default). The toggle is a pill segmented control
  (`role="group"`, `aria-pressed` per button) matching the mockup. Do not drop the toggle.
- **Quiet outlined trigger.** "Save a snapshot" is an outlined gold pill (`.btn-share`: transparent
  bg, gold border/text, subtle hover glow) placed in a right-aligned action row just beneath the
  headline stats — low-chrome, per the design philosophy.
- **Share sheet is a modal.** Fixed overlay with a blurred backdrop (`backdrop-filter: blur(6px)`),
  centered sheet, fade/scale-in transition. Closes on Esc and on backdrop click (not sheet click);
  focus moves to the close button on open and returns to the trigger on close.
- **Live canvas preview = the export.** The `<canvas>` inside `.preview` is scaled to 100% by CSS;
  the `.preview[data-ratio]` wrapper flips between `portrait` (4:5) and `square` (1:1) so the preview
  frame always matches the exported bitmap. A brief shimmer overlay can show while painting.
- **Actions.** Primary "Download PNG" (solid gold) is always present. "Share…" (secondary) appears
  only when `navigator.canShare({files})` is supported — progressive enhancement, mostly mobile.
- **Toast on save.** A centered bottom pill ("Snapshot saved ✓") fades in for ~2.2s after download.
- **Privacy hint** with a lock glyph in the sheet: "Rendered entirely in your browser. Your birthday
  is never sent anywhere." — reinforces the invariant at the share moment.
- **Brand fidelity.** Reuse existing `:root` tokens and both web fonts; gate first paint on
  `document.fonts.ready` so exports are on-brand.

## State Model

- **In-memory only.** `state` is recomputed on every input change and read by the snapshot
  renderer at open/toggle/paint time. The `<canvas>` is the single source of truth for both the
  preview and the downloaded bytes (preview = the same canvas scaled by CSS).
- **UI/session state:** `format` (default `"both"`), and overlay open/closed. Neither is persisted;
  format resets to `"both"` on reload. (Persisting format is out of scope — no added value, and the
  URL hash is reserved for birthday+lifespan.)
- **Persisted:** nothing new. The URL hash continues to carry only `b` (birthday) and `y`
  (lifespan), exactly as today. The PNG bytes and download filename are produced locally and never
  transmitted.
- **Privacy:** no network I/O is introduced. The static `og-image.png` is a fixed asset served by
  Cloudflare Pages; it contains no visitor data.

## Edge Cases

1. **No birthday entered / no valid render yet** — `state.hasData` is false; keep the "Save a
   snapshot" button `disabled`. Enable it inside `captureState()` once a valid render occurs.
2. **`render()` returns early on invalid date** — do not set `hasData`; button stays disabled.
3. **Fonts not yet loaded when sheet opens** — draw immediately with fallback, then repaint on
   `document.fonts.ready`. Guard: only repaint if the overlay is still open.
4. **`lived === 0`** (birthday today/future-clamped) — grid is all "future"; stats show 0 / 0.0%.
   Renderer must not divide-by-zero: `pct` uses `totalWeeks` (always ≥ 52) as denominator.
5. **`lived` clamped to `totalWeeks`** (very old birthday vs short lifespan) — reuse the existing
   clamp from `render()`; the "current" dot is the last cell, no out-of-range index.
6. **Very tall grid (large lifespan, e.g. 120 yrs)** — `drawLifeGrid` computes `cell` as the min of
   width- and height-constrained sizes and vertically centers, so the grid always fits its band
   regardless of row count. No overflow.
7. **`toBlob` returns null / throws** (rare, memory pressure) — guard the callback; if `blob` is
   falsy, show an error toast ("Couldn't render — try again") and do not attempt download.
8. **`navigator.canShare` present but `canShare({files})` false** (e.g. desktop Chrome) — keep the
   native Share button hidden / no-op; Download PNG is always available. Detect with a real
   `{files:[testFile]}` check, not just `navigator.share` existence.
9. **User cancels the native share dialog** — `navigator.share` rejects with `AbortError`; swallow
   it silently (no error toast).
10. **Rapid open/close or toggle spamming** — `renderSnapshot()` is synchronous and idempotent;
    each call fully repaints. No debounce needed.
11. **Esc key / click-outside** — Esc closes only when overlay is open; click closes only when the
    click target is the overlay backdrop itself (not the sheet). Return focus to `#open-share`.
12. **Focus management / a11y** — move focus to the close button on open; trap is not required for
    MVP but focus must return to the trigger on close. Toggle buttons expose `aria-pressed`.
13. **Download filename** — `my-life-in-weeks-{format}.png`. Filename derived from `format` only,
    not from the birthday, to avoid leaking a personal detail into a shared filename.

## Dependencies

- **`src/index.html`** — existing live site; this LLD refactors `render()` to publish `state` and
  adds the button, modal, canvas renderer, and CSS. No other source files change behavior.
- **Reference mockup** — `design-mockups/share-as-png-snapshot.html` (live:
  `https://harennon.github.io/weeks/share-as-png-snapshot.html`). The production renderer, FORMATS
  table, share-sheet markup/interaction, and progressive-enhancement logic should match it, with
  the one deviation that production defaults to `format = "both"`.
- **Fonts** — DM Sans + Libre Baskerville already loaded via Google Fonts `<link>` in
  `src/index.html`; `document.fonts.ready` gates the first paint.
- **`src/_headers`** — existing security headers are compatible (no CSP that would block inline
  script/canvas or the fonts already used). Adding `og-image.png` needs no header change. If a CSP
  is added later, it must allow `img-src 'self' data: blob:` and the existing Google Fonts origins.
- **New asset** — `src/og-image.png` must be produced (static export from the "both"/branded frame
  at 1200×630 is a convenient way to generate it) and committed before the OG tags resolve.
- **No milestones dependency** — issue #2 depends on the `weekStyle`/`drawLifeGrid` extension point
  defined here, but this LLD does not depend on #2.

## Test Requirements

**Manual / QA (primary — no test runner in this static project):**
- Enter a birthday → "Save a snapshot" enables; open sheet → preview paints in "both" by default.
- Toggle Full grid / Summary card / Both → preview reshapes (4:5 ↔ 1:1) and repaints; `aria-pressed`
  tracks the active button.
- Download PNG on desktop → a `.png` downloads and visibly shows the grid + a headline stat (the
  acceptance criterion). Open the file; confirm brand fonts rendered (not fallback), gold accent,
  wordmark, and `weeks.danbing.app` footer.
- On a mobile browser supporting file share → "Share…" appears and invokes the native sheet with the
  PNG attached; on desktop it is hidden.
- Esc, click-outside, and the × button all close the sheet and restore focus to the trigger.
- Reload with a fully-loaded page (cold cache) → first snapshot still uses brand fonts (validates
  `fonts.ready` gating).

**Rendering correctness (visual):**
- Extreme lifespans (1 yr, 120 yrs) fit within the grid band with no clipping/overflow.
- `lived === 0` and `lived === totalWeeks` render without NaN/`Infinity` in stat text.
- The exported bitmap dimensions equal `FORMATS[format].w/h` exactly.

**Privacy / security (must verify):**
- Network panel shows **zero** requests triggered by opening the sheet, toggling, downloading, or
  sharing (beyond already-loaded fonts). No birthday value in any request, filename, or the static
  `og-image.png`.
- `og-image.png` is static and identical for every visitor (contains no personal data).
- Confirm no dynamic/personalized OG generation path exists.

**Regression:**
- Existing MVP flow (input → grid → live stats → URL-hash persistence) is unchanged; the `render()`
  refactor still updates the DOM stats and hash exactly as before.
