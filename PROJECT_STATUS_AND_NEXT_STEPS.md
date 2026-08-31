# Belmont Reliability Premium Leaderboard — Project Status & Handoff

_Last updated: August 31, 2026 — month history added (schema 2). Rewritten from the code._

This note lets anyone — Joel, or a fresh Claude Code session on another computer — pick up this
project without the original chat history. Read this first, then open `index.html`.

**Verify before you trust this file.** It has drifted from reality once already. If the code and
this doc disagree, the code wins.

---

## What this is
A shared, web-hosted reliability leaderboard for Belmont Clean + Restore. Two locations
(Carbondale + Gypsum) each get their own team view; the owner sees both via a toggle.
Public links are view-only; a small number of managers can edit.

Five metrics per role, up to $500/month + a $300 quarterly streak bonus.

## The working file
- **`index.html`** — the entire app. One self-contained file: HTML, CSS, and JS, ~890 lines.
  There is no build step and no dependencies beyond the Supabase JS client loaded from a CDN.
- `PROJECT_STATUS_AND_NEXT_STEPS.md` — this file.

That's it. Older side files referenced by the previous version of this doc
(`Belmont_Reliability_Leaderboard_Cloud.html`, the offline fallback, the deployment brief,
the schema SQL) are **not in the repo** — if they still exist, they're on Joel's desktop only.

## How editing actually works (read this before you touch anything)
Joel works from **two Macs** — a desktop at home and a travel MacBook Pro — and syncs
by uploading `index.html` through the GitHub **web UI** (Add file → Upload files). That's why
every commit message reads "Add files via upload."

**The hazard:** a checkout on either machine can be many commits behind with no visible sign,
and uploading a stale file silently reverts the other machine's work. On 2026-08-31 the laptop
was 11 commits behind and still held superseded July edits.

**Always do this first:** `git fetch origin && git status` — confirm you are up to date before
reading or editing. Never upload a file you didn't just pull.

### If you are the other machine, reading this after a gap
The desktop and the laptop each have their own checkout. Whichever one you are on may be behind.
Before touching `index.html`:

```bash
git fetch origin
git status                 # "behind by N" means your copy is stale
git stash                  # only if you have local edits worth keeping
git merge --ff-only origin/main
```

**Uploading a pre-schema-2 `index.html` over the current board breaks the live site.** The old
code expects `people[].scores`, which no longer exists; it throws during render and the board
comes up empty below the header. Tested: the stored data is *not* corrupted by this — the old
code dies before it writes anything, and re-uploading the correct file restores the board. So it
is recoverable, but the team sees a broken board until it is fixed. Pull before you edit.

A file dated before **August 31, 2026** is pre-schema-2.

## Where it's hosted
- **Live site (owner view, has the location toggle):** https://joelbelmont.github.io/belmont-leaderboard-/
- **Carbondale team link:** https://joelbelmont.github.io/belmont-leaderboard-/?loc=carbondale-417
- **Gypsum team link:** https://joelbelmont.github.io/belmont-leaderboard-/?loc=gypsum-315
- **GitHub repo:** `joelbelmont/belmont-leaderboard-` — **public**; GitHub Pages serves `index.html` at root.
- **Supabase project ref:** `vhyfazcwhaiathwpmnez` (dashboard at supabase.com)

> **The repo is public.** Never commit real employee names, scores, or exported board data to it.
> Live data belongs in Supabase only. Keep backups outside the repo folder.

## Data model
Three Supabase tables:

| Table | Shape | Who can read | Who can write |
|---|---|---|---|
| `board` | ONE row, `id = 1`, whole board as a single JSON blob in `state` | everyone (powers the view-only links) | users listed in `editors` |
| `editors` | `id` = auth user id, `scope` = `owner` \| `carbondale` \| `gypsum` | — | — |
| `incident_log` | dated per-person notes, keyed by the person's `id` | owner, or a manager whose location matches | same |

`loadState()` reads the board row; `persist()` writes it back wholesale. If the Supabase keys are
blank the app falls back to `localStorage` (key `belmont_reliability_v2`).

The `state` blob is **schema 2** — a roster plus a map of months:

```
state = {
  schema: 2,
  people: [ {id, name, role, group, location, seedStreak} ],       // roster only, no scores
  months: {
    "2026-08": { label, week, labels, makeup, scores:{ personId:[5] } },
    "2026-09": { ... }
  },
  current: "2026-09",        // the LIVE month the team's links land on
  colWidths: [7]
}
```

Scores, metric names and makeup rules live **inside a month**, so changing September never
rewrites August. `week` is 1–4, or 5 = "Month complete". `seedStreak` is a per-person carry-in
for perfect months earned before this board existed — it is editable only on the earliest month.

`migrate()` runs on every load. It is **additive only** — it seeds missing fields and never
touches existing people or scores. It also upgrades a schema-1 board in place: the old flat
board is wrapped as its first month, with the month key derived from the old free-text `period`.
Keep it additive.

## Sign-in
Email + password (the magic-link flow was turned off — Supabase's built-in emailer is
rate-limited). Users are created by hand in Supabase → Authentication → Users, then given a row
in `editors` with the right scope. Reset a password in the same place.

The **anon/publishable** key is in `index.html` on purpose and is safe to be public.
The Supabase **secret** key must never go in the file or be shared.

## Role groups
Seven keys in `ROLE_SETS`, six of them displayed (`GROUPS` controls display order):
`tech`, `senior_tech`, `pm_senior`, `bdr_sales` (shown as "BDR/PM"), `bdr`, `office`.

`lead` ("Lead Tech / PM") is the **legacy** group, kept only so old saved data still renders.
It is not in `GROUPS` and should not be assigned to new people.

## What has shipped
- **Locations (Phase 1).** Obscured per-team links, managers auto-land on their own board,
  owner gets the toggle. Podium, Gold Club, streaks and Most Improved all compute per-team.
  See `resolveView()`, `buildLocToggle()`, and the filter in `render()`.
- **Editable metric names.** The five column labels per role group are editable in the Edit view
  and stored in `state.labels`. (The metric *definitions* in `ROLE_SETS` — type and band — are
  still hard-coded. Only the display name is editable.)
- **Self-updating "Making It Right" (was Phase 2 item 2).** `buildMakeupEditor()` gives one row
  per distinct metric: can't be made up / back to $50 / back to $100, plus the note shown on the
  board. `renderMakeup()` rebuilds the two lists from `state.makeup`, and `distinctLabels()`
  derives the metric list from the *current* labels — so renaming a metric re-keys its rule
  automatically.
- **Private incident log (Phase 3a).** Dated per-person notes in `incident_log`, invisible to the
  public view-only links.
- **Adjustable column widths**, drag-to-resize, persisted in `state.colWidths`.
- **Month history with ‹ › navigation** — see below.

## How months work
Each month is its own record. The arrows in the header move between them.

- **Paging back.** Everyone — including the team's view-only links — can page back to any past
  month and see exactly what it looked like: its scores, its metric names, and its own
  "Making It Right" rules.
- **Building ahead.** At the newest month the owner's right arrow becomes **+**. Pressing it
  creates the next month, copying the current month's metric names and makeup rules forward as a
  starting point. Nothing about the earlier month is touched.
- **Drafts are private.** A month later than `state.current` is labelled *Draft — not visible to
  the team* and only the owner can see or reach it. October can be built out through September
  without anyone seeing it.
- **Going live.** "Make this the live board" sets `state.current`. From the 1st of a month whose
  board isn't live yet, the owner also gets a banner offering to start or promote it. Nothing
  switches over automatically — promoting is always a deliberate click.
- **Rollover is calculated, not typed.** "Last mo. $" and the Gold Club streak are derived from
  the stored months (`lastTotalBefore()`, `streakAt()`) and shown read-only in the editor. A month
  still in progress neither counts toward a streak nor breaks it — nothing is earned until the
  period closes.

### Who can edit which month
| | Past / closed months | The live month | Future drafts |
|---|---|---|---|
| **Owner** | edit | edit | edit + create |
| **Location manager** | read-only | edit while open; locked at "Month complete" | cannot see |
| **View-only link** | read-only | read-only | cannot see |

Enforced in `canEditMonth()` and `visibleKeys()`. Note this is **UI-level** — Supabase RLS still
guards the row as a whole, so a manager technically retains write access to the board row. The
month lock is there to prevent accidents, not to defend against a determined editor.

## Backing up the board
The board is public-readable, so a backup needs no credentials:

```bash
KEY="sb_publishable_fVDVzLeVUgac8jEp2m3qIw_l0rM11J3"
curl -s -H "apikey: $KEY" -H "Authorization: Bearer $KEY" \
  "https://vhyfazcwhaiathwpmnez.supabase.co/rest/v1/board?id=eq.1&select=state,updated_at"
```

Save the result **outside this repo** (the repo is public). Existing backups live in
`../belmont-backups/`.

## NEXT UP
1. **Fully editable metric definitions** — a metric can be renamed per month, but the count (5),
   the type (`build` / `clean`) and the band still come from the hard-coded `ROLE_SETS`. Letting a
   metric be added, removed or re-banded per month is the remaining half of the old Phase 2.
2. **Reporting (Phase 3b)** — weekly/monthly reports over a date range: per-person breakdown,
   per-location summary, printable/PDF, CSV export. Month history now exists, so this is mostly a
   read over `state.months` rather than new plumbing.
3. **Trimming old months** — nothing prunes history. The whole board is one JSON row, so after a
   couple of years it is worth checking the row size and archiving old months out.

## Small known cruft
- `ensureMakeup()` now folds away makeup keys with stray whitespace (live data had an orphaned
  `" EOD by 6pm"` with a leading space). Cleaned on the first save.
- Gypsum currently has no people; all staff are on Carbondale.
- `shortName()` is dead code — nothing calls it.
- The previous version of this doc listed the three sign-in email addresses in this public repo.
  They've been removed here — look them up in Supabase → Authentication → Users instead.
