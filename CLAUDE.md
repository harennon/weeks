# CLAUDE.md

**weeks** — a single-page toy that renders your whole life as a scrollable grid of
weeks (one dot per week), in the tradition of Tim Urban's "Your Life in Weeks." Lives
at `weeks.danbing.app`, one of the danbing.app portfolio projects.

## Project Structure

- `src/` — the deployed site (HTML, CSS, vanilla JS). Cloudflare Pages serves from here.
- `design-mockups/` — throwaway HTML mockups for visual design exploration.
- `docs/` — design docs; `docs/lld/` holds low-level designs written by the pipeline.

## Tech Stack

- **Pure HTML + CSS + vanilla JS** — no framework, no build step.
- **Hosting:** Cloudflare Pages (deploys from `src/`), subdomain `weeks.danbing.app`.
- **State:** entirely client-side. The birthday + lifespan (and later, milestones) are
  encoded in the URL hash so a link reconstructs the view. **Nothing is ever sent to a
  server** — this is a hard privacy invariant, since users enter a personal detail.

## Design Philosophy

Small, personal, emotionally sticky — the kind of thing you look at for 30 seconds and
send to a friend. Visual language echoes the danbing.app portfolio: dark background
with a subtle dot-grid, gold (`#c9a84c`) accent, DM Sans for UI and Libre Baskerville
for display text. The week-dots ARE the interface; minimal chrome.

## Principles

- **Client-side only.** No backend, no accounts, no persistence beyond the URL. Do not
  introduce a server, database, or analytics that transmits the birthday.
- **Instant + zero-setup.** A first-time visitor sees their life-in-weeks and a headline
  stat within ~15 seconds, on mobile or desktop, no signup.
- **Deploy cheap.** Static files only; no build step. Anything that would require a
  build pipeline needs explicit justification.
- **Shareable artifacts.** Favor features that produce something a person wants to post
  (the URL-hash share link; later, a snapshot PNG).

## Agent Routing

| Trigger | Agent |
| --- | --- |
| Visual design, mockups, layout | `frontend-architect` |
| Design / write an LLD | `architect` |
| Review an LLD before implementation | `design-reviewer` |
| Build / implement from an approved LLD | `implementer` |
| Review code after implementation | `code-reviewer` |
| Verify UX | `qa` |
| What to build next / prioritize | `ceo` |

## Deployment

Push to `main` → Cloudflare Pages auto-deploys from `src/`.
