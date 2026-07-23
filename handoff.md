# Game Jam Planner — Handoff

## Status: Kanban board MVP complete ✅

Full loop works end to end: GitHub login (Device Flow) → repo picker → project init/open → Kanban board with drag-and-drop, rich cards,
speech dictation, collaborator assignment. This is a good checkpoint — next session should pick a direction from "What's next" below
rather than debug anything known-broken.

## Where this came from

Started from `planning.md` — a GitHub-backed, local-first React SPA vision (no custom backend, GitHub as auth/persistence/collaboration
layer, `.gamejam/` folder of JSON files as the project format). Standalone project (not tied to ExcaliburIDE). Single self-contained
`.html` file — no build tools, no npm.

## Current deliverables

- **`gamejam-planner.html`** — the app. React 18 (UMD, CDN) + Babel Standalone (CDN) for JSX, transformed manually at load time (see
  "Don't re-break these" below).
- **`gamejam-planner-auth-relay.js`** — Cloudflare Worker, deployed and live. Stateless CORS relay for GitHub's device-flow endpoints.

Hosted at `https://jyoung4242.github.io/GameJam-Planner/` (must stay named `index.html` in that repo for GitHub Pages to serve it).

## CONFIG — fully set, confirmed working

```js
const CONFIG = {
  clientId: "Ov23liy7kjuppgIY5yU1", // classic OAuth App, not the GitHub App from earlier
  relayBaseUrl: "https://little-grass-e4c2gamejam-planner.justin-dean-young.workers.dev",
  scopes: ["repo"],
};
```

**This is a classic OAuth App**, not the GitHub App (`gamejamplanner`) registered earlier in the project — that one hit an unresolved
404 on its install-app flow (possibly a known GitHub-side bug) and was abandoned in favor of OAuth App, which needs no install step at
all. The old GitHub App can be deleted from Settings → Developer settings → GitHub Apps if you want to tidy up, or left alone — it's
unused either way.

## What's built

### Auth & repo access

- GitHub Device Flow login via the CORS relay
- `GET /user/repos` for the repo picker (classic OAuth semantics)
- `GET /repos/{owner}/{repo}/collaborators` for the assignee dropdown

### Project init/open (`ProjectGate`)

- Detects `.gamejam/project.json`; if missing, shows `InitProjectForm` (name/description/jam/theme) which calls `initProject()` to
  scaffold `project.json`, `board.json`, `milestones.json`, `team.json`, `assets.json` with an initial commit
- If found, loads straight into the board

### Storage layer (`GitHubStorage`)

All methods `Workspace` needs are implemented and cross-checked against each other: `listUserRepos`, `listCollaborators`,
`detectProject`, `initProject`, `readManifest`, `readBoard`, `listCardIds`, `readCard`, `writeCard`, `deleteCard`, `commitBatch`, plus
low-level `getFile`/ `putFile`. Conflict detection via git blob sha — a stale sha on write comes back as 409/422, caught and converted
to `StorageConflictError`.

### Workspace layer (`Workspace` class + `useWorkspace` hook)

- Dirty-state tracking per file (board.json, each card)
- Manual **Save** button (disabled when nothing's dirty) + autosave every 3 minutes if anything's dirty
- Deletions handled separately from the batch write (Contents API delete isn't part of the batch shape)
- **Conflict handling is minimal by design**: on a 409/422, shows a banner with a count and a "Reload" button that refetches everything
  and discards local changes to the conflicting files. This is NOT a field-level merge — planning.md's "prompt only on genuine field
  conflicts, auto-merge the rest" is still unbuilt. Fine for solo/small team use so far, but worth flagging if multiple people start
  editing concurrently.
- Auto-seeds four default columns (Backlog/In Progress/Blocked/Done) on first load of an empty board

### Kanban board (`BoardWorkspace`)

- Four columns, add/move/delete cards
- **Drag-and-drop** between columns (native HTML5 DnD, no library) — the ◀▶ buttons still work too, kept both intentionally
- Column height is capped (`max(320px, 100vh - 220px)`) with its own independent scroll + themed thin scrollbar, so a column with many
  cards doesn't blow out the page or hide the add-card row
- Card face shows tag pills + assignee initials badge when present; status-colored left border accent per column
- Text overflow: card titles use `overflow-wrap: anywhere` (break long unbroken strings rather than overflow); tag pills truncate with
  ellipsis at 120px (pills are meant to stay short, full text is still editable in the modal)

### Card detail modal (`CardDetailModal`)

- Edit title, description, status (dropdown), tags (comma-separated), assignee (dropdown from `listCollaborators`, with a fallback so a
  removed collaborator's existing assignment doesn't silently vanish)
- **Speech-to-text dictation** for the description field, via the browser's native `SpeechRecognition` API (Chrome/Edge only — Firefox/
  Safari support is patchy/absent, gated with a disabled button + tooltip). Appends finalized speech chunks incrementally; shows a live
  interim preview while listening; surfaces mic/permission errors inline. **Note**: if dictation shows "Listening…" but nothing is ever
  transcribed, that's very likely a browser/OS microphone routing issue (wrong input device selected), not a bug in the app — confirmed
  this session by reproducing the same silence on Google's own speech demo. Nothing to fix in code for that; it's an environment thing.

## Don't re-break these (technical gotchas from earlier sessions)

1. **No TypeScript-in-Babel.** Plain JS only. Don't reintroduce `data-presets="react,typescript"`.
2. **App code must live in `<script type="text/plain" id="app-source">`**, transformed manually via
   `Babel.transform(source, { presets: [["react", { runtime: "classic" }]] })` at the bottom of the file — not a live
   `<script type="text/babel">` tag. Babel's default "automatic" JSX runtime injects an `import` statement that breaks on `file://`;
   the manual transform with `runtime: "classic"` avoids it.
3. **GitHub's device-flow endpoints don't support CORS** — the relay Worker exists because of this, is stateless, and needs no secret.
4. Before editing `index.html`, remember the **verification workflow** used throughout this project for every change: extract the
   `#app-source` block, run it through the same `Babel.transform` call the browser will use, and `node --check` the output. Catches
   syntax errors and missing-import mistakes before they reach the browser. (Node's `@babel/standalone` package was installed locally
   in `/home/claude/gamejam-planner` for this — reinstall if starting a fresh session: `npm install --no-save @babel/standalone`.)

## What's next — not yet decided, pick a direction

Remaining MVP items from planning.md not yet built:

- **Countdown** (to jam deadline)
- **Milestones** (`milestones.json` is scaffolded on init but nothing reads/writes/displays it yet)
- **Dashboard** (overview screen — currently the board is the only view)
- **Team** (`team.json` scaffolded but unused — collaborators are fetched live from GitHub each time instead)
- **Assets tracking** (`assets.json` scaffolded, completely unused)

Long-term / stretch items from planning.md, not started: live presence, WebRTC planning sessions, AI-assisted planning, engine
templates, Panic Mode, build checklist.

Also worth considering, not in planning.md but came up naturally this session: a real field-level conflict merge (vs. the current
reload-and-discard), card reordering within a column (currently cardIds order is preserved but nothing lets you manually reorder,
drag-and-drop only moves between columns not within one), and card descriptions aren't previewed on the card face at all (modal-only).
