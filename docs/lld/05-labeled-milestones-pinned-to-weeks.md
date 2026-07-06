# LLD 5: Add labeled milestones pinned to weeks

## Scope

Let a visitor attach a small set of labeled milestones ("started school", "moved cities")
to specific weeks on the life grid. Each milestone is a `{label, date}` pair; the date maps
to a week index, the week's dot is visually marked, and the whole set is encoded into the
URL hash alongside `b` (birthday) and `y` (lifespan) so a shared link reconstructs everything.
This builds directly on `src/index.html` as it stands after LLD 3 (the snapshot feature).

**In scope:**
- A slim single-row add UI beneath the grid: text label + date input + "+ Pin it". No modal, no CRUD panel.
- Marking the milestone week in the **live DOM grid**: brighter gold dot + soft halo + pop-in animation; label revealed as an accessible tooltip on hover/focus.
- Clicking an **empty** dot prefills the add row's date with that week's date (fast-add affordance).
- Removal two ways: (a) removable age-tagged chips listing current milestones; (b) clicking a **pinned** dot removes that milestone.
- A compact, URL-length-conscious `m` param in the existing `URLSearchParams` hash; validate/sanitize on restore.
- **Desktop layout (direction A):** labels sit in a right-hand gutter as "margin flags," each tied to its year-row by a thin gold rule.
- **Mobile / narrow layout (direction C):** quiet gold dots on the grid + an ordered timeline legend below (no gutter flags).
- Milestone dots + labels also render into the **PNG snapshot** (extend `weekStyle`/`drawLifeGrid`/layout renderers from LLD 3), so native share does not regress.

**Explicitly NOT in scope:**
- No backend, storage, accounts, or analytics. Nothing is transmitted (hard privacy invariant).
- No milestone editing in place (edit = remove + re-add). No drag-to-reposition.
- No categories, colors-per-milestone, icons, or notes — label + date only.
- No unbounded list. A hard cap (see Edge Cases) keeps the URL share-friendly.
- No new fonts, no build step, no dependencies.

## Approach

**Milestones are a first-class part of URL state, next to `b`/`y`.** `render()` currently
writes `new URLSearchParams({ b, y })` and `restoreFromHash()` reads `b`/`y`. We add a third
key `m`. The in-memory source of truth is a module-level `milestones` array; `render()`
serializes it into the hash, `restoreFromHash()` parses+sanitizes it back. This keeps a
single write path (the hash is still the only persistence) and satisfies the acceptance
criterion (add → pin → shareable URL reconstructs it).

**Compact `m` encoding (URL-length-conscious).** Each milestone is a `label|weekIndex` pair,
milestones joined by `~`. We store the **week index** (integer, derived from the milestone
date relative to the birthday) rather than a full ISO date — it is far shorter and is exactly
what the grid needs to place the dot. The label is percent-decoded/encoded by
`URLSearchParams` itself (the value is placed via the params object, so `~` and `|` are our
own delimiters inside one param value and must be stripped from labels — see Edge Cases).

```
#b=1990-05-01&y=90&m=started+school|1092~moved+cities|1560
```

Rationale for week-index over date: (1) shortest encoding; (2) the grid is week-quantized
anyway, so no information is lost that the grid can show; (3) reconstructing a display date is
trivial (`birthday + weekIndex*7 days`) for the chip's age tag. The date input in the add row
is the user-facing capture; we convert date → week index on pin and never store the raw date.

**Reuse LLD 3's extension seams; do not fork the renderers.** LLD 3 deliberately left two seams:
- `weekStyle(i)` returns the per-dot fill decision. We extend its return to `{ fill, milestone }`
  where `milestone` is the matching milestone object (or `null`). Both the live DOM builder and
  the canvas grid consult this, so marking logic lives in one place.
- `drawLifeGrid(left,right,top,bottom)` already computes `originX/originY/cell/gap`. We expose
  those via a returned geometry object (or a shared closure) so a **post-pass** can draw the
  milestone halo + label flags on the PNG without touching layout math. The live DOM equivalent
  is CSS classes on the affected `.wk` nodes.

**Direction A (desktop) with C (mobile), chosen at render time by viewport width.** A single
CSS breakpoint (`min-width: 700px` ≈ the point where the 900px-max grid has room for a gutter)
decides which presentation is active. Both presentations read the same `milestones` array; only
the DOM/CSS differs. This avoids a JS layout engine — CSS media queries flip between the gutter
flags (A) and the timeline legend (C). The grid dots themselves (halo + brighter gold) are
identical in both; only the *labels'* placement changes.

**Marking = decoration, never a layout change.** A pinned dot keeps its grid cell; we add a
class (`.wk.milestone`) that brightens the fill, adds a halo (`box-shadow`), and runs a one-shot
pop-in animation. The label is exposed for a11y via `aria-label` / `title` on the dot and, on
desktop, mirrored as a visible gutter flag. This keeps the 52-column grid math untouched.

**Client-side only.** Milestone labels are personal free text; they live only in memory and the
URL hash, exactly like the birthday. No network I/O is added. Labels are sanitized on both write
(strip delimiters/control chars, length-cap) and read (same, before rendering) to keep the hash
well-formed and prevent injection into `title`/`aria-label`/canvas text.

## Interfaces / Types

All code is inline in `src/index.html` (vanilla JS, no modules), matching the current file.

**In-memory milestone model (module-level, single source of truth):**

```js
// Ordered by weekIndex ascending after every mutation.
let milestones = []; // Array<Milestone>
// Milestone: { label: string, week: number }
//   label: sanitized, ≤ MAX_LABEL_LEN chars, no delimiters/control chars
//   week:  integer week index, 0 <= week < state.totalWeeks

const MAX_MILESTONES = 12;   // hard cap (URL-length + visual clarity)
const MAX_LABEL_LEN  = 40;   // per-label char cap
const M_SEP   = "~";         // between milestones
const M_KV    = "|";         // between label and week within one milestone
```

**Encoding / decoding (pure, testable functions):**

```js
function sanitizeLabel(raw: string): string
//   trim; remove M_SEP, M_KV, and control chars; collapse whitespace; slice to MAX_LABEL_LEN.

function encodeMilestones(list: Milestone[]): string
//   -> "label|week~label|week..."  (empty string if list empty)
//   labels are already sanitized so delimiters are safe.

function decodeMilestones(raw: string, totalWeeks: number): Milestone[]
//   split on M_SEP; for each part split once on M_KV; sanitizeLabel(label);
//   week = parseInt; DROP entries where: label empty, week NaN,
//   week < 0, or week >= totalWeeks; de-dupe by week (keep first);
//   sort by week asc; cap to MAX_MILESTONES. Never throws.

function weekForDate(birthDate: Date, dateStr: string): number | null
//   floor((date - birthday)/MS_PER_WEEK); null if before birthday or invalid.

function dateForWeek(birthDate: Date, week: number): Date
//   birthday + week*7 days (for chip age label / prefill display).

function ageLabel(week: number): string
//   -> "age 21" (Math.floor(week / WEEKS_PER_YEAR)) for the chip tag.
```

**Extended LLD-3 seams (specifications):**

```js
function weekStyle(i): { fill: string, milestone: Milestone | null }
//   unchanged fill logic; additionally returns the milestone whose .week === i, else null.

function milestoneAt(week: number): Milestone | null   // lookup helper used by weekStyle + DOM/click.
```

**Mutation API (all call render() at the end, which re-serializes hash + repaints):**

```js
function addMilestone(label: string, dateStr: string): void
//   sanitize label; week = weekForDate(...); reject (toast) if invalid/duplicate/at-cap; push; sort; render().
function removeMilestone(week: number): void          // filter out by week; render().
function prefillDate(week: number): void              // set add-row date input to dateForWeek(...).
```

**New DOM (inside `src/index.html`, after the grid, before/near the action row):**

- Add row: `<form id="ms-add" class="ms-add">` containing
  `<input id="ms-label" maxlength="40" placeholder="e.g. started school">`,
  `<input id="ms-date" type="date">`, and `<button type="submit">+ Pin it</button>`.
  Submit calls `addMilestone`; `preventDefault`. Disabled until `state.hasData`.
- Chips: `<ul id="ms-chips" class="ms-chips">` — one `<li>` per milestone: label + `age N` tag +
  a remove `<button aria-label="Remove {label}">×</button>`.
- Desktop gutter (direction A): `<div id="ms-gutter" class="ms-gutter" aria-hidden="true">` — flags
  positioned to align with their year-row; a thin gold rule connects flag → row. `aria-hidden`
  because the chips list is the accessible source of truth; the gutter is decorative reinforcement.
- Mobile legend (direction C): reuse the `#ms-chips` list styled as an ordered timeline below the grid.
- Grid dots gain, when pinned: `class="wk milestone"`, `tabindex="0"`, `role="button"`,
  `aria-label="{label} — age N"`, and `title="{label}"`. Non-pinned dots stay non-interactive
  except empty (future) dots which get a click handler for prefill.

**CSS additions (reuse `:root` tokens, no new colors beyond derived brighter gold):**

```css
.wk.milestone { background: var(--accent-gold); box-shadow: 0 0 0 2px rgba(201,168,76,.35), 0 0 8px 2px rgba(201,168,76,.45); animation: ms-pop .32s ease-out; }
@keyframes ms-pop { 0%{transform:scale(.4);opacity:.3} 60%{transform:scale(1.25)} 100%{transform:scale(1);opacity:1} }
/* direction A gutter flags: absolute-positioned within .grid-wrap; thin gold connector rule. */
/* direction C timeline: #ms-chips as ordered list, shown < 700px, gutter hidden. */
@media (prefers-reduced-motion: reduce){ .wk.milestone{ animation: none } }
```

**PNG (extend LLD-3 canvas):** milestone dots draw with the brighter fill + a stroked halo in a
post-pass inside `drawLifeGrid` (reuse the current-week ring pattern at lines ~757-764). Labels
render as small gutter flags on the right margin in the portrait layouts (`both`, `full`);
`summary` shows a count line ("+ N milestones") rather than every label (space).

## State Model

- **In-memory:** `milestones` array is the single source of truth, kept sorted by `week`.
  Mutated only by `addMilestone` / `removeMilestone`, each of which ends by calling `render()`.
- **Persisted (URL hash only):** `render()` appends `m=encodeMilestones(milestones)` to the
  existing `URLSearchParams({ b, y })` (omit `m` when empty). `restoreFromHash()` reads `m` and
  calls `decodeMilestones(raw, totalWeeks)`. This is the **only** persistence — no localStorage,
  no server. A copied URL fully reconstructs birthday + lifespan + milestones.
- **Restore ordering dependency:** week indices are validated against `totalWeeks`, which depends
  on lifespan (`y`). `restoreFromHash()` must parse `b`/`y` first, then decode `m` using the
  resulting `totalWeeks`; `render()` re-clamps anyway. On boot: `restoreFromHash()` → `render()`.
- **Derived, never stored:** the milestone's calendar date (recomputed via `dateForWeek` for chip
  age tags and prefill) and the `age N` label.
- **UI/session (not persisted):** add-row input contents; which presentation (A vs C) is active
  is purely a CSS media-query outcome, not JS state.
- **Privacy:** labels are personal text and are treated exactly like the birthday — memory + hash
  only, never transmitted; sanitized on both encode and decode.

## Edge Cases

1. **Milestone date before birthday** — `weekForDate` returns `null`; reject with toast
   ("That's before your birthday"). Do not add.
2. **Milestone date beyond lifespan** (week ≥ totalWeeks) — reject with toast ("That's past the
   grid — try a longer lifespan"). Keeps every dot on-grid.
3. **Lifespan later shortened** so an existing milestone falls off-grid — `decodeMilestones` /
   `render` drop weeks ≥ new `totalWeeks`; those milestones silently disappear from grid + chips
   (and thus from the next hash write). Documented, not an error.
4. **Empty / whitespace-only label** — `sanitizeLabel` yields ""; reject add with toast ("Add a
   short label"). On decode, entries with empty labels are dropped.
5. **Duplicate week** (two milestones same week) — on add, reject with toast ("You already pinned
   that week"). On decode, de-dupe keeping the first.
6. **At cap (`MAX_MILESTONES`)** — disable/reject further adds with toast ("Up to 12 milestones");
   decode caps to the first 12.
7. **Label contains delimiters (`~`, `|`) or control chars** — stripped by `sanitizeLabel` on both
   add and decode, so the encoding cannot be broken or the DOM/canvas injected.
8. **Overlong label** — capped to `MAX_LABEL_LEN`; `maxlength` on the input plus slice in
   `sanitizeLabel` (defense in depth for hostile hashes).
9. **Malformed `m` param** (missing `|`, non-numeric week, stray separators, empty segments) —
   `decodeMilestones` drops bad entries and never throws; a partially valid hash still restores
   the good milestones.
10. **Clicking a pinned dot** removes it; **clicking an empty (future) dot** prefills the add-row
    date. Lived/current non-milestone dots: click prefills date too (any dot is a valid target as
    long as its week is on-grid) — but the current/lived state does not block pinning.
11. **No birthday yet** — add row + chips are disabled/hidden (`state.hasData` false); pinning
    requires a birthday to map dates → weeks.
12. **URL length** — 12 milestones × (~40-char label + ~5-char week + 2 delimiters) ≈ well under
    2 KB; browsers/CDNs handle multi-KB URLs. Cap + week-index encoding keep it share-friendly.
13. **Reduced motion** — `@media (prefers-reduced-motion: reduce)` disables the pop-in animation;
    the halo/brighter fill remain.
14. **Two milestones in the same year-row (desktop gutter)** — flags may collide; stack/offset
    vertically within the row band or, if still overlapping, collapse to a single "+N" flag that
    the chips list disambiguates. Timeline legend (mobile) always lists all.
15. **PNG with milestones** — dots + halos always draw; if labels would overflow the portrait
    gutter, truncate with an ellipsis; `summary` layout shows a "+N milestones" line only.
16. **Keyboard access** — pinned dots are focusable (`tabindex=0`, `role=button`); Enter/Space
    removes (mirrors click); the tooltip/label is reachable via focus for screen readers, not just
    hover (satisfies the mobile-reveal + keyboard requirement).

## Dependencies

- **`src/index.html` post-LLD-3** — this LLD extends the existing `weekStyle(i)` (lines ~648-652),
  `drawLifeGrid` (lines ~724-766), `render()` hash write (line ~1083), `restoreFromHash()`
  (lines ~1090-1094), and the boot sequence (`restoreFromHash(); render();`). No new source files.
- **LLD 3 (`docs/lld/03-share-as-png-snapshot.md`)** — must be implemented first; the milestone
  PNG rendering layers onto its `drawLifeGrid` post-pass and layout renderers. Coordinate so the
  snapshot `toBlob`/download/native-share flow is not regressed (the halo/labels are additive draws
  before `toBlob`).
- **Reference mockup** — `https://harennon.github.io/weeks/labeled-milestones.html` (A/B/C switcher).
  Production ships **direction A on desktop, direction C on mobile** (owner-locked); B is not shipped.
- **Fonts / tokens** — reuse existing `:root` gold token and DM Sans / Libre Baskerville already
  loaded; canvas milestone labels use DM Sans and are gated by the existing `document.fonts.ready`.
- **`src/_headers`** — no change; no new origins or network calls introduced.

## Test Requirements

**Manual / QA (primary — no test runner in this static project):**
- Add a milestone (label + date) → its week dot brightens with halo + pop-in; chip appears with
  correct `age N`; URL hash gains `m=...`. Reload the URL → milestone reappears (acceptance).
- Copy the URL to a fresh tab/incognito → birthday, lifespan, and all milestones reconstruct.
- Click an empty dot → add-row date prefills to that week; click a pinned dot → it's removed and
  the hash updates. Remove via chip × → same.
- Desktop (≥700px): gutter flags align to the correct year-rows with connector rules; two
  milestones in one row don't overlap illegibly (stack/collapse rule holds).
- Mobile / narrow (<700px): gutter hidden; ordered timeline legend lists milestones; tapping a
  pinned dot reveals its label accessibly and removes on tap-again/confirm per interaction spec.
- Keyboard: Tab to a pinned dot, focus shows the label, Enter/Space removes it.
- Reduced-motion OS setting: no pop-in animation; halo still present.
- Open the snapshot sheet → milestone dots (halo) and labels appear in `both`/`full`; `summary`
  shows "+N milestones"; Download PNG contains them; native share (where supported) attaches the
  same image. Existing snapshot flows (toggle, download, share, esc/close) still work.

**Encoding correctness (pure-function level, exercised manually or via a scratch harness):**
- Round-trip: `decodeMilestones(encodeMilestones(list), totalWeeks)` equals a sanitized, sorted,
  de-duped, capped `list`.
- `sanitizeLabel` strips `~`, `|`, control chars, collapses whitespace, caps length.
- `decodeMilestones` never throws on: empty string, missing `|`, non-numeric week, out-of-range
  week, >MAX_MILESTONES entries, duplicate weeks, embedded delimiters.
- `weekForDate` returns `null` before birthday and for invalid dates; correct index otherwise.

**Privacy / security (must verify):**
- Network panel shows **zero** requests from adding/removing milestones or copying the URL. Labels
  never appear in any request.
- Hostile hash (`m` with script-y label, delimiters, huge length) restores safely: labels are
  text-only in `title`/`aria-label`/canvas (no HTML injection), over-cap/over-length trimmed.

**Regression:**
- MVP + LLD-3 flows unchanged when no milestones exist: `m` is omitted from the hash; grid, stats,
  and snapshot render exactly as before.
