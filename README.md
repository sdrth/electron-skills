# electron-skills

**Developer guide and Cursor skills for AI agents building Electron desktop apps.**

Use this repository when an agent (or human) needs production-grade Electron patterns: process model, secure defaults, IPC, packaging, updates, auth, notifications, and capability-oriented package choices—grounded in how real public Electron apps are structured.

## For AI agents

1. Read [`AGENTS.md`](AGENTS.md) first.
2. Pick an archetype from the architecture reference.
3. Implement with the development guide + [`electron-app-development`](skills/electron-app-development/SKILL.md) skill.
4. Choose packages from the capabilities reference.
5. Optionally study more macOS app **layouts** with [`electron-macos-pattern-study`](skills/electron-macos-pattern-study/SKILL.md) (pattern learnings only—never extract or republish vendor source).

## Docs

| Doc | Purpose |
|-----|---------|
| [AGENTS.md](AGENTS.md) | Instructions for AI agents using this guide |
| [Electron Development Guide](docs/electron-development-guide.md) | Scaffold, secure defaults, IPC, windows, workers, update, package |
| [Architecture Reference](docs/electron-app-architecture-reference.md) | Archetypes and pattern citations from public Electron apps |
| [Capabilities & Packages Reference](docs/electron-capabilities-packages-reference.md) | Auth, updates, notifications, tray, storage, agents, popular packages |

## Skills

| Skill | Purpose |
|-------|---------|
| [`electron-app-development`](skills/electron-app-development/SKILL.md) | Build Electron apps using these docs (secure defaults, archetypes, IPC, packaging) |
| [`electron-macos-pattern-study`](skills/electron-macos-pattern-study/SKILL.md) | Inspect installed macOS `.app` bundles for architecture **patterns** only |

Install into Cursor by copying or symlinking each skill folder to `.cursor/skills/` (project) or `~/.cursor/skills/` (user).

## What this repo is not

- Not a dump of proprietary app source or asar extracts
- Not a reverse-engineering, cracking, or license-bypass toolkit
- Not an official guide from Electron or any cited vendor

## License & trademarks

- Code/docs license: [LICENSE](LICENSE) (MIT)
- Trademark / no-affiliation / no-proprietary-redistribution: [NOTICE](NOTICE)
