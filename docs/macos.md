# macOS — launcher and engine

The launcher is this app; the engine (`3sx.app`) is built and released
separately from [gootecks/3sxtra](https://github.com/gootecks/3sxtra).
The launcher finds and starts the engine — it does **not** update itself
(see [Updater](#updater)).

## Building

```sh
bun install --bun && bun run tauri dev          # dev
bun install --bun && bun run tauri build        # bundle (host arch)
bun install --bun && bun run tauri build --target universal-apple-darwin
```

The release profile keeps host proc-macros unstripped
(`[profile.release.build-override] strip = false` in `src-tauri/Cargo.toml`):
stripped proc-macro dylibs fail to `dlopen` on recent macOS
("mis-aligned LINKEDIT string pool").

## Engine discovery

`3sx.app` is located as the newest (by `ENGINE_VERSION`, line 1 UTC ISO
build time; missing counts as oldest, ties prefer the earlier entry) of:

1. `~/Library/Application Support/CrowdedStreet/3SX/engine/3sx.app`
   (installed by the launcher's updater)
2. an embedded `Contents/Resources/engine/3sx.app` — only when the launcher is
   a combined bundle (standalone launcher bundles have no match here)
3. `3sx.app` beside the launcher app
4. `/Applications/3sx.app`
5. `~/Applications/3sx.app`

Config and mappings are placed under `get_pref_path()`: portable mode first
(if the discovered engine's `Contents/Resources/config/` directory exists)
then
`~/Library/Application Support/CrowdedStreet/3SX`
(the standard SDL `GetPrefPath` location). Logs and engine installs always
use the standard path.

## Preference directory

The launcher uses two different base paths:

| Item | Path | Base |
|------|------|------|
| Config | `<pref>/config` | `get_pref_path()` — portable or standard |
| Mappings | `<pref>/mappings.ini` | `get_pref_path()` — portable or standard |
| ROM file | `<pref>/resources/SF33RD.AFS` | `get_pref_path()` — portable or standard |
| Engine logs | `<stdpref>/logs/engine-{stdout,stderr}.log` | `standard_pref_path()` — **always** standard |
| Engine install | `<stdpref>/engine/` (updater) | `standard_pref_path()` — **always** standard |

`get_pref_path()` checks for a `config/` directory inside the discovered
engine's `Contents/Resources` first (portable mode), then falls back to
`standard_pref_path()` = `~/Library/Application Support/CrowdedStreet/3SX`.

`standard_pref_path()` is always `~/Library/Application Support/CrowdedStreet/3SX`
— logs and the engine install directory use this regardless of portable mode.

## Updater

The updater fetches the latest stable engine from `gootecks/3sxtra` via the
GitHub Releases API (`/releases/latest`). The current stable channel provides
a universal `3SX-<sha>-macos-universal.zip`. It downloads and expands the
archive with `ditto` into `~/Library/Application Support/CrowdedStreet/3SX/engine`,
then strips quarantine attributes
(`xattr -dr com.apple.quarantine`).

**The launcher updates the engine only** — there is no launcher self-update
mechanism. To update the launcher, download the new binary and replace the
application bundle manually.

## ROM (SF33RD.AFS)

See [docs/features.md](features.md) for the ROM panel walkthrough.

The ROM panel probes for `SF33RD.AFS` (case-insensitive) in 6 stages:
pref resources → engine portable dirs → 3sxw layouts → launcher adjacency →
user download/game directories → mounted disc images (via `hdiutil`).

Import copies the file (or extracts from ISO via `hdiutil attach -readonly`)
into `<pref>/resources/` (uses `get_pref_path()` — portable or standard) and reports whether its SHA-256 matches the known
dump (`f9fa50f3…5604`).

## Support commands

```sh
<launcher binary> --diagnose   # JSON: engine candidates, chosen engine, ROM status
<launcher binary> --launch     # start the engine without the UI
```

The engine is launched via `/usr/bin/open -n`. Its stdout/stderr go to
`~/Library/Application Support/CrowdedStreet/3SX/logs/engine-stdout.log` and
`engine-stderr.log` (always standard pref, regardless of portable mode).

## Release builds

macOS builds produce a **bare universal binary** (no `.app`, no `.dmg`, no
codesign/notarization). The `tauri.macos.conf.json` config (`productName:
3SXtra`, `identifier: com.gootecks.3sxtra`, `minimumSystemVersion: 13.0`) is
currently only partially exercised. A signed `.app`/`.dmg` distribution would
require Developer ID + notarization setup.