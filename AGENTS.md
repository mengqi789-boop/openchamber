# OpenChamber - AI Agent Reference

## Core purpose

OpenChamber provides UI runtimes (web/desktop/VS Code) for interacting with an OpenCode server (local auto-start or remote URL). Official OpenCode traffic goes through `@opencode-ai/sdk`; OpenChamber-owned runtime capabilities go through `RuntimeAPIs`, `runtimeFetch`, and browser/realtime URL helpers.

## Runtime architecture (IMPORTANT)

- **Desktop** (Electron) boots the web server **in the same Node process** as the Electron main, then loads the web UI from `http://127.0.0.1:<port>`. No sidecar subprocess.
- Backend/domain logic lives in `packages/web/server/*` (and `packages/vscode/*` for VS Code bridge/runtime parity). Electron owns the desktop shell/security boundary: windows, menus, dialogs, notifications, updater, deep-links, runtime host switching, local IPC gates, and SSH/tunnel management.
- **Do not add OpenCode feature backends to the native shell.** Shared UI features should remain server/runtime APIs unless the capability is inherently native.

### Desktop Shell

- Desktop work goes into `packages/electron/`.
- Desktop-side changes (IPC handlers, native integrations, window/quit/notification behavior) land in `main.mjs` + `preload.mjs`.
- Electron imports the server via `@openchamber/web/server/index.js` and calls `startWebUiServer({...})`. The returned handle has `getPort()` / `stop()`. Notifications flow via an `onDesktopNotification` callback injected at startup — no stdout-parsing IPC.
- **Windows OS**: any non-user-visible `child_process` call should run the target executable directly with `windowsHide: true`. Avoid `cmd.exe /c` pipelines and wrappers that spawn console grandchildren. If a delayed/background operation must outlive the app process, use a single hidden first-level helper (e.g. `powershell.exe -WindowStyle Hidden -EncodedCommand ...`) or a native Node/Electron API.

## Tech stack & Monorepo layout

**Runtime/tooling**: Bun (`packageManager`), Node >=20 (`engines`)
**UI**: React, TypeScript, Vite, Tailwind v4
**State**: Zustand stores and sync layer (`packages/ui/src/stores/`, `packages/ui/src/sync/`)
**UI primitives**: Base UI (`@base-ui/react` — primary source for dropdown/select/dialog/menu/tooltip), HeroUI, Remixicon as SVG sprite source only (use shared `Icon`, never direct `@remixicon/react` imports)
**Server**: Express (`packages/web/server/index.js`)
**Desktop**: Electron 41 (`packages/electron/`)
**VS Code**: extension + webview (`packages/vscode/`)

| Workspace | Contents |
|---|---|
| `packages/ui` | Shared React UI, sync layer, runtime API contracts, stores |
| `packages/web` | Web app, Express server, CLI |
| `packages/electron` | Desktop shell |
| `packages/vscode` | VS Code extension |

## Build / dev commands

All scripts in `package.json`. Key commands:

- Validate: `bun run type-check`, `bun run lint`
- Build all: `bun run build`
- Desktop: `bun run electron:build` (build), `bun run electron:dev` (dev)
- VS Code: `bun run vscode:build` (build)
- Release smoke: `bun run release:test`

## Runtime entry points

| Target | Entry point |
|---|---|
| Web bootstrap | `packages/web/src/main.tsx` |
| Web server | `packages/web/server/index.js` |
| Web CLI | `packages/web/bin/cli.js` |
| Desktop | `packages/electron/main.mjs` |
| VS Code extension | `packages/vscode/src/extension.ts` |
| VS Code webview | `packages/vscode/webview/main.tsx` |

## OpenCode integration

- **UI client**: `packages/ui/src/lib/opencode/client.ts` (imports `@opencode-ai/sdk/v2`)
- **Sync/event pipeline**: app roots mount `SyncProvider` from `packages/ui/src/sync/sync-context.tsx`; OpenCode SSE/WS handling in `packages/ui/src/sync/event-pipeline.ts`
- **Server embed**: `packages/web/server/index.js` (`createOpencodeServer`)
- **External server**: Set `OPENCODE_HOST` (full base URL) or `OPENCODE_PORT`, plus `OPENCODE_SKIP_START=true`

## Key UI patterns (reference files)

| Area | Path |
|---|---|
| Settings shell | `packages/ui/src/components/views/SettingsView.tsx` |
| Settings primitives | `packages/ui/src/components/sections/shared/` |
| Settings sections | `packages/ui/src/components/sections/` |
| Chat UI | `packages/ui/src/components/chat/` |
| Theme + typography | `packages/ui/src/lib/theme/`, `packages/ui/src/lib/typography.ts` |
| Terminal UI | `packages/ui/src/components/terminal/` (uses `ghostty-web`) |

## External / system integrations

| Integration | Client | Server |
|---|---|---|
| Runtime API contracts | `packages/ui/src/lib/api/types.ts` | — |
| Runtime transport/auth | `packages/ui/src/lib/runtime-fetch.ts`, `packages/ui/src/lib/runtime-url.ts` | — |
| Git | `packages/ui/src/lib/gitApi.ts` | `packages/web/server/lib/git/service.js` (`simple-git`) |
| Terminal PTY | — | `packages/web/server/lib/terminal/runtime.js` (`bun-pty`/`node-pty`) |
| Skills catalog | `packages/ui/src/components/sections/skills/` | `packages/web/server/lib/skills-catalog/` |

## Working Guidelines

### Agent constraints

- Do not modify `../opencode` (separate repo).
- Do not run git/GitHub commands unless explicitly asked.
- Keep baseline green (`bun run type-check`, `bun run lint` before finalizing).

### Code of conduct

- Prefer the smallest correct change.
- Preserve working behavior before improving structure.
- Do not add cleverness where a direct implementation is enough.
- Do not infer critical state from weak signals when a stronger source exists.
- Do not encode policy only in UI; enforce it in core logic.
- Do not hide data loss, partial failure, or fallback behavior. Make it explicit in code.
- Finish work end-to-end: implementation, verification, and cleanup.

### Development rules

- Keep diffs tight; avoid drive-by refactors.
- Follow local precedent; inspect nearby code before introducing new patterns.
- Backend changes: keep web, desktop, and VS Code behavior consistent when they share contracts.
- TypeScript: avoid `any`, blind casts, and shape guessing.
- React: prefer function components + hooks; use classes only when required.
- Control flow: prefer early returns and explicit branching over nested ternaries.
- Styling: Tailwind v4, typography via `packages/ui/src/lib/typography.ts`, theme vars via `packages/ui/src/lib/theme/`.
- Shared UI patterns: reuse shared primitives before introducing feature-local markup.
- Toasts: use the wrapper from `@/components/ui`; do not import `sonner` directly.
- No new deps unless asked. Never add secrets or log sensitive data.

### CLI Parity and Safety Policy (MANDATORY)

**Principle**: policy-first, UX-second. All safety and correctness rules MUST be enforced in core command logic, independent of output mode. Interactive/pretty UX (`@clack/prompts`) is a presentation layer only — it must never be the only place where validation is enforced.

Required parity across all modes (interactive TTY, non-interactive shells, `--quiet`, `--json`, pre-specified flags):

- Invalid operations MUST fail with non-zero exit code and deterministic error semantics.
- Core validators MUST run even when prompts are unavailable or skipped.
- `--quiet` suppresses non-essential output only; it does not weaken validation.
- `--json` changes output shape only; it does not weaken validation.

Detailed patterns in `clack-cli-patterns` skill — load the skill before implementing CLI commands.

## Project Skills (MANDATORY)

Before editing, agents **MUST** load every skill whose trigger matches the work; if multiple rows apply, load all of them.

| Work being done | Required skill call |
|---|---|
| Terminal CLI commands, prompts, or output formatting | `skill({ name: "clack-cli-patterns" })` |
| Shared UI data access, `RuntimeAPIs`, `runtimeFetch`, OpenCode SDK calls, VS Code bridges, Electron runtime switching | `skill({ name: "ui-api-decoupling" })` |
| UI components, styling, visual elements, colors, buttons, or icons | `skill({ name: "theme-system" })` |
| User-facing UI text: labels, buttons, placeholders, aria labels, toasts, dialogs, settings copy | `skill({ name: "locale-ui-patterns" })` |
| Settings pages, settings dialogs, configuration UI | `skill({ name: "settings-ui-patterns" })` |
| Drag-to-reorder, sortable lists/chips/grids with `@dnd-kit` | `skill({ name: "drag-to-reorder" })` |
| Ultra-compressed communication (triggered by "talk like caveman", "less tokens", etc.) | `skill({ name: "caveman" })` |

Skill docs are the source of truth for detailed patterns.

## Architecture & Performance rules

These rules exist because violating them has caused measurable regressions (render cascades, memory bloat, UI jank). They apply to all UI and sync layer work.

### Architecture principles

- **Thin entrypoints, focused modules**: Keep `index.js`, bridge, bootstrap files, and provider roots thin. Move route/domain/runtime logic into focused modules with clear ownership.
- **Strong source of truth**: Prefer deterministic state over heuristics. Use live server/session state for live activity — don't let historical anomalies masquerade as current execution. If a fallback is necessary, scope it narrowly and treat it as temporary.
- **Live state vs historical state**: Derive live UI behavior from live state channels, not persisted history. Use historical records to restore context, not to infer that work is still in progress.
- **Cross-runtime parity**: If web defines a route or payload contract that shared UI depends on, keep VS Code and desktop parity. Do not ship a web-only assumption into shared UI.
- **Partial-failure-safe flows**: Cross-directory and multi-entity operations must tolerate partial failure. Never leave optimistic state or local caches stranded after failure.
- **Distinguish fetch failure from empty success**: Authoritative API methods (bootstrap, reconnect resync, retry loops) **must signal failure distinctly** from empty success. Pick one of two existing patterns — **throw on failure** (e.g. `listAgents`) or **return `T | null` on failure** (e.g. `getSessionStatusForDirectory`). Never swallow errors inside the method while returning the same type as success. Retry loops require a failure signal.
- **Reconnect-loop pacing**: Respect `navigator.onLine` (use long backoff ~60s when offline), `document.visibilityState` (use long cap when hidden), and HTTP status of the last failure (permanent 4xx → jump to long cap; 408/429 → normal backoff). Use real exponential growth, not constant delays.

### Store and render discipline

- **Treat common stores as render fanout boundaries.** An unnecessary reference change in shared state can re-render large parts of the app.
- **Update only the fields that changed.** Preserve references for untouched state branches.
- **Prefer leaf selectors over container selectors.** Subscribe to the smallest stable value that satisfies the component.
- **Isolate hot consumers.** If a value changes often and only a few components need it, move it to a narrower store or consume it in a memoized child.
- **Zustand referential equality**: Every new object/array reference triggers a re-render. Never spread all state fields in an update. Select leaf values, not containers. Preserve item identity when presentation-relevant fields are unchanged.
- **Store splitting**: Group state by change frequency and subscriber set. Cross-store reads use `.getState()` — imperative, no subscription. Never add unrelated state to an existing store just because it's convenient.
- **Extract high-frequency hook consumers into separate `React.memo` components.** Use custom comparators for message rows.

### Event pipeline and SSE

- **Gate expensive operations on the hot path.** During streaming (`message.part.delta` / `message.part.updated` ~60/sec), any `findIndex`, `filter`, or iteration multiplies across every event. Gate behind a cheap boolean check first.
- **Skip no-op updates.** If an incoming event doesn't change state, return `false` from the reducer to avoid creating new references.
- **Coalesce by key.** Same-entity events should replace earlier ones in the queue, not accumulate.
- **Two-phase polling**: Run cheap change detection first; only run heavy status fetches for directories that actually changed. Do not let lightweight polling erase rich fields.

### Optimistic updates

- Use the shadow Map pattern. Insert optimistic data into the store for instant UI, AND register it in a separate tracking Map. Cleanup happens via `mergeOptimisticPage` on the next data fetch — not via heuristics in the event reducer.
- Pass client-generated IDs to the server. Use the same ID format as the server (hex-encoded timestamps). Pass `messageID` to `promptAsync` so the server echoes back the same ID — prevents duplicates and enables in-place replacement.
- Rollback on error. Remove the optimistic entry from both the store and the shadow Map.

### Session/input consistency

- **Capture send config at queue time.** Queue items must include provider/model/agent/variant snapshot; do not re-resolve from mutable live state at send time.
- **Do not let text input state repaint unrelated chrome.** Typing should not force unrelated controls, menus, indicators, or toolbars to re-render on every keystroke. Extract slow-changing chrome from hot input paths behind memoized boundaries.

### Scroll and DOM

- **Never use `await waitForFrames()` for scroll preservation.** Use `useLayoutEffect` to adjust scroll synchronously after React commits DOM.
- **Capture scroll state before the state change, restore in layout effect.** Save `scrollHeight`/`scrollTop` into a ref before triggering the update, consume it in `useLayoutEffect`.
- **Do not let viewport resizes masquerade as content growth.**
- **Disable native/browser scroll anchoring when custom scroll logic exists.**
- **Autosize textareas without transient collapse on growth.** Avoid `height='auto'` shrink/expand cycles on every character.

### List ordering and view consistency

- **Do not sort structural lists directly from high-churn live fields.** If live updates are frequent, sorting directly from them causes reorder thrash. Freeze order during high-frequency updates and apply a one-shot reorder at an intentional lifecycle edge.
- **Use one ordering source for all views of the same data.** Do not let each surface re-derive ordering independently.

### Caching and memory

- **Cap in-memory caches with both count and byte limits.** Entry count alone doesn't prevent memory bloat from large files. Use dual-constraint LRU (e.g. 40 entries OR 20MB).
- **Set store session limits to match loaded data.** Otherwise the next SSE event triggers trimming that silently removes sessions.
- **Invalidate caches on mutations.** File content cache must clear entries on write, delete, rename. Prefetch cache must clear on session eviction.
- **Use TTLs to prevent redundant fetches.** If a session was fetched <15s ago, skip re-fetching — SSE events keep it current.

### Directory context

- **Never cache directory strings in closures.** Directory can change at any time (worktree switch). Read it dynamically from `opencodeClient.getDirectory()` at call time.
- **Pass directory hints when the source of truth isn't available yet.** Newly created sessions aren't in the sync store until SSE delivers them. Pass the known directory as a parameter.

### Bootstrap resilience

- **Treat startup 502/503 as transient.** Retry bootstrap/session-list flows with bounded retries/intervals, especially in VS Code where API readiness can lag bridge startup.
- **Use polling recovery when failures are swallowed.** If an async loader resolves without throwing on failure, recover with interval retries gated by loaded-state checks.

## Regression-prevention checklist

- When adding fallback logic: can stale persisted data keep this path active forever?
- When deriving UI state: is this live state, historical state, or inferred state?
- When adding store fields: who reads this, how often does it change, and should it live elsewhere?
- When touching polling or bootstrap: can a lighter payload erase richer existing data?
- When handling optimistic updates: where is rollback, reconciliation, and duplicate prevention?
- When changing shared routes or state contracts: what breaks in web, desktop, and VS Code?
- When fixing a bug with a heuristic: prefer narrowing the heuristic over widening it.

## Validation expectations

- Run `bun run type-check` and `bun run lint` before finalizing.
- For hot-path changes: verify behavior under streaming or repeated events, not just static render.
- For sync or startup changes: verify fresh load, retry/failure, and restart behavior.
- For session changes: verify create, stream, abort, permission, archive/delete, and revisit flows when relevant.
