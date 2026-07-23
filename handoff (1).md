# Game Jam Planner — Handoff

## Status: Board + Dashboard + Team tabs complete ✅

Full loop works end to end: GitHub login → repo picker → project
init/open → tabbed shell (Dashboard / Board / Team) with a persistent
header (project info, Save button, conflict banner) shared across all
three views. Confirmed working this session. Next up is a real feature
request, not a bug fix — see "What's next" at the bottom.

## Where this came from

Started from `planning.md` — a GitHub-backed, local-first React SPA
vision (no custom backend, GitHub as auth/persistence/collaboration
layer, `.gamejam/` folder of JSON files as the project format).
Standalone project. Single self-contained `.html` file — no build
tools, no npm.

## Current deliverables

- **`gamejam-planner.html`** — the app. React 18 (UMD, CDN) + Babel
  Standalone (CDN) for JSX, transformed manually at load time (see
  "Don't re-break these").
- **`gamejam-planner-auth-relay.js`** — Cloudflare Worker, deployed and
  live. Stateless CORS relay for GitHub's device-flow endpoints.

Hosted at `https://jyoung4242.github.io/GameJam-Planner/` (must stay
named `index.html` in that repo's Pages path).

## CONFIG — fully set, confirmed working

```js
const CONFIG = {
  clientId: "Ov23liy7kjuppgIY5yU1",              // classic OAuth App
  relayBaseUrl: "https://little-grass-e4c2gamejam-planner.justin-dean-young.workers.dev",
  scopes: ["repo"],
};
```

Classic OAuth App (not the earlier abandoned `gamejamplanner` GitHub
App — that hit an unresolved install-flow 404, unused now, safe to
delete from GitHub settings if you want to tidy up).

## Architecture — component tree

```
App
└─ ProjectGate            (detects .gamejam/project.json; init form or →)
   └─ ProjectShell         (owns useWorkspace + collaborator logins,
                             renders persistent header + tab bar)
      ├─ Dashboard          (stats, team panel, activity feed)
      ├─ BoardView          (Kanban columns, drag-and-drop, card modal)
      └─ TeamPage           (live collaborators + Sync to team.json)
```

**Key point from this session's refactor**: `useWorkspace` used to live
inside the board component alone. It's now owned by `ProjectShell`,
one level up, so switching tabs doesn't lose unsaved changes or
re-fetch from GitHub — `Dashboard`, `BoardView`, and `TeamPage` all
read from the same `Workspace` instance via props. Any new tab/view
should follow this pattern: read from props passed down by
`ProjectShell`, don't call `useWorkspace` again.

## What's built (cumulative)

### Auth & repo access
GitHub Device Flow login via the CORS relay. `GET /user/repos` for the
picker. `GET /repos/{owner}/{repo}/collaborators` (two flavors: plain
logins via `listCollaborators`, full objects with permission + avatar
via `listCollaboratorsWithPermissions`).

### Storage layer (`GitHubStorage`) — all methods, cross-checked against every caller
`listUserRepos`, `listCollaborators`, `listCollaboratorsWithPermissions`,
`listRecentActivity`, `detectProject`, `initProject`, `readManifest`,
`readBoard`, `readTeam`, `writeTeam`, `listCardIds`, `readCard`,
`writeCard`, `deleteCard`, `commitBatch`, plus low-level `getFile`/
`putFile`. Conflict detection via git blob sha (409/422 → `StorageConflictError`).

`listRecentActivity(repo, limit)` — `GET .../commits?path=.gamejam` so
the Dashboard's feed only shows planner-related commits, not unrelated
game-source commits sitting in the same repo.

### Workspace layer (`Workspace` class + `useWorkspace` hook)
Dirty tracking per file, manual Save (disabled when clean), autosave
every 3 minutes if dirty, deletions handled separately from the batch
write. Auto-seeds 4 default columns on first load of an empty board.

**Conflict handling is still minimal**: a 409/422 on save shows a
banner with a "Reload" button that discards local changes to the
conflicting files and refetches everything (`reloadDiscardingLocalChanges`).
Not a field-level merge. This matters more now — see "What's next."

### Kanban board (`BoardView`)
Drag-and-drop (native HTML5, no library) + ◀▶ fallback buttons. Column
height capped with independent scroll + themed scrollbar. Card face:
status-accent left border, tag pills (ellipsis-truncated at 120px),
assignee initials badge. Title text uses `overflow-wrap: anywhere` so
long unbroken strings break instead of overflowing the fixed-width
column.

### Card detail modal (`CardDetailModal`)
Title/description/status/tags/assignee (dropdown from live
collaborators, with a fallback so a since-removed collaborator's
existing assignment doesn't vanish). **Speech-to-text dictation** on
description via native `SpeechRecognition` (Chrome/Edge only,
feature-detected). If dictation shows "Listening…" but nothing
transcribes, that's a browser/OS mic-routing issue, not an app bug —
confirmed this session by reproducing identical silence on Google's own
speech demo.

### Dashboard
Card stats (total, % done, per-status breakdown), live team panel,
recent activity feed. All fetched fresh on mount — no caching yet
(relevant to "What's next").

### Team page
Live collaborators with GitHub permission level + a "Sync to team.json"
button that snapshots the current list into the repo. `team.json`
deliberately mirrors GitHub's own permission levels rather than custom
jam-specific roles — GitHub stays the source of truth, this is just a
persisted cache of it (explicit decision this session, keeping scope
simple).

## Don't re-break these (technical gotchas from earlier sessions)

1. **No TypeScript-in-Babel** — plain JS only in `#app-source`.
2. **App code must live in `<script type="text/plain" id="app-source">`**,
   transformed manually via `Babel.transform(source, { presets:
   [["react", { runtime: "classic" }]] })` — not a live
   `text/babel` tag (Babel's default "automatic" JSX runtime injects an
   `import` that breaks on `file://`).
3. **GitHub's device-flow endpoints don't support CORS** — that's what
   the relay Worker is for. Stateless, no secret needed.
4. **Verification workflow** for every edit to `index.html`: extract
   the `#app-source` block, run it through the same `Babel.transform`
   call the browser uses, `node --check` the output. Catches syntax
   errors before they reach the browser. Needs `@babel/standalone`
   installed locally (`npm install --no-save @babel/standalone` in
   `/home/claude/gamejam-planner` if starting a fresh session/container).

## What's next: periodic resync with the repo

**The ask**: pull down updated project data from GitHub periodically
while the app is open, so changes made by a collaborator (or from
another browser tab/device) show up without a manual reload.

This is the *opposite direction* from what exists today — autosave
**pushes** local dirty changes out on a timer; this is about **pulling**
remote changes in on a timer. Not yet designed or built. Things worth
deciding before writing code:

- **Polling vs. something smarter**: GitHub's REST API doesn't offer
  push notifications for file changes without a webhook (which needs a
  backend — against this project's "no custom backend" principle,
  same class of problem the CORS relay already sidesteps for a
  different reason). Polling on a timer is the realistic option here,
  similar cadence to autosave (a few minutes?) or faster if it should
  feel closer to real-time.
- **Cheap staleness check vs. full refetch**: don't need to re-download
  every file every poll — comparing each tracked file's cached git
  blob sha (`FileHandle.revision`, already stored per file in
  `Workspace`) against GitHub's current sha is enough to detect "did
  this change," and only refetch the files that actually did.
  `listCardIds` also needs polling to catch newly-added/removed card
  files, not just changed ones.
- **Interaction with dirty local state — this is the crux of it**: if a
  file is dirty locally (unsaved edit) AND changed remotely, that's
  the same conflict `commitBatch` already detects on save — but now it
  needs to surface *before* the user tries to save, from a background
  poll, not just on write. If a file is clean locally and changed
  remotely, that's a safe silent refresh — no conflict, no user
  decision needed, just update `Workspace`'s in-memory copy and emit.
  Existing `reloadDiscardingLocalChanges()` is too heavy-handed for
  this (nukes all local changes, not just the ones that actually
  conflict) — likely needs a new, more surgical method: refresh clean
  files silently, flag dirty-and-changed files as conflicts (reusing
  the existing conflict banner UI) without touching other in-flight edits.
- **Where to hook it in**: `Workspace` already has `startAutosave`/
  `stopAutosave` and an `onChange` pub/sub — a `startPolling`/
  `stopPolling` pair alongside those, started/stopped in the same
  `useWorkspace` effect, is probably the natural home for this rather
  than a separate mechanism.
- **Dashboard's activity feed and Team page** also have no live-refresh
  today (fetched once on mount) — worth deciding whether "periodic
  resync" is scoped to just the board/cards, or should refresh those
  too.

Suggested first step next session: design the surgical
refresh-clean-flag-conflicts method on `Workspace` before touching any
UI, since that's the part with real design risk (getting the
dirty/remote-changed interaction wrong could silently lose someone's
edits, which is worse than the current fully-manual reload).

## Still open from earlier handoffs, not yet revisited
- Countdown to jam deadline
- Milestones (`milestones.json` scaffolded, unused)
- Assets tracking (`assets.json` scaffolded, unused)
- Card reordering within a column (drag-and-drop only moves between
  columns right now, not within one)
- Card descriptions aren't previewed on the card face, modal-only
