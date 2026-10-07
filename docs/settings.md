# Settings Reference

All 28 engine settings exposed by the 3SX Launcher, with engine consumption
contract, known no-ops, and workarounds.

- [Format & Save Contract](#format--save-contract)
- [Setting Matrix](#setting-matrix)
  - [Window (5)](#window)
  - [Rendering (7)](#rendering)
  - [Netplay (6)](#netplay)
  - [Training (7)](#training)
  - [Mods (3)](#mods)
- [Controls / Mappings Contract](#controls--mappings-contract)
- [Known Issues & Workarounds](#known-issues--workarounds)
- [Safe Manual Editing Workflow](#safe-manual-editing-workflow)
- [Investigation](#investigation)

---

## Format & Save Contract

The engine's config file is a **flat key=value** file at
`<pref>/config` — no section headers, one key-value per line.

**The launcher**:
- Reads on startup via `read_flat_ini` (`rust-ini` 0.21.3, sectionless keys
  only, `ListOrderedMultimap` preserving duplicate keys).
- Writes **immediately** on every UI interaction — toggle, keystroke, or select
  change. There is **no Apply / Save** button.
- Uses `save_config(key, value)` → `rust-ini` `with_section(None).set(key,value)`
  → rewrites the **entire file** (dropping comments, normalizing whitespace).
- Bool values: `toggleBool` writes `"1"` or `"0"`. The `isBoolEnabled` helper
  considers both `"1"` and `"true"` as enabled.
- Select values: `<select value={val || options[0].value}>` shows the first
  option when the stored value is absent/empty, but **does not persist it**.
- No debounce on int/string inputs (each keystroke triggers a write —
  transiently empty values can be persisted).

**The engine**:
- `Config_Init` loads the file once at startup (`config.c:295-300`).
- Type-parses each value: `true`/`false` → `CFG_BOOL`, digits → `CFG_INT`,
  anything else → `CFG_STRING`.
- `find_entry`: if a key exists in `default_entries` but the read type does not
  **exactly** match the default's type, the **default is returned**.
- Getters (`Config_GetBool`, `Config_GetInt`, `Config_GetString`): when the key
  has no matching `default_entries` entry, or when the getter's own type check
  fails against the parsed value, the getter returns `false` / `0` / `NULL`
  (not the compiled default). For keys with a default entry, type mismatch
  triggers the `find_entry` default substitution instead.
- `Config_Save()` (at game exit) rewrites the whole file from its own in-memory
  entries — **launcher edits made while the game is running are overwritten**.

**Key consequence**: all boolean settings written by the launcher are stored as
`1`/`0` (CFG_INT), but the engine expects `true`/`false` (CFG_BOOL). Every
launcher boolean toggle is **silently ignored** unless the user manually edits
the file.

---

## Setting Matrix

**Source revision**: engine source `9cd552a` (macOS stable tag `macos-v0.1.0`).
Verdicts are from static code analysis (anchors in comments). **Runtime-tested
subset** (on installed build `cc69ae0`): `fullscreen`, `window-size`, `vsync`,
and exit clobber — those rows are marked "(verified)". Other verdicts are
source-supported only and may differ in other engine builds.

Legend:
- **Verdict**: WORK = value reaches engine; PARTIAL = honored with caveats;
  NO-OP = written but ignored due to type mismatch or no consumer; INERT = no
  consumer at boot (runtime-side only).
- **Engine default**: the default value compiled into the engine (used when
  config key is missing or type-mismatched).
- **Restart**: whether the engine must be restarted for the change to apply.
  (All boot-read settings require restart; no live-reload exists.)

### Window

| # | Key | Type | Default | Consumer | Verdict | Restart |
|---|-----|------|---------|----------|---------|---------|
| 1 | `fullscreen` | bool | `true` (BOOL) | Window creation | **NO-OP** — `"0"` → CFG_INT ≠ BOOL → engine returns default `true`; window is always fullscreen after a launcher toggle | yes |
| 2 | `window-width` | int | `1920` (INT) | Window size | **PARTIAL** — written value is honored, but only when not fullscreen; `--window-size` CLI flag overrides | yes |
| 3 | `window-height` | int | `1080` (INT) | Window size | **PARTIAL** — same caveat as width | yes |
| 4 | `vsync` | bool | — (no default) | **Never read at boot** | **NO-OP** — `Config_GetBool(CFG_KEY_VSYNC)` is called **nowhere** in the engine; only written at exit. Always `OFF (OpenGL, native pacing)` in logs | N/A |
| 5 | `skip-intro` | bool | `false` (BOOL) | attract/demo/opening | **NO-OP** — `"1"` → CFG_INT ≠ BOOL → default `false` | yes |

### Rendering

| # | Key | Type | Default | Consumer | Verdict | Restart |
|---|-----|------|---------|----------|---------|---------|
| 6 | `scale-mode` | select | `"nearest"` (STRING) | Scale mode selection | **WORK** — all 5 launcher options accepted (`nearest`, `linear`, `soft-linear`, `integer`, `square-pixels`); engine also accepts `"pixel-art"` (not in launcher) | yes |
| 7 | `hd-stages` | bool | — (no default) | Modded stage loader | **NO-OP** — `"0"`/`"1"` → `Config_GetBool` returns false → HD stages cannot be enabled | yes |
| 8 | `bezel-enabled` | bool | — (no default) | Bezel overlay | **NO-OP** — source consumer at `sdl_app.c:1086-1087` reads the key, but `0`/`1` → CFG_INT vs BOOL default → value ignored | yes |
| 9 | `shader-path` | string | `""` (STRING) | Shader config loader | **WORK** — file path is read. ⚠️ Windows-style backslashes (`C:\shaders\x`) are mangled by INI parser's escape processing | yes |
| 10 | `draw-players-above-hud` | bool | — (no default) | HUD draw order | **NO-OP** — type mismatch on GetBool | yes |
| 11 | `renderer` | select | `"gl"` (STRING) | Renderer selection | **WORK** — one of `gl`/`gpu`/`sdl`/`classic`; `--renderer` CLI flag overrides at boot | yes |
| 12 | `gpu-driver` | select | `"auto"` (STRING) | GPU device init | **WORK** — only effective when `renderer=gpu`; `auto` → macOS uses Metal | yes |

### Netplay

| # | Key | Type | Default | Consumer | Verdict | Restart |
|---|-----|------|---------|----------|---------|---------|
| 13 | `lobby-display-name` | string | — (no default) | Identity / netplay UI | **WORK** — string value reaches netplay components | yes (netplay init) |
| 14 | `netplay-ft` | int | `2` (INT) | Netplay game logic | **WORK** — integer clamped 1–10 | yes |
| 15 | `netplay-region-lock` | bool | — (no default) | Netplay UI region filter | **NO-OP** — type mismatch on GetBool | yes |
| 16 | `netplay-max-ping` | int | — (no default) | Netplay UI ping filter | **WORK** | yes |
| 17 | `netplay-block-wifi` | bool | — (no default) | Netplay UI WiFi check | **NO-OP** — type mismatch on GetBool | yes |
| 18 | `lobby-auto-connect` | bool | `true` (BOOL) | Auto-join lobby | **NO-OP** — `"0"` → CFG_INT ≠ BOOL → default `true`; always auto-connects | yes |

### Training

| # | Key | Type | Default | Consumer | Verdict | Restart |
|---|-----|------|---------|----------|---------|---------|
| 19 | `training-hitboxes` | bool | `true` (BOOL) | Training menu / HUD (runtime) | **INERT** — not read at boot; runtime reads `g_training_menu_settings` struct initialized to `{0}` at `sdl_app.c:591`; config is **write-only** | N/A |
| 20 | `training-pushboxes` | bool | `true` (BOOL) | Training menu / HUD (runtime) | **INERT** — same as #19 | N/A |
| 21 | `training-hurtboxes` | bool | `true` (BOOL) | Training menu / HUD (runtime) | **INERT** — same as #19 | N/A |
| 22 | `training-advantage` | bool | `false` (BOOL) | Training menu (runtime) | **INERT** — not read at boot; runtime in-memory only | N/A |
| 23 | `training-stun` | bool | `true` (BOOL) | Training menu (runtime) | **INERT** | N/A |
| 24 | `training-frame-meter` | bool | `true` (BOOL) | Frame display (runtime) | **INERT** | N/A |
| 25 | `training-inputs` | bool | `true` (BOOL) | Input display (runtime) | **INERT** | N/A |

### Mods

| # | Key | Type | Default | Consumer | Verdict | Restart |
|---|-----|------|---------|----------|---------|---------|
| 26 | `modded-bgm-enabled` | bool | `false` (BOOL) | Modded BGM init | **NO-OP** — `"1"` → CFG_INT ≠ BOOL → default `false`; cannot enable | yes |
| 27 | `modded-voice-enabled` | bool | `false` (BOOL) | Modded voice init | **NO-OP** — same as #26 | yes |
| 28 | `arcade-balance` | bool | — (no default) | Arcade balance feature | **NO-OP** — type mismatch on GetBool | yes |

**Tally**: 9 WORK (2 PARTIAL), 12 NO-OP, 7 INERT.

---

## Controls / Mappings Contract

The Controls tab writes to `<pref>/mappings.ini` in flat format.

**Format**: `p1_mapping=<Action>,<Input>` — one line per binding, all lines
share the bare key `p1_mapping`.

**Engine expectations**:
- **Action** must be a full name from `game_actions[]`:
  `Up` / `Down` / `Left` / `Right` / `Light Punch` / `Medium Punch` /
  `Hard Punch` / `Light Kick` / `Medium Kick` / `Hard Kick` / `Start` / `Select`
- **Input** must use SDL scancode naming:
  `Key <scancode>` (e.g. `Key W`, `Key Up`, `Key Keypad 4`, `Key Space`),
  or `Joy Button N` / `Joy Axis N±` / `Joy Hat N Up` for gamepad.

**What the launcher actually writes**:
- `Key_<e.code>` where `e.code` may be `KeyW`, `Space`, or `ArrowUp`.
  Result: `Key_W`, `Key_Space`, `Key_ArrowUp`. The engine expects
  `Key W` (space separator, underscore-free) and SDL scancode names
  (`Key Up`, `Key Space`, `Key Keypad 4`).
- Actions use short codes: `LP` / `MP` / `HP` / `LK` / `MK` / `HK` (engine
  expects full names).

**Result**: **keyboard bindings created by the launcher have zero effect
in-game** — `get_action_flag("LP")` returns 0 and the input token
(e.g. `Key_Space`) is unrecognized — `get_input_id` returns
`INPUT_ID_UNKNOWN`.

Only **manually edited** `mappings.ini` lines using the correct grammar will
work from the launcher. The engine's own in-game controls configuration menu
writes valid mappings and is the recommended alternative.

---

## Known Issues & Workarounds

### 1. All boolean settings are no-ops
**Root cause**: launcher writes `0`/`1` but engine requires `true`/`false`.
**Manual workaround**: edit `<pref>/config` with a text editor and ensure
boolean values are the literal strings `true` or `false`. The launcher will
re-read and display them, and toggling again will revert to `0`/`1`.

This workaround applies only to the **11 boolean keys the engine reads at
startup** (identified in the Matrix above as having a default entry with
BOOL type). The 7 `training-*` booleans are never read from config at boot
(INERT), and `vsync` has no startup consumer — editing their values to
`true`/`false` has no effect either.

### 2. Immediate per-keystroke writes
Every keystroke in int/string inputs calls `save_config`. Typing `-` or
backspace through the entire value persists an empty or negative string.
**Workaround**: no mitigation in the launcher currently; use manual editing
for int values.

### 3. Exit-time clobber (verified)
The engine overwrites `<pref>/config` at exit (`Config_Save`). A launcher edit
made while the game is running (e.g. `window-width`) is visible on disk mid-game
but **reverts to the pre-launch value** after the engine exits. Edits to keys
the engine did not read at startup are deleted entirely.
**Workaround**: edit config only when the engine is not running.

### 4. Inline comments survive as value text; full-line comments dropped on rewrite
`save_config` is a whole-file rewrite through `rust-ini`. When the launcher
reads a line with an inline comment, e.g. `fullscreen=false # inline`, the
comment text (` # inline`) is included as part of the value. If the user
edits a **different** key via the UI, `save_config` writes the original
back unchanged — the comment is preserved. If the same key (`fullscreen`)
is toggled, the value is replaced with `0` or `1`, and the inline comment
disappears. Full-line comments (lines starting with `#` or `;`) are dropped
by the parser and never re-emitted.

Additionally, `rust-ini`'s default parser has escape processing enabled. A
value like `C:\shaders\custom.slangp` is loaded as `C:shaderscustom.slangp`
(backslashes consumed as escape characters). The writer then re-escapes; the
damage is done on the first read-through-save cycle.
**Workaround**: keep a backup of manual edits; avoid using the launcher UI
for `shader-path` if it contains backslashes.

### 5. Malformed section header can cause data loss (source-confirmed)
If the config file contains an **unterminated** section header (e.g.
`[broken` on its own line before `fullscreen=false`), `rust-ini`'s parser
returns an `Err`. The launcher's `save_config` uses `.unwrap_or_default()`,
which discards the error and writes a **new file containing only the edited
key** — all other settings are lost. This is a narrow edge case: properly
formed INI files, section headers (even unused ones), and inline comments all
parse without error.

### 6. Select fallback does not persist
When a stored value is absent, the `<select>` UI shows the first option, but
does **not** write it to disk. The engine reads whatever `config` had (no
value = engine default). To ensure a select takes effect, make an explicit
selection in the UI.

### 7. VSync toggle is decorative
The `vsync` key is **never read** by the engine — it is only written at exit.
The setting has no effect regardless of value.

### 8. Training mode settings are inert at boot
The 7 training keys are not consumed during boot — the training menu uses an
in-memory struct initialized to `{0}`. The engine's Training menu writes to
the struct (not config) and calls `Config_SetBool`, so config persistence for
these keys is write-only. Values only take effect through the in-game Training
menu, not the launcher UI.

### 9. Engine truncation at spaces
The engine's `%127s` sscanf reads only until the first space. A shader path
or display name containing spaces is silently truncated.
**Workaround**: avoid spaces in string setting values.

---

## Safe Manual Editing Workflow

To make a setting take effect reliably:

1. **Close the engine** if running.
2. Open `<pref>/config` in a text editor.
3. For booleans, write `true` or `false` (not `1`/`0`).
4. For strings with backslashes or spaces, test carefully — paths should
   use forward slashes where possible.
5. Save the file.
6. Start the engine (via launcher or directly).
7. **Do not use the launcher's Settings tab** for keys you've hand-edited,
   as toggling will rewrite with the wrong encoding.

The launcher will display whatever values are on disk at its next launch.

---

## Investigation

See [docs/settings-investigation.md](settings-investigation.md) for the
detailed root-cause analysis (boolean type encoding, exit clobber,
VSync/scale-mode/input mapping audits).