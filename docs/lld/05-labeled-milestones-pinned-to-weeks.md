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

**Reuse LLD 3's canvas seam; refactor the DOM builder through the shared decision.** LLD 3 left
`weekStyle(i)` as the per-dot decision point, but *only* the canvas `drawLifeGrid()` consults it
(index.html line 751); the live-grid builder in `render()` (lines 1064-1071) currently inlines its
own class string and never calls `weekStyle()`. To make marking logic live in one place we:
- Extend `weekStyle(i)` to return `{ fill, kind, milestone }` where `kind ∈ {"lived","current","future"}`
  (the existing fill decision, made explicit) and `milestone` is the matching milestone object (or
  `null`). This is a pure lookup with no side effects.
- **Refactor `render()`'s DOM builder to call `weekStyle(i)`** instead of its inline ternary, so the
  `.wk`/`.wk.lived`/`.wk.current` classes are derived from `kind` and the `.milestone` class is
  appended when `milestone !== null`. This is the explicit live-grid marking path. Without this
  refactor milestones would never appear on the live grid (the current defect the reviewer flagged).
  The class-string mapping is: base `wk`; append ` lived`/` current` per `kind`; append ` milestone`
  when a milestone matches (see State Model for how `.milestone` composes with `.current`).
- `drawLifeGrid(left,right,top,bottom)` already computes `originX/originY/cell/gap`. Its per-dot loop
  already reads `weekStyle(i)`; we extend that same loop's post-pass to draw the milestone halo when
  `milestone !== null` (reusing the current-week ring pattern at lines 757-764). No layout math changes.

**Milestone affordances live OUTSIDE the `role="img"` grid.** `#grid` is `role="img"` (index.html
line 524); assistive tech ignores its descendants, so a focusable `role=button`/`aria-label` *inside*
a `.wk` dot would be invisible to screen readers and unreachable by keyboard — it cannot satisfy the
a11y edge case or the keyboard/SR test requirements. We therefore keep the grid a single opaque image
and put **all** interactive + accessible milestone affordances in sibling elements that are real
document content:
- The `#ms-chips` list (and, on mobile, the timeline legend it becomes) is the **accessible, focusable,
  keyboard-operable** representation of every milestone: each chip carries the label, `age N`, and a
  real `<button>` to remove it. This is where screen-reader users read labels and remove milestones.
- The grid `.wk.milestone` dots remain **purely decorative** (no `tabindex`, no `role`, no
  per-dot `aria-label`): brighter fill + halo + pop-in only. `#grid`'s single `aria-label` is updated
  to summarize milestone count (e.g. `"Life in weeks grid, 3 milestones marked"`) so the presence of
  milestones is announced without exposing per-descendant semantics AT would drop anyway.
- Click/tap on a `.wk.milestone` dot is a **mouse/touch convenience** to remove that milestone (a
  redundant shortcut to the chip's remove button, not the primary a11y path), handled by the delegated
  listener below.

**One delegated click listener on `#grid`, never per-dot handlers.** With ~4,680 dots, attaching a
handler per node is wasteful. We add a **single** delegated `click` listener on `#grid`: on click,
find the `.wk` target (`e.target.closest(".wk")`), compute its week index from its position among the
grid's children (`Array.prototype.indexOf` on `gridEl.children`, or a `data-w` attribute set during
build — prefer `data-w` to avoid an O(n) index scan), then: if that week has a milestone →
`removeMilestone(week)`; else → `prefillDate(week)`. The listener is attached once at boot, survives
`replaceChildren`, and is the only interactivity mechanism for the grid.

**Direction A (desktop) with C (mobile), chosen at render time by viewport width.** A single
CSS breakpoint (`min-width: 700px` ≈ the point where the 900px-max grid has room for a gutter)
decides which presentation is active. Both presentations read the same `milestones` array; only
the DOM/CSS differs. This avoids a JS layout engine — CSS media queries flip between the gutter
flags (A) and the timeline legend (C). The grid dots themselves (halo + brighter gold) are
identical in both; only the *labels'* placement changes.

**Marking = decoration, never a layout change.** A pinned dot keeps its grid cell; we add a
class (`.wk.milestone`) that brightens the fill, adds a halo (`box-shadow`), and runs a one-shot
pop-in animation. The label is NOT placed on the dot (the dot lives inside `role="img"`; see above);
a11y comes from the sibling `#ms-chips` list, and on desktop the label is mirrored as a visible
(decorative, `aria-hidden`) gutter flag. This keeps the 52-column grid math untouched.

**Visual precedence when a milestone lands on the current week.** The current week already renders
distinctly: `.wk.current` in the DOM and a stroked gold ring in canvas (lines 757-764). A milestone
on that same week must not erase the "you are here" signal. Resolution: **the two compose, they do
not replace each other.** The dot keeps its `current` fill/ring; the `.milestone` class adds only its
outer halo (`box-shadow` beyond the current ring) and the pop-in animation — it does NOT override the
fill to plain gold when `current` is present. Concretely, in the DOM the class list is
`wk current milestone` and `.wk.current.milestone` is specified so the current ring sits inside the
milestone halo (halo radius > ring radius). In canvas, `drawLifeGrid` draws the current-week ring
first (existing code), then the milestone post-pass strokes a second, larger halo ring outside it.
The chip/gutter still labels it normally. This is the single documented answer to edge case 10's
current-vs-milestone conflict.

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
function weekStyle(i): { fill: string, kind: "lived"|"current"|"future", milestone: Milestone | null }
//   fill: unchanged decision. kind: same decision made explicit so the DOM builder can derive its
//     class string ("wk" + " lived"/" current" per kind). milestone: the milestone whose .week === i,
//     else null. Pure; no side effects. Called by BOTH the DOM builder in render() (after refactor)
//     and the canvas drawLifeGrid() loop, so marking logic lives in exactly one place.

function milestoneAt(week: number): Milestone | null   // O(cap) lookup over milestones; used by weekStyle + delegated grid click.
```

**Live-grid class mapping (the refactored `render()` DOM builder):** for each week `i`, read
`{ kind, milestone } = weekStyle(i)`; build class = `"wk"` + (`kind==="lived"` → `" lived"`,
`kind==="current"` → `" current"`, else `""`) + (`milestone` → `" milestone"`, else `""`). Set the
dot's `data-w = i` so the delegated `#grid` listener can recover the week index in O(1). No per-dot
event handlers, no `tabindex`/`role`/`aria-label` on dots (grid is `role="img"`).

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
  This list is the accessible, keyboard-operable representation of milestones in BOTH layouts; only
  its CSS presentation changes across the breakpoint.
- Grid dots gain, when pinned, only `class="wk milestone"` (composed with `lived`/`current` per
  `kind`) and `data-w="{i}"`. **No `tabindex`, `role`, `aria-label`, or `title` on any dot** — dots
  are decorative descendants of `#grid` (`role="img"`), which AT does not traverse. All interaction
  and accessibility for milestones is provided by `#ms-chips` and the single delegated `#grid`
  listener; the grid's own `aria-label` is updated to include the milestone count.
- Grid interactivity: one delegated `click` listener on `#grid` (attached once at boot, survives
  `replaceChildren`). It resolves `e.target.closest(".wk")`, reads `data-w` → week `i`, then removes
  the milestone if `milestoneAt(i)` is truthy, else calls `prefillDate(i)`. No per-dot handlers.
- Grid `aria-label`: `render()` sets it to `"Life in weeks grid"` when there are no milestones, and
  `"Life in weeks grid, N milestone(s) marked"` when `milestones.length > 0`.

**CSS additions (reuse `:root` tokens, no new colors beyond derived brighter gold):**

```css
.wk.milestone { background: var(--accent-gold); box-shadow: 0 0 0 2px rgba(201,168,76,.35), 0 0 8px 2px rgba(201,168,76,.45); animation: ms-pop .32s ease-out; cursor: pointer; }
/* Compose with current: keep the current fill/ring, add the milestone halo OUTSIDE it (no fill
   override). Halo box-shadow radius > current ring so both remain legible ("you are here" + pinned). */
.wk.current.milestone { background: /* current fill from base .wk.current */; box-shadow: 0 0 0 2px var(--accent-gold), 0 0 0 5px rgba(201,168,76,.35), 0 0 10px 3px rgba(201,168,76,.45); }
@keyframes ms-pop { 0%{transform:scale(.4);opacity:.3} 60%{transform:scale(1.25)} 100%{transform:scale(1);opacity:1} }
/* direction A gutter flags: absolute-positioned within .grid-wrap; thin gold connector rule. */
/* direction C timeline: #ms-chips as ordered list, shown < 700px, gutter hidden. */
@media (prefers-reduced-motion: reduce){ .wk.milestone{ animation: none } }
```
(Exact fill/shadow values are illustrative; the invariant is: `.milestone` alone → gold fill + halo;
`.current.milestone` → current fill + current ring + a larger outer halo, never a plain-gold override.)

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
  age tags and prefill), the `age N` label, each dot's class string (from `weekStyle(i)`), and the
  `#grid` `aria-label` milestone-count suffix — all recomputed each `render()`.
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
10. **Clicking a pinned dot** removes it (via the delegated `#grid` listener); **clicking any
    non-pinned dot** (future, lived, or current) prefills the add-row date with that week's date.
    Any on-grid week is a valid pin target. **Milestone on the current week:** current state does not
    block pinning, and the two visual signals *compose* — the dot keeps its `current` fill + gold ring
    and gains an additional outer milestone halo (see Approach "Visual precedence" and the
    `.wk.current.milestone` CSS). The current ring is never replaced by a plain-gold milestone fill;
    in canvas the ring draws first, then a larger halo ring outside it. The chip labels it normally.
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
16. **Keyboard / screen-reader access** — accessibility does NOT rely on the grid dots (they are
    decorative descendants of `#grid`, which is `role="img"`; AT ignores them and they carry no
    `tabindex`/`role`/`aria-label`). Instead, the `#ms-chips` list is the accessible source of truth:
    each chip shows the label + `age N` and holds a real `<button aria-label="Remove {label}">` that
    is natively focusable and Enter/Space-activatable. Keyboard users Tab to a chip's remove button
    and press Enter/Space to remove; screen readers read each chip's label. The grid's own
    `aria-label` announces the milestone count. This is the same accessible path on desktop (chips)
    and mobile (same list styled as a timeline), so the mobile-reveal + keyboard + SR requirements are
    met without any interactive descendant inside `role="img"`.

## Dependencies

- **`src/index.html` post-LLD-3** — this LLD extends/refactors:
  - `weekStyle(i)` (lines ~648-652): extend return to `{ fill, kind, milestone }`.
  - `render()`'s live-grid DOM builder (lines ~1064-1071): **refactor** the inline class ternary to
    call `weekStyle(i)` and derive classes from `kind` + `milestone`, set `data-w`, and update the
    `#grid` `aria-label` with the milestone count. (This is the required live-grid marking path.)
  - `drawLifeGrid` (lines ~724-766): add the milestone halo post-pass in the existing per-dot loop
    (which already calls `weekStyle(i)`), reusing the current-week ring pattern (lines 757-764).
  - `render()` hash write (line ~1083): append `m=` when non-empty.
  - `restoreFromHash()` (lines ~1090-1094): parse `b`/`y` first, then decode `m`.
  - Boot sequence (`restoreFromHash(); render();`) and a one-time delegated `click` listener on `#grid`.
  - `#grid` element (line 524) keeps `role="img"`; milestone dots are decorative descendants and all
    interactive/accessible affordances live in sibling DOM (`#ms-chips`). No new source files.
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
- Mobile / narrow (<700px): gutter hidden; ordered timeline legend (`#ms-chips`) lists milestones
  with labels + `age N`; tapping a pinned dot removes it; the label is read from the legend, not the
  dot.
- Keyboard: Tab reaches each chip's remove `<button>` (dots are NOT in the tab order and carry no
  role/label); Enter/Space on the button removes the milestone.
- Screen reader: with a screen reader active, verify milestone labels are announced from `#ms-chips`
  (not from grid dots — those are inside `role="img"` and must not be individually announced), and
  the `#grid` `aria-label` includes the milestone count.
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

**Visual / a11y composition (must verify):**
- Pin a milestone on the **current** week: the dot still shows the current fill + gold ring AND gains
  the outer milestone halo (neither signal is lost); same in the PNG (ring then outer halo). Removing
  it restores the plain current dot.
- Grid dots expose no per-dot `role`/`tabindex`/`aria-label`; the accessibility tree shows `#grid` as
  a single image whose label includes the milestone count, with milestones reachable only via
  `#ms-chips`.

**Regression:**
- MVP + LLD-3 flows unchanged when no milestones exist: `m` is omitted from the hash; grid, stats,
  and snapshot render exactly as before. `render()`'s `weekStyle`-driven class output must produce
  byte-identical `.wk`/`.wk.lived`/`.wk.current` classes to the pre-refactor inline ternary for every
  week when `milestones` is empty (the DOM-builder refactor must not change existing dot styling).
