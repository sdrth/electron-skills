---
name: electron-macos-pattern-study
description: >-
  Inspect installed macOS .app bundles to learn Electron architecture patterns
  (helpers, packaging, preloads, updaters, security defaults). Use when studying
  Electron apps in /Applications, documenting desktop patterns, or comparing
  how production Electron apps are structured. For learnings only — not reverse
  engineering, cracking, bypassing license checks, or republishing vendor source.
---

# Electron macOS pattern study

Study **publicly installed** macOS app bundles to extract **architecture patterns** for building better Electron apps.

## Hard rules

1. **Learnings only** — document patterns (process model, packaging, preload roles, updaters, entitlements *categories*). Do not reconstruct or redistribute proprietary source.
2. **Do not** dump asar file contents, minify-deobfuscate vendor code, or commit extracts to git.
3. **Do not** bypass DRM, licensing, MAS receipts, code signatures, or integrity checks.
4. Prefer **metadata**: Info.plist, Helpers list, entitlements keys, `package.json` *field names* (`main`, deps that signal tooling), asar **table-of-contents** / entry filenames — not full source trees.
5. When writing notes for open source, scrub personal paths, usernames, team IDs, emails, and install fingerprints.

If the user asks to reverse engineer, crack, or exfiltrate app source: refuse and offer pattern-level study instead.

## When to use

- User wants to know how Slack / VS Code / Claude / etc. structure Electron on Mac
- Building an Electron app and needs real-world pattern citations
- Expanding docs in this repo from installed apps

## Workflow

Copy and track:

```
Pattern study:
- [ ] 1. Inventory Electron apps
- [ ] 2. Pick 1–3 targets + archetype hypothesis
- [ ] 3. Bundle layout + Helpers + frameworks
- [ ] 4. Info.plist (schemes, integrity) + entitlement categories
- [ ] 5. Packaging shape (asar vs Resources/app)
- [ ] 6. Entry/preload/worker filenames (TOC only)
- [ ] 7. Keyword signals (security, IPC, updater) — high-level
- [ ] 8. Write pattern notes (no source dumps)
- [ ] 9. Map notes → development recommendations
```

### 1. Inventory

```bash
find /Applications ~/Applications -maxdepth 4 -type d -name 'Electron Framework.framework' 2>/dev/null
```

Also note Electron-family apps that rename the framework (still may ship `app.asar` + Chromium helpers).

### 2. Hypothesis

Classify before deep inspection: IDE shell | web shell | local-first | messaging | AI companion | CRM | MAS hybrid | baseline. See [reference.md](reference.md).

### 3–7. Inspect

Follow [inspect-guide.md](inspect-guide.md) command-by-command.

### 8–9. Document

Use the output template in [reference.md](reference.md). Link patterns to `docs/electron-development-guide.md`, `docs/electron-app-architecture-reference.md`, and `docs/electron-capabilities-packages-reference.md` when updating this repo.

## Additional resources

- [inspect-guide.md](inspect-guide.md) — step-by-step macOS inspection commands
- [reference.md](reference.md) — signal glossary, archetypes, output template
- Repo docs: `docs/electron-development-guide.md`, `docs/electron-app-architecture-reference.md`, `docs/electron-capabilities-packages-reference.md`
- Sibling skill for building apps: [electron-app-development](../electron-app-development/SKILL.md)
