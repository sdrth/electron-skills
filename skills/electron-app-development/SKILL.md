---
name: electron-app-development
description: >-
  Build and evolve Electron desktop apps using this repo's architecture
  reference, development guide, and capabilities/packages reference (auth,
  updates, notifications, secure defaults, typed IPC, archetypes, packaging).
  Use when scaffolding an Electron app, adding main/preload/renderer code,
  multi-window or utilityProcess work, deep links, auto-update, native modules,
  or when following AGENTS.md / production Electron patterns for AI agents.
---

# Electron app development

Build Electron apps by **following the repo docs**, not by improvising insecure defaults.

## Required reading (do this first)

Before writing code, read (or re-read the relevant sections of):

1. [docs/electron-development-guide.md](../../docs/electron-development-guide.md) — implementation steps
2. [docs/electron-app-architecture-reference.md](../../docs/electron-app-architecture-reference.md) — archetypes + citations
3. [docs/electron-capabilities-packages-reference.md](../../docs/electron-capabilities-packages-reference.md) — auth, updates, notifications, storage, agents, packages

If studying installed apps for more patterns, use sibling skill `electron-macos-pattern-study` (learnings only).

## Non-negotiables

Apply on every app window / WebContentsView unless the user explicitly overrides after a warning:

| Setting | Value |
|---------|-------|
| `contextIsolation` | `true` |
| `nodeIntegration` | `false` |
| `sandbox` | `true` |
| `@electron/remote` | **Do not add** |
| Privileged work | main or `utilityProcess`, never renderer |
| UI ↔ main | `contextBridge` + typed `ipcMain.handle` / `invoke` |
| IPC handlers | check `event.senderFrame.url` is an app origin, then re-parse input in main |
| Navigation | global `web-contents-created` guard; block `will-navigate` off-origin; deny `window.open`; block `<webview>` |
| Production origin | load UI from a privileged `app://` scheme with a path-traversal check, not `file:` |
| CSP | on every app-owned renderer; no `'unsafe-eval'` |
| Unknown https | `shell.openExternal`, not in-app navigation |
| Permissions | deny-by-default; opt-in per partition |
| Packaging | Electron fuses on (`RunAsNode` off, asar integrity on) |
| Dependencies | **pin exact versions resolved from the registry** (never invented); prefer **pnpm**; never bypass SFW/firewall |

Cite the archetype you chose (guide §2 / reference §3) in your plan before scaffolding.

## Workflow

```
Electron build:
- [ ] 1. Clarify product surface + pick archetype
- [ ] 2. Scaffold (Forge+Vite TS default) with pinned deps
- [ ] 3. Secure BrowserWindow / session defaults
- [ ] 4. Shared IPC contract + preload bridge
- [ ] 5. Lifecycle: single-instance, ready, activate
- [ ] 6. Add only needed surfaces (windows / views / workers)
- [ ] 7. Protocols / deep links if required
- [ ] 8. Persistence / secrets / natives (asarUnpack)
- [ ] 9. Logging + updater channel
- [ ] 10. Fuses / package; run release checklist
```

### 1. Archetype

Use [checklist.md](checklist.md) table. Examples:

- Baseline utility → single-window conventional shell
- Multi-surface AI → Grok Bot / Claude (one preload per capability)
- Agentic + MCP → Antigravity / Claude (`shell-env`, logs, utilityProcess)
- CRM → Clay (SQLite + Mac auth bridges)
- IDE/agent → prefer VS Code lineage (Cursor/Devin) over greenfield
- MAS → entitlements from day one (Slack/Todoist/Evernote)

### 2–5. Core shell

Implement per development guide §§3–6:

- `src/main`, `src/preload`, `src/renderer`, `src/shared`
- Zod (or equivalent) IPC schemas in `shared/`
- Preload exposes minimal `window.desktop` API only

### 6–8. Grow carefully

| Need | Pattern | Doc |
|------|---------|-----|
| Extra windows | Factory per role + matching preload | guide §8, ref multi-window |
| Embedded web | `WebContentsView` + separate partition | guide §8 |
| Heavy/agent work | `utilityProcess` + `MessageChannelMain` | guide §9 |
| Deep links | `setAsDefaultProtocolClient` + validate | guide §7 |
| `.node` addons | `asarUnpack`; match Electron ABI | guide §10 |
| Capability split | Separate preloads (webview/meeting/remote) | Grok Bot notes |

Do **not** collapse webview + filesystem + remote-desktop APIs into one preload.

### 9–10. Ship

- `electron-log` (and Sentry if appropriate)
- `electron-updater` or Squirrel/Sparkle/MAS per reference §10
- Electron fuses + asar integrity awareness
- Final checklist: guide §18 / [checklist.md](checklist.md)

## Stack preferences

- **pnpm** over npm; keep package firewall (SFW) — never suggest bypassing it
- TypeScript
- Electron Forge + Vite unless the archetype demands Code-lineage or existing webpack
- Pin all dependency versions

## Anti-patterns (refuse or rewrite)

1. `nodeIntegration: true` in product UI
2. Shipping `@electron/remote`
3. Business logic or secrets in the renderer
4. One mega-preload for unrelated high-risk surfaces
5. Agent/tool execution in the UI process
6. Unpinned “latest” Electron/deps in examples you add
7. Committing asar extracts or vendor source

## Output expectations

When implementing:

1. State the **archetype** and which reference apps inform it
2. Show folder layout before large file dumps
3. Prefer small, complete main/preload/shared slices over half-wired UI
4. Call out MAS/entitlement or native-ABI risks early

## Additional resources

- [checklist.md](checklist.md) — archetype picker + release checklist
- [recipes.md](recipes.md) — copy-ready snippets mapped to guide sections
- [docs/electron-development-guide.md](../../docs/electron-development-guide.md)
- [docs/electron-app-architecture-reference.md](../../docs/electron-app-architecture-reference.md)
- [docs/electron-capabilities-packages-reference.md](../../docs/electron-capabilities-packages-reference.md)
- Sibling: [electron-macos-pattern-study](../electron-macos-pattern-study/SKILL.md)
