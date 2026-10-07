# Architecture

The launcher is a **Tauri 2** desktop app: a Rust backend linked via IPC to a
React 19 + TypeScript + Vite frontend, rendered in the system WebView
(WebKit on macOS, WebView2 on Windows, WebKitGTK on Linux).

```
┌─────────────────────────────────────────────┐
│  Frontend (React 19 + Vite)                 │
│  ┌─────────────────────────────────────────┐│
│  │  App.tsx (tab state, settings UI, news) ││
│  │  components/Remapper.tsx   (pad UI)     ││
│  │  components/RomPanel.tsx  (ROM import)  ││
│  └─────────────────────────────────────────┘│
│                    ║                         │
│               @tauri-apps/api/core.invoke()   │
│                    ║                         │
├────────────────────╬─────────────────────────┤
│  Tauri IPC (JSON-RPC over IPC)               │
├────────────────────╬─────────────────────────┤
│  Backend (Rust)   ║                          │
│  ┌────────────────╨────────────────────────┐ │
│  │  lib.rs — 14 Tauri commands, 1 event    │ │
│  │  engine.rs — macOS .app discovery/launch│ │
│  │  paths.rs  — pref/install/root resolution│ │
│  │  rom.rs    — SF33RD.AFS search/import   │ │
│  │  updater.rs — release fetch/download    │ │
│  │  main.rs   — entry point                │ │
│  └─────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
```

**Total Rust backend**: ~1,360 lines (main.rs 6, lib.rs 298, engine.rs 215,
paths.rs 123, rom.rs 385, updater.rs 340).

**Total TypeScript frontend**: ~930 lines (App.tsx 656, Remapper.tsx 153,
RomPanel.tsx 113, main.tsx 10).

---

## Tauri IPC Surface

The backend registers **14 commands** and emits **1 event**:

### Commands

| Command | Arguments | Returns | Module |
|---------|-----------|---------|--------|
| `is_game_installed` | — | `bool` | lib.rs |
| `launch_game` | — | `Result<String, String>` | lib.rs (calls `engine::launch`) |
| `get_config` | — | `Vec<GameConfig>` | lib.rs → `read_flat_ini(paths::get_config_file_path())` |
| `save_config` | `key: String, value: String` | — | lib.rs → ini write |
| `get_mappings` | — | `Vec<GameConfig>` | lib.rs → `read_flat_ini(paths::get_mappings_file_path())` |
| `save_mapping` | `player: String, action: String, input: String` | — | lib.rs → ini write |
| `get_rom_status` | — | `RomStatus` | rom.rs |
| `find_rom_candidates` | — | `Vec<RomCandidate>` | rom.rs |
| `import_rom` | `path: String` | `ImportResult` | rom.rs |
| `check_updates` | — | `Option<UpdateManifest>` | updater.rs → GitHub Releases API |
| `download_and_extract_archive` | `url, extractPath, markerFile, stripRoot, versionId` | — | updater.rs |
| `check_file_exists` | `path: String` | `bool` | lib.rs |
| `get_local_version` | — | `String` | lib.rs |
| `get_launcher_build_date` | — | `String` | lib.rs → compile-time `BUILD_DATE` env |

### Event

| Event | Payload | When |
|-------|---------|------|
| `download-progress` | `f64` (0–100) | During archive download |

---

## Engine Discovery (`engine.rs`)

The module is **unconditional** — support helpers (version comparison,
candidate iteration) compile on all platforms. Platform-specific branches
for macOS `.app` layout and `open -n` launch use `#[cfg(target_os = "macos")]`
where needed; `#[cfg_attr(not(target_os = "macos"), allow(dead_code))]`
suppresses unused‑code warnings on non‑macOS.

The engine is a separate `3sx.app` bundle. Discovery probes 5 candidate
locations in fixed order, selecting the **newest** by `ENGINE_VERSION` (line 1
of `Contents/Resources/ENGINE_VERSION`, an ISO-8601 UTC timestamp). Missing
timestamp counts as oldest; ties go to the earlier candidate.

1. `<pref>/engine/3sx.app` — installed by the launcher's updater
2. `Contents/Resources/engine/3sx.app` inside the launcher bundle (combined
   bundle only)
3. Beside the launcher app
4. `/Applications/3sx.app`
5. `~/Applications/3sx.app`

Launch uses `/usr/bin/open -n` via LaunchServices; stdout/stderr redirect to
`<standard-pref>/logs/engine-{stdout,stderr}.log` — the **standard** pref
path is always used for logging and engine install, regardless of portable
mode.

---

## Path Resolution (`paths.rs`)

Mirrors the engine's own `paths.c`. Three path categories:

- **Standard pref path**: `SDL_GetPrefPath("CrowdedStreet", "3SX")` → macOS:
  `~/Library/Application Support/CrowdedStreet/3SX`, Windows:
  `%APPDATA%/CrowdedStreet/3SX`, Linux:
  `~/.local/share/CrowdedStreet/3SX`. Used for logs, engine install, and
  updater version tracking.
- **Portable preference/data**: if a `config/` directory exists at the engine's
  base path, the launcher uses that directory for config, mappings, and ROM
  resources (matching the engine's own portable-detection logic). The portable
  path does not replace the standard pref — it only overrides the
  data-file locations (config, mappings.ini, `resources/` for ROM). Engine
  install and logs remain at the standard pref path unconditionally.
- **Install root** (`install_root()`): where downloads and engine updates
  land. macOS: `<standard-pref>/engine/`. Windows/Linux: `<game_root>/` (the
  directory above `tools/launcher/`).

---

## Config Lifecycle (`lib.rs`)

The engine's settings are stored in a **flat `key=value` file** (the `config`
file at `<pref>/config`). There are no INI sections. The launcher uses
`rust-ini` with the `General` / `None` section to read and write it — this is
a hack: `rust-ini` is a section-based library, and the flat format is
maintained only because `rust-ini` treats the first key without a section
header as belonging to a synthetic default section.

**Write path**: `save_config(key, value)` → opens `config`, sets `key =
value`, writes back. The engine overwrites the same file on exit, so there is
a **race condition**: if the launcher writes a change and the engine writes
its own state on shutdown, the launcher's change is lost.

**Mappings** (`mappings.ini`) use the same flat `key=value` storage. The
launcher writes entries as `p1_mapping=Action,Key_<code>` (repeating the key
`p1_mapping` for every action). `rust-ini`'s multimap behavior preserves
duplicate keys — this is intentional and correct for the format.

Both config and mappings are **read in full** on app init (via `get_config` /
`get_mappings` calls in `useEffect`). Individual writes update the in-memory
copy optimistically without re-reading from disk.

### Known contract gaps

The launcher declares 28 settings in `SETTING_CATEGORIES`, but the engine does
not read all of them at startup. Specifically:

- **VSync** (`vsync`): never read at engine boot.
- **Training** (7 options): written to config but the engine's
  `g_training_menu_settings` struct is never populated from config at startup.
  Training-dock toggles from the in-game menu do write, but start-from-scratch
  configs are ignored.
- **GPU Driver** (`gpu-driver`): the engine reads `.gpu` at boot (device 139)
  — the key name `gpu-driver` with values `auto`/`metal`/`vulkan` matches
  engine conventions.
- **Boolean encoding**: the launcher writes `"1"` / `"0"`; the engine's own
  config-parser expects `true` / `false`. Many boolean settings are therefore
  silently ignored.
- **Scale-mode values**: the launcher offers 5 options (`nearest`, `linear`,
  `soft-linear`, `integer`, `square-pixels`) — the engine accepts all 5 plus
  `pixel-art`. No mismatch here, but the full set is wider than the launcher's
  declared enum.

A separate investigation document (`docs/settings-investigation.md`, created
by the parent) details each setting's runtime disposition.

---

## ROM System (`rom.rs`)

Discovers `SF33RD.AFS` (the PS2 disc image of Street Fighter 3: 3rd Strike)
across 6 probe stages. Maximum 4,000 directory entries scanned; user-directory
scans limited to 2 levels deep.

**Expected SHA-256**: `f9fa50f3a124ec9fa9465aa9c8546c2d867887eb39f711a070762a0324ba5604`.

**Import** copies the file (or extracts `THIRD/SF33RD.AFS` from a `.iso` on
macOS via `hdiutil attach -readonly -nobrowse`) to `<pref>/resources/`.
Checksum verification is advisory — a non-matching SHA-256 warns but does not
block.

---

## Updater (`updater.rs`)

**macOS** (stable channel): queries
`api.github.com/repos/gootecks/3sxtra/releases/latest`. Expects a single
universal binary asset named `3SX-<sha>-macos-universal.zip`. Unpacks via
`ditto` (preserves exec bits, symlinks, framework layout), then strips
quarantine attributes from the installed `.app` bundle.

**Windows/Linux** (rolling channel): queries
`api.github.com/repos/gootecks/3sxtra/releases/tags/rolling-pre-release`.
Downloads tar.gz or zip archives, strips an optional root prefix
(`strip_root`), and writes to `install_root()`.

Progress is emitted as `download-progress` events (0–100 `f64`) during
download; the frontend reads these via Tauri event listener.

---

## Network Dependencies

| Dependency | Role | Bundled? |
|------------|------|----------|
| `reqwest` | HTTP client (GitHub API, file downloads) | Static link into binary |
| GitHub API (unauthenticated) | `releases/latest`, `releases/tags/rolling-pre-release`, commit feed | Runtime (60 req/hr rate-limit per IP) |
| Unsplash CDN | Background images for news cards | Runtime (only if CSP allows, which it does — `"csp": null`) |

---

## Release & CI

### Platform Targets

| Label | Runner | Target Triple | Dependencies (critical) |
|-------|--------|--------------|-------------------------|
| `windows-x86_64` | `windows-latest` | `x86_64-pc-windows-msvc` | MSVC toolchain (built-in), WebView2 (OS) |
| `macos-universal` | `macos-latest` | `universal-apple-darwin` (via lipo) | Xcode Command Line Tools |
| `linux-x86_64` | `ubuntu-22.04` | `x86_64-unknown-linux-gnu` | `libwebkit2gtk-4.1-dev`, `libgtk-3-dev`, `libayatana-appindicator3-dev`, `librsvg2-dev`, `libssl-dev` |
| `linux-arm64` | `ubuntu-22.04` | `aarch64-unknown-linux-gnu` (cross) | `gcc-aarch64-linux-gnu`, arm64 ports: all *-dev packages above |

### Build Duration (empirical)

Measured from `.github/workflows/build.yml` runs on `gootecks/3sx-launcher`.

**Run URLs** (click each for step-level timing):

- [Cold build 37598538177](https://github.com/gootecks/3sx-launcher/actions/runs/37598538177) — first CI run, no cached Cargo artifacts
- [Warm build 37631820028](https://github.com/gootecks/3sx-launcher/actions/runs/37631820028) — main @596d1ed, Cargo cache hit
- [Release build 37626781955](https://github.com/gootecks/3sx-launcher/actions/runs/37626781955) — v0.1.0 tag (warm, cache hit)

Times are seconds for the **Build Tauri** step (cargo compilation + linker).
Job totals include step overhead (checkout, dep install, cargo cache, etc.).

**Cold cache (run 37598538177):**

| Platform | Job total | Build step | Notes |
|----------|----------|------------|-------|
| **macos-universal** | 9m49s (589s) | 8m37s (517s) | Two cargo --release builds + lipo; **cold long pole** |
| linux-x86_64 | 5m00s (300s) | 3m55s (235s) | Native gnu compile |
| linux-arm64 | 6m44s (404s) | 4m27s (267s) | Cross gcc; dep install 95s |
| windows-x86_64 | 6m16s (376s) | 5m04s (304s) | MSVC link |
| **Workflow total** | **9m58s (598s)** | — | macOS lane gated finish |

The cold macOS lane (589s job / 517s compile) is the single-longest platform
in this snapshot, not ARM64. The ARM64 lane (404s job / 267s compile + 95s
dep install) is second.

**Warm cache (run 37631820028):**

| Platform | Job total | Build step | Notes |
|----------|----------|------------|-------|
| macos-universal | 4m56s (296s) | 4m12s (252s) | Still two cargo targets + lipo |
| linux-x86_64 | 2m32s (152s) | 1m04s (64s) | Fastest single target |
| linux-arm64 | 7m05s (425s) | 2m04s (124s) | Dep install 4m30s (270s — install variability uninvestigated) |
| windows-x86_64 | 3m34s (214s) | 2m16s (136s) | Cache restore 30s |
| **Workflow total** | **7m07s (427s)** | — | ARM64 lane now gates finish |

With warm Cargo cache, macOS compile dropped to 252s (−51%). ARM64 became the
warm long pole because its apt cross‑dependency install took 270s (the
install‑time variability between runs was uninvestigated).

The macOS universal build's 252s warm includes two separate `tauri build`
invocations (aarch64 + x86_64) followed by `lipo -create` — switching to a
single universal binary via cargo would save the intermediate step but not the
dual compilation.

### Release modes

Two release triggers (both via `release.yml`):

| Mode | Trigger | Tag | Assets | Prerelease? |
|------|---------|-----|--------|-------------|
| Rolling pre-release | `workflow_dispatch` | `rolling-pre-release` (force-moved) | 4 platform binaries | Yes |
| Stable release | `v*` tag push | `v0.1.0`, `v0.2.0`, … | 4 platform binaries | Only if semver pre-release suffix (`v0.1.0-alpha.1`) |

Both build the same 4 platform binaries. The `rolling-pre-release` tag enables
the communiqué release-notes tooling to reference a moving target; stable tags
are permanent.

---

## macOS Bundle & Minimum Version

The Tauri config applies `macOS` `minimumSystemVersion: "13.0"` (from
`tauri.macos.conf.json`, which overrides the base `tauri.conf.json` on macOS).

The bundled product is named **"3SXtra"** with identifier
`com.gootecks.3sxtra` on macOS, while the base config produces **"3SX
Launcher"** / `com.crowdedstreet.3sx-launcher`. This dual identity reflects
the combined `3SXtra.app` that bundles both launcher and engine.

**Packaging gap**: the CI workflow produces raw binaries only (no `.dmg`,
`.app` bundle, or notarization). The `--no-bundle` flag is passed to `npx tauri
build` on all platforms. macOS `.app` bundling and code signing must be done
manually or added to the pipeline. For local development, `npm run tauri build`
(without `--no-bundle`) creates the `.app` bundle normally — this is used for
ad-hoc local testing, where signing is **not required** for running the
built app.

---

## Dependencies (Rust Cargo.toml)

| Crate | Version | Purpose |
|-------|---------|---------|
| `tauri` | 2 | App framework, IPC, window management |
| `tauri-plugin-opener` | 2 | `openUrl` for news links |
| `tauri-plugin-shell` | 2 | Shell access |
| `serde` / `serde_json` | 1 | Serialization for IPC payloads |
| `reqwest` | 0.12 | GitHub API + file downloads (json + stream features) |
| `tokio` | 1 | Async runtime (full features) |
| `rust-ini` | 0.21 | INI read/write (flat config via General section trick) |
| `directories` | 5.0 | User home, config paths |
| `futures-util` | 0.3 | Async stream combinators for download progress |
| `zip` / `tar` / `flate2` | 0.6 / 0.4 / 1 | Archive extraction |
| `sha2` | 0.10 | ROM SHA-256 verification |
| `chrono` | 0.4 | Build-date embedding (build.rs only) |