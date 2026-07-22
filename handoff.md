# Game Jam Planner — Handoff

## Where this came from

Started from `planning.md` — a GitHub-backed, local-first React SPA vision (no custom backend, GitHub as auth/persistence/collaboration
layer, `.gamejam/` folder of JSON files as the project format).

**Important correction mid-session:** this is a standalone project, not tied to ExcaliburIDE. Deliverable is a **single self-contained
`.html` file** — no build tools, no npm, matching Justin's usual pattern (Shader Playground, SFXR generator). All architecture below
reflects that.

## Current deliverables

- **`gamejam-planner.html`** — the app itself. React 18 (UMD, CDN) + Babel Standalone (CDN) for JSX, transformed manually at load time.
  Currently implements: GitHub Device Flow login → repo picker (via GitHub App installations). Repo selection is wired but stops there
  — see "Next up."
- **`gamejam-planner-auth-relay.js`** — a Cloudflare Worker. Stateless CORS relay for GitHub's device-flow endpoints (see "Key
  technical findings" below for why this exists).

## Key technical findings from this session

1. **In-browser TypeScript via Babel Standalone doesn't work reliably** here — hit a hard parse error
   (`Unexpected reserved word 'interface'`) and separately a `file://` origin warning from the ESM `data-type` attribute. Dropped TS
   entirely; the file is plain JS now. Don't reintroduce `data-presets="react,typescript"` or `data-type="module"`.

2. **Babel's default JSX runtime ("automatic") injects an `import` statement** (`import { jsx } from "react/jsx-runtime"`) regardless
   of the `data-type` attribute, which breaks `file://` usage entirely (`Cannot use import statement outside a module`). Fix: don't let
   Babel auto-scan `<script type="text/babel">` tags at all. Instead the app source sits in
   `<script type="text/plain" id="app-source">` (inert to the browser), and a small bootstrap script at the bottom manually calls
   `Babel.transform(source, { presets: [["react", { runtime: "classic" }]] })` and injects the result as a real `<script>`. This forces
   classic `React.createElement` calls, no imports. **Any future edit to the app code must stay inside that `#app-source` block**, not
   a live `text/babel` tag.

3. **GitHub's device-flow endpoints don't support CORS** (`/login/device/ code` and `/login/oauth/access_token`) — confirmed against
   current GitHub docs and multiple corroborating issues. A browser can't call them directly no matter how the auth flow is shaped.
   Device Flow doesn't need a client secret though (only `client_id`), so the relay that works around this is genuinely stateless — no
   secrets, just a CORS-adding pass-through. That's `gamejam-planner-auth-relay.js`.

4. **This ended up registered as a GitHub App, not a classic OAuth App** (client ID `Iv23lijW75Uw61RKbn2I` confirms it — GitHub Apps
   use `Iv1.`/`Ov23li`-style client IDs; OAuth Apps use a 20-char hex string). This has real consequences, already reflected in the
   code:
   - No OAuth-style scopes — permissions are configured on the app itself (**Contents: Read & write**, **Metadata: Read-only** —
     confirm these are set under Permissions & webhooks).
   - `GET /user/repos` does **not** reliably return a GitHub App's accessible repos. The correct flow is `GET /user/installations` →
     `GET /user/installations/{id}/repositories`, flattened. Already implemented in `GitHubStorage.listUserRepos()`.
   - The app only sees repos it's been **installed** on. The UI has an "Install the app on a repository" link
     (`https://github.com/apps/{appSlug}/installations/new`) for the empty state, plus a persistent "manage installations" link once
     repos are showing.

## CONFIG values — current state

In `gamejam-planner.html`, near the top of the app-source block:

```js
const CONFIG = {
  clientId: "Iv23lijW75Uw61RKbn2I", // ✅ set
  relayBaseUrl: "https://your-relay.example.workers.dev", // ❌ placeholder — needs the deployed Worker URL
  appSlug: "YOUR_APP_SLUG", // ❌ placeholder — the dash-cased name from https://github.com/apps/<this>
  scopes: [], // sent but ignored for GitHub Apps
};
```

## GitHub App settings checklist

- [x] Device Flow enabled
- [x] Client ID obtained (`Iv23lijW75Uw61RKbn2I`)
- [ ] Homepage URL set to `https://jyoung4242.github.io/GameJam-Planner/`
- [ ] Authorization callback URL set to the same (ignored by device flow, but the form requires it)
- [ ] Permissions & webhooks → Repository permissions → **Contents: Read and write**
- [ ] Permissions & webhooks → Repository permissions → **Metadata: Read-only**

## Relay deployment — in progress

Was walking through deploying `gamejam-planner-auth-relay.js` to Cloudflare Workers via the dashboard (no CLI):

1. dash.cloudflare.com → Workers & Pages → Create → Create Worker
2. Name it (e.g. `gamejam-planner-auth-relay`), Deploy the placeholder
3. Edit code → paste in the relay file's contents → Deploy
4. Copy the resulting `*.workers.dev` URL → paste into `CONFIG.relayBaseUrl`

Also already set `ALLOWED_ORIGIN` in the relay to `https://jyoung4242.github.io` (origin only, no path) since the Pages origin is now
known.

**This step wasn't confirmed finished** — pick up here if resuming.

## GitHub Pages deployment note

The HTML file must be named **`index.html`** in the repo/path that resolves to `https://jyoung4242.github.io/GameJam-Planner/`, not
`gamejam-planner.html`, or the URL will 404.

## Next up (once login/repo-picker is verified working end to end)

The `App` component currently stops after repo selection with a stub:

```js
// Next step: detectProject() -> either open the existing .gamejam/
// workspace or offer to initialize one, per planning.md's
// "GitHub Flow" section.
```

`GitHubStorage` already has the methods needed for this: `detectProject`, `initProject`, `readManifest`, `readBoard`, `getFile`,
`putFile` (conflict detection via git blob sha — a stale `sha` on write comes back as 409/422 from GitHub's Contents API, caught and
converted to `StorageConflictError`).

Not yet ported into the single-file build: the **Workspace** abstraction (dirty-state tracking per file, batched autosave every few
minutes, manual save, conflict events) that was designed earlier in the session before the pivot to single-file JS. The design is sound
and worth reusing — dirty tracking keyed by file path/card id, `save()` batches only dirty files, autosave timer calls `save()`
periodically — just needs to be written as plain JS inside the `#app-source` block rather than the original TS module version (which
was deleted).

Suggested order from here:

1. Confirm relay + full login → repo list flow works live.
2. Wire `detectProject()` on repo selection: if `.gamejam/project.json` exists, load it; if not, prompt to initialize (per
   planning.md's "New Project Wizard" / "GitHub Flow" sections).
3. Port the Workspace/dirty-tracking/autosave logic into the single file.
4. Build the actual Kanban board UI on top of that.
