# AGENTS.md — Electron development guide for AI agents

You are helping build or evolve an **Electron** desktop application. Follow this repository’s docs and skills. Prefer cited production patterns over ad-hoc “website in a BrowserWindow” setups.

## Read in order

1. This file (`AGENTS.md`)
2. [`docs/electron-development-guide.md`](docs/electron-development-guide.md) — implementation
3. [`docs/electron-app-architecture-reference.md`](docs/electron-app-architecture-reference.md) — archetypes
4. [`docs/electron-capabilities-packages-reference.md`](docs/electron-capabilities-packages-reference.md) — packages by feature

Load skill [`skills/electron-app-development/SKILL.md`](skills/electron-app-development/SKILL.md) when implementing.

Load skill [`skills/electron-macos-pattern-study/SKILL.md`](skills/electron-macos-pattern-study/SKILL.md) only when the user asks to study installed macOS apps for **patterns**.

## Hard rules

### Security (every UI window / WebContentsView)

- `contextIsolation: true`
- `nodeIntegration: false`
- `sandbox: true`
- Do **not** add `@electron/remote`
- Expose APIs only via `contextBridge` + typed `ipcMain.handle` / `invoke`
- In every IPC handler, verify `event.senderFrame.url` is an app origin and re-parse the input in main
- Install a global navigation guard (`app.on('web-contents-created')`): block off-origin `will-navigate`, deny `window.open`, block `<webview>` attachment
- Serve production UI from a privileged `app://` scheme with a path-traversal check, not `file:`
- Ship a CSP on app-owned renderers; never add `'unsafe-eval'`
- Deny session permissions by default; allow per partition/feature
- Open unknown `https:` URLs with `shell.openExternal`, not in a privileged window
- Enable Electron fuses on packaged builds

### Process placement

| Work | Where |
|------|--------|
| UI | Renderer |
| Bridge API | Preload (minimal) |
| Windows, protocols, secrets, natives, updater | Main |
| Heavy CPU/IO, agents, MCP hosts, parsers | `utilityProcess` (or dedicated worker process) |

### One preload per high-risk capability

Do not share one privileged preload across main UI + embedded webview + remote desktop + meetings. Split preloads (multi-surface AI pattern).

### Dependencies & tooling

- Prefer **pnpm**
- **Pin** exact dependency versions (no `^`, `~`, or `latest` in manifests you write)
- Resolve versions from the registry (`npm view <pkg> version`, or `pnpm add -E`); never write a version number from memory
- Never bypass package-manager firewall / SFW protections
- Unpack native `.node` modules (`asarUnpack`); rebuild for the app’s Electron ABI

### macOS app study (optional)

If inspecting `/Applications/*.app`:

- Document **patterns** (helpers, packaging shape, preload *names*, updater family)
- Do **not** bulk-extract asars, commit vendor source, bypass signatures/DRM, or publish team IDs, emails, home paths, or install fingerprints
- Scrub notes before adding them to any public repo

### Refuse

Reverse engineering for source theft, cracking, license bypass, or redistributing proprietary binaries/source.

## Workflow checklist

```
- [ ] Clarify product surface → pick archetype (architecture reference §3)
- [ ] Scaffold (Forge + Vite + TypeScript default unless archetype demands otherwise)
- [ ] Secure BrowserWindow + session defaults
- [ ] shared IPC contract + preload
- [ ] Lifecycle: single-instance lock, ready, activate
- [ ] Add windows / WebContentsView / utilityProcess only as needed
- [ ] Auth / updates / notifications / storage via capabilities reference
- [ ] Logging + updater channel
- [ ] Fuses / package; run development guide release checklist
```

## Output expectations

When implementing for a user:

1. State the **archetype** and which public apps inform it
2. Show intended folder layout before large code dumps
3. Prefer complete main/preload/shared slices over half-wired UI
4. Call out MAS entitlements, native ABI, and updater-channel risks early
5. Cite patterns as `AppName-style` / doc section—not pasted proprietary code

## Doc map

| Need | Doc |
|------|-----|
| How to code it | `docs/electron-development-guide.md` |
| Why / who does this in production | `docs/electron-app-architecture-reference.md` |
| Which package for auth/updates/etc. | `docs/electron-capabilities-packages-reference.md` |
| Agent build skill | `skills/electron-app-development/` |
| Optional macOS layout study | `skills/electron-macos-pattern-study/` |
