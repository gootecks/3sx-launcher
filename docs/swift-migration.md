# Swift/MacOS Native Migration Assessment

## Scope

This document evaluates replacing the **Tauri 2 (Rust + React) WebView host
shell** on macOS with a native **Swift + AppKit** host, keeping the **3SX
engine** (C/C++ SDL) unchanged. The assessment is grounded in measured CI
build costs, a complete inventory of the backend surface, and the
cross-platform requirement that Windows and Linux builds continue unchanged.

No Swift implementation has been written or measured. All references to Swift
feasibility describe *what would need to be built*, not runtime performance.

---

## Current Architecture (Baseline)

The launcher's Rust backend is **~1,360 source lines** across 6 files:

| File | Lines | Role |
|------|-------|------|
| `main.rs` | 6 | Entry point, calls `launcher_lib::run()` |
| `lib.rs` | 298 | 14 Tauri commands, config/mapping read/write, `--diagnose`/`--launch` |
| `engine.rs` | 215 | macOS `3sx.app` discovery, version comparison, `open -n` launch |
| `paths.rs` | 123 | Pref path, install root, game root, portable-mode detection |
| `rom.rs` | 385 | SF33RD.AFS search (6 probe stages), ISO mount, SHA-256 import |
| `updater.rs` | 340 | GitHub Releases API fetch, zip/tar.gz download + extract, macOS `ditto` |

See [`docs/features.md`](features.md) for the complete frontend feature
inventory.

---

## macOS Native Replacement Inventory

### Foundation / AppKit Replacements

| Rust Crate | macOS Native | Complexity |
|------------|-------------|-----------|
| `directories` | `FileManager.default.urls(for:.applicationSupportDirectory, …)` + `NSHomeDirectory()` | Trivial — path constants |
| `sha2` | `CryptoKit SHA256` (`import CryptoKit`) | Trivial — one-liner |
| `rust-ini` | Faithful line‑preserving parser — comments, blank lines, escapes, duplicate keys, unknown keys must survive round‑trip unchanged | Not trivial — the flat config has no section header, and the engine's parser may tolerate specific whitespace/escape conventions. A naive `[String: String]` split/lossy would break the existing file. Must re-read `lib.rs:read_flat_ini`† and `lib.rs:save_config` behaviors to guarantee byte‑identical non‑edited lines |
| `reqwest` (GitHub API) | `URLSession` async/await | Moderate — no streaming JSON‑by‑chunk equivalent to `reqwest::Response::chunk()` |
| `reqwest` (download progress) | `URLSessionDownloadDelegate` callbacks | Moderate — maps to current Tauri `download-progress` event |
| `zip` / `tar` / `flate2` | macOS migration only: engine updates arrive as `ditto`‑made `.zip` and the updater already uses `ditto`‑via‑`Command` to extract (not the Rust `zip` crate on macOS). On‑macOS extraction already delegates to `/usr/bin/ditto`. A pure‑Swift host would call `ditto` or `/usr/bin/unzip` the same way; no Rust‑crate replacement needed on this platform. For rolling‑pre‑release (tar.gz, zip‑with‑strip‑root) on macOS the same `Process` route works. | Low — delegate to system tools |
| Tauri IPC (14 commands, 1 event) | WKWebView message handler + `evaluateJavaScript` | **Structural** — every `invoke()` call in the React frontend gets a replacement message‑handler on the Swift side |

†`read_flat_ini` propagates parse errors upward (`map_err` + `?`) — it does
not silently discard data. `save_config` uses `unwrap_or_default()` on load
failure, which replaces the entire file with a fresh key‑only write (losing
comments and unrecognised lines). The Swift replacement should preserve all
comments, blank lines, escapes, unknown keys, and duplicate keys — matching
the engine's own tolerance and better than the current writer.

### What Stays (Not Replaced)

- **3SX engine (C/C++ SDL)** — separate binary; discovery and `open -n`
  launch logic would be ported (rewritten in Swift) but the engine itself is
  unchanged.
- **React/TypeScript frontend** — could be kept in WKWebView (macOS system
  WebView is the same WebKit engine Tauri uses). Switching to SwiftUI is a
  **separate project** — a full UI rewrite with its own scope, cost, and
  quality targets. This document evaluates the host‑shell replacement only.

---

## Build Cost: Current CI Baseline

Measured from CI runs on `gootecks/3sx-launcher`:

**Run URLs** (click each for step-level timing):

- [Cold build 37598538177](https://github.com/gootecks/3sx-launcher/actions/runs/37598538177) — first CI run after repo creation, no cached Cargo artifacts
- [Warm build 37631820028](https://github.com/gootecks/3sx-launcher/actions/runs/37631820028) — main @596d1ed, Cargo cache hit
- [Release build 37626781955](https://github.com/gootecks/3sx-launcher/actions/runs/37626781955) — v0.1.0 tag (warm, cache hit)

All times are wall‑clock seconds for the **Build Tauri** step (cargo
compilation + linker). The workflow builds all 4 platforms in parallel.

### Cold cache (run 37598538177)

| Platform | Job total | Build step | Notes |
|----------|----------|------------|-------|
| **macos-universal** | 9m49s (589s) | 8m37s (517s) | Two cargo --release builds + lipo; **long pole cold** |
| windows-x86_64 | 6m16s (376s) | 5m04s (304s) | LLVM / MSVC link |
| linux-x86_64 | 5m00s (300s) | 3m55s (235s) | Native gnu compile |
| linux-arm64 | 6m44s (404s) | 4m27s (267s) | Cross gcc; dep install 95s |
| **Workflow total** | **9m58s (598s)** | — | macOS lane gated the finish |

The cold macOS universal lane is the single‑longest step (517s compile +
overhead = 589s job). ARM64 cross‑compile is second at 267s + 95s dep
install = 404s job.

### Warm cache (run 37631820028)

| Platform | Job total | Build step | Notes |
|----------|----------|------------|-------|
| **macos-universal** | 4m56s (296s) | 4m12s (252s) | Still two cargo targets + lipo |
| windows-x86_64 | 3m34s (214s) | 2m16s (136s) | Cache restore 30s |
| linux-x86_64 | 2m32s (152s) | 1m04s (64s) | Fastest, dep install 56s |
| linux-arm64 | 7m05s (425s) | 2m04s (124s) | Dep install 4m30s (270s — install variability uninvestigated) |
| **Workflow total** | **7m07s (427s)** | — | ARM64 lane gated the finish |

With a warm Cargo cache, macOS shrank from 517s→252s compile (‑51%). The
ARM64 lane became the **warm long pole** at 425s job total, primarily from
re‑installing apt arm64 cross‑dependencies (270s — the install‑time
variability between runs was uninvestigated), not from compilation.

**npm/Vite build**: measured per platform across runs (negligible relative to
Rust compile). The frontend build time is constant across cold/warm cache
states.

---

## Build Cost Discussion

No Swift build times have been measured for this project. The following are
*structural contrasts*, not estimates:

1. **macOS lane removes Cargo compilation.** The 252s (warm) / 517s (cold) Rust
   compile of the Tauri host crate, `reqwest`, `zip`, `sha2`, etc. is the
   dominant cost of the macOS lane today. A Swift/AppKit host replaces that
   compilation with `xcodebuild` (or `swift build`) of an approximately
   equivalent dependency surface.

2. **npm build is unchanged.** The Vite + tsconfig compilation is needed
   whether the host is Tauri‑Rust or Swift‑AppKit (assuming the same React
   frontend is kept in WKWebView). The actual frontend build time is
   negligible relative to the Rust compile step.

3. **Universal binary CI cost is unchanged.** The CI already performs two
   sequential Tauri builds (`aarch64-apple-darwin` + `x86_64-apple-darwin` +
   `lipo`). An Xcode universal build would also compile twice. Neither
   approach is dominant over the other.

4. **No binary‑size claims.** The current release binary (12,521,872B
   universal FAT: aarch64 + x86_64) is a stripped multi‑arch binary. No Swift
   counterpart exists for comparison; binary size is not a constraint.

### Effect on total workflow wall clock

In the **warm cache snapshot**, the ARM64 lane (425s total, of which 270s is
apt‑dep install and 124s is compile) gates the finish. Replacing the macOS
lane alone with a native Swift build would *not reduce total workflow
wall‑clock time in that snapshot* — the workflow still waits for ARM64.

In the **cold snapshot**, the macOS lane (589s) is the gating factor. A faster
macOS build would reduce cold workflow time, though by how much depends on the
Swift build (unmeasured).

The long‑term critical‑path status of each lane depends on CI runner
availability, caching improvements, and build‑tool evolution — no guarantee
exists that a particular lane will always bottleneck.

---

## Architectural Options

| Option | Frontend | macOS Backend | Windows/Linux Backend | Migration Cost |
|--------|----------|---------------|----------------------|----------------|
| **A: Status quo** | React in WKWebView | Rust (Tauri) | Rust (Tauri) | None |
| **B: Swift host + React** | React in WKWebView | Swift (explicit new bridge, 14 commands) | Rust (Tauri, unchanged) | Moderate |
| **C: Swift host + SwiftUI** | SwiftUI | Swift | Rust (Tauri, unchanged) | **High** — UI rewrite + IPC + full regression |

**Option B** replaces the Tauri macOS host with a thinner Swift/AppKit shell
that:
- Implements a **purpose‑built WKWebView message bridge** exposing *only* the
  14 commands the current React frontend invokes (no emulation of Tauri's
  internal IPC layer).
- Handles frameless‑window chrome, minimize/maximize/close, drag region
- Provides the existing engine‑discovery, config, ROM, and updater logic
  (ported to Swift, see replacement table above)
- Leaves the React + Vite frontend unchanged inside WKWebView

**Option C** additionally replaces the React frontend with SwiftUI
views. This is a separate, larger project — a full UI rewrite with its own
milestones, testing scope, and platform‑behavior targets. This document
does **not** evaluate build‑time tradeoffs for a SwiftUI rewrite since no
implementation exists.

### Recommendation: stabilize, then build the bridge

1. **Fix the engine‑config mismatches first** (boolean encoding, training
   no‑ops, VSync never‑read — documented in
   [`docs/settings-investigation.md`](settings-investigation.md) and
   [`docs/settings.md`](settings.md)). Debugging a static‑config contract
   against the C engine is easier in Rust where the engine's C codebase is
   directly inspectable. Porting a moving spec multiplies the investigation
   surface.

2. **Extract a testable Swift domain library**
   (`3SXLauncherKit.framework` or SPM package) that implements the domain
   logic: engine discovery, paths, config read/write, ROM search, updater.
   Develop and CI‑verify this alongside the existing Rust backend — the Rust
   backend stays as the reference for cross‑platform builds.

3. **Build the minimal Swift host shell** with an explicit WKWebView message
   bridge for the 14 commands. The Linux/Windows CI continues using the
   unchanged Tauri/Rust backend. Swift replaces only the macOS host crate.

4. **Do not emulate Tauri internals.** The bridge maps *exactly* the command
   set the React frontend calls. No shim, no Tauri‑API polyfill. The frontend
   code changes from `invoke("command", args)` to a
   `window.webkit.messageHandlers.launcher.postMessage({command, args})`
   pattern in the TypeScript adapter.

7. **Keep the flat‑config contract.** The engine reads `<pref>/config` as flat
   `key=value`. The Swift parser must not add sections, JSON, or a database.
   It must also preserve comments, blank lines, unknown keys, and escape
   sequences during write — the current `rust-ini` writer drops comments and
   normalises whitespace, which a faithful replacement should improve on.

---

## Key Risks

1. **Dual‑backend maintenance.** After a macOS‑native migration, every
   platform‑specific fix (config parsing, engine discovery, network behavior)
   needs attention in Swift for macOS and in Rust for Windows/Linux, unless
   the domain logic is kept in a shared Rust core.

2. **No build‑speed improvement if ARM64 remains bottleneck.** In the warm
   snapshot, ARM64 gates total workflow at 7m07s regardless of macOS lane
   speed. Improving ARM64 CI (faster runner, cached apt packages, native ARM
   runner) would reduce total workflow time independently of the macOS
   migration.

3. **Notarization and code signing gap.** The existing CI produces unsigned
   raw binaries only. A Swift/AppKit app requires the same Apple Developer
   signing pipeline (`codesign`, `notarytool`) — this is orthogonal to the
   migration but must be added either way.

4. **Faithful config parser is non‑trivial.** The current Rust backend
   *discards* comments and malformed lines (via `rust-ini`'s default
   `unwrap_or_default`). A Swift replacement must be equal or better at
   preservation — no line that the engine writes should vanish. This rules out
   a naive `split("=")` dictionary and demands a line‑preserving round‑tripper.

---

## Summary

| Dimension | Current (Tauri 2 + Rust + React) | Option B (Swift host + React in WKWebView) |
|-----------|----------------------------------|---------------------------------------------|
| macOS build time (warm) | 252s compile (run 37631820028) | Unmeasured — `xcodebuild` replaces `cargo` |
| Cold long pole | macOS 589s | Unknown until built |
| Warm long pole | ARM64 425s | ARM64 still 425s — unchanged |
| macOS required tooling | Rust toolchain, Node.js | Xcode, Node.js |
| Code signing pipeline | None | None — orthogonal addition |
| Multi‑platform | One Rust backend for 4 targets | Rust for Windows/Linux + Swift for macOS |
| Config parser fidelity | Write-side loses comments/unknown keys via `unwrap_or_default` | Must preserve all lines faithfully |
| IPC surface | 14 Tauri commands | 14 WKWebView message handlers |
| Frontend | React 19 in WKWebView | React 19 in WKWebView (unchanged) |

**Bottom line**: a pure‑Swift macOS host that keeps the existing React
frontend in WKWebView is architecturally feasible and would eliminate the
Cargo compile step from the macOS CI lane. The practical CI benefit depends
on the ARM64 cross‑build situation (currently the warm bottleneck). The
config parser must be built with line‑preserving care. A SwiftUI rewrite is a
separate, larger project not evaluated here. The recommended sequence is:
stabilise the engine‑config contract first, extract a testable Swift domain
library, then replace the Tauri host with a minimal WKWebView bridge — not an
emulation of Tauri internals.