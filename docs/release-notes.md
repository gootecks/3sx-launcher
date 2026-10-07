# Release Notes & Changelog

Uses [Communiqué](https://github.com/jdx/communique) via the
[LLMGateway](https://llmgateway.io) to auto-generate release notes and
changelog entries from git history.

**Important:** This tooling manages release *notes* only — it does **not**
create tags, build binaries, or upload assets.  The CI workflow handles the
mechanical release process; run these tasks after a tag is pushed or to
preview what the next release will say.

## Prerequisites

- [mise](https://mise.jdx.dev) — pins `communique` and `fnox` versions
- [fnox](https://fnox.jdx.dev) — resolves secrets from 1Password;
  run `fnox check` to verify configuration
- [1Password CLI](https://developer.1password.com/docs/cli) — `op` must be
  authenticated and the `Developer/LLMGateway` item accessible

## Workflow

This is separate from the **3sxtra engine** release process.  The launcher's
release workflow works like this:

1. A `v*` tag push triggers `.github/workflows/release.yml`.
2. It builds all platform binaries and moves the `rolling-pre-release` tag to
   the commit, then creates/updates the **rolling-pre-release** GitHub release
   with a static Downloads table.
3. The **generated release notes** step is manual — run `mise run release-notes`
   after the CI workflow finishes to replace only the note body.

The `--preserve-sections` flag keeps the Downloads table and other non-communique
content intact by only rewriting content between
`<!-- communique:start -->` / `<!-- communique:end -->` HTML comment markers.
If no markers exist yet (first run), communique appends a new managed section
with the generated notes.

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

# Smoke-test with HEAD (no tag, just working tree):
mise run preview HEAD

# Direct invocation:
support/communique-release generate rolling-pre-release --dry-run --concise
```

### Generate CHANGELOG.md entry

```sh
mise run changelog rolling-pre-release
mise run changelog rolling-pre-release 0f5c0f69a3980fe14c7c99806553ce26ee056ce5
```

The `--concise` flag produces a condensed changelog entry.  The entry is
appended to `CHANGELOG.md` in the repo root.

### Update an existing GitHub release

The release must already exist (created by `release.yml` on a `v*` tag push).
Downloads and assets are preserved by `--preserve-sections`.

```sh
mise run release-notes rolling-pre-release
mise run release-notes rolling-pre-release 0f5c0f69a3980fe14c7c99806553ce26ee056ce5
```

The wrapper validates the release exists and sets up `GITHUB_TOKEN` before
spending any LLM tokens.  Requires `gh auth login` with `repo` scope, or set
`GITHUB_TOKEN`/`GH_TOKEN` in the environment.

## Tag notes

- The `rolling-pre-release` tag is force-moved by CI on every build; communique
  references it by name, so the release body is always updated for the latest
  build.
- The root commit (`0f5c0f69a3980fe14c7c99806553ce26ee056ce5`) is the default
  PREV baseline for all mise tasks.  This is **cumulative** — it includes the
  entire project history.  For subsequent stable tags (`v0.2.0`, `v0.3.0`, …),
  explicitly supply the **actual** previous tag so the generated notes reflect
  only new changes.

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