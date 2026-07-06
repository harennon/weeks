# Customer Experience — weeks

The complete user experience for **weeks** (`weeks.danbing.app`): a single-page toy
that renders your whole life as a scrollable grid of weeks — one dot per week — in the
tradition of Tim Urban's "Your Life in Weeks."

This is the spec QA validates against. It documents the one screen, its states, the core
flows, and every user-facing edge case. Sections marked **(deferred)** describe flows for
features that are planned but not yet built (tracked as open issues); QA should treat them
as target behavior, not as something to test until the feature ships.

---

## Interaction Principles

1. **Instant, zero-setup.** A first-time visitor sees their life-in-weeks and a headline
   stat within ~15 seconds — no signup, no onboarding, no modal. One input (birthday)
   is the only required step.
2. **The dots are the interface.** The week-grid is the product. Chrome is minimal: a
   title, two inputs, three stats, the grid. Nothing competes with the visualization.
3. **State is always visible.** Weeks lived, weeks remaining, and percent elapsed are
   shown as soon as a birthday is entered. The current week is visually distinct from
   past and future weeks.
4. **Privacy is absolute and legible.** The birthday never leaves the browser. It lives
   only in the URL hash. The UI says so plainly, because we ask for a personal detail.
5. **Shareable by construction.** The URL *is* the saved state. Copying the address bar
   reproduces the exact view for anyone who opens it — no account, no server round-trip.
6. **No dead ends.** The empty state tells you exactly what to do. Every input change
   re-renders immediately. There is never a spinner, an error page, or a stuck state.

---

## Users

There is **one** user type. No accounts, no guest/registered distinction, no roles.

| Attribute    | Value                                                                       |
| ------------ | --------------------------------------------------------------------------- |
| Identity     | None. No login, no cookie, no fingerprint.                                  |
| Data entered | Birthday (required), lifespan in years (optional, defaults to 90).          |
| Where stored | URL hash only (`#b=YYYY-MM-DD&y=90`). Never transmitted to any server.      |
| Persistence  | Whatever the user bookmarks or shares. Nothing is stored server-side, ever. |

**First-time visitor** and **returning visitor** differ only in whether the URL hash is
already populated:
- **First-time / bare URL** → empty state, waiting for a birthday.
- **Returning via shared/bookmarked link** → inputs pre-filled from the hash, grid
  rendered immediately on load.

---

## The Screen

One screen, top to bottom: **header → controls → stats → grid**. Responsive; the same
layout reflows for mobile and desktop. The grid is 52 dots per row (one row = one year).

```
┌───────────────────────────────────────────────────────────┐
│                    your life in weeks                       │
│         Every dot is one week. The gold ones are behind you.│
│                                                             │
│    Your birthday            Lifespan (years)                │
│    [ 1990-06-15 ]           [ 90        ]                   │
│                                                             │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐                 │
│   │  1,881   │  │  2,799   │  │  40.2%   │                 │
│   │weeks lived│  │ remaining │  │ elapsed  │                 │
│   └──────────┘  └──────────┘  └──────────┘                 │
│                                                             │
│   each row = one year (52 weeks)      90 years · 4,680 wks  │
│   ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● …      │  ← gold = lived
│   ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● …      │
│   ● ● ● ● ◉ · · · · · · · · · · · · · · · · · · · · · …      │  ← ◉ = current week
│   · · · · · · · · · · · · · · · · · · · · · · · · · · · …      │  ← dim = future
│   · · · · · · · · · · · · · · · · · · · · · · · · · · · …      │
│                          … scrolls …                        │
│                                                             │
│   Everything stays in your browser. Your birthday is never  │
│   sent anywhere — it lives only in this page's URL.         │
│   a danbing.app toy · inspired by Tim Urban's Life in Weeks │
└───────────────────────────────────────────────────────────┘
```

### Dot states

| Dot        | Appearance                                     | Meaning                          |
| ---------- | ---------------------------------------------- | -------------------------------- |
| **Lived**  | Solid gold (`#c9a84c`)                          | A week entirely in the past      |
| **Current**| Light fill (`#f5f0e8`) with a gold ring         | The week the user is living now  |
| **Future** | Faint (`rgba(255,255,255,0.12)`)                | A week not yet lived             |

### Stats

| Stat            | Definition                                                  |
| --------------- | ----------------------------------------------------------- |
| Weeks lived     | Whole weeks between birthday and today, floored, min 0.     |
| Weeks remaining | `lifespan × 52 − weeks lived`, min 0.                        |
| % elapsed       | `weeks lived / (lifespan × 52) × 100`, one decimal place.   |

---

## User Flows

### 1. First visit → see your life (happy path)

1. Visitor lands on `weeks.danbing.app` with no hash.
2. Header, tagline, and the two inputs are visible. The grid area shows the empty state:
   *"Enter your birthday above to see your weeks."* Stats and grid-meta are hidden.
3. Visitor picks a birthday in the date input.
4. On that input event, the grid renders, the three stats appear, and the grid-meta row
   ("each row = one year… / N years · N weeks") appears. The empty state disappears.
5. The URL hash updates to `#b=YYYY-MM-DD&y=90` so the view is now shareable/bookmarkable.

**Edge cases:**

- Birthday left empty (or cleared after entry) → return to the empty state; stats and
  grid hidden; hash reflects no birthday. No error.
- Birthday is today or within the current week → weeks lived = 0; the first dot is the
  current week; all others future.
- Birthday in the future (date after today) → weeks lived clamps to 0 (never negative);
  grid is all future dots with the first as current. No crash, no negative stat.
- Very old birthday whose age already exceeds the lifespan → weeks lived clamps to the
  total; weeks remaining = 0; % elapsed = 100.0%. The grid is entirely gold with no
  distinct current-week dot beyond the last.

### 2. Adjust lifespan

1. With a birthday entered, the visitor changes the lifespan number.
2. Grid, stats, and the "N years · N weeks" label re-render immediately.
3. The hash updates its `y` value.

**Edge cases:**

- Lifespan below 1, above 120, blank, or non-numeric → clamped into the range 1–120
  (blank/NaN falls back to 90). The grid never renders zero rows or an absurd count.
- Lifespan reduced below current age → weeks remaining = 0, % elapsed = 100.0%, grid
  all gold (see flow 1 clamping).

### 3. Return via a shared or bookmarked link (happy path)

1. Someone opens a URL containing `#b=1990-06-15&y=90`.
2. On load, the birthday and lifespan inputs are pre-filled from the hash **before**
   first render.
3. The grid and stats render immediately — no empty state flash for a valid hash.
4. The view matches what the original sharer saw, recomputed against *today* (so weeks
   lived reflects the viewer's current date, not the sharer's — this is expected: the
   picture ages with time).

**Edge cases:**

- Hash present but malformed (`b` missing, unparseable date, garbage params) → the app
  falls back to the empty state rather than rendering a broken grid. Any valid params
  that are present are still applied (e.g. `y` alone pre-fills lifespan).
- Hash with only `y` and no `b` → lifespan pre-filled, empty state shown (still needs a
  birthday).
- Extremely long or injected hash content → treated as opaque query params; unknown keys
  are ignored; no script executes from the hash.

### 4. Share the current view

The primary share mechanism is the **URL itself** — the address bar always reflects the
current birthday + lifespan. Copying and sending it reproduces the view. This works today
with zero UI: it is a property of the hash-encoded state.

**(deferred) Explicit "Share…" / copy-link affordance** — a button that copies the URL
(and, on supporting devices, invokes the native share sheet). Tracked in the backlog.
When built, its happy path is:

1. Visitor clicks "Share".
2. On devices supporting the Web Share API, the native share sheet opens with the current
   URL. On others, the URL is copied to the clipboard and brief confirmation is shown.

Edge cases QA must cover when this ships: clipboard permission denied → visible fallback
(e.g. select-the-URL affordance); native share invoked without a valid user-activation
gesture must not silently fail (see backlog issue on `navigator.share()` losing activation
inside async work).

**(deferred) Share as PNG snapshot** — render the grid to an image the visitor can save or
post. Tracked in the backlog. Must remain fully client-side (canvas/`toBlob`), preserving
the privacy invariant — no upload.

### 5. Mark milestones (deferred)

Planned feature: pin labeled events (graduation, a move, a child's birth) to specific
weeks so the grid becomes a personal timeline. Tracked in the backlog.

Target behavior when built:
- Milestones are encoded in the URL hash alongside birthday and lifespan (still no
  server), so a shared link carries them.
- A milestone marks its week dot distinctly and shows its label on hover/tap.
- Adding, editing, and removing milestones re-renders immediately and updates the hash.

Edge cases to cover when it ships: a milestone dated before birth or after the lifespan
end; overlapping milestones in the same week; a milestone label long enough to overflow;
a hash carrying more milestones than the grid can legibly mark.

---

## Edge Case Summary (current MVP)

QA should be able to walk this table against the live site today.

| # | Case                                       | Expected behavior                                                  |
| - | ------------------------------------------ | ------------------------------------------------------------------ |
| 1 | No birthday entered                        | Empty state; stats + grid hidden; no error                         |
| 2 | Birthday today / this week                 | Weeks lived = 0; first dot is current                              |
| 3 | Birthday in the future                     | Weeks lived clamps to 0; no negative stats                         |
| 4 | Age already exceeds lifespan               | Remaining = 0; % = 100.0; grid all gold                            |
| 5 | Lifespan blank / non-numeric               | Falls back to 90                                                   |
| 6 | Lifespan < 1 or > 120                       | Clamped to 1–120                                                   |
| 7 | Valid shared hash on load                  | Inputs pre-filled; grid renders immediately, no empty flash        |
| 8 | Malformed hash on load                     | Falls back to empty state; valid partial params still applied      |
| 9 | Clearing birthday after a render           | Returns to empty state                                             |
| 10| Leap-year / 52-vs-52.18-weeks drift        | Rows are a fixed 52 weeks; minor calendar drift is accepted by design |
| 11| Very long lifespan (e.g. 120)              | ~6,240 dots render without perceptible lag                         |
| 12| Mobile viewport                            | Layout reflows; dots scale down; grid stays 52 columns, tappable   |

**Note on week accounting (edge case 10):** the grid uses a fixed 52 weeks per row and a
7-day week, so it does not track the ~52.18 real weeks per year. This is an intentional
simplification for a "toy," not a bug — QA should not flag the minor drift as a defect,
but *should* flag any off-by-one in the current-week highlight or the lived/remaining
split relative to that model.

---

## What This Product Is NOT

To keep QA and future design honest about scope:

- **No backend, database, or account.** Any flow that implies server persistence, login,
  or transmitting the birthday is out of scope and a privacy violation.
- **No analytics that transmit the birthday.** The personal detail stays local.
- **No build step.** Static HTML/CSS/JS served from `src/`.
- **Not a calendar or planner.** It's a single reflective visualization, not a tool for
  scheduling or tracking ongoing data.
