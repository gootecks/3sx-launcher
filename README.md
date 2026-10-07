# 3SX Launcher

A cross-platform companion launcher for the **3SX engine** (Street Fighter III:
3rd Strike Open Port). Built with Tauri 2 (Rust) + React 19, it handles engine
discovery, updates, ROM setup, and configuration — presented in a CRT-arcade UI.

## Quick Links

| Doc | Description |
|-----|-------------|
| [Features](docs/features.md) | Full walkthrough: tabs, sidebar, play/update, news, ROM, CLI, platform behavior |
| [Settings](docs/settings.md) | Complete 28-setting reference: type contract, engine consumer, no-op verdicts, workarounds |
| [macOS Guide](docs/macos.md) | macOS-specific: engine discovery, update, pref paths, building |
| [Architecture](docs/architecture.md) | Rust backend, Tauri frontend seam, cross-platform deps, CI system |
| [Architecture Review](docs/architecture-review-20261007-launcher.html) | Visual render of launcher's architecture & Swift-migration overview |
| [Swift/Migration](docs/swift-migration.md) | Analysis of a macOS Swift/AppKit replacement for the Tauri shell |
| [Settings Investigation](docs/settings-investigation.md) | Root-cause analysis of why most settings have no effect |
| [Release Notes](docs/release-notes.md) | Communiqué release-note tooling |

## Features

- **Engine management**: discovers the latest installed `3sx.app` (macOS) or
  `3sx` binary (Windows/Linux), fetches engine updates from GitHub Releases.
- **Settings panel**: 28 engine settings across 5 categories (Window,
  Rendering, Netplay, Training, Mods) saved to disk on each interaction
  (most have no effect — see [docs/settings.md](docs/settings.md)).
- **Button configuration**: SVG arcade-pad remapper for keyboard bindings
  (p1 only; keyboard-only; **engine-incompatible action/key tokens** — see
  [docs/settings.md](docs/settings.md)).
- **News feed**: live commits from the 3SX engine GitHub repo, displayed as
  cards with Unsplash imagery.
- **ROM import**: auto-scans for `SF33RD.AFS`, copies from local paths or
  mounted disc images, verifies integrity via SHA-256.
- **Frameless window**: custom minimize/maximize/close chrome with
  keyboard navigation (Q/E or Arrow keys).
- **Support CLI**: `--diagnose` (JSON discovery dump) and `--launch` (engine
  without UI).
- **Cross-platform**: released for macOS (universal), Windows x86_64,
  Linux x86_64 and ARM64.

## Getting Started

### Prerequisites

- [Bun](https://bun.sh/)
- [Rust](https://rustup.rs/) latest stable
- Platform build tools (see CI deps in [docs/architecture.md](docs/architecture.md))

### Development

```sh
bun install
bun run tauri dev
```

In dev mode, the game root maps relative to `package.json` in the project tree.

### Production Build

```sh
bun run tauri build
```

Output binaries at `src-tauri/target/release/`.

## Legal

This launcher is an open-source tool for the 3SX project.
Street Fighter III: 3rd Strike trademarks belong to Capcom.
No ROM files or game runtime data are distributed in this repository.