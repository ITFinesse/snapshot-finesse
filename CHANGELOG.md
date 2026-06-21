# Changelog

## [0.3.1] — 2026-06-17

### Security
- Upgraded `fastify` from `^4.26.0` to `^5.8.5` to fix 5 high-severity CVEs in `fast-uri` (GHSA-q3j6-qgpj-74h6, GHSA-v39h-62p7-jpjc) and related transitive deps (`@fastify/ajv-compiler`, `fast-json-stringify`, `@fastify/fast-json-stringify-compiler`)

### Changed
- `src/server.ts`: Updated Fastify instantiation to use v5 default export pattern (`fastifyModule.default ?? fastifyModule`)
- `src/server.ts`: Added comment noting Fastify v5 automatic JSON body parsing for `POST /api/settings`

## [0.3.0] — 2026-06-17

### Added
- `dashboard/index.html` — standalone dashboard UI file served directly by Fastify (no inline HTML string)
- `src/dependencyAnalyser.ts` — restored; regex-based dependency scanner warns before restore
- `src/skeletonIndex.ts` — regex structural extractor; stores symbols per snapshot in `index.json`
- `src/previewManager.ts` — preview mode state machine (enter → discard / save)
- `src/timerManager.ts` — interval-based auto-snapshot timer
- `src/statusBar.ts` — status bar item with preview mode amber state
- `SPEC.md` — full technical specification
- Dashboard: sidebar nav, structural diff panel, settings page with save
- Dashboard: auto-refresh every 30 seconds

### Fixed
- Server no longer inlines HTML as a string in `server.ts`
- Status bar correctly shows amber warning colour during preview mode
- Pre-preview snapshots excluded from maxSnapshots prune count

## [0.2.0] — 2026-06-16

### Added
- Initial working extension with snapshot save, restore, diff, tree view, and Fastify dashboard
