# Electron App Architecture Reference

**Audience:** AI agents and developers building Electron apps.  
**Source:** Patterns from **publicly distributed** Electron desktop applications (macOS bundle layouts and common packaging shapes). Electron majors drift; treat as illustrative.  
**Method:** Bundle layout (`Contents/`), `Info.plist` / entitlement *categories*, Helper sets, public `package.json` / `product.json` fields, asar entry names / TOC, and high-level API-presence signals—not proprietary source dumps.  
**Scope:** Architecture patterns for production Electron apps. Not a vendor security audit.

**How to use this doc**
1. Skim §1–3 to pick an **archetype**.
2. Implement with [electron-development-guide.md](./electron-development-guide.md) (or [`electron-app-development`](../skills/electron-app-development/SKILL.md)).
3. Pick packages from [electron-capabilities-packages-reference.md](./electron-capabilities-packages-reference.md).
4. Optional layout study: [`electron-macos-pattern-study`](../skills/electron-macos-pattern-study/SKILL.md) (learnings only—not reverse engineering).
5. Agents: follow [`AGENTS.md`](../AGENTS.md).

**Disclaimer:** Product names are trademarks of their respective owners. Citations describe *patterns* only—not permission to copy proprietary code.

---

## 1. Inventory (representative public Electron apps)

Illustrative packaging shapes and Electron **major** lines. Not a personal install census; omit on-disk sizes.

| App | Electron (approx. major) | Packaging shape | Notes |
|-----|--------------------------|-----------------|-------|
| Slack | 44.x | Multi-arch asar + MAS | Mac App Store; sandboxed |
| Notion | 43.x | Webpack asar + many renderers | Tabs / BrowserView-style surfaces |
| Cursor | 42.x | VS Code–style `Resources/app/` | Code fork; Squirrel updaters |
| Devin | 42.x | VS Code–style `Resources/app/` | Agent IDE; Plugin helper lineage |
| Grok Bot | 42.x | `dist/electron-main` + role preloads | Meetings / webview / VNC surfaces |
| Todoist | 42.x | Webpack asar + native Swift frameworks | MAS + widget/extension bridges |
| Antigravity | 41.x | `dist/main.js` asar | Agentic desktop; MCP + updater/log |
| Notion Calendar | 41.x | `build/main` + preload bundle | Calendar shell; `electron-updater` |
| Claude | 40.x | Electron Forge + Vite (`.vite/`) | Multi-window; MCP-oriented natives |
| Obsidian | 39.x | Thin asar (`main.js` + natives) | Local-first; may still ship `@electron/remote` |
| GitHub Desktop | 38.x | Unpacked `app/` | Bundled `git/`, keytar, trampoline |
| LM Studio | 38.x | Electron Forge webpack (`.webpack/`) | Tray; `electron-updater` |
| Postman | 37.x | Fat asar + many workers | Agent / execution / proxy processes |
| Evernote | 37.x | Large asar (multi-shell HTML entries) | MAS; background conduit worker UI |
| VS Code | 34.x | `Resources/app/out` + `node_modules.asar` | Extension host Helper (Plugin) |
| Figma | 30.x | Shell + Rust `.node` + tray | Hosted web UI in a desktop shell |
| Clay | 25.x | Classic asar | CRM desktop; SQLite + Mac OS bridges |
| Raindrop.io | 19.x | `src/` + `webapp/` | Older Electron major; Sentry |
| ChatGPT | Electron-family (custom framework label) | `app.asar` + Sparkle | Hybrid shell; may not use stock framework name |

**Baseline conventional shells** (single window, stock Helpers, standard asar) are common in small utilities—start there before IDE/MCP complexity.

Non-Electron stacks often confused with Electron: Spotify (CEF), WhatsApp / Telegram / Zoom / Chrome / Brave (other stacks).

---

## 2. System design map (what Electron actually is)

Mature Electron apps follow the same Chromium process graph, renamed per product:

```
┌─────────────────────────────────────────────────────────────┐
│ Main process (app executable)                               │
│  - app lifecycle, BrowserWindow / WebContentsView            │
│  - ipcMain, protocol, sessions, auto-update, native addons   │
└───────────────┬─────────────────────────────┬───────────────┘
                │                             │
    ┌───────────▼──────────┐      ┌───────────▼──────────────┐
    │ Helper (Renderer)    │      │ Helper (GPU)             │
    │  Chromium renderers  │      │  compositing / WebGL     │
    └───────────┬──────────┘      └──────────────────────────┘
                │
    ┌───────────▼──────────┐      ┌──────────────────────────┐
    │ Preload (isolated)   │─────▶│ Optional: Helper (Plugin)│
    │ contextBridge API    │      │  IDE / extension hosts   │
    └──────────────────────┘      └──────────────────────────┘
```

**Seen widely:** `* Helper.app`, `* Helper (GPU).app`, `* Helper (Renderer).app`, `* Helper (Plugin).app` plus `Electron Framework.framework` — Slack, VS Code, Cursor, Devin, Notion, Obsidian, Figma, Postman, Claude, GitHub Desktop, Todoist, LM Studio, Evernote, Antigravity, Grok Bot, Clay, and similar.

**VS Code lineage:** Plugin helper used as extension host / Node utility plane — VS Code, Cursor, Devin, and other agent/IDE forks that keep the Code helper set.

**ChatGPT note:** Ships an Electron-family runtime under a custom framework name plus Sparkle for updates. Treat as Electron-like packaging with a custom macOS integration layer.

---

## 3. Architectural archetypes

### A. Editor / IDE shell (multi-process workbench)

**Examples:** Visual Studio Code, Cursor, Devin.

**Shape:**
- App code lives in `Contents/Resources/app/` (not only a single opaque `app.asar`).
- Entry: `package.json` → `./out/main.js` (Code/Cursor) or equivalent workbench main.
- `product.json` (or fork equivalent) configures branding, `dataFolderName`, update endpoints, marketplace.
- Helpers: Main / GPU / Renderer / **Plugin**.
- Updater stack often includes `Squirrel.framework` (+ Mantle / ReactiveObjC on many Mac builds).
- Agent IDEs (Devin) keep the same helper topology as Code while growing disk footprint for agent tooling/assets.

**Patterns to copy:**
1. **Product config file** separate from code (`product.json`) — VS Code / Cursor / Devin-class forks.
2. **Privileged custom schemes** via `protocol.registerSchemesAsPrivileged` (e.g. webview / file-resource schemes) — VS Code / Cursor.
3. **UtilityProcess + MessageChannelMain** for isolated Node work — VS Code / Cursor / Claude / LM Studio.
4. **Session permission gates** (`setPermissionRequestHandler`) and `webRequest` header rewriting — VS Code / Cursor / Claude.
5. **safeStorage** for secrets at rest — VS Code / Cursor / Claude.
6. Secure defaults for secondary surfaces: partitioned views with `contextIsolation: true`, `nodeIntegration: false`, `sandbox: true` — Cursor.
7. **Broad file-type / URL ownership** when the product is an editor shell — Cursor / Devin-style plists register many document types.

### B. Messaging / realtime desktop client

**Example:** Slack.

**Shape:**
- **Universal Mac build with arch-specific asars:** a thin bootstrap `app.asar` selects `app-arm64.asar` or `app-x64.asar` at runtime (via app-path switching).
- Bundled main/boot entrypoints under `dist/`; multiple role-specific preloads (main UI, child windows, login flows).
- Build toolchain signals: Rspack-style bundling, Yarn modern releases, Electron Forge packaging, `@electron/fuses`, Sentry.
- State libraries historically include Redux-family stacks on the desktop shell.
- Platform natives: notification / focus helpers, prefs bridges, native auth helpers.
- **Mac App Store:** `_MASReceipt` + App Sandbox entitlements (camera/mic/downloads/network as needed).

**Patterns to copy:**
1. Thin bootstrap asar + arch payload (universal Mac apps with native modules).
2. Separate preload entry points per window/role.
3. Desktop-only native helpers for OS notification/focus policy.
4. CSP enforcement wired through the bundler where possible.

### C. Web-app shell (desktop wrapper around a hosted UI)

**Examples:** Notion, Figma, Raindrop (`webapp/`).

**Notion-like:**
- Webpack/Forge layout with **named renderer packages** (tabs, popups, sqlite helper views, etc.), each with its own preload.
- Locals: SQLite (`better-sqlite3`), first-party desktop-native addons, prefs bridges.

**Figma-like:**
- Thin desktop shell (`main`, shell CSS/JS, separate web vs shell binding scripts).
- Performance-critical natives (e.g. Rust `.node`) living in `app.asar.unpacked`.
- First-class **Tray** assets (template + indicator variants).
- File watching for local extension/plugin folders.

**Patterns to copy:**
1. Desktop shell owns windowing/tray/deep links; UI can stay a web bundle or remote origin.
2. Split “shell bindings” vs “web app bindings” preloads (Figma-style).
3. Unpack only `.node` binaries (`asarUnpack`) — Figma, Notion, Claude, Obsidian.

### D. Local-first content app

**Example:** Obsidian.

**Shape:** Minimal asar — main entry + small natives. Documents/vaults live on disk; custom URL scheme for deep links.

**Caution:** Some local-first apps still vendor `@electron/remote`. Prefer `contextBridge` + explicit IPC in new apps (as Claude / Cursor do).

### E. API / agent / automation workstation

**Example:** Postman.

**Shape:** Main process plus **many side processes** co-located in the asar:
- Agent / execution / proxy worker entrypoints
- Dataset / analytics helpers (e.g. DuckDB-style services)
- MCP-oriented control-plane workers in newer builds
- Multiple preloads for desktop, payments, visualizers, migrations
- Sandboxed tester HTML surfaces
- Natives such as keytar / PTY; some builds still include `@electron/remote`

**Patterns to copy:**
1. Treat execution, proxy, and agent work as separate processes (crash isolation + permissions).
2. Dedicated preload per privileged UI surface.
3. Local query engines beside the UI when datasets grow.

### F. AI desktop companion

**Examples:** Claude, ChatGPT, LM Studio, Grok Bot, Antigravity.

**Claude-like (strong modern reference):**
- **Electron Forge + Vite**; multi-window renderers (main, quick capture, about, find-in-page, teach flows).
- Security defaults observed: `nodeIntegration: false`, `contextIsolation: true`, `sandbox: true`; `contextBridge.exposeInMainWorld` in preloads.
- `WebContentsView` for embedded web content beside a native shell window.
- `session.fromPartition` + **deny-by-default** permission handlers.
- Custom privileged schemes + `webRequest` enforcement.
- `utilityProcess.fork` for background services; `MessageChannelMain` for ports.
- Domain natives (Swift/Node addons), optional `node-pty`, MCP/agent SDKs, typed IPC codegen.
- Observability: Sentry / crashReporter.
- Custom URL scheme registration via `setAsDefaultProtocolClient`.

**Grok Bot-like (multi-surface AI client):**
- Clear split: `dist/electron-main/*` vs `dist/electron-preload/*`.
- **Role-specific preloads** for high-risk surfaces: webview, meeting, VNC/remote desktop, desktop mount panel — each bridge is narrower than a single mega-preload.
- Main process modularized (`main`, `main-app`, `main-core`, meetings, import workers, password-manager connection services).
- Native addons for parsing/process introspection (e.g. tree-sitter family, process-list helpers) living beside the Electron shell.

**Antigravity-like (agentic desktop):**
- Thin `dist/main.js` entry with `electron-updater` + `electron-log`.
- Pulls in **MCP / DevTools-oriented** agent tooling (`chrome-devtools-mcp`-style deps) and `shell-env` so the agent sees a realistic user shell PATH — not only Electron’s sanitized env.
- Pattern: agent runtimes need explicit environment bootstrapping and noisy, durable file logs.

**LM Studio-like:**
- Forge webpack main under `.webpack/`.
- Tray icons; `app-update.yml` feeding `electron-updater` (object-storage backends are common).
- UtilityProcess / MessageChannelMain / CSP / electron-log / electron-store signals.

**ChatGPT-like:**
- Hybrid Electron-family asar + **Sparkle** updates + custom native framework + dock integrations.
- Rich entitlements (JIT, camera/mic, Apple Events, keychain / app groups as product features require).

### G. CRM / GTM productivity desktop

**Example:** Clay.

**Shape:**
- Classic asar main; local **SQLite** (`better-sqlite3`) for offline/workspace data.
- Desktop UX kits: `electron-window-state`, `electron-context-menu`, `custom-electron-titlebar`.
- Updates via `electron-updater`; config via `electron-store`; scheduling via `toad-scheduler`.
- Observability: `@sentry/electron`.
- Push / realtime: Superhuman-style push-receiver packages.
- Mac OS bridges: `electron-native-auth`, Sign in with Apple, optional Contacts / Permissions natives.

**Patterns to copy:**
1. Treat CRM desktops as **local database + sync** apps, not pure web wraps.
2. Prefer small, purpose-built Mac natives for auth/contacts/permissions over bespoke Objective-C from scratch.
3. Persist window geometry and use a scheduler in main for reminders/sync ticks.
4. Plan an Electron major upgrade path early — Clay-class apps can lag several majors behind messaging/IDE peers.

### H. Calendar / lightweight SaaS shell

**Example:** Notion Calendar.

**Shape:** Compiled `build/main` + single `preload-bundle`; `electron-updater` YAML in extra resources; signing metadata / trusted-publisher manifests beside the app; pnpm-workspace traces in packaging.

**Pattern:** Smaller surface area than full Notion — one main + one preload bundle, still first-class auto-update and code-signing hygiene.

### I. Conventional desktop shell

**Example:** Typical small Electron utilities (single primary window).

**Shape:** Stock Helper set, `app.asar` + `app.asar.unpacked`, current Electron majors. Useful as a baseline: if your app does not need IDE plugins, MCP, or MAS widgets, this topology is enough.

**Pattern:** Do not over-structure; adopt multi-preload / UtilityProcess only when a surface earns it (Grok Bot / Postman / Claude).

### J. Native-feeling productivity with OS extensions

**Example:** Todoist (MAS).

**Shape:** Electron Helpers **plus** first-party Apple frameworks (widgets, share extensions, StoreKit, etc.) and a Mac extensions bridge package. Webpack-bundled asar. PKCE-style auth helpers in desktop utility packages.

**Pattern:** Electron for cross-platform UI; Swift/ObjC frameworks for widgets, extensions, IAP.

### K. Notes suite with background conduit

**Example:** Evernote (MAS).

**Shape:** Multiple shell HTML/JS entrypoints (main, tray helper, conduit worker, update flows) plus conduit packages, SQLite, Sentry, `electron-updater`.

**Pattern:** Dedicated hidden/worker windows for sync pipelines; separate tray helper document.

---

## 4. Packaging & distribution patterns

| Pattern | Where seen | When to use |
|---------|------------|-------------|
| Single `app.asar` + `app.asar.unpacked` for `.node` | Notion, Claude, Figma, Obsidian, Postman, Clay, Antigravity, Grok Bot, and many utilities | Default shipping format |
| Unpacked `Resources/app/` tree | VS Code, Cursor, Devin, GitHub Desktop, LM Studio | Large trees, IDE/agent-style |
| Arch-specific asars + bootstrap | Slack (`app-arm64.asar` / `app-x64.asar`) | Universal app with arch-native addons |
| Extra asars / asset packs | ChatGPT-style secondary asars; Slack media assets | Feature modules / assets |
| `ElectronAsarIntegrity` in Info.plist | Slack, VS Code, LM Studio, ChatGPT, … | Tamper evidence for asar |
| `@electron/fuses` | Slack, Claude packaging metadata | Lock down risky Electron runtime flags |
| Squirrel.framework | Most non-MAS Electron Mac apps sampled | Direct-download Mac auto-update |
| `electron-updater` + `app-update.yml` | LM Studio, Evernote, Clay, Antigravity, Notion Calendar | Generic update channel (GitHub/S3/R2/etc.) |
| Sparkle.framework | ChatGPT-style | Native Mac updater UX |
| Mac App Store + App Sandbox | Slack, Todoist, Evernote | MAS distribution; expect entitlement friction |

**GitHub Desktop extras:** ships a full `git/` runtime, trampoline helpers, keytar, and a dedicated crash window — pattern: bundle CLI tooling the UI depends on rather than hoping `PATH` is correct.

---

## 5. Process model & windowing

### 5.1 Chromium helpers (baseline)

Always plan for GPU + Renderer helpers; enable Plugin helper if you run extension hosts or Node child semantics like VS Code.

### 5.2 Multi-window product surfaces

| Surface | Seen in |
|---------|---------|
| Main + quick capture / palette window | Claude |
| About / find-in-page auxiliary | Claude |
| Tabs / BrowserView / WebContentsView hosts | Notion; Cursor; Claude |
| Meetings / A/V companion surfaces | Grok Bot (meeting preload) |
| Remote desktop / VNC bridge surfaces | Grok Bot (VNC preload) |
| Webview / mount-panel bridges | Grok Bot |
| Tray-only / tray helper document | Figma; Evernote; LM Studio |
| Crash reporter window | GitHub Desktop |
| Auth / proxy interstitial HTML | Postman; Slack login/auth views |
| Update force/check UI | Evernote |

### 5.3 Side processes (beyond BrowserWindow)

| Mechanism | Seen in | Role |
|-----------|---------|------|
| `utilityProcess.fork` | VS Code, Cursor, Claude, LM Studio | Isolated Node services |
| Named `*Process.js` / `*Worker.js` in asar | Postman; Grok Bot import workers | Agent, execution, proxy, MCP, chrome import |
| Hidden conduit worker window | Evernote | Sync/data plane |
| Dedicated sqlite / shell workers | Claude | DB / env work off main |
| Bundled `git` + trampoline | GitHub Desktop | Privileged CLI bridge |
| Password-manager / secret bridge services | Grok Bot | Out-of-band credential connections |
| Main-process schedulers | Clay (`toad-scheduler`-style) | Reminders / sync ticks |

**Design rule:** Main process stays thin — windowing, IPC routing, lifecycle. CPU/IO-heavy or crash-prone work goes to utility/worker processes (Postman, Claude, VS Code).

---

## 6. Security architecture (what production apps actually set)

### 6.1 Renderer hardening ladder

| Level | webPreferences (observed) | Examples |
|-------|---------------------------|----------|
| Modern default | `contextIsolation: true`, `nodeIntegration: false`, `sandbox: true` + `contextBridge` | Claude; Cursor partitioned views |
| Mixed / legacy | `nodeIntegration: true`, `contextIsolation: false` | GitHub Desktop main window (historical pattern) |
| Legacy remote module still shipped | `@electron/remote` present | Obsidian, Postman (as observed) |

**Recommendation for new apps:** Follow Claude/Cursor-style defaults — never enable `nodeIntegration` in app UI; expose a minimal API via `contextBridge`; keep privileged operations in main/utility processes.

### 6.2 Session & navigation policy

Observed repeatedly:
- `setPermissionRequestHandler` (VS Code, Cursor, Claude, LM Studio) — embedded web sessions often deny by default.
- `shell.openExternal` for http(s) egress instead of unconstrained in-app navigation.
- `webRequest` hooks for auth, CSP, or scheme enforcement.
- `registerSchemesAsPrivileged` for app-owned schemes.

### 6.3 Secrets & IPC

- **safeStorage** — VS Code, Cursor, Claude.
- **keytar** — GitHub Desktop, Postman, VS Code-related password-store flows.
- **Typed / schema-first IPC** — Claude-style codegen — prefer contracts over ad-hoc channel strings.
- **electron-store** — Claude, LM Studio, Clay, Antigravity-class stacks (encrypt sensitive keys).

### 6.4 macOS entitlements (recurring)

Chromium-based apps commonly request:
- `com.apple.security.cs.allow-jit` (V8)
- Camera / microphone usage strings when calls or voice matter

MAS apps add App Sandbox + download/user-selected file entitlements. Non-MAS AI companions may keep sandbox off while still declaring network/file/device/keychain entitlements.

### 6.5 Integrity & lockdown

- `ElectronAsarIntegrity` plist key — widespread.
- Electron fuses — Slack / Claude packaging metadata.
- Preload-only bridges — Claude-style `exposeInMainWorld`.

---

## 7. IPC & preload patterns

1. **One preload per window role** — Slack, Notion, Postman, Claude, **Grok Bot** (webview / meeting / VNC / mount-panel).
2. **contextBridge namespaces** — expose focused objects rather than a giant `window.api`.
3. **Main handles + invoke** — GitHub Desktop / VS Code structured IPC modules.
4. **Message ports** — `MessageChannelMain` paired with `utilityProcess`.
5. **Cross-process Redux** — older messaging clients; newer apps prefer explicit RPC.
6. **Modular main entrypoints** — Grok Bot-style `main` / `main-app` / `main-core` / feature mains instead of one 10k-line file.

---

## 8. Native capability matrix (what people leave JS for)

| Capability | Typical approach | Seen in |
|------------|------------------|---------|
| SQLite | `better-sqlite3` | Notion, Evernote, Clay; Claude workers |
| PTY / terminals | `node-pty` | Claude, Postman, VS Code ecosystem |
| Keychain secrets | `keytar` | GitHub Desktop, Postman |
| Rust perf / engine | custom `.node` | Figma |
| Swift / OS integration | custom `.node` / frameworks | Claude, Todoist |
| macOS prefs | `cf-prefs`-style addons | Slack, Notion |
| Notifications policy | OS notification-state helpers | Slack |
| Widgets / extensions | first-party Apple frameworks | Todoist |
| Git | Bundled `git/` + trampoline | GitHub Desktop |
| Local LLM runtime | app-specific binaries beside Electron | LM Studio |
| Contacts / Apple auth | `node-mac-contacts`, Sign in with Apple, `electron-native-auth` | Clay |
| macOS permission prompts | `node-mac-permissions`-style | Clay |
| Push receive | push-receiver packages | Clay |
| Tree-sitter / code intel natives | prebuilt `.node` grammars | Grok Bot |
| Process introspection | custom process-list natives | Grok Bot |
| Agent MCP / DevTools bridges | MCP SDK + chrome-devtools MCP packages | Antigravity, Claude, Postman |
| Realistic shell PATH for agents | `shell-env`-style bootstrap | Antigravity |

**Rule:** Put `.node` files in `asar.unpacked`. Rebuild natives against the exact Electron ABI.

---

## 9. Deep links, protocols, file associations

| Scheme / pattern | App |
|------------------|-----|
| `slack://` | Slack |
| `notion://` | Notion |
| `obsidian://` | Obsidian |
| `figma://` | Figma |
| `claude://` | Claude |
| Product-specific + editor URL protocols | VS Code / Cursor |
| GitHub client URL types | GitHub Desktop |
| Extensive file-type ownership | Editor / agent shells (VS Code, Cursor, Devin) |

Implement with: `setAsDefaultProtocolClient`, `open-url` / second-instance argv parsing, and validation before navigating any renderer.

---

## 10. Auto-update & crash/observability

| Approach | Apps |
|----------|------|
| Squirrel (framework present) | VS Code, Cursor, Devin, Notion, Obsidian, Figma, Claude, GitHub Desktop, LM Studio, Postman, … |
| electron-updater + YAML | LM Studio, Evernote, Clay, Antigravity, Notion Calendar |
| Sparkle | ChatGPT-style |
| Sentry | Slack, Claude, Evernote, Raindrop, Clay, … |
| electron-log | LM Studio, Antigravity, Clay-class stacks |
| Dedicated crash window | GitHub Desktop |
| Electron `crashReporter` | VS Code, Claude, LM Studio |

Also consider **minimum supported desktop version** gates (Slack-style) when protocol/API compatibility breaks.

---

## 11. Build toolchain cheat sheet

| Toolchain | Apps |
|-----------|------|
| Electron Forge + Vite | Claude |
| Electron Forge + Webpack | LM Studio, Notion |
| Rspack-style bundling | Slack |
| Classic Webpack tree in asar | Todoist, Evernote |
| VS Code custom → `out/` / `Resources/app` | VS Code, Cursor, Devin |
| `dist/electron-main` + `dist/electron-preload` | Grok Bot |
| Thin `dist/main.js` + updater deps | Antigravity |
| `build/main` + preload bundle | Notion Calendar |
| electron-builder / electron-updater signals | Clay, Antigravity, LM Studio, Evernote, many |

---

## 12. Recommended reference architecture for a new Electron app

Synthesized from the strongest modern signals (Claude + Cursor/Devin + Slack packaging + Figma natives + Grok Bot preload split + Clay local data):

```
app/
  package.json              # main → out/main/index.js
  electron.vite|forge config
  src/
    main/
      index.ts              # lifecycle, single-instance lock
      windows/              # one factory per window role
      ipc/                  # schema-generated handlers
      protocols/            # privileged schemes
      sessions/             # partition + permission policy
      updates/              # squirrel or electron-updater
      security/             # fuses, CSP, navigation allowlists
      workers/              # utilityProcess entrypoints
    preload/
      main.ts               # contextBridge API only
      quick.ts
    renderer/
      main-app/
      quick-capture/
    shared/
      ipc-contract.ts       # zod / json-schema
  natives/                  # optional .node → asarUnpack
```

**Non-negotiables (aligned with Claude / Cursor):**
1. `sandbox: true`, `contextIsolation: true`, `nodeIntegration: false` on all app windows.
2. No `@electron/remote`.
3. Deny-by-default permissions; `openExternal` for unknown URLs.
4. Global navigation guard, CSP, and production UI served from a traversal-safe `app://` origin.
5. One preload per surface; typed IPC; every handler validates the sender frame and re-parses input.
6. Unpack natives; fuse production builds; enable asar integrity.
7. Heavy work in `utilityProcess`, not `BrowserWindow` hacks — unless you need DOM (then Evernote-style hidden window).

Working code for each item is in the [development guide](./electron-development-guide.md) §4–§7 and §12.

**Pick an archetype early:**
- Hosted SaaS UI → Figma/Notion shell pattern (or Notion Calendar for smaller surfaces).
- Local documents → Obsidian-like thin main + disk vaults (but modern security).
- IDE/agent → VS Code / Cursor / Devin helper/plugin + UtilityProcess model.
- Multi-surface AI client → Grok Bot-style per-role preloads (never one preload for webview + VNC + meetings).
- Agentic desktop with MCP → Antigravity-style env bootstrap + durable logs + updater.
- CRM / GTM → Clay-style SQLite + Mac auth/contacts natives + window state.
- Baseline product → conventional single-window asar shell; grow complexity only when earned.
- MAS → study Slack/Todoist entitlements before adding natives.

---

## 13. Per-app field notes (pattern anchors)

### Slack
- Arch-specific asars behind a bootstrap; MAS sandbox; multiple preloads; desktop notification natives; Redux-family desktop state; Forge/fuses/Sentry packaging signals.

### Visual Studio Code
- `Resources/app/out/main.js`, `product.json`, `node_modules.asar`.
- Privileged webview schemes; UtilityProcess; safeStorage; Squirrel updates.

### Cursor
- Same skeleton as Code with distinct product branding/data folder.
- Secure webPreferences on partitioned contents; WebContentsView; Tray.

### Devin
- VS Code helper topology (including Plugin helper) applied to an agent IDE.
- Broad document-type registration like other editor shells; expect a larger resources tree than a thin SaaS shell.
- Learning: fork Code when you need extensions/terminals/file graphs; do not re-implement the workbench.

### Notion
- Multi-renderer webpack graph; sqlite + desktop-native; tab/window preloads.

### Notion Calendar
- Smaller sibling shell: `build/main` + one preload bundle; first-class `electron-updater` and signing metadata.
- Learning: product siblings can share brand without sharing Notion’s full multi-renderer complexity.

### Obsidian
- Minimal main asar; custom URL scheme; local vaults; may still include `@electron/remote`.

### Figma
- Shell/web binding split; Rust native module; tray; unpacked `.node` files.

### Postman
- Process mesh (agent/execution/proxy/MCP/analytics); many preloads; tester sandbox HTML.

### Claude
- Forge+Vite; multi-window; WebContentsView; sandbox+contextIsolation; utilityProcess; Swift/OS natives; MCP; typed IPC; Sentry; custom URL scheme.

### Grok Bot
- Modular electron-main vs electron-preload trees.
- Separate preloads for webview, meetings, VNC, and desktop mount — isolates high-risk capabilities.
- Main services for meetings, chrome import, and password-manager connections; tree-sitter / process natives.
- Learning: AI clients that embed remote or meeting surfaces should never share one privileged preload.

### Antigravity
- Agentic desktop on a thin `dist/main.js`; `electron-updater` + `electron-log`.
- MCP / chrome-devtools agent deps; `shell-env` so agents inherit a real user PATH.
- Learning: agent products fail quietly without env bootstrapping and persistent logs.

### GitHub Desktop
- Unpacked webpack app; historically permissive `nodeIntegration`; bundled git; keytar; crash window; Squirrel.

### LM Studio
- Forge webpack; tray; electron-updater; UtilityProcess; electron-log/store.

### Clay
- Classic asar CRM desktop: SQLite, window-state, context menus, custom titlebar.
- `electron-updater` + `electron-store` + scheduler; Sentry; push-receiver.
- Mac bridges for native auth, Sign in with Apple, contacts, permissions.
- Learning: GTM tools need local DB + OS identity APIs; also budget Electron upgrades (can lag majors).

### Conventional baseline shell
- Stock Helper + asar topology without IDE/MCP specialization.
- Learning: valid default starting point; add Grok Bot / Postman / Claude complexity only when a surface requires it.

### Todoist
- MAS; Electron + Swift bridges for widgets/extensions; webpack asar.

### Evernote
- MAS; multi-shell main/tray/conduit/update windows; sqlite; electron-updater; Sentry.

### ChatGPT
- Custom Electron-family framework + Sparkle + asar modules; computer-use oriented assets; rich keychain/app-group entitlements.

---

## 14. Anti-patterns observed (avoid in new code)

1. **`nodeIntegration: true` + `contextIsolation: false`** in primary UI — convenient, unsafe for any untrusted content.
2. **Shipping `@electron/remote`** — prefer explicit bridges.
3. **One mega-renderer doing network execution** — split processes (Postman-style).
4. **Assuming PATH tools exist** — bundle CLI deps when they are load-bearing (GitHub Desktop-style).
5. **Ignoring MAS sandbox until late** — Slack/Todoist show how constrained entitlements become.
6. **Stale Electron majors** — Clay (~25.x) and Raindrop (~19.x) vs Slack (~44.x); plan upgrade cadences with Chromium security.
7. **One privileged preload for webview + remote desktop + meetings** — Grok Bot splits these; collapsing them widens blast radius.
8. **Agent processes without shell-env / logging** — Antigravity shows both as first-class dependencies for agentic desktops.

---

## 15. How to reproduce this kind of study

1. Locate apps that contain `Electron Framework.framework` (or Electron-family equivalents).
2. Record Helpers, entitlements, and `Info.plist` URL types.
3. Inspect `Resources/app` or asar **table of contents** / public `package.json` metadata — avoid republishing proprietary source.
4. Keyword-scan main/preload entrypoints for `webPreferences`, `utilityProcess`, `contextBridge`, updaters, and natives.
5. Document **patterns**, not copied implementation.

Keep extracted vendor files out of git. This repository intentionally does not ship asar extracts.

---

## 16. Quick checklist for your next Electron app

- [ ] Choose archetype (shell / local-first / IDE-agent / multi-surface AI / CRM / MAS / baseline).
- [ ] Lock security webPreferences to modern defaults (sandbox + contextIsolation, no nodeIntegration).
- [ ] Define IPC contract + **per-window / per-capability** preloads (especially webview vs remote vs meetings).
- [ ] Decide update channel (Squirrel vs electron-updater vs MAS vs Sparkle).
- [ ] Plan asarUnpack for natives; consider arch-specific asars if universal + native.
- [ ] Add UtilityProcess for heavy/privileged work.
- [ ] If building agents: bootstrap real shell PATH + durable logs (Antigravity-style).
- [ ] If building CRM/GTM: local DB + Mac auth/contacts permissions early (Clay-style).
- [ ] Register deep link scheme + single-instance routing.
- [ ] Wire crash reporting + readable logs.
- [ ] Enable asar integrity + electron fuses before release.
- [ ] If macOS-native UX matters, budget Swift bridges early.
- [ ] Schedule Electron major upgrades; do not freeze on mid-20s majors without a plan.

---

*Patterns only. Vendor binaries and versions change; re-verify against current releases before relying on a citation.*
