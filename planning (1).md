# Game Jam Planner - GitHub-Backed Architecture

## Vision

Build a local-first React SPA that uses GitHub as the persistence,
collaboration, authentication, and authorization layer. The application
acts as a visual editor for a Game Jam project rather than a generic
project management tool.

## Core Principles

-   No custom backend
-   No custom user accounts
-   Local-first
-   GitHub is the source of collaboration
-   Projects are human-readable JSON files
-   Works for solo developers and small teams

## Architecture

``` text
React SPA
    │
Local State (Zustand)
    │
IndexedDB Cache
    │
Storage Adapter
    │
GitHub (Octokit)
    │
Private Repository
```

## Repository Layout

``` text
.gamejam/
├── project.json
├── board.json
├── milestones.json
├── assets.json
├── team.json
├── cards/
│   ├── movement.json
│   ├── combat.json
│   └── enemy-ai.json
├── notes/
│   ├── ideas.md
│   └── retrospective.md
└── templates/
```

Each card is stored independently to reduce merge conflicts.

## Authentication

Users authenticate with their own GitHub credentials (PAT for MVP or
OAuth later).

The application never impersonates users.

## Authorization

A private repository is shared using GitHub collaborators.

Permissions come directly from GitHub:

-   Admin
-   Write
-   Read

The application adapts its UI based on the authenticated user's
repository permissions.

## Collaboration

Every collaborator edits the same repository.

Commits are authored by the current GitHub user.

Git history becomes the activity log.

## Saving Strategy

Do not commit every interaction.

Instead:

-   Maintain local dirty state.
-   Batch changes.
-   Save manually or autosave every few minutes.
-   Create meaningful commits such as:
    -   "Planning updates"
    -   "Autosave - Friday Planning"

## Sync

1.  Pull latest
2.  Merge JSON
3.  Push changes

Automatic merge for different fields.

Prompt the user only when the same field has conflicting edits.

## Storage Abstraction

``` ts
interface ProjectStorage {
  login(): Promise<void>;
  openProject(id: string): Promise<Project>;
  saveProject(project: Project): Promise<void>;
  sync(): Promise<void>;
}
```

Possible implementations:

-   GitHubStorage
-   LocalFolderStorage
-   DropboxStorage
-   OneDriveStorage

## MVP

1.  Create/Open project
2.  GitHub repository support
3.  Kanban board
4.  Rich task cards
5.  Countdown
6.  Milestones
7.  Dashboard
8.  Sync

## Long-Term

-   Live presence
-   WebRTC planning sessions
-   AI-assisted planning
-   Engine templates
-   Panic Mode
-   Asset tracking
-   Build checklist

## Elevator Pitch

> A local-first, GitHub-backed game jam planning tool that feels like
> Trello, understands game development, and requires no custom backend.


## Project Initialization Workflow

### Startup

- New Project
- Open Local Project
- Open GitHub Project
- Recent Projects

Authentication is only required when GitHub storage is selected.

### New Project Wizard

1. Project Details
   - Name
   - Description
   - Jam
   - Theme

2. Choose Storage
   - Local Folder
   - GitHub Repository

### GitHub Flow

If GitHub is selected:

1. Authenticate with GitHub.
2. Choose:
   - Create Repository
   - Use Existing Repository

If creating a repository:

- Create public/private repository.
- Create `.gamejam/` folder.
- Commit initial planner files.

If using an existing repository:

- Detect `.gamejam/project.json`.
- If found, open the workspace.
- Otherwise prompt to initialize the planner.

### Workspace Layout

```
MyGame/
├── src/
├── assets/
├── docs/
└── .gamejam/
    ├── project.json
    ├── board.json
    ├── milestones.json
    ├── assets.json
    ├── team.json
    ├── cards/
    └── notes/
```

### Workspace Abstraction

Treat every project as a Workspace rather than exposing Git concepts.

```ts
interface Workspace {
  storage: StorageProvider;
  project: Project;
}
```

Storage providers:

- GitHub
- Local Folder
- Dropbox (future)
- OneDrive (future)

The UI should expose **Save**, **Sync**, and **History** rather than Commit, Push, Pull, and Merge.
