# Rooms: Cloudroom remade inside zapfleet — design spec

**Status:** approved scope, pre-implementation
**Date:** 2026-11-05
**Decisions locked by owner:**
- Scope: **full rooms inside zapfleet** — one window, one identity. Not a launcher, not concepts-only.
- Reuse: **clean reimplement with a port bar** — rebuild the shell in zapfleet style, but lift Cloudroom's proven, tested, pure-logic packages with license headers intact. No naive rewrite of what already works; no blind copy of UI.
- Release: **unresolved** — `v0.2.0` is not tagged until the owner says so. Rooms work lands on `main` behind a feature flag so Studio can ship first if that call is made.

## 1. What a Room is

A **Room** is a zapfleet-native surface that hosts one or more **threads** (agent conversations with Codex / Claude Code / OpenCode / Pi) against one **project** (a folder on this machine), with terminals, diffs, and Studio reachable from the same pane.

Cloudroom's vocabulary maps onto zapfleet like this:

| Cloudroom | zapfleet Rooms | Notes |
|---|---|---|
| Project | Project (existing `projects` table) | Reuse. Rooms attach to the existing project model; no second project store. |
| Thread (Local) | Thread | New. One agent process, one event log, one timeline. |
| Thread (Cloud) | **Phase 3** | Needs the Rust `core/` API. Not in v1. |
| Environment / host | Machine = this PC | zapfleet's pty + settings layer already covers it. |
| Pending interaction | Approval | Agent asks → user approves/denies in the Room, or from a terminal. |
| Queued message | Queued message | Ported type; "steer vs queue" semantics preserved. |
| Timeline | Timeline | Ported projection; zapfleet-styled rows. |
| Sidebar | Room switcher | Rewritten. |

Studio becomes a tool inside a Room (promote an asset into the project folder → shows in the thread's diff), not a headline tab.

## 2. Port bar — what moves, what doesn't

Source: `github.com/davidondrej/cloudroom` (`gui/packages/*`), Apache-2.0, derived from BB (MIT). Every ported file keeps its header; `NOTICE` at repo root gains both attributions and the source commit SHA.

### 2.1 Port (lift, keep headers, keep tests)

| Package | Lands at | Why port |
|---|---|---|
| `agent-runtime` | `electron/rooms/runtime/` | Provider process lifecycle, turn state, recovery, interactive approvals, watchdogs. ~80 KB runtime, ~400 KB tests across Codex/Claude/ACP topologies. Rewriting this is where a year goes. |
| `thread-view` | `shared/rooms/thread-view/` | Event → timeline projection. Pure functions, zero UI deps. |
| `provider-bridge-protocol` | `shared/rooms/bridge/` | Thread-delta grammar, handshake, conformance tests. |
| `domain` (subset) | `shared/rooms/domain/` | `thread.ts`, `thread-events.ts`, `thread-lifecycle.ts`, `thread-status.ts`, `queued-message.ts`, `pending-interactions.ts`, `provider-event.ts`, `provider-types.ts`, `shared-types.ts`, `stored-thread-event.ts`, `reasoning-*`, `model-levels.ts`. **Dropped:** plugin-*, theme/code-theme, ui-preferences, keybindings, marketplace, experiments. |
| `db` (schema + migrate) | `database/migrations/004_rooms_*.sql` + `electron/rooms/db/` | Same driver (`better-sqlite3`). Ported as a *new* migration series appended to zapfleet's existing `MIGRATIONS`; zapfleet tables untouched. |

Port rule: a file is ported only if it (a) has tests upstream or is pure types, and (b) has no import from `client-core`, `shared-ui`, `bb-app`, or `plugin-*`. Anything failing (b) is adapted or rewritten.

### 2.2 Adapt (take the behaviour, write zapfleet code)

- `client-core/{thread,timeline,prompt,sidebar}` — view-model logic coupled to BB layout. Behaviour is re-expressed as zapfleet React components using existing primitives (`Palette`, toolbar, `styles.css` tokens: ink / signal / paper / 2px rules).
- `core-ui/pending-interaction-*` — approval formatting; adapt to zapfleet copy.

### 2.3 Skip (zapfleet already has it)

- `secret-storage` → zapfleet `safeStorage` vault.
- `host-daemon`, `host-workspace`, `host-watcher`, `process-utils` → zapfleet pty + fs layer.
- `desktop`, `server`, `connect`, `tunnel-*`, `vps` → not needed for Local threads.

### 2.4 Rewrite (the "remake")

- Room switcher, Room canvas (thread list + active timeline + composer), presence strip (which threads are idle / working / blocked), approvals inbox, Studio hook-in, shell chrome, shortcuts, Palette entries.

### 2.5 Out of v1

- Rust `core/`, Cloud threads, cloud VM, tunnels, plugin marketplace, computer-use, themes.

## 3. Architecture

```
renderer (src/)                       main (electron/)                    shared/
┌──────────────────────┐   IPC        ┌──────────────────────────┐        ┌──────────────────┐
│ Rooms/               │◄────────────►│ ipc/rooms.ts             │        │ rooms/domain     │ (ported)
│  RoomSwitcher        │ rooms:*      │  thread CRUD, send,      │        │ rooms/thread-view│ (ported)
│  RoomCanvas          │              │  approve/deny, cancel    │        │ rooms/bridge     │ (ported)
│   ThreadList         │  stream      ├──────────────────────────┤        │ rooms.ts (IPC    │
│   Timeline (adapted) │◄─────────────│ rooms/runtime/ (ported)  │        │  channel names)  │
│   Composer           │ rooms:event  │  provider processes      │        └──────────────────┘
│   Approvals          │              ├──────────────────────────┤
│  PresenceStrip       │              │ rooms/db/ (ported)       │
└──────────────────────┘              │  thread + event tables   │
                                      └──────────────────────────┘
```

- **Runtime** owns one provider process per running thread, emits `ThreadEvent`s, persists them via `rooms/db`, and forwards deltas to the renderer over `rooms:event`.
- **Renderer** keeps a per-thread event buffer and runs the ported `thread-view` projection to render the timeline; no projection logic lives in components.
- **Approvals** travel as `PendingInteraction` (ported type) → `rooms:approve` / `rooms:deny`.
- **Studio** promote target gains "into current Room's project" so promoted assets appear in the thread diff.

## 4. Data

New migration `004_rooms` (ported schema, trimmed):

- `rooms_threads` — id, project_id (FK → existing `projects`), title, provider, model, reasoning_level, status, parent_thread_id, created_at, archived_at.
- `rooms_thread_events` — id, thread_id, seq, kind, payload (JSON), created_at. Append-only.
- `rooms_queued_messages` — id, thread_id, body, mode (`steer` | `queue`), created_at, sent_at.
- `rooms_pending_interactions` — id, thread_id, kind, payload, resolved_at, resolution.

Retained-output trimming and compaction rules come with the ported `db` logic. All ids are zapfleet UUIDs; Cloudroom's `raw-thread-id` prefixing is dropped.

## 5. Surface

- **Entry:** Rooms is the default view when `rooms.enabled` is on (settings flag, default `false` until v1 ships). Palette entry `rooms`, shortcut `Ctrl+Shift+R`, toolbar `Rooms` toggle — same pattern as Studio.
- **Room switcher:** left rail; one row per project with thread count and presence dot (idle / working / blocked).
- **Canvas:** thread list → active timeline → composer. Composer supports `steer` (default while running) and `queue`.
- **Approvals inbox:** global badge; one-click approve/deny; also answerable from any thread-scoped terminal via a tiny `zapfleet-room` CLI shim (later phase, mirrors `room-cli thread interactions`).
- **Presence strip:** footer line, all running threads across Rooms with status glyphs.
- **Style:** zapfleet tokens only. No BB/Cloudroom CSS crosses over.

## 6. Providers (v1)

Codex CLI and Claude Code via the ported runtime adapters; OpenCode via the ACP bridge. Keys come from zapfleet's existing vault and are injected per process exactly as agent sessions do today. Install detection reuses `AGENT_PROFILES`.

## 7. Error handling

- Provider not installed / not logged in → thread shows a blocked card with the exact fix (`claude` → `/login`), never auto-installs.
- Process crash → ported recovery path restarts the turn once, then surfaces the error event in the timeline.
- Watchdog on turn start (ported) → blocked state with "stuck" affordance.
- DB write failure → event kept in memory ring, retried, surfaced as a banner if it persists.

## 8. Testing

- Ported packages bring their Vitest suites; they run under zapfleet's `node:test` harness via a thin shim (`tests/rooms/*.test.ts` importing ported modules). Target: all ported suites green before any UI lands.
- New: migration round-trip, IPC param validation, projection snapshot per provider fixture, approval flow, e2e Playwright: open Room → send prompt to a fake provider → see timeline → approve a tool call.

## 9. Build order (each phase ships green, flag-gated)

1. **Port foundation** — `domain` subset, `bridge`, `thread-view`, `db` migration. Tests green. No UI.
2. **Runtime** — port `agent-runtime`, wire `ipc/rooms.ts`, preload bridge, settings flag. Fake-provider e2e.
3. **Room shell** — switcher, canvas, timeline, composer, approvals, presence. Palette/shortcut/toolbar.
4. **Studio hook-in** — promote into Room project; Studio demoted to a Room tool.
5. **Terminal shim** — `zapfleet-room` CLI for approvals from a pane.
6. **Cloud threads** — against Rust core API. Separate spec.

## 10. Licensing

- `NOTICE` gains: Cloudroom (Apache-2.0, davidondrej/cloudroom @ commit SHA at port time) and BB (MIT, Michael Yong).
- Ported files keep their original header; adapted files add `// Adapted from cloudroom <path> @ <sha>: <reason>`.
- zapfleet stays MIT; Apache-2.0 §4 notices are satisfied by the above.

## 11. Open questions for the owner

1. Release: tag `v0.2.0` (Studio) first, or roll Rooms into it?
2. Default provider for a new thread: Codex, Claude Code, or "ask every time"?
3. Should Rooms replace the current terminal-first home screen when enabled, or sit beside it?
