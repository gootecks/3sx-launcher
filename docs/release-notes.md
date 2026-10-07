# Release Notes & Changelog

Uses [Communiqué](https://github.com/jdx/communique) via the
[LLMGateway](https://llmgateway.io) to auto-generate release notes and
changelog entries from git history.

**Important:** This tooling manages release *notes* only — it does **not**
create tags, build binaries, or upload assets.  The CI workflow handles the
mechanical release process; run these tasks after a workflow run finishes to
replace only the note body.

## Prerequisites

- [mise](https://mise.jdx.dev) — pins `communique` and `fnox` versions
- [fnox](https://fnox.jdx.dev) — resolves secrets from 1Password;
  run `fnox check` to verify configuration
- [1Password CLI](https://developer.1password.com/docs/cli) — `op` must be
  authenticated and the `Developer/LLMGateway` item accessible

## Workflow

This is separate from the **3sxtra engine** release process.  The launcher
has two release modes:

### Rolling pre-release (manual dispatch)

1. Running the workflow via **`workflow_dispatch`** from the GitHub UI
   triggers `.github/workflows/release.yml`.
2. It builds all platform binaries and force-moves the `rolling-pre-release`
   tag to the commit, then creates/updates the **rolling-pre-release** GitHub
   release (prerelease, not latest).
3. After the workflow finishes, add LLM-generated notes:

   ```sh
   mise run release-notes rolling-pre-release
   mise run release-notes rolling-pre-release <previous-sha-or-tag>
   ```

### Stable release (tag push)

1. Pushing a `v*` tag (e.g. `v0.1.0`, `v0.2.0`) triggers
   `.github/workflows/release.yml`.
2. It builds all platform binaries and creates/updates a **stable** GitHub
   release at that tag name (latest, non-prerelease).  The rolling tag is
   **not** moved.
3. Tags with a semver pre-release suffix (e.g. `v0.1.0-alpha.1`,
   `v0.1.0-beta.1`) are created as **prereleases** instead of latest stable.
4. After the workflow finishes, add LLM-generated notes:

   ```sh
   mise run release-notes v0.1.0
   mise run release-notes v0.1.0 <previous-tag>
   ```

The `--preserve-sections` flag keeps the Downloads table and other non-communique
content intact by only rewriting content between
`<!-- communique:start -->` / `<!-- communique:end -->` HTML comment markers.
Stable release bodies include these markers from the start; for rolling releases
they are added on first communique invocation.

## Secret mapping

Communiqué reads `OPENAI_API_KEY` first, then `LLM_API_KEY`.  The wrapper
`support/communique-release`:

1. Resolves `LLMGATEWAY_API_KEY` from the fnox `release` profile
   (`op://Developer/LLMGateway/communique invoicegen`)
2. Re-exports it as `LLM_API_KEY` for communique to find
3. Unsets `OPENAI_API_KEY` in the subprocess only (no global side effects)

For `--github-release`, the wrapper also reads `GITHUB_TOKEN` (or `GH_TOKEN`,
or `gh auth token`) and validates the release exists before calling the LLM.

## Commands

### Preview (dry-run, no publishing)

```sh
# Preview against the root baseline (cumulative — includes all history):
mise run preview rolling-pre-release

# Preview against a specific baseline (narrower range):
mise run preview rolling-pre-release <previous-sha-or-tag>

# Preview a stable tag range:
mise run preview v0.1.0 <previous-tag-or-sha>

# Smoke-test with HEAD (no tag, just working tree):
mise run preview HEAD

# Direct invocation:
support/communique-release generate rolling-pre-release --dry-run --concise
```

### Generate CHANGELOG.md entry

```sh
# Rolling:
mise run changelog rolling-pre-release
mise run changelog rolling-pre-release 0f5c0f69a3980fe14c7c99806553ce26ee056ce5

# Stable:
mise run changelog v0.1.0
mise run changelog v0.1.0 v0.0.1
```

The `--concise` flag produces a condensed changelog entry.  The entry is
appended to `CHANGELOG.md` in the repo root.

### Update an existing GitHub release

The release must already exist (created by `release.yml` on a `v*` tag push
or workflow_dispatch).  Downloads and assets are preserved by
`--preserve-sections`.

```sh
# Rolling:
mise run release-notes rolling-pre-release
mise run release-notes rolling-pre-release 0f5c0f69a3980fe14c7c99806553ce26ee056ce5

# Stable:
mise run release-notes v0.1.0
mise run release-notes v0.1.0 v0.0.1
```

The wrapper validates the release exists and sets up `GITHUB_TOKEN` before
spending any LLM tokens.  Requires `gh auth login` with `repo` scope, or set
`GITHUB_TOKEN`/`GH_TOKEN` in the environment.

## Tag notes

- The `rolling-pre-release` tag is force-moved by CI on every
  `workflow_dispatch`; communique references it by name, so the rolling
  release body is always updated for the latest build.
- Stable version tags (`v0.1.0`, `v0.2.0`, …) are permanent — they pin a
  specific release and are never force-moved.
- The root commit (`0f5c0f69a3980fe14c7c99806553ce26ee056ce5`) is the default
  PREV baseline for all mise tasks.  This is **cumulative** — it includes the
  entire project history.  For subsequent stable tags, explicitly supply the
  **actual** previous tag so the generated notes reflect only new changes.

## Configuration files

| File | Purpose | Committed? |
|------|---------|-----------|
| `mise.toml` | Tool versions (communique 1.5.0, fnox 1.28.0) + tasks | Yes |
| `fnox.toml` | 1Password `LLMGATEWAY_API_KEY` ref (release profile only) | Yes |
| `fnox.local.toml` | Personal overrides/locally-resolved cache | **No** (gitignored) |
| `communique.toml` | Communiqué model/repo/provider config | Yes |
| `support/communique-release` | Wrapper that resolves LLM_API_KEY + handles GITHUB_TOKEN | Yes |

## Smoke testing

```sh
fnox check
mise run preview HEAD
support/communique-release generate HEAD --dry-run --concise
```