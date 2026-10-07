# macOS: launcher and engine

The launcher is this app; the engine (`3sx.app`) is built and released
separately from [gootecks/3sxtra](https://github.com/gootecks/3sxtra).
FreeFighter downloads the two independently; the launcher finds and starts
the engine.

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

1. `<pref>/engine/3sx.app` (installed by the launcher's updater)
2. an embedded `Contents/Resources/engine/3sx.app` (only when this app is a
   combined bundle; a standalone launcher app has no match here)
3. `3sx.app` beside this app
4. `/Applications/3sx.app`
5. `~/Applications/3sx.app`

`<pref>` is `~/Library/Application Support/CrowdedStreet/3SX` (the ROM,
config and logs use the portable `config/` directory instead when one
exists, as the engine does).

## Updater

The updater fetches the latest stable engine from `gootecks/3sxtra` via the
GitHub Releases API (`/releases/latest`). The
current stable channel provides a universal `3SX-<sha>-macos-universal.zip`
(`macos-v0.1.0` as of this writing; future semantic releases are picked up
automatically). It downloads and expands the archive with `ditto` into
`<pref>/engine`, then strips quarantine attributes. The launcher itself
updates by replacing the app bundle.

## ROM (SF33RD.AFS)

The ROM panel looks for `SF33RD.AFS` (case-insensitive) in:

1. `<pref>/resources/`
2. each known engine's portable `config/resources/` and `rom/`
3. 3sxw layouts: `resources/` beside a `3SX*.app`, inside it, and `3sxw*` folders
4. beside and inside the launcher app
5. `~/Downloads`, `~/Documents`, `~/Desktop`, `~/Games`, `~/ROMs` (depth ≤ 2,
   also `*.iso`)
6. mounted discs: `/Volumes/*/THIRD/SF33RD.AFS`, `/Volumes/*/SF33RD.AFS`

Import copies the file (or extracts `THIRD/SF33RD.AFS` from an ISO via
`hdiutil`) into `<pref>/resources/` and reports whether its SHA-256 matches
the known dump. The engine's own folder picker accepts a folder containing
`THIRD/SF33RD.AFS` or `SF33RD.AFS`.

## Support commands

```sh
<launcher binary> --diagnose   # JSON: engine candidates, chosen engine, ROM status
<launcher binary> --launch     # start the engine without the UI
```

The engine's stdout/stderr go to `<pref>/logs/engine-stdout.log` and
`engine-stderr.log`.
