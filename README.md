# Snapshot Finesse

> **Local-first, Git-free workspace snapshots for Visual Studio Code.**
> Compressed, deduplicated, time-travel backups with a full analytics dashboard — no cloud, no AI, no external dependencies.

---

## Table of Contents

- [Overview](#overview)
- [Feature Highlights](#feature-highlights)
- [Full Feature Checklist](#full-feature-checklist)
- [How It Works](#how-it-works)
- [Architecture](#architecture)
  - [Source Modules](#source-modules)
- [Features In Depth](#features-in-depth)
  - [Snapshot Engine](#snapshot-engine)
  - [Test Point](#test-point)
  - [Group Diff](#group-diff)
  - [Side Panel](#side-panel)
  - [Status Bar](#status-bar)
  - [Analytics Dashboard](#analytics-dashboard)
  - [Exclusions Manager](#exclusions-manager)
  - [Dependency Warning](#dependency-warning)
  - [Skeleton Index](#skeleton-index)
- [Storage Format](#storage-format)
- [Configuration](#configuration)
- [Commands & Keybindings](#commands--keybindings)
- [Tech Stack & Frameworks](#tech-stack--frameworks)
- [Design Decisions](#design-decisions)
- [Project Structure](#project-structure)
- [Spec](#spec)
- [License](#license)

---

## Overview

Snapshot Finesse is a VS Code extension that gives developers a fast, lightweight backup system that lives entirely inside their workspace. Unlike Git it has no staging area, no concept of branches, and requires zero ceremony — a snapshot is just a compressed copy of every file at a point in time, stored in a hidden `.snapshotfinesse` directory alongside your code.

The core value proposition is **time-travel without friction**: take a snapshot before a risky refactor, experiment freely, and roll back with a single click. The extension also ships a full analytics dashboard so you can understand exactly how your codebase is evolving over time.

---

## Feature Highlights

- 📸 **Snapshot Now** — `Ctrl+Alt+S` or click the button in the sidebar
- ⏱ **Auto Snapshots** — on every save, or on a configurable timer interval (default 5 mins)
- ⚡ **Test Point** — safely explore a past snapshot; Keep or Restore when done
- 🔄 **Restore** — entire workspace snapshot or single file, with confirmation
- 🔍 **Compare / Diff** — VS Code's native diff viewer for any file in any snapshot group
- 🗜 **Brotli Compression** — content-addressable chunks; 60–80% storage savings vs raw copies
- 🌐 **Dashboard** — full analytics UI served at `http://127.0.0.1:1212` with charts, heatmap, settings, snapshot browser
- 🏷 **Rename Snapshots** — inline from the sidebar or tree view
- ♻️ **Deduplication** — identical file content is stored once regardless of how many snapshots reference it
- ⚠️ **Dependency Warnings** — warns which files will be affected before you restore
- 🧠 **Skeleton Index** — structural code map stored per snapshot for future semantic diff
- 🗑 **Delete All** — one-click wipe of all snapshots and blobs, with confirmation

---

## Full Feature Checklist

### Sidebar Panel

- [x] Snapshot Now button (`Ctrl+Alt+S`)
- [x] Auto-save toggle (on every file save)
- [x] Timer snapshots (every N minutes, configurable)
- [x] Rename snapshot (inline button on hover)
- [x] Delete snapshot (inline button, with confirmation)
- [x] Open Diff — VS Code native diff viewer for any file in a group
- [x] Restore snapshot (full group or single file, with confirmation)
- [x] Test Point (amber banner, Keep / Restore to Before)
- [x] Delete All Snapshots (danger button with modal confirmation)
- [x] Refresh
- [x] Stats card — total backups, files tracked, storage used, auto-save state

### Status Bar

- [x] Shows last snapshot time · backup count · storage size
- [x] Turns amber / flashes confirmation text after each snapshot
- [x] Click to toggle auto-save on/off

### Dashboard (port 1212 · Fastify · `dashboard/index.html`)

- [x] Dark / light mode toggle (persisted to localStorage)
- [x] KPIs — total snapshots, files tracked, storage used, space saved %, dedup rate, largest file
- [x] Activity bar chart (last 14 days, Chart.js)
- [x] Storage growth line chart (cumulative KB over time)
- [x] Activity heatmap (12 weeks, 4-level colour intensity, hover tooltips)
- [x] Top files by snapshot count table
- [x] File churn table (most-changed files by unique content versions)
- [x] Full snapshot browser with collapsible group tree
- [x] Per-file snapshot history with changed/unchanged indicators
- [x] Analytics page — compression trend, dedup trend, per-file compression bar, avg size, hour-of-day frequency, change frequency table
- [x] Compression page — savings per day, cumulative dedup refs, animated progress bar, by-extension breakdown, dedup detail
- [x] Settings panel — edit all settings from the UI (saved via REST API)
- [x] Exclusions manager — three-column file tree → glob list UI
- [x] Auto-refreshes home every 30 seconds
- [x] Opens inside VS Code as a `WebviewPanel` (iframe), not an external browser

### Dependency Warnings (`dependencyAnalyser.ts`)

- [x] On restore — scans `import`, `require`, `href`, `src` across entire workspace
- [x] Warns if other files reference the file being restored
- [x] Shows which files would be affected before you confirm

### Skeleton Index (per snapshot entry)

- [x] Stored in `index.json` alongside each snapshot entry
- [x] Captures file structure: functions, classes, exports — via regex, no AST, no build step
- [x] Supported extensions: `.ts`, `.tsx`, `.js`, `.jsx`, `.vue`, `.svelte`
- [x] Designed for future structural diff: "function `processOrder` was removed in this snapshot"

---

## How It Works

### Taking a Snapshot

1. The extension walks the workspace root recursively, respecting all configured exclusion glob patterns and the built-in skip of `.git`, `node_modules`, and the `.snapshotfinesse` directory itself.
2. Every file that passes the size limit check has its content hashed with **SHA-256**.
3. If the hash matches the most recent snapshot for that file, the file is counted as **unchanged** and skipped — no new entry is written to disk.
4. For files that _have_ changed, the raw content is compressed with **Brotli** (quality level 1–11, default 11) and written as a content-addressable blob at `objects/<xx>/<remaining>.br`, where `<xx>` is the first two hex characters of the SHA-256 hash.
5. A flat JSON index (`index.json`) records metadata for every snapshot entry — file path, timestamp, original size, compressed size, hash, label, group ID, and an optional code skeleton.
6. If `newCount === 0` after scanning all files, the index is **not written** and the backup is silently skipped, preventing empty snapshot groups from accumulating.

### Deduplication

Blobs are stored by their content hash. If ten snapshots all contain the same version of a file, only one `.br` blob exists on disk. Metadata entries point at the shared blob. When a snapshot is deleted, the blob is only removed if no other entry still references it.

### Groups

Every call to `saveWorkspaceSnapshot` generates a `groupId` (a 10-character URL-safe random string). All file entries created in that call share the same `groupId`, so the entire workspace state can be restored atomically, diffed as a unit, or deleted together.

---

## Architecture

```
VS Code Extension Host
│
├── extension.ts          ← Activation, command registration, orchestration
│
├── snapshotManager.ts    ← Core engine: walk, hash, compress, deduplicate, index
├── panelProvider.ts      ← WebviewViewProvider: sidebar panel UI
├── treeProvider.ts       ← TreeDataProvider: file-based tree view
├── statusBar.ts          ← Status bar item with flash feedback
├── timerManager.ts       ← Interval-based auto-snapshot timer
├── previewManager.ts     ← Test Point / preview session guard
├── dependencyAnalyser.ts ← import/require/href/src regex scanner for restore warnings
├── skeletonIndex.ts      ← Code structure extractor (functions, classes, exports)
├── server.ts             ← Fastify v5 HTTP server exposing REST API for dashboard
└── types.ts              ← Shared TypeScript interfaces

dashboard/
└── index.html            ← Full analytics SPA served by Fastify (Chart.js, vanilla JS)
```

### Source Modules

| File | Responsibility |
|---|---|
| `extension.ts` | Activates the extension, registers all commands, wires file-save listeners, manages Test Point state, opens dashboard in a VS Code `WebviewPanel` |
| `snapshotManager.ts` | Core engine. Full lifecycle: walk → hash → compress → dedup → index → prune. Exposes `saveWorkspaceSnapshot`, `saveSnapshot`, `restoreSnapshot`, `restoreGroup`, `deleteGroup`, `deleteAllSnapshots`, and all query methods |
| `panelProvider.ts` | `vscode.WebviewViewProvider`. Renders the sidebar panel via injected HTML/CSS/JS, communicates bidirectionally via `postMessage`. Hosts stats card, action buttons, Test Point banner, and backup list |
| `server.ts` | [Fastify](https://fastify.dev/) v5 HTTP server (default port `1212`). Exposes REST API consumed by the dashboard: `/api/stats`, `/api/snapshots`, `/api/settings`, `/api/exclusions`, `/api/workspace-tree`, `/api/delete-all` |
| `treeProvider.ts` | `vscode.TreeDataProvider`-based file-grouped tree view with collapsible snapshot entries |
| `timerManager.ts` | `setInterval` loop that triggers workspace snapshots at the configured minute interval. Restarts automatically on config change |
| `statusBar.ts` | Clickable status bar item showing auto-save state; flashes a confirmation message after each snapshot |
| `previewManager.ts` | Guards snapshot writes during an active Test Point session so the safety snapshot cannot be overwritten mid-session |
| `dependencyAnalyser.ts` | Regex-based scanner for `import`, `require`, `href`, and `src` across all workspace files. Identifies which files reference the file about to be restored |
| `skeletonIndex.ts` | Extracts a lightweight structural index from `.ts/.tsx/.js/.jsx/.vue/.svelte` files (exported functions, classes, type aliases) stored with each snapshot entry |
| `types.ts` | Defines `SnapshotMeta`, `SnapshotIndex`, and `WorkspaceSnapshotResult` |

---

## Features In Depth

### Snapshot Engine

- **Workspace snapshots** — captures every file in the workspace in a single operation, grouped by a shared `groupId`
- **Per-file snapshots** — `saveSnapshot` captures a single file (used for on-save triggers)
- **Skip unchanged** — if no file content has changed since the last backup, the entire snapshot is aborted and no disk writes occur
- **Content-addressable blobs** — Brotli-compressed files stored at `objects/<hash[0:2]>/<hash[2:]>.br`; identical content is stored exactly once regardless of how many snapshots reference it
- **Automatic pruning** — per-file history is capped at `maxSnapshots` (default 50); oldest entries are evicted and orphaned blobs deleted
- **Configurable exclusions** — glob patterns (supporting `**`, `*`, `?`) filter files before any hashing occurs
- **File size cap** — files over `maxFileSizeKB` (default 10 MB) are skipped entirely

### Test Point

Test Point is the renamed and extended **Preview** feature. It allows developers to safely restore a past snapshot, experiment, and then either keep or discard their changes.

**Workflow:**
1. Click **Test Point** (⚡) on any snapshot row, or run `Snapshot Finesse: Start Test Point` from the command palette
2. A safety snapshot is automatically taken first, tagged `test-point-base`, capturing the current workspace state
3. The chosen snapshot group is restored atomically across all its files
4. An amber banner appears in the sidebar panel reading **TEST POINT ACTIVE**, showing the label of the safety snapshot
5. A notification toast offers a **Stop Test Point** shortcut
6. On stop, a modal asks:
   - **Keep Current State** — ends the session, keeps whatever is on disk now
   - **Restore to Before** — restores the safety snapshot, reverting all changes made during the test

### Group Diff

When diffing a workspace snapshot the `openDiff` command shows a **QuickPick** listing every file in the group. Selecting a file opens VS Code's native diff editor comparing the snapshot blob against the current file on disk.

### Side Panel

The `panelProvider.ts` webview renders inside VS Code's sidebar explorer panel, communicating with the extension host via `postMessage`.

- **Test Point banner** — amber top bar, visible only when a Test Point is active. Shows safety snapshot label and a Stop button
- **Stats card** — total backups, files tracked, storage used, auto-save status
- **Action buttons** — Snapshot Now, Auto-Save toggle, Open Dashboard, Test Point, Delete All Snapshots
- **Backup list** — one row per workspace snapshot group; hover reveals per-row icons: Rename, Diff, Test Point, Restore, Delete

### Status Bar

The status bar item lives in the bottom bar of VS Code at all times.

- Displays: last snapshot time · backup count · storage size
- Flashes a confirmation string (e.g. `$(history) Snapped!`) after every successful snapshot
- Turns amber and shows `PREVIEW` text while a Test Point session is active
- Click to toggle auto-save on/off

### Analytics Dashboard

The dashboard is `dashboard/index.html` — a self-contained SPA served by Fastify on port `1212` and embedded in VS Code via a `WebviewPanel` iframe. It never opens an external browser.

**Home** — KPI cards, activity bar chart, storage growth line chart, 12-week activity heatmap, top-files table, file churn table

**Snapshots** — collapsible group tree with per-file changed/unchanged indicators and size figures

**Analytics** — compression ratio trend, dedup rate trend, per-file compression bar chart (top 12), avg file size per day, snapshot frequency by hour of day, file change frequency table

**Compression** — savings per day chart, cumulative dedup refs chart, animated overall ratio bar, by-extension breakdown table, dedup detail panel

**Exclusions** — three-column workspace tree → glob list manager; add by clicking files or typing patterns; save via REST

**Quick Settings** — toggle switches and number inputs for all settings; saved via `POST /api/settings`

### Exclusions Manager

Glob matching has no third-party dependency. The custom `matchesGlob` function converts patterns to regular expressions, supporting `**`, `*`, and `?`. The `.snapshotfinesse` storage directory is always skipped regardless of user config.

### Dependency Warning

Before restoring a snapshot, `dependencyAnalyser.ts` scans all workspace files for `import`, `require`, `href`, and `src` references to the file being restored. If any are found, a warning lists the affected files before confirmation is requested.

### Skeleton Index

For `.ts`, `.tsx`, `.js`, `.jsx`, `.vue`, and `.svelte` files, `skeletonIndex.ts` extracts exported functions, classes, type aliases, and interface names via regex (no AST parser, no build step). The skeleton is stored with each snapshot entry in `index.json`, enabling future semantic diff without re-decompressing blobs.

---

## Storage Format

```
<workspace-root>/
└── .snapshotfinesse/
    ├── settings.json          ← Quick Settings saved from the dashboard
    ├── index.json              ← All snapshot metadata (flat JSON array)
    └── objects/
        ├── ab/
        │   └── cdef1234....br  ← Brotli blob, path = SHA-256 hash
        └── ff/
            └── 001a....br
```

### `index.json` entry shape

```jsonc
{
  "id": "aBcDeFgHiJ",           // 10-char nanoid (crypto.randomBytes)
  "fileRelPath": "src/app.ts",  // relative to workspace root
  "timestamp": 1750000000000,   // Unix ms
  "label": "17 Jun 2026 19:30", // human-readable label
  "groupId": "xYzAbCdEfG",      // shared across all files in one workspace snapshot
  "hash": "<sha256hex>",
  "originalSize": 14200,        // bytes before compression
  "compressedSize": 3100,       // bytes of .br blob on disk
  "tags": [],                   // e.g. ["test-point-base"]
  "skeleton": { ... }           // optional: functions, classes, exports
}
```

---

## Configuration

All settings live in `<workspace-root>/.snapshotfinesse/settings.json` and are edited from the dashboard's Quick Settings panel.

| Setting | Default | Description |
|---|---|---|
| `autoSnapshotEnabled` | `true` | Master switch for all automatic snapshots |
| `autoSnapshotOnSave` | `false` | Snapshot on every file save — off by default to avoid many tiny backups |
| `autoSnapshotInterval` | `5` | Minutes between timer snapshots (0 = disabled) |
| `maxSnapshots` | `50` | Max snapshots per file — oldest are pruned |
| `maxFileSizeKB` | `10240` | Skip files larger than this (default 10 MB) |
| `compressionLevel` | `11` | Brotli level 1–11 (1 = fast, 11 = maximum compression) |
| `dashboardPort` | `1212` | Local port for the Fastify dashboard server |
| `exclude` | `["**/node_modules/**", "**/.git/**", "**/dist/**", ...]` | Glob patterns to exclude from snapshots |

---

## Commands & Keybindings

| Command ID | Title | Shortcut |
|---|---|---|
| `snapshotFinesse.saveNow` | Snapshot Now | `Ctrl+Alt+S` / `Cmd+Alt+S` |
| `snapshotFinesse.restore` | Restore Snapshot | — |
| `snapshotFinesse.openDiff` | Open Diff | — |
| `snapshotFinesse.testPoint` | Start Test Point | — |
| `snapshotFinesse.stopTestPoint` | Stop Test Point | — |
| `snapshotFinesse.openDashboard` | Open Dashboard | — |
| `snapshotFinesse.openSettings` | Open Settings | — |
| `snapshotFinesse.deleteSnapshot` | Delete Snapshot | — |
| `snapshotFinesse.deleteAll` | Delete All Snapshots | — |
| `snapshotFinesse.renameSnapshot` | Rename Snapshot | — |
| `snapshotFinesse.toggleAutoSave` | Toggle Auto-Save | — |
| `snapshotFinesse.refresh` | Refresh Panel | — |

---

## Tech Stack & Frameworks

| Layer | Technology |
|---|---|
| Extension language | **TypeScript 5** compiled to CommonJS via `tsc` — no bundler, no Webpack/Vite for the extension itself |
| VS Code API | `vscode` ^1.85.0 — WebviewView, WebviewPanel, TreeDataProvider, commands, configuration, text documents, native diff editor |
| Compression | Node.js built-in **`zlib.brotliCompressSync`** / `brotliDecompressSync` |
| Hashing | Node.js built-in **`crypto.createHash('sha256')`** |
| HTTP server | **[Fastify](https://fastify.dev/) v5** — the only `dependencies` entry in `package.json`. Chosen over Express for lower overhead, built-in TypeScript types, and near-zero startup latency |
| Dashboard | `dashboard/index.html` — standalone HTML file served statically by Fastify. Vanilla JS, no framework, no bundler |
| Dashboard charts | **[Chart.js](https://www.chartjs.org/) v4.4** — loaded from jsDelivr CDN inside the dashboard |
| File system | Node.js built-in `fs` (sync APIs for consistent, low-latency reads/writes) |
| Glob matching | Custom regex-based `matchesGlob` — no `minimatch`, no `micromatch`, no `glob` |
| Storage | Flat JSON index + content-addressable Brotli blobs — no SQLite, no LevelDB, no database |
| Dependency analysis | Regex scanner in `dependencyAnalyser.ts` — no TypeScript compiler API, no AST |
| Skeleton extraction | Regex-based structural extractor in `skeletonIndex.ts` — no AST, no build step |

### Explicitly ruled out

| Candidate | Reason excluded |
|---|---|
| SQLite | Overkill for a flat metadata list; adds a native binary dependency |
| Git | Extension must work on any workspace, not just Git repos |
| AI / LLM | Out of scope by design — no telemetry, no network calls |
| Webpack / esbuild | Extension compiled with plain `tsc`; no bundling overhead |
| minimatch / micromatch | Supply-chain risk for a 10-line glob function |
| External browser | Dashboard opens in VS Code WebviewPanel, not a browser tab |

---

## Design Decisions

### No Git dependency
The extension works on any workspace. It makes no Git assumptions and does not call `git` at any point.

### Content-addressable storage
Blobs are keyed by SHA-256 hash. Fifty snapshots referencing the same file version store one blob. Deletion is reference-counted — a blob is removed only when no index entry references its hash.

### Skip-if-unchanged
The engine hashes each file before writing anything. If the hash matches the most recent snapshot for that file, no new index entry is created and the index file is not touched. If every file in the workspace is unchanged, the entire snapshot call is a no-op.

### `autoSnapshotOnSave` defaults to `false`
Even with dedup, saving dozens of times per session creates many genuine new entries per changed file. The timer (every 5 minutes) is the sensible default cadence; on-save is opt-in.

### Fastify v5 as the only dependency
Fastify starts in < 5 ms, has TypeScript types built in, and handles concurrent API requests from the dashboard without any configuration. It is significantly faster than Express and has no transitive risk from a large dependency tree.

### Dashboard in VS Code WebviewPanel
The Fastify server runs on `127.0.0.1:1212`. The `WebviewPanel` embeds it in an iframe with a CSP that permits `http://127.0.0.1:*`. The dashboard opens as a VS Code tab — no external browser process, no focus loss.

### Regex over AST for skeleton and dependency analysis
Using the TypeScript compiler API or a full parser would add hundreds of milliseconds to activation time and megabytes to the packaged extension. Regex patterns covering `export function`, `export class`, `import from`, and `require()` cover 95% of real-world cases at essentially zero cost.

### Skeleton index stored at snapshot time
Re-extracting structural information later requires decompressing every blob. Capturing it once at write time (< 1 KB of JSON per file) makes future features like "show snapshots where this function existed" trivially fast.

---

## Project Structure

```
snapshot-finesse/
├── src/
│   ├── extension.ts          # Entry point, command registration
│   ├── snapshotManager.ts    # Core snapshot engine
│   ├── panelProvider.ts      # Sidebar webview panel
│   ├── treeProvider.ts       # Explorer tree view (file-grouped)
│   ├── server.ts             # Fastify v5 dashboard REST API
│   ├── timerManager.ts       # Auto-snapshot interval timer
│   ├── statusBar.ts          # VS Code status bar item
│   ├── previewManager.ts     # Test Point session guard
│   ├── dependencyAnalyser.ts # import/require/href/src regex scanner
│   ├── skeletonIndex.ts      # Structural code extractor (no AST)
│   └── types.ts              # Shared TypeScript types
├── dashboard/
│   └── index.html            # Self-contained analytics SPA (Fastify-served)
├── package.json              # Extension manifest & VS Code contributions
├── tsconfig.json             # TypeScript compiler config
├── CHANGELOG.md
└── SPEC.md                   # Original design specification
```

---

## Spec

`SPEC.md` in the repository root contains the original design specification, covering:

- **Design Principles** — local-only, no Git, no AI, no SQLite, no AST, no build step
- **Technology Stack** — every layer with the exact library chosen and why
- **Storage Layout** — full `index.json` schema, CAS blob path format, deduplication logic
- **All 10 source files** — each file's responsibility spelled out
- **Every command & keybinding** — all commands with shortcuts
- **Sidebar tree structure** — node hierarchy, inline icons, context menu, toolbar buttons
- **Auto-snapshot triggers** — on-save and timer logic, suppression rules
- **Test Point state machine** — full flow: enter → keep / restore to before
- **Status bar states** — three modes with exact colours and text
- **Dependency analyser** — regex patterns, scanned reference types, warning display
- **Skeleton index** — captured symbol categories, supported extensions, diff output shape
- **Dashboard** — all API routes, all pages, layout, theming
- **All settings** — types, defaults, descriptions
- **Out of scope table** — explicitly documents what was ruled out and why

---

## License

MIT © Stephen S / [METAFinesse.com](https://METAFinesse.com)
