# Recipes — mapped to the development guide

Short pointers for agents. The full, working versions live in `docs/electron-development-guide.md` — copy from there, not from memory.
Resolve versions from the registry (`npm view <pkg> version` or `pnpm add -E`); never invent them.

---

## Secure BrowserWindow

See guide §4. Paths assume the Forge Vite template (`.vite/build/`).

```ts
webPreferences: {
  preload: path.join(__dirname, 'preload.js'),
  contextIsolation: true,
  nodeIntegration: false,
  sandbox: true,
  webviewTag: false,
}
```

Production UI loads from `app://bundle/...`, not `loadFile`, so there is exactly one trusted origin.

---

## Navigation guard + CSP

See guide §4.

- `app.on('web-contents-created')` → `will-navigate` blocks non-app origins, `setWindowOpenHandler` denies and opens https externally, `will-attach-webview` is blocked.
- CSP meta tag on every app-owned renderer; no `'unsafe-eval'`.

---

## Typed IPC + preload

See guide §5.

1. Define channel + zod schemas in `src/shared/ipc.ts`
2. `ipcMain.handle` in main: **check `event.senderFrame.url` against app origins**, then re-parse input
3. `contextBridge.exposeInMainWorld('desktop', { … })` in preload
4. Renderer calls only `window.desktop.*`

Main-side validation is the security boundary; preload-side parsing is only ergonomics.

---

## `app://` scheme

See guide §7.

`registerSchemesAsPrivileged` before `ready`, `protocol.handle` after. The handler must resolve the path and reject anything where `path.relative(root, target)` escapes the root.

---

## Deny-by-default session

See guide §4.

```ts
session.defaultSession.setPermissionRequestHandler((_wc, _perm, cb) => cb(false));
```

Opt in per `session.fromPartition('persist:feature')` when a feature needs cam/mic/etc.

---

## Single-instance + deep link

See guide §§6–7.

- `app.requestSingleInstanceLock()`; put all other setup in the lock-acquired branch
- `registerSchemesAsPrivileged` and the `open-url` listener **before** `ready`
- `setAsDefaultProtocolClient`
- macOS: queue `open-url` until windows exist. Windows/Linux: read the URL from `second-instance` argv and from `process.argv` on cold start
- Route by an allowlist of hosts; treat all params as attacker-controlled

---

## Capability-separated preloads

See guide §5 (Grok Bot / Claude pattern).

| File | May expose |
|------|------------|
| `preload/main.ts` | App UI APIs |
| `preload/webview.ts` | Navigation messaging only |
| `preload/meeting.ts` | A/V-related bridges |
| `preload/remote.ts` | Remote-desktop bridges |

Never give webview preload filesystem or keychain APIs.

---

## UtilityProcess worker

See guide §9.

```ts
utilityProcess.fork(workerPath, [], { serviceName: 'agent-worker' });
// MessageChannelMain for RPC
```

Use for agents, parsers, MCP hosts — not for UI.

---

## asarUnpack natives

See guide §10.

Unpack `**/*.node` (and packages like `better-sqlite3`). Rebuild against the app’s Electron ABI.

---

## Updater choice

See guide §11 + architecture reference §10.

| Distribution | Mechanism |
|--------------|-----------|
| Direct download Mac (classic) | Squirrel lineage |
| Generic hosting | `electron-updater` + `app-update.yml` |
| Custom Mac UX / hybrid | Sparkle |
| App Store | MAS only |

---

## Archetype → first folders

| Archetype | Create first |
|-----------|--------------|
| Baseline | `main/index`, `windows/main-window`, `preload/main`, `shared/ipc` |
| Multi-surface AI | Above + `preload/webview`, `sessions/`, `workers/` |
| Agentic | Above + `workers/agent`, logging init, `shell-env` bootstrap |
| CRM | Above + `db/` migrations, window-state, optional Mac natives |
| IDE agent | Evaluate VS Code OSS fork before greenfield |
