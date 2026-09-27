# Reference — Electron macOS pattern signals

Companion to [SKILL.md](SKILL.md) and [inspect-guide.md](inspect-guide.md).

---

## Archetypes (hypothesis labels)

| ID | Archetype | Typical signals |
|----|-----------|-----------------|
| A | Editor / IDE / agent IDE | `Resources/app`, `product.json`, Helper (Plugin), Squirrel, many file types |
| B | Messaging | Multi preload, notification natives, often MAS, arch asars |
| C | Web SaaS shell | Thin main, tray, `web_*binding*`, remote UI |
| D | Local-first | Thin asar, custom URL scheme, disk vaults |
| E | API / automation | `*Worker*`, `*Process*`, proxy/execution entries |
| F | AI companion | Multi-window, MCP deps, utilityProcess, role preloads |
| G | CRM / GTM | SQLite, window-state, Mac auth/contacts, electron-updater |
| H | Calendar / small SaaS | `build/main` + single preload bundle |
| I | Baseline shell | Single asar, stock helpers, few preloads |
| J | MAS + native extensions | `_MASReceipt`, sandbox, Swift frameworks beside Electron |
| K | Notes + conduit | Tray helper + worker HTML shells, sqlite, updater |

---

## Signal glossary

### Process / helpers

| Signal | Meaning for builders |
|--------|----------------------|
| Helper (GPU) | Compositor process — always expect it |
| Helper (Renderer) | UI web contents |
| Helper (Plugin) | Extension / node host territory (IDE lineage) |
| Mantle + ReactiveObjC + Squirrel | Common Mac Electron updater stack |

### Packaging

| Signal | Meaning |
|--------|---------|
| `app.asar` + `asar.unpacked` | Default; natives unpacked |
| `Resources/app/` tree | Large/editable app payload (Code lineage) |
| Bootstrap + `app-arm64.asar` | Universal binary with arch natives |
| `ElectronAsarIntegrity` | Integrity metadata in plist |
| Fuses (packaging metadata) | Runtime lockdown |

### Security API presence

| API / flag | Good default? |
|------------|----------------|
| `contextIsolation: true` | Yes |
| `sandbox: true` | Yes |
| `nodeIntegration: false` | Yes |
| `nodeIntegration: true` | Legacy — do not copy |
| `@electron/remote` | Legacy — do not copy |
| `contextBridge` | Yes — expose minimal API |
| `setPermissionRequestHandler` | Yes — deny by default |
| `safeStorage` | Yes for secrets |
| `utilityProcess` | Yes for heavy/agent work |

### Updaters

| Signal | Stack |
|--------|-------|
| Squirrel.framework | Classic Electron Mac |
| `app-update.yml` / electron-updater | Generic hosting |
| Sparkle.framework | Native Mac updater |
| `_MASReceipt` | App Store channel |

### Dependency name hints (from root package.json only)

| Name contains | Hint |
|---------------|------|
| `electron-updater` / `electron-log` / `electron-store` | Standard desktop ops |
| `better-sqlite3` | Local DB in main/utility |
| `node-pty` | Terminals / agents |
| `keytar` | OS keychain |
| `@sentry/electron` | Crash/error reporting |
| `electron-window-state` | UX persistence |
| MCP / chrome-devtools | Agentic tooling |
| `shell-env` | Real user PATH for agents |

---

## Output template (paste into notes / PR)

```markdown
# Pattern notes: <AppName>

- Date: YYYY-MM-DD
- Bundle: /Applications/<AppName>.app
- Electron: <major.x or family>
- Archetype hypothesis: <A–K>

## Layout
- Helpers: …
- Frameworks of note: …
- Packaging: asar | Resources/app | multi-arch | hybrid

## Product surface
- URL schemes: …
- MAS: yes/no
- Updater: Squirrel | electron-updater | Sparkle | MAS

## Structure signals
- main entry field: …
- preload count / roles: …
- workers/processes: …
- natives (.node) locations: …

## Security signals (presence only)
- [ ] contextIsolation
- [ ] sandbox
- [ ] nodeIntegration disabled
- [ ] permission handler
- [ ] safeStorage
- [ ] utilityProcess

## Learnings to apply
1. …
2. …

## Explicitly not copied
- No source extracts retained
- No team IDs / emails / home paths in this note
```

---

## Mapping to repo docs

| Study finds… | Update / follow |
|--------------|-----------------|
| New archetype | `docs/electron-app-architecture-reference.md` |
| Implementation steps | `docs/electron-development-guide.md` |
| New package / capability pattern | `docs/electron-capabilities-packages-reference.md` |
| Inspection procedure change | this skill’s `inspect-guide.md` |

---

## Public release scrub list

Remove before publishing notes:

- Home-directory absolute paths (use `/Applications/AppName.app` if a path is required)
- Apple Team IDs / keychain group prefixes
- Maintainer emails from extracted package.json
- Exact on-disk sizes that only fingerprint a private install inventory (prefer ranges)
- Full dependency trees copied from vendor packages
- Any asar payload files
