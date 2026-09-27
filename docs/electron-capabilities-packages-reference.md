# Electron Capabilities & Packages Reference

Popular **packages** and **implementation patterns** for common Electron product capabilities. Intended for **AI agents and developers** choosing libraries by feature. Pair with:

- [AGENTS.md](../AGENTS.md) — agent rules
- [electron-development-guide.md](./electron-development-guide.md) — how to wire them
- [electron-app-architecture-reference.md](./electron-app-architecture-reference.md) — where patterns show up in production apps

**Conventions**
- Prefer **built-in Electron APIs** when enough; add a package when it encodes painful OS differences or battle-tested flows.
- **Pin versions** in your app; versions below are illustrative only.
- **Native addons** (`.node`) must match your Electron ABI and usually need `asarUnpack`.
- Citations mean “seen in shipped apps / common stacks,” not endorsements.

---

## Quick matrix

| Capability | First choice | Common packages | Production cites |
|------------|--------------|-----------------|------------------|
| Auth (OAuth / PKCE) | System browser + loopback / deep link | `electron-native-auth`, app-specific PKCE helpers | Clay, Todoist, Slack |
| Auth (Sign in with Apple) | Native bridge | `node-mac-sign-in-with-apple` | Clay |
| Secrets at rest | `safeStorage` | `keytar` when storing shared creds | VS Code, Cursor, Claude; GH Desktop, Postman |
| Auto-update | Channel-dependent | `electron-updater`; Squirrel; Sparkle | LM Studio, Clay; Code lineage; ChatGPT-style |
| Notifications | `Notification` API | `macos-notification-state`, focus-assist helpers | Slack |
| Push (background) | Platform push bridge | push-receiver style packages | Clay |
| Tray / menu bar | `Tray` + `Menu` | template PNGs; optional menubar kits | Figma, LM Studio, Evernote |
| Window state | manual bounds | `electron-window-state` | Clay |
| Config / settings | — | `electron-store` | Clay, Claude, LM Studio |
| Logging | — | `electron-log` | LM Studio, Antigravity, Clay |
| Crash / errors | `crashReporter` | `@sentry/electron` | VS Code, Claude; Slack, Clay, Evernote |
| Local DB | — | `better-sqlite3` | Notion, Clay, Evernote |
| Terminals / agents | — | `node-pty` | Claude, Postman, VS Code ecosystem |
| MCP / tools | utilityProcess host | `@modelcontextprotocol/sdk`, chrome-devtools MCP | Claude, Antigravity, Postman |
| Shell PATH for agents | — | `shell-env` | Antigravity |
| Context menus | custom Menu | `electron-context-menu` | Clay |
| Title bar | `titleBarStyle` / traffic lights | `custom-electron-titlebar` | Clay |
| Deep links | `setAsDefaultProtocolClient` | — | Slack, Notion, Obsidian, Claude, … |
| Scheduling | — | `toad-scheduler` | Clay |
| Mac permissions | usage strings + session | `node-mac-permissions` | Clay |
| Contacts | — | `node-mac-contacts` | Clay |
| Prefs (native) | — | `cf-prefs`-style natives | Slack, Notion |
| File watching | — | `chokidar` (+ `fsevents` on Mac) | Figma |
| Packaging | Forge or builder | `@electron/fuses`, makers | Slack, Claude, LM Studio |

---

## 1. Authentication & identity

### Patterns

| Pattern | When to use | How |
|---------|-------------|-----|
| **System browser OAuth + PKCE** | SaaS login | Open `https://…/authorize` via `shell.openExternal` or `electron-native-auth`; complete via **loopback** `http://127.0.0.1:port` or **custom URL scheme**; exchange code in **main** |
| **Deep-link callback** | Desktop-installed clients | `myapp://auth/callback?code=…` → validate state/PKCE in main |
| **Sign in with Apple** | MAS / Apple-centric identity | Native module; keep tokens in `safeStorage` / keychain |
| **Cookie / webview session login** | Wrapping an existing web app | Isolated `session.fromPartition`; never share app preload with guest web |
| **Device / machine binding** | Licensing | Prefer OS keychain + server attestation; avoid home-rolled disk secrets |

### Packages & APIs

| Item | Role |
|------|------|
| Electron `safeStorage` | Encrypt tokens/secrets for the current OS user |
| `keytar` | Cross-platform keychain/credential store (native) |
| `electron-native-auth` | Desktop-oriented auth helpers (seen Clay / Slack stacks) |
| `node-mac-sign-in-with-apple` | Apple identity on macOS (Clay) |
| In-house PKCE helper module (Todoist-style) | Pattern: keep PKCE in a small main-process util, not the renderer |

### Do / don’t

- **Do** keep client secrets and refresh-token handling in main (or utilityProcess).
- **Do** validate `state` and redirect URI strictly.
- **Don’t** put refresh tokens in `localStorage` of a non-sandboxed or node-integrated window.
- **Don’t** enable `nodeIntegration` “so auth is easier.”

**Cites:** Clay (native auth + Apple + store); Todoist (PKCE utils); Slack (`electron-native-auth`); Grok Bot (password-manager connection service pattern); VS Code/Cursor/Claude (`safeStorage`).

---

## 2. Auto-update & distribution

### Choose a channel

| Channel | Mechanism | Typical hosting | Cites |
|---------|-----------|-----------------|-------|
| Direct download (classic Electron Mac) | **Squirrel.framework** (+ Mantle/ReactiveObjC) | Vendor update CDN | VS Code, Cursor, Devin, Notion, Figma, Claude, … |
| Cross-platform generic | **`electron-updater`** + `app-update.yml` | GitHub Releases, S3, R2, generic HTTP | LM Studio, Clay, Antigravity, Evernote, Notion Calendar |
| Native Mac updater UX | **Sparkle** | Appcast XML | ChatGPT-style hybrid shells |
| Mac App Store | **MAS** only | App Store | Slack, Todoist, Evernote |

### Packages

| Package | Notes |
|---------|-------|
| `electron-updater` | Works with electron-builder / many Forge setups; supports multiple providers |
| `@electron/fuses` | Lock down runtime flags at package time (Slack, Claude) |
| Electron Forge makers (`maker-dmg`, `maker-zip`, `maker-squirrel`, …) | Platform artifacts |
| electron-builder | Alternative packaging + publish pipeline |

### Patterns

1. **Staging channel** before prod (separate feed / quality flag — Code `product.json` quality mindset).
2. **Minimum supported version** gate when IPC/protocol breaks (Slack-style metadata).
3. **Silent download + notify** vs forced update UI (Evernote-style dedicated update windows).
4. MAS builds: **no** Sparkle/Squirrel private update path; use Store.

### Minimal electron-updater sketch

```ts
import { autoUpdater } from 'electron-updater';
import log from 'electron-log';

autoUpdater.logger = log;
autoUpdater.checkForUpdatesAndNotify();
```

Ship provider config (`app-update.yml` or publish config). **Cite:** LM Studio, Clay, Antigravity (generic HTTP / object-storage style hosting is common).

---

## 3. Notifications & push

### OS notifications (foreground / local)

| API / package | Role | Cite |
|---------------|------|------|
| Electron `Notification` | Local banners | Widespread |
| `Notification.isSupported()` / permission flows | Gate before spam | — |
| `macos-notification-state` | Respect macOS focus / DND-like state | Slack |
| `windows-focus-assist` | Windows Focus Assist awareness | Slack |
| Dock bounce / badge | `app.dock.bounce`, `app.setBadgeCount` | GH Desktop-style attention |

**Pattern:** Check notification policy **before** showing; provide in-app prefs to disable categories (messages vs marketing).

### Background push

| Approach | Notes | Cite |
|----------|-------|------|
| Push-receiver style packages | Bridge FCM/APNs-like delivery into Electron | CRM desktops (e.g. Clay) |
| MAS + app groups | Notification service extension / app group entitlements | ChatGPT-style entitlement categories |
| Polling + local `Notification` | Simpler; battery heavier | Small apps |

**Pattern:** Decrypt/verify push payloads in main; renderer only gets sanitized view models via IPC.

---

## 4. Tray, menu bar, dock

| Piece | Approach | Cite |
|-------|----------|------|
| Tray icon | `Tray` + **template** images (`*Template.png`) for macOS | Figma, LM Studio |
| Context menu | `Menu.buildFromTemplate` | Widespread |
| Status indicators | Swap tray images (idle / paused / unread) | LM Studio tray variants; Figma indicator assets |
| Hide-to-tray | `window.hide()` on close (Linux/Win prefs); macOS often keep dock | Evernote tray helper pattern |
| Dock menu | `app.dock.setMenu` | macOS productivity apps |

**Pattern:** Keep tray click handlers in main; use IPC to ask renderer for unread counts if needed.

---

## 5. Windowing & desktop UX

| Capability | Package / API | Cite |
|--------------|---------------|------|
| Remember size/position | `electron-window-state` | Clay |
| Frameless / custom titlebar | `titleBarStyle: 'hiddenInset'` or `custom-electron-titlebar` | Clay |
| Context menu on page | `electron-context-menu` | Clay |
| Multi-window | one factory + preload per role | Claude, Slack |
| Tabs / embedded web | `WebContentsView` / BrowserView | Claude, Cursor, Notion |
| Quick capture palette | separate small `BrowserWindow` | Claude quick window |
| Crash UI | dedicated `crash.html` window | GitHub Desktop |

---

## 6. Storage, config, secrets, database

| Need | Recommendation | Packages | Cite |
|------|----------------|----------|------|
| Non-secret settings | JSON config in userData | `electron-store` | Clay, Claude, LM Studio |
| Secrets | OS-backed encryption | `safeStorage`; optional `keytar` | VS Code, Claude; GH Desktop |
| Structured local data | SQLite in main/utility | `better-sqlite3` | Notion, Clay, Evernote |
| Large files | app `userData` / user-selected paths | `dialog.showOpenDialog` + bookmarks (MAS) | Slack MAS entitlements |
| Migrations | versioned SQL migrations beside DB | app code | Evernote/Notion-class |

**asarUnpack:** `better-sqlite3`, `keytar`, and any `.node`.

---

## 7. Logging, crash reporting, telemetry

| Layer | Package / API | Cite |
|-------|---------------|------|
| File + console logs | `electron-log` | LM Studio, Antigravity, Clay |
| Crash dumps | Electron `crashReporter` | VS Code, Claude |
| Error aggregation | `@sentry/electron` | Slack, Claude, Clay, Evernote, Raindrop |
| Privacy | gate telemetry; Code-style product flags | VS Code `enableTelemetry` mindset |

**Agent pattern:** Always enable durable file logs when running tools/MCP (Antigravity).

---

## 8. Networking, proxies, certificates

| Capability | Pattern | Cite |
|------------|---------|------|
| Corporate proxy | Honor `session.setProxy` / env; test with PAC | Postman proxy surfaces |
| Client certificates | `session` certificate events + dedicated UI | Postman `proxyAuth` HTML |
| Certificate errors | Fail closed; optional support override window | GH Desktop certificate error UX |
| Pinning / trust | Prefer OS trust store; document overrides | — |
| WebRequest mutation | `session.webRequest.onBeforeSendHeaders` | VS Code, Cursor, Claude |

---

## 9. Files, dialogs, watching, protocols

| Capability | API / package | Cite |
|------------|---------------|------|
| Open/save dialogs | `dialog` | Widespread |
| Reveal in folder | `shell.showItemInFolder` | GH Desktop |
| Open external | `shell.openExternal` (https allowlist) | Claude, VS Code |
| Watch vaults / extensions | `chokidar` (+ `fsevents`) | Figma local extension registry |
| Custom resource scheme | `protocol.registerSchemesAsPrivileged` | VS Code, Claude |
| Deep links | `setAsDefaultProtocolClient` | Slack, Notion, Obsidian, Figma, Claude |
| Bundle CLI tools | ship `git/` (or similar) in resources | GitHub Desktop |

---

## 10. Media, permissions, device access

| Capability | Approach | Cite |
|------------|----------|------|
| Camera / mic | Usage strings in Info.plist + `setPermissionRequestHandler` allowlist | Slack, Claude, ChatGPT, VS Code |
| Screen / computer use | Dedicated native addons + **separate** preload | Claude Swift/computer-use natives |
| Contacts | `node-mac-contacts` + permission module | Clay |
| Generic Mac TCC prompts | `node-mac-permissions` | Clay |
| Deny-by-default guest sessions | partition + permission handler → `false` | Claude embedded web |

---

## 11. Background work, agents, MCP, terminals

| Capability | Pattern | Packages | Cite |
|------------|---------|----------|------|
| CPU/IO off main | `utilityProcess.fork` + `MessageChannelMain` | built-in | VS Code, Cursor, Claude, LM Studio |
| Hidden DOM worker | Invisible `BrowserWindow` / HTML shell | — | Evernote conduit |
| Named process mesh | `*Worker.js` / `*Process.js` entries | app structure | Postman |
| PTY / integrated shell | native PTY | `node-pty` | Claude, Postman, VS Code |
| MCP tool host | utilityProcess + SDK | `@modelcontextprotocol/sdk` | Claude, Antigravity, Postman |
| Browser automation tools | MCP/devtools bridges | `chrome-devtools-mcp`-style | Antigravity |
| Realistic PATH | hydrate env before spawn | `shell-env` | Antigravity |
| Schedulers | interval jobs in main | `toad-scheduler` | Clay |
| Tree-sitter / code intel | prebuilds | `tree-sitter` grammars | Grok Bot |

**Rule:** Agent/tooling never runs with the UI preload’s privileges.

---

## 12. IPC, state, security hardening

| Capability | Package / pattern | Cite |
|------------|-------------------|------|
| Typed IPC | zod schemas + `invoke`/`handle` | Development guide §5 |
| IPC codegen | generate bridges from schema | Claude-style schema-generated IPC |
| IPC sender validation | check `event.senderFrame.url` in every handler | Development guide §5 |
| contextBridge | expose minimal API | Claude, modern apps |
| Redux across processes | avoid for new apps; explicit RPC | Slack historically `electron-redux` |
| CSP | bundler CSP plugins / `webRequest` headers | Slack CSP tooling; Claude enforcement |
| Fuses | `@electron/fuses` | Slack, Claude |
| Asar integrity | `ElectronAsarIntegrity` plist | Many modern Mac builds |

---

## 13. Packaging & native modules

| Concern | Package / setting | Cite |
|---------|-------------------|------|
| Scaffold | Electron Forge + Vite or Webpack | Claude; LM Studio/Notion |
| Universal Mac + arch natives | bootstrap asar + `app-arm64` / `app-x64` | Slack |
| Unpack natives | `asarUnpack: ['**/*.node']` | Figma, Notion, Claude |
| Notarize / sign | `@electron/notarize`, `@electron/osx-sign` | Forge/Slack tooling metadata |
| Prefs native | `cf-prefs` | Slack, Notion |
| Progress / UI native | first-party `.node` | Notion progress bar native |

---

## 14. Feature → starter shopping list

### Baseline utility (conventional shell)

`electron-log`, optional `electron-updater`, `electron-store`, zod.

### SaaS shell (Figma/Notion-like)

Above + deep links, tray assets, guest `WebContentsView` partition, optional `chokidar`.

### CRM (Clay-like)

`better-sqlite3`, `electron-window-state`, `electron-context-menu`, `electron-updater`, `electron-store`, `@sentry/electron`, `toad-scheduler`, Mac auth/contacts/permissions packages as needed, push-receiver if required.

### Multi-surface AI (Grok Bot / Claude-like)

Per-capability preloads, `utilityProcess`, `electron-log`, `@sentry/electron`, optional `node-pty`, MCP SDK, `safeStorage`.

### Agentic desktop (Antigravity-like)

`electron-updater`, `electron-log`, `shell-env`, MCP/devtools bridges, utilityProcess.

### IDE agent (Cursor/Devin-like)

Prefer VS Code OSS architecture; Squirrel updates; careful extension host isolation — don’t reinvent with random packages.

### MAS messaging (Slack-like)

Sandbox entitlements early; notification-state helpers; multi-preload; fuses; Sentry.

---

## 15. Anti-patterns

| Don’t | Prefer |
|-------|--------|
| `@electron/remote` | contextBridge + typed IPC |
| Tokens in renderer storage | `safeStorage` / keytar in main |
| One preload for UI + webview + VNC | Split preloads (Grok Bot) |
| Agents without file logs / PATH | `electron-log` + `shell-env` |
| Private updater inside MAS builds | Store-only updates |
| Unpacked secrets in asar | OS keychain / safeStorage |
| Unpinned “latest” natives | Pin + rebuild for Electron ABI |

---

## 16. How to extend this list

When you discover a new capability pattern via [`electron-macos-pattern-study`](../skills/electron-macos-pattern-study/SKILL.md):

1. Add a row to the **quick matrix**.
2. Document pattern + packages + cite (app name only).
3. Link an implementation note in the development guide if wiring is non-obvious.
4. Scrub paths, emails, team IDs before publishing.
