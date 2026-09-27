# Electron development checklist

Used by the `electron-app-development` skill with  
`docs/electron-development-guide.md` and `docs/electron-app-architecture-reference.md`.

---

## Archetype picker

| Building… | Archetype | Start like | First implementation focus |
|-----------|-----------|------------|------------------------------|
| Simple desktop utility | Baseline shell | Conventional utilities | 1 window, 1 preload, typed IPC |
| Hosted web product | Web SaaS shell | Figma, Notion | Shell window + tray/deep link; guest partition |
| Notes / vaults on disk | Local-first | Obsidian | Custom scheme + disk roots; no remote module |
| Chat / realtime | Messaging | Slack | Multi-preload; notification permissions plan |
| Editor / coding agent | IDE shell | VS Code, Cursor, Devin | Prefer Code lineage; Plugin helper mindset |
| Chat + webview + meetings/remote | Multi-surface AI | Grok Bot, Claude | **Separate preloads per capability** |
| MCP / computer-use agent | Agentic desktop | Antigravity, Claude | utilityProcess + shell-env + file logs |
| Sales / enrichment CRM | CRM / GTM | Clay | SQLite + window-state + OS auth bridges |
| Small calendar/SaaS companion | Light SaaS shell | Notion Calendar | `build/main` + one preload + updater |
| Mac App Store | MAS hybrid | Slack, Todoist, Evernote | Sandbox entitlements before natives |

---

## Secure defaults (every window)

- [ ] `contextIsolation: true`
- [ ] `nodeIntegration: false`
- [ ] `sandbox: true`
- [ ] `webviewTag: false` unless required (then dedicated preload)
- [ ] `setPermissionRequestHandler` deny-by-default (default session and every partition)
- [ ] Global `web-contents-created` guard: `will-navigate`, `setWindowOpenHandler`, `will-attach-webview`
- [ ] Production UI served from `app://` with a traversal-safe `protocol.handle`
- [ ] CSP on app-owned renderers
- [ ] Every `ipcMain.handle` validates the sender frame and re-parses input
- [ ] No `@electron/remote`

---

## Project structure (minimum)

- [ ] `src/main/` — lifecycle, windows, ipc, security, updates
- [ ] `src/preload/` — one file per surface
- [ ] `src/renderer/` — UI only
- [ ] `src/shared/` — IPC contracts (zod or equivalent)
- [ ] Pinned versions in `package.json`
- [ ] pnpm (SFW respected)

---

## Feature gates (add only when needed)

Pick packages from `docs/electron-capabilities-packages-reference.md`.

- [ ] Second window / quick capture → new preload
- [ ] Embedded web → `WebContentsView` + `persist:` partition
- [ ] Heavy jobs / agents → `utilityProcess`
- [ ] Auth (OAuth/PKCE / Apple) → main-process tokens + `safeStorage`/`keytar`
- [ ] Deep links → protocol registration + validation
- [ ] Notifications / push → `Notification` + policy helpers / push-receiver
- [ ] Secrets → `safeStorage` (or keytar if justified)
- [ ] SQLite / `.node` → `asarUnpack` + ABI match
- [ ] Tray / dock menu
- [ ] Auto-update channel chosen (Squirrel / electron-updater / Sparkle / MAS)
- [ ] electron-log (and optional Sentry)
- [ ] Electron fuses on release builds

---

## Release checklist

- [ ] Smoke: cold start, reload, quit, reopen (macOS activate)
- [ ] Single-instance lock behaves
- [ ] Deep link (if any) validated on happy + malicious URLs
- [ ] Guest content cannot call main filesystem APIs
- [ ] Updater works on staging
- [ ] Logs written in packaged app
- [ ] No unpinned deps introduced
- [ ] No vendor asar extracts in the repo
