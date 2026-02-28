# Mission Control — CLAUDE.md

A reference guide for AI assistants working in this repository.

---

## Project Overview

**Mission Control** is a single-page Kanban task management dashboard that uses GitHub as its data backend. It is a **zero-build, single-file SPA** (`index.html`) with no package manager, no bundler, and no test framework. Data is persisted as JSON files committed to this repository and read/written via the GitHub API.

The dashboard is designed to work alongside **MoltBot** — an AI agent that receives webhooks when tasks are moved to "In Progress" and can autonomously create, update, and move tasks by editing `data/tasks.json`.

---

## Repository Structure

```
mission-control/
├── index.html          # Entire application: HTML + embedded CSS + embedded JS (~3,200 lines)
├── data/
│   ├── tasks.json      # Live task board data (source of truth for the UI)
│   ├── crons.json      # Cron job definitions for scheduled automations
│   └── version.json    # App version info checked every 5 minutes for live reload
└── CLAUDE.md           # This file
```

There is no `package.json`, no build output directory, no `node_modules`, and no CI/CD pipeline.

---

## Key Architecture Decisions

### Single-File Application
Everything lives in `index.html`. CSS is in a `<style>` block (~lines 11–2180), HTML structure is in the body (~lines 2182–2844), and JavaScript is in a `<script>` block (~lines 2845–3187). Do not split these into separate files — the single-file design is intentional for GitHub Pages compatibility and zero-dependency deployment.

### GitHub as Database
All persistent data lives in `data/tasks.json` and `data/crons.json`, committed to the `main` branch. The app reads these files via the GitHub Contents API and writes them back using authenticated PUT requests. There is no server, no database, and no backend API.

### Authentication Model
- **Authenticated mode**: Users provide a GitHub Personal Access Token (classic, `repo` scope). Token stored in `localStorage['mc_token']`.
- **Read-only mode**: If no token is present and the repo is public, the app loads `data/tasks.json` directly (relative URL, works on GitHub Pages).
- **Demo mode**: If public read fails, `enableDemoMode()` loads `FALLBACK_DATA` embedded in the JS.

---

## Core Data Schema

### `data/tasks.json`
```json
{
  "tasks": [
    {
      "id": "unique_string",
      "title": "Task title",
      "description": "Markdown supported",
      "status": "permanent | backlog | in_progress | review | done | scheduled | templates",
      "project": "project_id",
      "tags": ["tag1"],
      "subtasks": [{ "id": "sub_001", "title": "Step", "done": false }],
      "priority": "high | medium | low",
      "comments": [{ "author": "Name", "text": "Note", "timestamp": "ISO8601" }],
      "createdAt": "ISO8601",
      "completedAt": "ISO8601"
    }
  ],
  "projects": [
    { "id": "project_id", "name": "Display Name", "color": "#hex", "icon": "emoji" }
  ],
  "activities": [
    { "type": "created | updated", "actor": "User", "task": "Task Name", "time": "relative" }
  ],
  "lastUpdated": "ISO8601"
}
```

### `data/crons.json`
```json
{
  "lastUpdated": "ISO8601",
  "crons": [
    {
      "id": "uuid",
      "name": "Cron Name",
      "emoji": "🕐",
      "schedule": "0 8 * * *",
      "enabled": true,
      "lastStatus": "ok | error | null",
      "lastRunAt": 1234567890000,
      "nextRunAt": 1234567890000
    }
  ]
}
```

### `data/version.json`
```json
{
  "version": "2.2.2",
  "buildHash": "6fc797b",
  "buildTime": "ISO8601"
}
```
The app polls this file every 5 minutes. If the version changes, it prompts the user to reload.

---

## JavaScript Architecture

All JS is global scope inside a single `<script>` tag. Key globals:

```javascript
const CONFIG = {
  owner: 'rdsthomas',
  repo: 'mission-control',
  branch: 'main',
  tasksFile: 'data/tasks.json'
};

let STATE = {
  user: null,         // GitHub user object
  token: null,        // GitHub PAT from localStorage
  data: null,         // Current tasks.json content
  originalData: null, // Snapshot for unsaved-change detection
  hasUnsavedChanges: false,
  isLoading: false,
  gatewayUrl: null    // MoltBot webhook gateway URL
};
```

### Key Functions
| Function | Purpose |
|---|---|
| `checkAuth()` | Entry point: validates token or attempts public read |
| `enableDemoMode()` | Loads `FALLBACK_DATA` when unauthenticated |
| `loadData()` | Fetches `tasks.json` from GitHub API |
| `switchView(viewName)` | Switches between Tasks / Docs / People tabs |
| `getGatewayUrl()` | Resolves MoltBot webhook URL (localStorage → localhost:3033 → null) |
| `setGatewayUrl(url)` | Persists gateway URL to `localStorage['mc_gateway_url']` |
| `handleDragOver/Leave/Drop()` | Kanban drag-and-drop handlers |
| `renderTasks(tasks)` | Renders all task cards into columns |
| `showToast(msg, type)` | Displays temporary notification |

---

## Board Columns (Status Values)

| Column Label | `status` value | Purpose |
|---|---|---|
| Recurring | `permanent` | Always-visible recurring tasks |
| Scheduled | `scheduled` | Time-triggered tasks |
| Templates | `templates` | Reusable task templates |
| Permanent | `permanent` | Persistent reference tasks |
| Backlog | `backlog` | Upcoming work |
| In Progress | `in_progress` | Active work; triggers MoltBot webhook |
| Review | `review` | Awaiting human approval |
| Done | `done` | Completed; can be archived |

Moving a task to `in_progress` triggers the MoltBot webhook at `STATE.gatewayUrl`.

---

## MoltBot Integration

MoltBot is an external AI agent that interacts with this board:
- **Receives**: GitHub webhook (`push` events) or a direct webhook call when tasks move to `in_progress`
- **Acts by**: Editing `data/tasks.json` directly and committing to the `main` branch
- **Can**: Create tasks, update subtasks, add comments, change status, delete/archive tasks
- **Webhook URL pattern**: `https://YOUR-TAILSCALE-URL/hooks/github?token=YOUR-TOKEN`

When editing `data/tasks.json` programmatically (as MoltBot would), always:
1. Preserve the full JSON structure including `projects`, `activities`, and `lastUpdated`
2. Update `lastUpdated` to the current ISO8601 timestamp
3. Append to `activities` array rather than replacing it
4. Use unique IDs for new tasks (e.g., `task_<timestamp>` or a UUID)

---

## CSS Conventions

- All styles are embedded in `<style>` in `index.html`
- CSS custom properties (variables) defined on `:root` for the dark theme color system
- Class naming is kebab-case (e.g., `.task-card`, `.column-header`, `.drag-over`)
- No CSS preprocessors; plain CSS only
- Responsive breakpoints: `768px` (mobile), `1100px` (tablet), plus specific media queries for foldable devices (Surface Duo, Galaxy Fold)
- Minimum touch target size: 44px

---

## Development Workflow

### Making Changes
Since there is no build step, editing `index.html` directly is the development workflow:
1. Edit `index.html` in this repository
2. Open the file in a browser (or push to GitHub Pages) to test
3. For data changes, edit `data/tasks.json` or `data/crons.json` directly

### Testing
There is no automated test suite. Manual testing is done by:
- Opening `index.html` in a browser
- Using demo mode (`enableDemoMode()`) to verify rendering without authentication
- Checking browser DevTools console for errors

### Deployment
Deployment is via **GitHub Pages**:
- Pages source: `main` branch, root directory
- The published URL serves `index.html` directly
- `data/*.json` files are served as static files read by the GitHub Contents API

### Updating Version
After significant changes, update `data/version.json`:
```json
{
  "version": "X.Y.Z",
  "buildHash": "<short git SHA>",
  "buildTime": "<current ISO8601 timestamp>"
}
```

---

## Git Conventions

- Default working branch: `main`
- Commit messages: imperative, lowercase (e.g., `fix task card drag behavior`, `add cron schedule editor`)
- No branching strategy enforced; direct commits to `main` are the norm
- AI assistant branches follow the pattern: `claude/<description>-<sessionId>`

---

## Constraints & Anti-Patterns to Avoid

- **Do not introduce a build system** (no webpack, Vite, npm, etc.)
- **Do not split `index.html`** into separate JS/CSS files without explicit instruction
- **Do not add a backend** — all persistence is GitHub API + JSON files
- **Do not use TypeScript** — this codebase is plain JavaScript
- **Do not add external CSS frameworks** (Bootstrap, Tailwind, etc.)
- **Do not add dependencies** beyond the existing mobile-drag-drop polyfill CDN script
- **Do not use `var`** — use `const`/`let` throughout
- When editing `data/tasks.json`, never remove the `projects`, `activities`, or `lastUpdated` top-level keys
- When adding tasks, never reuse existing task `id` values

---

## External Dependencies

| Library | Source | Purpose |
|---|---|---|
| `mobile-drag-drop@2.3.0-rc.2` | jsDelivr CDN | Touch/mobile drag-and-drop support |

No other runtime dependencies. No dev dependencies.
