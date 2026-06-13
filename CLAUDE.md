# Project Tracker — project guide for Claude Code

This repo **is** the live app. It's a single self-contained file, `index.html`, served as a website by
GitHub Pages at **https://mattyfloyd.github.io/project-tracker/**. There is **no build step** — editing
`index.html` and pushing to `main` redeploys the site automatically (~1 minute).

It's a personal **Milanote / Trello-style Kanban project tracker** for a 3D-visualisation / creative studio
(site-safety inductions, visual standards, VR/interactive training for construction & utility clients). It is
single-user, daily-use, not client-facing. The guiding principles throughout: **keep it one self-contained
file, keep it easy to use and low-effort, keep it polished, and make it work well on iPhone as a home-screen
PWA.**

---

## Working on this repo (incl. from a second machine, e.g. a laptop)

- **`index.html` is the source of truth.** Edit it directly, then `git add -A && git commit && git push`.
  GitHub Pages serves `index.html` at the URL above, so a push = a deploy.
- `icon.png` (512×512) is the app icon, served alongside `index.html`.
- **Always tell the user and get an OK before deploying**, and **batch changes** rather than pushing after
  every tiny tweak. (The user explicitly asked for this.)
- After a push, the live site can take ~1–3 min to rebuild and the CDN/browser cache the old copy — verify
  with a cache-busted fetch and tell the user to **hard-refresh (Ctrl+Shift+R)** / reopen the phone PWA.
- **Verify changes before deploying.** A local static server works for previewing
  (`python -m http.server`), but the app shows a **login screen** (cloud is enabled), so to test board/card
  behaviour without logging in, seed a state in the page console: `state = seed(); render();` then drive the
  functions / inspect computed styles. (On the original Desktop PC there's a `.claude/launch.json` + preview
  tooling; a laptop can just use `python -m http.server`.)
- **Never change the internal storage keys** `ptracker.v1` (localStorage) or `ptracker-fs` (IndexedDB) — doing
  so loses the user's local data. The on-screen name has changed over time; these keys must not.

> Historical note: the user's Desktop PC keeps a canonical `project-tracker.html` plus a `deploy.ps1` that
> copies it to `deploy/index.html` and pushes. That's a Desktop-only convenience. On any other machine, just
> treat `index.html` as canonical and edit/commit/push it directly. If you edit `index.html` here, the
> Desktop copy can drift — easiest is to standardise on editing `index.html`.

---

## Architecture (how the code is shaped)

- **Vanilla JS, no framework, no dependencies** except the Supabase JS client loaded from a CDN.
- One global **`state`** object (columns, cards, sideLinks, linkGroups, statuses, clients, quickNotes, dark,
  etc.). `render()` rebuilds the board from `state`; `save()` persists it (localStorage + debounced cloud
  upsert + optional local file). Most mutations call `save()` then `render()`.
- **Cards** have a `type`: `project` (full card), `note` (sticky), `checklist` (tickable items). Project cards
  carry: `title`, `projectNumber`, `notes`, `color`, `links`, `image`, `status`, `client`, `renderSpec`
  (render/output spec), `filePath`, `shootDate`, `priority`, plus `timer`/`logs` (see "removed" below).
- **Persistence / sync**: Supabase (Auth email+password + Postgres `boards` table, one `jsonb` row per
  `user_id`, protected by RLS). The Supabase **anon/public** key + project URL are embedded in `index.html`
  — this is safe to be public (it's a client key, guarded by RLS). The **service_role** key is NOT in the
  repo and must never be. Data lives in Supabase, so the same board appears on any device once logged in.
  There's also a localStorage cache and an optional File System Access auto-save-to-file.
- **Login** is sign-in only (the sign-up toggle was removed — there's a single account). Boot: `init()` →
  if a session exists, load the board from `boards`; else show the login screen.
- **Icons**: the browser-tab favicon is generated in-page (`drawAppIcon` → an "MF" monogram over a small
  kanban board); the iOS home-screen icon + PWA manifest icon use the custom **`icon.png`** (a planet with a
  kanban board, orbit ring and rocket). `installIcons()` wires these up + builds the web-app manifest.
- **Mobile/PWA**: a `@media (max-width: 640px)` block at the END of the `<style>` (placed last so it
  overrides desktop). The sidebar becomes a slide-in drawer; columns are full-height with "+ Add card"
  pinned to the bottom; `env(safe-area-inset-*)` padding keeps the toolbar clear of the iPhone status
  bar / Dynamic Island / home indicator. **CSS gotcha:** the base toolbar rule is `header.topbar`, so a
  mobile `.topbar` override loses on specificity — match the base selector. (Same class of bug bit the
  sidebar width earlier.)

---

## Current features

- **Board**: columns (add/rename/recolour/collapse/compact/reorder via the `⠿` grip), drag-drop cards,
  search, dark mode, undo (Ctrl+Z + toast), styled prompt/confirm/alert dialogs, a global right-click
  context menu, auto-backups (rolling localStorage snapshots) + Backups/Export/Import in the `🗂`/`•••` menu.
- **Top toolbar**: `☰` · brand · a **Board · Focus** view-switcher · search · **client filter** dropdown ·
  `•••` overflow menu (Archive, file-save, Backups, Export, Import, Shortcuts, theme, Sign out).
- **Cards**: project / note / checklist. **Card titles**: a single click/drag moves the card;
  **double-click the title to rename** (project & checklist). New cards open straight into title-editing.
  Whole-card colour tint; priority levels (Low/Medium/Urgent/Very-urgent) with auto-sort-to-top.
- **Checklist items**: tick the custom checkbox to mark done; **double-click the text to edit** (supports
  **bold/italic** via Ctrl+B/I, stored as sanitised HTML); **drag by the row** to reorder or drop onto
  another checklist to move it (live preview like the links, commit on dragend — no flicker).
- **Client tagging + filter**: tag a project with a client (editor dropdown, "＋ Add client…"); a coloured
  client pill shows on the card; a fixed-width **client filter** appears in the toolbar once clients exist.
- **Render/output field** on project cards (e.g. "3840×2160 · MP4 · 30s"), shown as a `🖥` row.
- **Focus page**: Very urgent / Urgent / Shoot-soon cards (a full page, switched via the toolbar tab).
- **Quick Notes**: a scratchpad (⚡ in the sidebar) — full-screen on phone, popup on desktop, synced.
- **Links sidebar**: client folders, drag-reorder/move links, smart labels for Google Drive links.
- **Drop / paste an image onto the board** → it becomes a card with that image as its cover; the image is
  auto-downscaled (~1100px JPEG) so it doesn't bloat saved data.
- **Inductions**: some cards have a live "shoot in N days" countdown (`shootDate`).

---

## REMOVED — time tracking & timesheet (taken out "for now", may be re-added)

The user removed all time-tracking UI but **liked it** and may want it back. The data model
(`card.timer`, `card.logs`) and most functions are still present (just unused/unwired). It's all recoverable
from git history — look at the commit **before** "Remove timesheet + time-tracking UI" and revert that
commit, or re-wire the UI.

What it did, for reference when re-adding:
- **Per-card timers**: a ▶/⏸ play button + `⏱` elapsed badge on each project card; **exclusive** (starting
  one paused the others); a floating **timer tray** listing active timers; click a time to edit it; "Confirm"
  banked the session into `card.logs`.
- **Today pill**: a topbar "📅 Today Xh / 8h" progress chip against an 8-hour daily target (`DAY_TARGET`).
- **Timesheet page** (Scoro-style, the user especially liked this): a full page with a **weekly grid** —
  a person summary card (initials, week %, total) + a column per weekday (Mon–Fri, weekend only if logged),
  each showing its time entries (project no., title, client, note, duration) with a coloured left-stripe, and
  a per-day total with an under/over pill vs 8h (red under, green over). Plus a Day/Week toggle and
  Prev/Today/Next navigation. Built from `daySummary(ts)` + `startOfWeek/addDays/sameDay/fmtHuman`.
- **Focus "⏱ Running now"** section, **manual time entry** (minutes or H:MM to a chosen date), and **CSV
  export** of all logs.
- It lived as switchable **pages** (`#focusPage`, `#timesheetPage` beside `#board`, toggled by `setView`),
  with a 1-second tick keeping the active page live while a timer ran.

---

## Conventions & preferences (the user has stated these)

- Keep it **one self-contained `index.html`**. No build tooling, no frameworks.
- **Free only** — no paid hosting, no card on file (GitHub Pages + Supabase free tiers).
- **Tell the user before each deploy**, batch changes, and verify before pushing.
- Keep it **simple, low-effort, polished, and mobile-friendly**. The user dislikes clutter and overcomplication
  (deliberately skipped: assignees, automations, Gantt, connector arrows, nested boards, collaboration).
- Don't reintroduce native `prompt/confirm/alert` — use the in-app `uiPrompt/uiConfirm/uiAlert`.
- Format file/PR references as links; reference code as `index.html:line`.
