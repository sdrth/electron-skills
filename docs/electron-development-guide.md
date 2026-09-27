# Electron Development Guide

Practical instructions for building production Electron apps—written so **AI agents** can follow them end-to-end. Pair with:

- [AGENTS.md](../AGENTS.md) — agent operating rules
- [electron-app-architecture-reference.md](./electron-app-architecture-reference.md) — real-world pattern citations
- [electron-capabilities-packages-reference.md](./electron-capabilities-packages-reference.md) — packages for auth, updates, notifications, storage, agents, etc.


**Stack defaults in this guide:** pnpm, TypeScript, Electron Forge + Vite (Claude-style) or a small manual main/preload/renderer layout. Pin dependency versions in your own `package.json`.

---

## 1. Mental model (implement this, not “a website in a window”)

| Process | Runs | Owns | Must not own |
|---------|------|------|--------------|
| **Main** | Node + Electron APIs | Windows, menus, tray, IPC handlers, protocols, auto-update, native `.node`, `utilityProcess` | Untrusted HTML/JS |
| **Preload** | Isolated world w/ limited Electron access | `contextBridge` API surface | Business logic, network, secrets |
| **Renderer** | Chromium | UI (React/Vue/Svelte/etc.) | `fs`, `child_process`, raw `ipcRenderer` without bridge |
| **Utility / worker** | Node (no DOM) | Heavy CPU/IO, agents, parsers | UI |

**Seen in production:** Claude / Cursor / VS Code (utilityProcess); Postman / Grok Bot (role workers + role preloads); Slack (multiple preloads).

---

## 2. Choose an archetype before scaffolding

| If you are building… | Start from | Cite |
|----------------------|------------|------|
| Editor / coding agent | VS Code-like `Resources/app` workbench orthodoxy; Plugin helper mindset | VS Code, Cursor, Devin |
| Hosted SaaS desktop | Thin shell + web UI + tray/deep links | Figma, Notion |
| Local documents | Thin main + disk folders + custom scheme | Obsidian |
| API / automation workstation | Main + execution/proxy workers | Postman |
| Multi-surface AI (webview, meetings, remote) | **One preload per capability** | Grok Bot, Claude |
| Agentic desktop + MCP | Env bootstrap (`shell-env`) + logs + updater | Antigravity, Claude |
| CRM / GTM | SQLite + Mac auth/contacts + window state | Clay |
| Baseline utility app | Single window + standard asar | Conventional utilities |
| Mac App Store | Sandbox entitlements from day one | Slack, Todoist, Evernote |

Do not start as a minimal shell and accidentally grow into “Postman” without splitting processes.

---

## 3. Scaffold (recommended: Forge + Vite)

```bash
# Prefer pnpm; keep Socket Firewall / sfw enabled — do not bypass it.
# Pin the scaffolder instead of @latest (check the current version first: `npm view create-electron-app version`).
pnpm create electron-app@7.11.2 my-app --template=vite-typescript
cd my-app

# Add runtime deps with exact pins (-E writes "x.y.z", not "^x.y.z").
pnpm add -E electron-log electron-updater zod
pnpm add -D -E @electron-forge/plugin-fuses @electron/fuses
```

**Pinning rules for agents**
- Never write a version you have not resolved from the registry. Use `npm view <pkg> version`, or let `pnpm add -E` resolve and record it.
- Keep the Vite / TypeScript majors the template installed. Newer majors can fall outside `@electron-forge/plugin-vite`'s supported range; check its `peerDependencies` before bumping.
- Commit the lockfile.

The Forge Vite template's defaults matter for the snippets below:

| Template fact | Value |
|---------------|-------|
| Built main / preload | `.vite/build/main.js`, `.vite/build/preload.js` |
| Dev server URL global | `MAIN_WINDOW_VITE_DEV_SERVER_URL` (undefined in packaged builds) |
| Renderer name global | `MAIN_WINDOW_VITE_NAME` |
| Entry files | set in `forge.config.ts` → `VitePlugin({ build: [...], renderer: [...] })` |

If you move entries into `src/main/index.ts` / `src/preload/main.ts` (layout below), update the `entry` paths in `forge.config.ts` to match.

If you already have a web app, add Electron beside it:

```text
my-app/
  package.json
  forge.config.ts          # or electron-builder.yml
  src/
    main/
      index.ts
      windows/main-window.ts
      ipc/
      protocols/
      security/
      updates/
      workers/
    preload/
      main.ts
    renderer/
      index.html
      src/...
    shared/
      ipc.ts               # zod schemas / channel names
```

**`package.json` shape** (versions resolved at time of writing — re-resolve, don't copy):

```json
{
  "name": "my-app",
  "private": true,
  "main": ".vite/build/main.js",
  "scripts": {
    "start": "electron-forge start",
    "package": "electron-forge package",
    "make": "electron-forge make"
  },
  "devDependencies": {
    "@electron-forge/cli": "7.11.2",
    "@electron-forge/maker-dmg": "7.11.2",
    "@electron-forge/maker-zip": "7.11.2",
    "@electron-forge/plugin-fuses": "7.11.2",
    "@electron-forge/plugin-vite": "7.11.2",
    "@electron/fuses": "2.1.3",
    "electron": "44.4.5"
  },
  "dependencies": {
    "electron-log": "5.4.4",
    "electron-updater": "6.8.9",
    "zod": "4.6.5"
  }
}
```

`typescript` and `vite` are omitted on purpose: keep whatever exact versions the template installed.

**Tooling map from the field:** Forge+Vite (Claude), Forge+Webpack (LM Studio, Notion), Rspack (Slack), VS Code custom `out/` (Cursor/Devin), thin `dist/main.js` (Antigravity).

---

## 4. Secure window defaults (do this first)

Every `BrowserWindow` / `WebContentsView` for app UI:

```ts
// src/main/windows/main-window.ts
import { BrowserWindow } from 'electron';
import path from 'node:path';

export function createMainWindow(): BrowserWindow {
  const win = new BrowserWindow({
    width: 1200,
    height: 800,
    show: false,
    webPreferences: {
      // Forge Vite emits main.js and preload.js side by side in .vite/build/
      preload: path.join(__dirname, 'preload.js'),
      contextIsolation: true,
      nodeIntegration: false,
      sandbox: true,
      webviewTag: false,
      spellcheck: true,
    },
  });

  win.once('ready-to-show', () => win.show());

  if (MAIN_WINDOW_VITE_DEV_SERVER_URL) {
    void win.loadURL(MAIN_WINDOW_VITE_DEV_SERVER_URL);
  } else {
    // Served by the app:// handler in §7 from .vite/renderer/
    void win.loadURL(`app://bundle/${MAIN_WINDOW_VITE_NAME}/index.html`);
  }

  return win;
}
```

Loading production UI from `app://bundle` instead of `loadFile` gives the app one exact origin to trust. With `file:`, every HTML file on disk shares the same origin, so a navigation to a downloaded file would pass origin checks.

**Sandboxed preloads cannot `require` arbitrary modules.** With `sandbox: true`, a preload only gets `electron` (renderer subset) and a few Node polyfills. Imports like `zod` or `../shared/ipc` work only because Vite bundles the preload into one file. If you drop the bundler, those imports fail at runtime.

**Aligns with:** Claude / Cursor partitioned views. **Avoid:** GitHub Desktop-era `nodeIntegration: true` + `contextIsolation: false`. **Avoid:** `@electron/remote` (still seen in Obsidian/Postman — do not add it).

### Navigation guard (every webContents, including ones you didn't create)

Setting `setWindowOpenHandler` on one window is not enough. Apply the policy globally so popups, guest views, and later windows inherit it:

```ts
// src/main/security/origins.ts
export const APP_ORIGINS = new Set<string>([
  'app://bundle',
  ...(MAIN_WINDOW_VITE_DEV_SERVER_URL ? [new URL(MAIN_WINDOW_VITE_DEV_SERVER_URL).origin] : []),
]);

export function isAppUrl(raw: string): boolean {
  try {
    return APP_ORIGINS.has(new URL(raw).origin);
  } catch {
    return false;
  }
}
```

```ts
// src/main/security/navigation.ts
import { app, shell } from 'electron';
import { isAppUrl } from './origins';

function openIfHttps(raw: string): void {
  if (new URL(raw).protocol === 'https:') void shell.openExternal(raw);
}

export function installNavigationGuards(): void {
  app.on('web-contents-created', (_event, contents) => {
    contents.on('will-navigate', (event, url) => {
      if (!isAppUrl(url)) {
        event.preventDefault();
        openIfHttps(url);
      }
    });

    contents.on('will-attach-webview', (event) => event.preventDefault());

    contents.setWindowOpenHandler(({ url }) => {
      openIfHttps(url);
      return { action: 'deny' };
    });
  });
}
```

Call `installNavigationGuards()` before creating any window. If a feature genuinely needs `<webview>`, replace the blanket `preventDefault` with a handler that strips `preload` / forces `nodeIntegration: false` and checks the `src` against an allowlist.

### Content Security Policy

Ship a CSP for every app-owned renderer. Start strict and loosen only for named origins:

```html
<!-- src/renderer/index.html -->
<meta
  http-equiv="Content-Security-Policy"
  content="default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; connect-src 'self' https://api.example.com; object-src 'none'; base-uri 'none'; frame-ancestors 'none'"
/>
```

- Vite's dev server needs its HMR websocket; if CSP blocks it in development, add the dev origin to `connect-src` only in dev builds.
- For remote content you load, set CSP as a response header via `session.webRequest.onHeadersReceived` instead of a meta tag.
- Never add `'unsafe-eval'` to satisfy a library; find a build that doesn't need it.

### Session permissions (deny by default)

```ts
import { session } from 'electron';

export function hardenSession(ses = session.defaultSession): void {
  ses.setPermissionRequestHandler((_wc, _permission, callback) => {
    callback(false); // opt-in per partition/feature later
  });

  ses.setPermissionCheckHandler(() => false);
}
```

**Seen in:** Claude embedded web sessions; VS Code / Cursor permission handlers.

---

## 5. Preload + typed IPC (required pattern)

### Shared contract

```ts
// src/shared/ipc.ts
import { z } from 'zod';

export const Ipc = {
  ping: {
    channel: 'app:ping',
    req: z.object({ nonce: z.string().min(1) }),
    res: z.object({ pong: z.literal(true), at: z.number() }),
  },
  openExternal: {
    channel: 'shell:openExternal',
    req: z.object({ url: z.url({ protocol: /^https$/ }) }),
    res: z.object({ ok: z.boolean() }),
  },
} as const;
```

(Zod 4 syntax. On Zod 3, use `z.string().url()` plus a protocol check in the handler.)

### Preload (only expose what the UI needs)

```ts
// src/preload/main.ts
import { contextBridge, ipcRenderer } from 'electron';
import { Ipc } from '../shared/ipc';

contextBridge.exposeInMainWorld('desktop', {
  ping: async (nonce: string) => {
    const req = Ipc.ping.req.parse({ nonce });
    const raw = await ipcRenderer.invoke(Ipc.ping.channel, req);
    return Ipc.ping.res.parse(raw);
  },
  openExternal: async (url: string) => {
    const req = Ipc.openExternal.req.parse({ url });
    const raw = await ipcRenderer.invoke(Ipc.openExternal.channel, req);
    return Ipc.openExternal.res.parse(raw);
  },
});
```

```ts
// src/renderer/src/desktop.d.ts
export {};
declare global {
  interface Window {
    desktop: {
      ping: (nonce: string) => Promise<{ pong: true; at: number }>;
      openExternal: (url: string) => Promise<{ ok: boolean }>;
    };
  }
}
```

### Main handlers

```ts
// src/main/ipc/register.ts
import { ipcMain, shell, type IpcMainInvokeEvent } from 'electron';
import { Ipc } from '../../shared/ipc';
import { isAppUrl } from '../security/origins';

// Any frame that reaches ipcRenderer can call these handlers, including a
// navigated or injected frame. Reject senders that are not app-owned pages.
function assertTrustedSender(event: IpcMainInvokeEvent): void {
  const frame = event.senderFrame;
  if (!frame) throw new Error('IPC sender frame is gone');
  if (!isAppUrl(frame.url)) throw new Error(`Untrusted IPC sender: ${frame.url}`);
}

export function registerIpc(): void {
  ipcMain.handle(Ipc.ping.channel, (event, unknownReq) => {
    assertTrustedSender(event);
    Ipc.ping.req.parse(unknownReq);
    return Ipc.ping.res.parse({ pong: true as const, at: Date.now() });
  });

  ipcMain.handle(Ipc.openExternal.channel, async (event, unknownReq) => {
    assertTrustedSender(event);
    const { url } = Ipc.openExternal.req.parse(unknownReq);
    await shell.openExternal(url);
    return { ok: true };
  });
}
```

**Validate twice, trust once:** the preload parses for developer ergonomics, but main must re-parse because a compromised renderer can call `ipcRenderer.invoke` with anything. The sender check and the main-side parse are the security boundary; the preload parse is not.

If you add a guest `WebContentsView` with its own preload, give it a **separate channel namespace** and a sender check that accepts only its partition's origin.

**Field notes:** Claude generates its IPC bridge from a schema instead of hand-writing channels. Grok Bot uses **separate preloads** per risky surface — copy that when you add webview / remote / meetings.

### One preload per capability

| Surface | Preload | Why |
|---------|---------|-----|
| Main app | `preload/main.ts` | General UI API |
| Quick capture | `preload/quick.ts` | Smaller API |
| Embedded webview | `preload/webview.ts` | No filesystem APIs |
| Meeting / A/V | `preload/meeting.ts` | Media-only bridge |
| Remote / VNC | `preload/remote.ts` | Tight allowlist |

**Cite:** Grok Bot, Slack, Notion, Postman, Claude.

---

## 6. App lifecycle (main entry)

```ts
// src/main/index.ts
import { app, BrowserWindow } from 'electron';
import path from 'node:path';
import { createMainWindow } from './windows/main-window';
import { registerIpc } from './ipc/register';
import { registerAppScheme, serveAppScheme } from './protocols/app-scheme';
import { initDeepLinks, flushPendingDeepLinks, handleDeepLinkArgv } from './protocols/deep-links';
import { hardenSession } from './security/session';
import { installNavigationGuards } from './security/navigation';
import { setupAutoUpdater } from './updates/updater';

// Must run before `ready`: privileged schemes, open-url listener, lock.
registerAppScheme();
initDeepLinks();

if (!app.requestSingleInstanceLock()) {
  app.quit();
} else {
  // Windows/Linux deliver deep links to the already-running instance via argv.
  app.on('second-instance', (_event, argv) => {
    const win = BrowserWindow.getAllWindows()[0];
    if (win) {
      if (win.isMinimized()) win.restore();
      win.focus();
    }
    handleDeepLinkArgv(argv);
  });

  app.whenReady().then(() => {
    installNavigationGuards();
    hardenSession();
    serveAppScheme(path.join(__dirname, '../renderer')); // .vite/renderer/
    registerIpc();
    createMainWindow();
    flushPendingDeepLinks();
    handleDeepLinkArgv(process.argv); // cold start on Windows/Linux
    setupAutoUpdater();

    app.on('activate', () => {
      if (BrowserWindow.getAllWindows().length === 0) createMainWindow();
    });
  });

  app.on('window-all-closed', () => {
    if (process.platform !== 'darwin') app.quit();
  });
}
```

**Patterns:** single-instance lock (all serious desktops); macOS `activate` recreates the window; privileged schemes and the `open-url` listener are registered **before** ready (VS Code / Claude). Everything after the lock check lives in the `else` branch so a second instance never creates windows or registers handlers before it quits.

---

## 7. Custom protocols & deep links

### Privileged scheme (local app resources)

```ts
// src/main/protocols/app-scheme.ts
import { net, protocol } from 'electron';
import path from 'node:path';
import { pathToFileURL } from 'node:url';

export function registerAppScheme(): void {
  // Before `ready`.
  protocol.registerSchemesAsPrivileged([
    {
      scheme: 'app',
      privileges: { standard: true, secure: true, supportFetchAPI: true },
    },
  ]);
}

export function serveAppScheme(root: string): void {
  // After `ready`. Serves app://bundle/<path> from `root` only.
  const base = path.resolve(root);
  protocol.handle('app', (request) => {
    const { host, pathname } = new URL(request.url);
    if (host !== 'bundle') return new Response('Not found', { status: 404 });

    const target = path.resolve(base, '.' + decodeURIComponent(pathname));
    const rel = path.relative(base, target);
    if (rel.startsWith('..') || path.isAbsolute(rel)) {
      return new Response('Forbidden', { status: 403 });
    }
    return net.fetch(pathToFileURL(target).toString());
  });
}
```

The `path.relative` check is the part agents most often drop. Without it, `app://bundle/../../etc/passwd`-style requests read arbitrary files. Add `corsEnabled` only if a specific cross-origin fetch needs it.

### Deep link (`myapp://…`)

```ts
// src/main/protocols/deep-links.ts
import { app } from 'electron';

const SCHEME = 'myapp';
const pending: string[] = [];
let ready = false;

export function initDeepLinks(): void {
  if (process.defaultApp && process.argv.length >= 2) {
    app.setAsDefaultProtocolClient(SCHEME, process.execPath, [process.argv[1]]);
  } else {
    app.setAsDefaultProtocolClient(SCHEME);
  }

  // macOS: can fire before `ready` (cold start from a link) — queue until windows exist.
  app.on('open-url', (event, url) => {
    event.preventDefault();
    if (ready) route(url);
    else pending.push(url);
  });
}

export function flushPendingDeepLinks(): void {
  ready = true;
  pending.splice(0).forEach(route);
}

export function handleDeepLinkArgv(argv: string[]): void {
  const url = argv.find((arg) => arg.startsWith(`${SCHEME}://`));
  if (url) route(url);
}

function route(raw: string): void {
  let u: URL;
  try {
    u = new URL(raw);
  } catch {
    return;
  }
  if (u.protocol !== `${SCHEME}:`) return;

  // Allowlist actions; treat every query param as attacker-controlled.
  switch (u.host) {
    case 'open':
      // e.g. focus a document id after validating its format
      break;
    case 'auth':
      // hand code/state to the auth module; verify state + PKCE there
      break;
    default:
      return;
  }
}
```

Never `loadURL` a deep-link value into a privileged window, and never pass its params into shell commands or file paths without validation.

**Cite:** `slack://`, `notion://`, `obsidian://`, `figma://`, `claude://`, VS Code URL protocol + `--open-url`.

---

## 8. Multi-window & WebContentsView

```ts
import { BrowserWindow, WebContentsView, session } from 'electron';
import { hardenSession } from '../security/session';

export function attachGuestView(
  parent: BrowserWindow,
  opts: { partition?: string; preload?: string } = {},
): WebContentsView {
  const partition = opts.partition ?? 'persist:guest';
  hardenSession(session.fromPartition(partition));

  const view = new WebContentsView({
    webPreferences: {
      partition,
      contextIsolation: true,
      nodeIntegration: false,
      sandbox: true,
      // Omit preload entirely unless the guest needs a (separate, minimal) bridge.
      ...(opts.preload ? { preload: opts.preload } : {}),
    },
  });

  parent.contentView.addChildView(view);
  const resize = () => {
    const { width, height } = parent.getContentBounds();
    view.setBounds({ x: 0, y: 80, width, height: height - 80 });
  };
  resize();
  parent.on('resize', resize);
  return view;
}
```

**Cite:** Claude / Cursor `WebContentsView`; Notion tab/browser views; Figma shell vs web bindings.

**Rules:**
- Guest content gets its own `session.fromPartition`.
- Deny permissions on that session by default.
- Do not reuse the main app preload for guest content.

---

## 9. UtilityProcess for heavy / agent work

```ts
import { utilityProcess, MessageChannelMain } from 'electron';
import path from 'node:path';

export function startWorker(): Electron.UtilityProcess {
  const child = utilityProcess.fork(path.join(__dirname, '../workers/agent.js'), [], {
    serviceName: 'agent-worker',
  });

  const { port1, port2 } = new MessageChannelMain();
  child.postMessage({ type: 'init' }, [port1]);
  // keep port2 in main for RPC

  child.on('exit', (code) => {
    console.error('agent-worker exited', code);
  });

  return child;
}
```

**Cite:** VS Code, Cursor, Claude, LM Studio. Prefer this over hidden `BrowserWindow` unless you need DOM (Evernote conduit pattern).

**Agent extras (Antigravity-style):**
- Bootstrap PATH via `shell-env` (or equivalent) before spawning tools.
- Use `electron-log` file transport — agents fail silently without durable logs.

---

## 10. Persistence, secrets, OS integrations

| Need | Approach | Cite |
|------|----------|------|
| Window size/position | `electron-window-state` | Clay |
| Simple config | `electron-store` (encrypt secrets) | Clay, Claude, LM Studio |
| Secrets | `safeStorage.encryptString` | VS Code, Cursor, Claude |
| Keychain username/password | `keytar` (native) | GitHub Desktop, Postman |
| Local DB | `better-sqlite3` in main or utility | Notion, Clay, Evernote |
| Tray | `Tray` + template images | Figma, LM Studio |
| Scheduler | `toad-scheduler` in main | Clay |
| Mac contacts / permissions / Apple auth | purpose-built natives | Clay |
| Widgets / IAP / share extensions | Swift frameworks + bridge | Todoist |

Unpack all `.node` binaries:

```ts
// forge/electron-builder equivalent idea
asarUnpack: ['**/*.node', '**/better-sqlite3/**']
```

**Cite:** Figma `app.asar.unpacked` for Rust/bindings nodes.

---

## 11. Auto-update

### electron-updater (common non-MAS)

```ts
import { app } from 'electron';
import { autoUpdater } from 'electron-updater';
import log from 'electron-log/main';

export function setupAutoUpdater(): void {
  if (!app.isPackaged) return; // dev builds have no update feed
  autoUpdater.logger = log;
  autoUpdater.checkForUpdatesAndNotify().catch((err) => log.error(err));
}
```

Use `app.isPackaged`, not `NODE_ENV`: packaged apps don't reliably set `NODE_ENV`.

On macOS, `electron-updater` only applies updates to **signed** builds, and you must ship the `zip` target alongside `dmg` — the updater consumes the zip.

Ship `app-update.yml` / publish config for your provider (GitHub Releases, S3, R2 — LM Studio-style).

### When to use what

| Channel | Use when | Cite |
|---------|----------|------|
| Squirrel | Classic Mac direct-download Electron | VS Code lineage |
| electron-updater | Cross-platform generic hosting | LM Studio, Clay, Antigravity, Evernote |
| Sparkle | Custom Mac updater UX / hybrid shells | ChatGPT-style |
| MAS | App Store only — no custom sparkle path | Slack, Todoist, Evernote |

Add a **minimum supported version** gate when breaking IPC/protocol (Slack-style).

---

## 12. Packaging, fuses, integrity

1. Enable **Electron fuses** at package time — Slack / Claude packaging. Forge config:

```ts
// forge.config.ts (plugins array)
import { FusesPlugin } from '@electron-forge/plugin-fuses';
import { FuseV1Options, FuseVersion } from '@electron/fuses';

new FusesPlugin({
  version: FuseVersion.V1,
  [FuseV1Options.RunAsNode]: false,                          // blocks ELECTRON_RUN_AS_NODE abuse
  [FuseV1Options.EnableCookieEncryption]: true,
  [FuseV1Options.EnableNodeOptionsEnvironmentVariable]: false,
  [FuseV1Options.EnableNodeCliInspectArguments]: false,       // blocks --inspect on the shipped app
  [FuseV1Options.EnableEmbeddedAsarIntegrityValidation]: true,
  [FuseV1Options.OnlyLoadAppFromAsar]: true,
}),
```

   `RunAsNode: false` breaks code that spawns `process.execPath` as Node. Use `utilityProcess.fork` for that instead.
2. Asar integrity validation writes `ElectronAsarIntegrity` into `Info.plist`; you'll see that key on modern Mac builds.
3. Universal Mac + arch-native modules: consider Slack’s **bootstrap asar + `app-arm64.asar` / `app-x64.asar`** pattern.
4. MAS: design entitlements before adding natives (camera/mic/downloads/bookmarks as needed).

---

## 13. Observability

```ts
import { app } from 'electron';
import log from 'electron-log/main';

log.initialize();
log.info('app starting', { version: app.getVersion() });
```

- **electron-log** — LM Studio, Antigravity, Clay-class.
- **Sentry** — Slack, Claude, Clay, Evernote.
- **crashReporter** — VS Code / Claude pre-boot patterns.
- Optional dedicated crash window — GitHub Desktop.

---

## 14. Testing & quality bar

1. Unit-test shared IPC zod schemas (no Electron required).
2. Smoke-test the packaged app with Playwright's Electron support (`_electron.launch` from `playwright`). Spectron is deprecated; don't use it.
3. Manual checklist before release:
   - [ ] All windows use secure `webPreferences`
   - [ ] No `nodeIntegration` in UI
   - [ ] Navigation guard installed via `web-contents-created`
   - [ ] Every `ipcMain.handle` checks the sender and re-parses input
   - [ ] CSP present on app-owned renderers
   - [ ] `app://` handler rejects path traversal
   - [ ] External links use `openExternal`, https only
   - [ ] Deep links validated (including argv on Windows/Linux)
   - [ ] Fuses applied to the packaged build
   - [ ] Natives unpacked and ABI-matched
   - [ ] Updater works on a staging channel
   - [ ] Logs land on disk in prod

---

## 15. Implementation recipes by archetype

### Baseline app (conventional shell)

1. One `BrowserWindow`, one preload, typed IPC.
2. `electron-log` + optional `electron-updater`.
3. Stop. Do not add UtilityProcess until measured need.

### Multi-surface AI (Grok Bot / Claude-like)

1. Main modules: `windows/`, `sessions/`, `workers/`.
2. Preloads: `main`, `webview`, `meeting`, `remote` — zero shared privileged APIs.
3. Guest partitions deny permissions.
4. MCP / PTY / computer-use natives only in main or utility.

### Agentic desktop (Antigravity-like)

1. `shell-env` before tool spawn.
2. MCP server control in utilityProcess.
3. File logs always on; updater always on for rapid agent fixes.

### CRM (Clay-like)

1. `better-sqlite3` schema migrations in main.
2. `electron-window-state` + tray optional.
3. Mac auth/contacts via maintained native modules; request entitlements/usage strings early.

### IDE agent (Devin / Cursor-like)

1. Prefer extending VS Code OSS architecture over greenfield.
2. Keep Plugin helper / extension host separation.
3. Register only the file types you truly own.

---

## 16. Anti-patterns (do not ship)

1. `nodeIntegration: true` in product UI.
2. `@electron/remote`.
3. One preload that can both read the filesystem and drive a webview/VNC surface.
4. Running agent/tool execution inside the UI renderer.
5. Assuming `git`/CLIs exist on `PATH` — bundle or detect (GitHub Desktop).
6. Freezing on ancient Electron majors without a patch plan (Clay/Raindrop vs Slack).
7. Committing asar extracts or vendor `package.json` dumps to git.

---

## 17. Suggested learning loop

1. Read [architecture reference](./electron-app-architecture-reference.md) for citations.
2. Implement with the **electron-app-development** skill (follows this guide).
3. Use **electron-macos-pattern-study** to inspect installed apps **for patterns only** (metadata, layout, public entry names — not source redistribution).
4. Port one pattern at a time into your scaffold (preload split → then utilityProcess → then updater).

---

## 18. Quick start checklist

- [ ] Pick archetype (§2)
- [ ] Scaffold with pnpm + pinned Electron (§3)
- [ ] Secure `webPreferences`, navigation guard, CSP, session deny-by-default (§4)
- [ ] Typed IPC + contextBridge + sender validation (§5)
- [ ] Single-instance + lifecycle (§6)
- [ ] Deep link / protocol if needed (§7)
- [ ] Split windows/views/workers as the product requires (§8–9)
- [ ] Persistence/secrets/natives (§10)
- [ ] Updater + fuses + logs (§11–13)
- [ ] Smoke test + release checklist (§14)
