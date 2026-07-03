# Belmont Reliability Leaderboard — Project Status & Handoff

_Last updated: June 18, 2026_

This note lets anyone (Joel, or a fresh Claude Code session on another computer) pick up
this project without the original chat history. Read this first, then open `index.html`.

---

## What this is
A shared, web-hosted reliability leaderboard for Belmont Clean + Restore. Two locations
(Carbondale + Gypsum) each get their own team view; the owner sees both via a toggle.
Public links are view-only; three people can edit.

## The working file
- **`index.html`** in this folder = the deployed app. This is the one that gets uploaded to GitHub.
- `Belmont_Reliability_Leaderboard_Cloud.html` = identical working copy / source of truth we edit.
- `Belmont_Reliability_Leaderboard.html` = original OFFLINE fallback (leave as-is).
- `Belmont_Leaderboard_Deployment_Brief.md` + `..._Supabase_Schema.sql` = original build docs.
- **To edit from another computer:** download the latest `index.html` from GitHub (below),
  edit it, then re-upload it to the repo (Add file → Upload files → same filename → Commit).

## Where it's hosted (all cloud — reachable from any browser)
- **Live site (owner/company view, has the toggle):** https://joelbelmont.github.io/belmont-leaderboard-/
- **Carbondale team link:** https://joelbelmont.github.io/belmont-leaderboard-/?loc=carbondale-417
- **Gypsum team link:** https://joelbelmont.github.io/belmont-leaderboard-/?loc=gypsum-315
- **GitHub repo:** joelbelmont/belmont-leaderboard-  (Public; GitHub Pages serves `index.html` at root)
- **Supabase project ref:** vhyfazcwhaiathwpmnez  (dashboard at supabase.com)

## How the data + security work
- The whole board is ONE JSON row in Supabase table `board` (id = 1). `loadState()` reads it,
  `persist()` writes it. Offline fallback to localStorage if Supabase keys are blank.
- Table `editors` (id = auth user id, scope = owner | carbondale | gypsum) decides who can edit.
  RLS: everyone can READ the board (powers the view-only links); only users in `editors` can WRITE.
- **Sign-in = email + password** (we switched OFF the email magic-link because Supabase's built-in
  emailer is rate-limited). Three users exist, created in Supabase → Authentication → Users:
  - joel@belmont.com → owner
  - office@belmontclean.com → carbondale
  - officevv@belmontclean.com → gypsum
  - Reset a password anytime in Authentication → Users → click person → set new password.
- The public **anon/publishable** key is in `index.html` on purpose (safe to be public).
  The Supabase **secret** key must NEVER go in the file or be shared.

## Location behavior (Phase 1 — DONE)
- `?loc=carbondale-417` / `?loc=gypsum-315` = obscured team links; each locks to its team, no toggle.
- Signed-in managers auto-land on their own team's board.
- Owner / plain link = Carbondale|Gypsum toggle; podium, Gold Club, streaks, Most Improved are
  all computed per-team. See `resolveView()`, `buildLocToggle()`, and the filter in `render()`.
- The board is currently seeded EMPTY. Add real people via owner → Edit scores → set each person's Location.

## NEXT UP — Phase 2 (not started)
1. **Editable metrics per month.** Some of the 5 metrics change month to month. Make the metric
   definitions (name/type/band per role group) editable in the Edit view, not hard-coded in `ROLE_SETS`.
2. **Self-updating "Making It Right."** When a metric is swapped, the "Can be made up / Can't be
   made up" section should automatically update to reflect the current metric in that slot.

## Also planned later — Phase 3 (reporting, discussed, not started)
- A "logbook" that saves a dated snapshot of the board on each save (new Supabase table), so Joel
  can pull **weekly/monthly reports for a date range**: per-person breakdown, per-location summary,
  printable/PDF, and spreadsheet (CSV) export. NOTE: reporting can only cover data from the day the
  logbook is switched on — worth turning on early.
