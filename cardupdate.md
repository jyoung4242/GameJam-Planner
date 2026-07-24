# Card UI + GitHub Issues Integration

## Lightweight Card Model (MVP)

---

# Vision

The planner should remain a **fast, lightweight Kanban board** designed specifically for game jams.

GitHub Issues provide collaboration, history, and persistence.

The planner provides a streamlined planning experience without attempting to replicate the full GitHub Issues interface.

The guiding principle is:

> **If GitHub already solves it well, don't duplicate it. Build only the planning experience that GitHub is missing.**

---

# Design Goals

- One Card = One GitHub Issue
- Keep card editing fast
- Minimize required fields
- Avoid feature bloat
- Support game jam workflows first
- Leverage GitHub rather than replace it

---

# Card Data Model

```ts
interface Card {
  // Planner identity
  id: string;

  // GitHub
  issueNumber?: number;
  issueUrl?: string;

  // GitHub-backed
  title: string;
  description: string;

  assignee?: string;

  tags: string[];

  // Planner metadata
  status: CardStatus;

  priority?: Priority;

  category?: Category;

  dueDate?: string;

  createdAt: string;
  updatedAt: string;
}
```

---

# Card Layout

```
----------------------------------

Title

Description (Markdown)

----------------------------------

Status

Assignee

Priority

Category

Tags

Due Date

----------------------------------

Checklist

----------------------------------
```

The editor should remain intentionally minimal.

No secondary tabs.

No advanced configuration.

Everything visible on one screen.

---

# GitHub Synchronization

## GitHub Owns

- Issue Title
- Issue Description
- Labels
- Assignee
- Open / Closed State

## Planner Owns

- Board Status
- Priority
- Category
- Due Date
- Card Layout
- Visual Workflow

---

# Title

Maps directly to

```
Issue.title
```

---

# Description

Use Markdown.

Synchronizes directly with

```
Issue.body
```

No rich text editor.

Markdown is sufficient for technical teams.

---

# Assignee

Support a single assignee.

```ts
assignee?: string;
```

Although GitHub supports multiple assignees, the planner intentionally limits ownership to one person per task.

This keeps responsibility clear during fast-moving game jams.

---

# Category

Categories are first-class properties.

Suggested values:

- Programming
- Art
- Animation
- Audio
- UI
- Design
- QA
- Writing

Categories are intended for filtering and organization.

Unlike tags, they are selected from a predefined list.

---

# Tags

Tags remain lightweight and user-defined.

Examples:

- player
- shader
- gameplay
- boss
- bug
- optimization

Internally, tags synchronize directly to GitHub Labels.

Users continue interacting with simple "Tags" in the UI without needing to think about GitHub labels.

```
Planner Tags

↓

GitHub Labels
```

---

# Priority

Priority remains a planner feature but is synchronized as GitHub labels.

Suggested priorities:

- Critical
- High
- Normal
- Low

Example mapping:

```
Critical

↓

priority:critical
```

This allows filtering both inside the planner and directly within GitHub.

---

# Due Date

Support a simple date picker.

The planner can later highlight:

- Due Today
- Due Tomorrow
- Overdue

Due dates remain planner metadata.

They are not synchronized to GitHub.

---

# Checklist

The checklist is one of the primary workflow features.

Instead of maintaining a separate checklist model, use GitHub Markdown checkboxes inside the Issue body.

Example:

```md
## Tasks

- [ ] Player movement
- [ ] Animation
- [x] Input
- [ ] Audio
```

The planner should parse Markdown checklists and render them as native interactive checkboxes.

When a checkbox is toggled:

- Update the Markdown
- Synchronize the Issue
- Save the planner card

This avoids duplicate data while providing a much nicer editing experience.

---

# Status

Planner workflow remains independent from GitHub.

Suggested columns:

- Backlog
- In Progress
- Blocked
- Done

Synchronization:

```
Done

↓

Close GitHub Issue
```

Moving a card back out of Done automatically reopens the Issue.

---

# Features Intentionally Deferred

The following features are intentionally excluded from the MVP to keep the experience lightweight and maintainable:

- Multiple assignees
- Milestones
- Dependencies
- Attachments
- Embedded comments
- GitHub discussion thread
- Acceptance Criteria
- Story estimates
- Pull Request viewer
- Branch management
- Commit history
- Activity timeline

Most of these are already handled well by GitHub or introduce additional complexity that is unnecessary during a typical game jam.

---

# GitHub API Changes

## Create Issue

```
POST /repos/{owner}/{repo}/issues
```

Populate:

- title
- body
- labels
- assignee

---

## Update Issue

```
PATCH /repos/{owner}/{repo}/issues/{number}
```

Synchronize:

- title
- body
- labels
- assignee
- state

---

## Load Labels

```
GET /repos/{owner}/{repo}/labels
```

Used to populate the Tag autocomplete.

---

## Load Collaborators

Continue using the existing collaborator endpoint.

Single selection only.

---

# User Experience Goals

The ideal interaction should feel effortless:

- Create a card in seconds.
- Assign ownership immediately.
- Add a category and a few tags.
- Set a priority if needed.
- Pick a due date.
- Track progress with a simple checklist.
- Drag between board columns.
- Let GitHub handle everything else.

The planner should feel like a purpose-built game jam tool—not a clone of GitHub Projects or Jira.

---

# Future Roadmap

Potential enhancements after the MVP proves itself:

- Acceptance Criteria
- Story Estimates
- Custom Categories
- Saved Filters
- Keyboard Shortcuts
- Card Templates
- Burndown Dashboard
- Game Jam Countdown Widget
- Build Readiness Dashboard
- Personal "My Tasks" View

These features should only be added if they clearly improve the game jam workflow without increasing the cognitive load of creating and
managing tasks.
