# Inspect guide — macOS Electron pattern study

**Purpose:** Learn packaging and process-architecture patterns from apps you already have installed.  
**Not for:** Extracting or republishing proprietary source, bypassing security, or cracking apps.

Replace `APP` with a path like `/Applications/Example.app`.

---

## 0. Ethics checklist (every session)

- [ ] Studying patterns for implementation learnings
- [ ] Will not commit asar extracts or vendor source to git
- [ ] Will scrub usernames, home paths, team IDs, emails from notes
- [ ] Will describe *what exists* (Helpers, `main` field, preload *names*), not paste proprietary implementations

---

## 1. Confirm it is Electron-family

```bash
APP="/Applications/Example.app"

# Classic Electron
ls "$APP/Contents/Frameworks" | head
test -d "$APP/Contents/Frameworks/Electron Framework.framework" && echo "Electron Framework: yes"

# Framework version (approximate Electron)
/usr/libexec/PlistBuddy -c 'Print :CFBundleVersion' \
  "$APP/Contents/Frameworks/Electron Framework.framework/Resources/Info.plist" 2>/dev/null \
  || /usr/libexec/PlistBuddy -c 'Print :CFBundleVersion' \
  "$APP/Contents/Frameworks/Electron Framework.framework/Versions/A/Resources/Info.plist" 2>/dev/null

# Electron-family without that exact name: look for app.asar + Chromium helpers
ls "$APP/Contents/Resources" | head
```

**Record:** Electron / Electron-family / not Electron (CEF, native, etc.).

---

## 2. Bundle map (always)

```bash
ls -la "$APP/Contents"
ls "$APP/Contents/MacOS"
ls "$APP/Contents/Frameworks"
ls "$APP/Contents/Resources" | head -60
```

**Look for:**

| Signal | Pattern learning |
|--------|------------------|
| `Helper (GPU/Renderer/Plugin).app` | Standard Chromium split; Plugin ⇒ IDE/extension host likely |
| `Squirrel.framework` | Direct-download auto-update lineage |
| `Sparkle.framework` | Native Mac updater (often hybrid shells) |
| `_MASReceipt` | Mac App Store build |
| `app.asar` (+ `.unpacked`) | Classic Electron packaging |
| `app/` directory (no asar / alongside) | VS Code-style unpacked app tree |
| `app-arm64.asar` / `app-x64.asar` | Universal + arch-specific payloads |
| `app-update.yml` | `electron-updater` |
| Extra `*.asar` | Feature modules / secondary UIs |

---

## 3. Identity & deep links (Info.plist)

```bash
plutil -p "$APP/Contents/Info.plist" | rg -i 'CFBundle(Identifier|Name|Executable|ShortVersionString)|URLTypes|URLSchemes|ElectronAsarIntegrity|LSHandler'
```

**Record:** bundle id, version, URL schemes, whether asar integrity metadata exists, document-type ownership breadth (editors register many).

---

## 4. Entitlement categories (not secrets)

```bash
codesign -d --entitlements :- "$APP" 2>/dev/null | plutil -p - 2>/dev/null | head -80
```

**Record categories only:**

- App Sandbox on/off (MAS often on)
- `allow-jit` (expected for Chromium/V8)
- Camera / microphone / Apple Events
- Network client/server
- Downloads / user-selected files
- Keychain / application-groups (presence — do not copy team IDs into public docs)

---

## 5. Packaging shape

### A. Unpacked `Resources/app` (IDE lineage)

```bash
ls "$APP/Contents/Resources/app" | head -40
test -f "$APP/Contents/Resources/app/package.json" && \
  python3 -c "import json;p=json.load(open('$APP/Contents/Resources/app/package.json'));print({k:p.get(k) for k in ['name','main','version']})"
test -f "$APP/Contents/Resources/app/product.json" && \
  python3 -c "import json;p=json.load(open('$APP/Contents/Resources/app/product.json'));print({k:p.get(k) for k in ['nameShort','applicationName','dataFolderName','quality'] if k in p})"
```

**Learning:** `main` → entry; `product.json` ⇒ branding/update/data folder separation (VS Code / Cursor / Devin-class).

### B. `app.asar` (most apps)

**Allowed for learnings:** list entry **names** and read **root `package.json` fields** (`name`, `main`, dependency *names* that signal tooling).

**Avoid:** bulk-extracting the asar into the repo; publishing minified vendor bundles; paraphrasing large proprietary implementations.

```bash
# TOC-oriented listing with Node if @electron/asar is available in a throwaway env;
# otherwise note: Resources layout + whether package.json main is visible via tooling.
ls -la "$APP/Contents/Resources/app.asar" "$APP/Contents/Resources/app.asar.unpacked" 2>/dev/null
ls "$APP/Contents/Resources/app.asar.unpacked" 2>/dev/null | head
```

If you parse asar headers locally for TOC:

- Keep output to: top-level folders, `*preload*`, `*worker*`, `*main*`, `*.node` paths.
- Delete extracts after note-taking; never commit them.

**Multi-arch:** if `app-arm64.asar` / `app-x64.asar` exist with a tiny bootstrap `app.asar`, note Slack-style universal packaging.

---

## 6. Filename signals (pattern gold)

From TOC or unpacked trees, classify paths:

| Filename pattern | Likely learning |
|------------------|-----------------|
| `preload*.js` / `*preload*` | Preload bridge(s) — count them |
| `webview*preload*` vs `meeting*preload*` vs `vnc*preload*` | Capability-separated bridges (Grok Bot-like) |
| `utility` / `worker` / `*Process.js` | Side process mesh (Postman-like) |
| `.webpack/` / `.vite/` | Forge webpack vs Vite |
| `out/main.js` | VS Code-like |
| `dist/electron-main` + `dist/electron-preload` | Explicit main/preload split |
| `better_sqlite3.node` / `*.node` in unpacked | Native modules → asarUnpack |
| `*Worker*.html` / `tray*` HTML | Hidden/worker or tray windows |
| `app-update.yml` | electron-updater |

---

## 7. High-level keyword signals (optional)

Only on **entry files you already identified** (e.g. `out/main.js`, `main.js`), search for presence — not for copying code:

```bash
# Example against an unpacked entry (IDE-style). Do not paste matched code into public docs.
ENTRY="$APP/Contents/Resources/app/out/main.js"
rg -o "contextIsolation|nodeIntegration|sandbox|utilityProcess|MessageChannelMain|contextBridge|safeStorage|registerSchemesAsPrivileged|setPermissionRequestHandler|electron-updater|autoUpdater" "$ENTRY" | sort -u
```

**Record a checklist of which APIs appear**, e.g.:

- Secure defaults present? (`contextIsolation` / `sandbox` / no `nodeIntegration`)
- UtilityProcess / MessageChannelMain?
- safeStorage / permission handlers / privileged schemes?
- Updater family?

---

## 8. Compare two apps (recommended)

Pick one “baseline” and one “complex”:

| Baseline | Complex |
|----------|---------|
| Small asar, one preload | Many preloads / workers |
| No Plugin helper | Plugin helper (IDE) |
| electron-updater yaml | Squirrel or Sparkle |
| Non-MAS | MAS sandbox |

Write diffs as pattern deltas, not file diffs.

---

## 9. Finish — notes hygiene

Before saving anything to a public repo:

1. Remove home-directory absolute paths; cite `/Applications/AppName.app` only when needed.
2. Remove Apple team IDs, emails, machine sizes if they only fingerprint an install.
3. Prefer approximate Electron majors (`40.x`) over exact patch builds unless needed.
4. Cite **pattern + app name**, not pasted source.
5. Ensure `.gitignore` excludes `research/`, `*.asar`, extracts.

---

## 10. Map learnings → implementation

For each finding, add one actionable line:

| Finding | Implementation action |
|---------|---------------------|
| Three preloads | Split preload per window role in our app |
| `utilityProcess` strings | Move agent jobs off main |
| `app-update.yml` | Adopt electron-updater staging channel |
| MAS sandbox | Defer native module X or add entitlement plan |
| Plugin helper | Consider VS Code OSS instead of greenfield IDE |

Then update or follow `docs/electron-development-guide.md`.
