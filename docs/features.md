# 3SX Launcher — Feature Guide

A cross-platform desktop companion for the **3SX engine** (Street Fighter III:
3rd Strike Open Port). Built with Tauri 2 (Rust) + React 19, the launcher
handles engine discovery, updates, ROM setup, and configuration — presented in
a CRT-arcade UI.

- [Tabs & Layout](#tabs--layout)
- [Sidebar & Navigation](#sidebar--navigation)
- [Window Controls](#window-controls)
- [News Feed](#news-feed)
- [Game Launch & Engine Updates](#game-launch--engine-updates)
- [Settings Panel](#settings-panel)
- [Button Configuration (Controls Tab)](#button-configuration-controls-tab)
- [ROM Import](#rom-import)
- [Support CLI](#support-cli)
- [Platform Behavior](#platform-behavior)
- [Architecture](#architecture)
- [Swift/Migration Assessment](#swiftmigration-assessment)

---

## Tabs & Layout

Three tabs form the main UI, navigated via the sidebar:

| Tab | Purpose |
|-----|---------|
| **News** | Developer commit feed from the 3SX engine repo |
| **Settings** | 28 engine settings in 5 accordion categories |
| **Controls** | SVG arcade-pad button remapper |

`Q`/`ArrowLeft` cycle to the previous tab; `E`/`ArrowRight` cycle to the next.

No audio, display, gameplay/system, standalone netplay, update, or version tab
exists.

---

## Sidebar & Navigation

The left sidebar (320 px) contains:

- **Logo**: "3rd Strike / 3SXtra" styled text
- **Nav buttons**: News, Settings, Controls — highlight the active tab
- **Status line**: text banner below nav (e.g. "SYSTEM READY", "CHECKING FOR
  UPDATES", "ROM MISSING — IMPORT SF33RD.AFS")
- **Progress bar**: visible only during engine download/update
- **PLAY / INSTALL GAME / UPDATING button**: launches the engine; if the engine
  is not installed, triggers the update flow
- **Version footer**: `ENGINE {buildDate} · LAUNCHER {launcherDate}` — build
  dates from the installed engine version tag and the launcher compile time
- **Keyboard hint**: `[Q] / [E] PREV/NEXT TAB` — also accepts
  `ArrowLeft`/`ArrowRight`

The keyboard shortcuts are suppressed when an `<input>` or `<select>` element
has focus (e.g. during settings text entry).

---

## Window Controls

The window is **frameless** (`decorations: false`). The launcher draws its own
title-bar chrome:

| Button | Action |
|--------|--------|
| **Minimize** (—) | `getCurrentWindow().minimize()` |
| **Maximize** (□) | `getCurrentWindow().toggleMaximize()` |
| **Close** (✕) | `getCurrentWindow().close()` |

A `data-tauri-drag-region` attribute on the chrome bar enables window dragging.

The launcher window is 1100×700 (min 900×600), centered on first launch.
Window-width/height settings apply only to the engine, not the launcher.

---

## News Feed

The News tab displays 3 cards in a grid, each with an Unsplash background
image, a date label, a title, a summary excerpt, and a "READ FULL" button.

**Source**: the GitHub Commits API
(`https://api.github.com/repos/gootecks/3sxtra/commits?per_page=3`) —
**unauthenticated**, rate-limited to 60 requests/hour per IP.

**Known limitations**:
- The tag is *always* `"DEV UPDATE"` — the button label only changes to
  "PATCH NOTES" if the tag includes `"PATCH"`, which never happens from the
  commit API.
- Three hardcoded Unsplash image URLs cycle per card.
- Title truncated at 50 chars; summary at 130 chars.
- No caching, no offline fallback, no error-state UI — on fetch failure the
  grid is simply empty.
- Background images load directly from Unsplash at render time (the launcher's
  CSP is `null`, allowing this).

---

## Game Launch & Engine Updates

The **PLAY** button (or **INSTALL GAME** / **UPDATING** depending on state)
triggers:

1. **Auto-update for installed engines**: on first launch (or any launch where
   `isGameInstalled` is true), `performUpdate` runs automatically at startup
   and checks for engine updates.
2. **First-run with no installed engine**: the UI shows a status of
   `"FIRST RUN - DOWNLOAD REQUIRED"` and waits for the user to click
   **PLAY/INSTALL GAME**, which then runs the update flow.
3. On macOS, launches via `/usr/bin/open -n` — output is redirected to
   `~/Library/Application Support/CrowdedStreet/3SX/logs/engine-stdout.log`
   and `engine-stderr.log`.
4. On successful launch, the launcher **minimizes** itself.

**Update flow** (`performUpdate`):
1. Calls `check_updates` — fetches the latest stable release from
   `gootecks/3sxtra` GitHub Releases API (`/releases/latest` on macOS,
   `releases/tags/rolling-pre-release` on non-macOS).
2. Iterates release archives, checks their marker files.
3. Downloads + extracts each archive with `download-progress` events.
4. macOS: engine is installed to
   `~/Library/Application Support/CrowdedStreet/3SX/engine/3sx.app` via
   `ditto`, then quarantine attributes stripped.
5. Sets status to one of: `GAME UP TO DATE` / `SYSTEM READY` /
   `OFFLINE — PLAY AVAILABLE` / `OFFLINE — INSTALL REQUIRED` /
   `ROM MISSING — IMPORT SF33RD.AFS`.
6. Progress bar jumps to 100% at the end (not a smooth final segment).

**Important**: the updater only updates the **engine** — there is no launcher
self-update path. The launcher must be replaced manually. (The engine update
also writes `launcher_version.txt` at the install root, despite the name.)

---

## Settings Panel

The Settings tab shows 5 accordion categories containing 28 settings total.
See [docs/settings.md](settings.md) for the complete reference, including
engine-consumer contract, known no-ops, and safe manual workaround.

In short: **many settings saved by the launcher have no effect** in the engine
  due to boolean type encoding ("1"/"0" vs `true`/`false`) and race conditions
  with the engine's own save-on-exit.

---

## Button Configuration (Controls Tab)

The Controls tab renders an SVG arcade-layout pad with buttons for all 6 face
actions, the 4 directions, Start, and Select.

**How it works**:
1. Click a pad button → enters "listening" mode.
2. Press a keyboard key → captures `e.code` (e.g. `KeyW`, `Space`, `ArrowUp`).
3. Emits `save_mapping(player="p1", action, input="Key_<code>")`.

**What is stored**:
- Format: `p1_mapping=Action,Key_<code>` in `<pref>/mappings.ini`
- Only **p1** is ever captured; no p2, no gamepad, no joystick.
- The format `Key_Numpad4` / `Key_Space` / `Key_ArrowUp` is **not** what the
  engine expects — the engine parses `Key Keypad 4` / `Key Space` / `Key Up`
  (SDL scancode naming). As a result, **keyboard bindings created by the
  launcher have no effect in-game**.

**Known issues**:
- Gamepad remapping is **absent**: no joystick/button capture exists. The
  format-key display (`formatKeyName`) and a commented-out `applyPreset`
  function reference gamepad tokens, but no actual binding path supports them.
- Non-letter key codes generate `Key_Space`, `Key_ArrowUp`, `Key_ShiftLeft`,
  etc. — the engine does not use underscore-prefixed scancode names.
- Action tokens are short (`LP`, `MP`, `HP`, `LK`, `MK`, `HK`) but the engine
  expects full names (`Light Punch`, `Medium Punch`, `Hard Punch`, …).
- No cancel key while binding — any keydown is captured and `preventDefault`-ed.
- The `applyPreset` code (XBOX/PS5/STICKS) is **commented out** and would pass
  incorrect argument names to `save_mapping` — it is dead code.

---

## ROM Import

The ROM panel appears on every tab (as an overlay above content) whenever
`romStatus.installed` is `false`.

**Detection**: probes 6 documented stages for `SF33RD.AFS` (case-insensitive):

1. `<pref>/resources/`
2. Each known engine's portable `config/resources/` and `rom/`
3. 3sxw layouts: `resources/` beside a `3SX*.app`, inside it, and
   `3sxw*` folders
4. Beside and inside the launcher app
5. `~/Downloads`, `~/Documents`, `~/Desktop`, `~/Games`, `~/ROMs` (depth ≤ 2,
   also `*.iso`)
6. Mounted discs: `/Volumes/*/THIRD/SF33RD.AFS`, `/Volumes/*/SF33RD.AFS`

**Import**: copies the file (or extracts from ISO via `hdiutil attach -readonly
-nobrowse`) to `<pref>/resources/SF33RD.AFS`, verifies SHA-256 against the
expected hash `f9fa50f3…5604`. Non-matching hash raises a warning but does
not block.

**UI**: lists discovered candidates with per-row **Import** button, a manual
path input + **Import** button, and a **Rescan** button.

---

## Support CLI

The launcher binary accepts two CLI flags (processed before the Tauri GUI
starts):

```
<launcher> --diagnose   # JSON dump of engine/pref/ROM discovery
<launcher> --launch     # start engine without the UI window
```

**`--diagnose`** prints:
- `launcher_exe` — current binary path
- `engine_exe` — resolved engine binary
- `engine_app` — macOS: discovered `.app`
- `engine_candidates` — all candidate locations with exists/version check
- `pref_path` — effective preference directory
- `install_root` — update install root
- `rom_status` — installed/path
- `rom_candidates` — all discovered ROM copies
- `local_version` — engine version string

**`--launch`** starts the engine and exits (exit code 1 on failure).

---

## Platform Behavior

### macOS

The engine is a separate `3sx.app` bundle. Discovery order (newest by
ENGINE_VERSION build time wins; missing = oldest):

1. `~/Library/Application Support/CrowdedStreet/3SX/engine/3sx.app` (installed by the launcher's updater)
2. Embedded in launcher bundle (`Contents/Resources/engine/3sx.app`)
3. Beside the launcher app
4. `/Applications/3sx.app`
5. `~/Applications/3sx.app`

`<pref>` = `~/Library/Application Support/CrowdedStreet/3SX` (unless a
portable `config/` directory exists inside the engine `.app`).

Game output logs and engine install always use the standard preference path
(`~/Library/Application Support/CrowdedStreet/3SX/logs/`, `…/engine/`),
regardless of portable mode.

### Windows / Linux

- Engine exe lives at `<game_root>/3sx[.exe]`.
- Pref path: `%APPDATA%/CrowdedStreet/3SX` (Windows) /
  `~/.local/share/CrowdedStreet/3SX` (Linux).
- Non-macOS updates use `rolling-pre-release` tag and handle asset+engine
  zip/tar.gz downloads.

### Settings Ineffectiveness

See [docs/settings.md](settings.md) for the full analysis of settings that
are written but have no runtime effect. The root causes — boolean type
encoding, VSync never read, and exit-time config
clobber — are documented there.

---

## Architecture

See [docs/architecture.md](architecture.md) for the Rust backend (Tauri
commands, engine discovery, path resolution, updater, ROM), Tauri frontend
seam, cross-platform dependencies, CI system, and build/release workflow.

## Swift/Migration Assessment

See [docs/swift-migration.md](swift-migration.md) for an analysis of a macOS
Swift/AppKit host as a replacement for the Tauri/Rust shell.