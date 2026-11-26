---
title: "gemtop v0.8.4–v0.8.5: Binary E2E Tests, Gemini 4 & CI Security"
slug: "gemtop-gemini-4-security-v0-8-5"
date: 2026-11-25T23:03:00Z
draft: false
categories: ["Diary"]
tags: ["gemtop", "golang", "testing", "security", "github-actions", "gemini-4"]
summary: "Two gemtop releases covering compiled-binary E2E tests, Gemini 4 defaults, free CI security scanners, stale-model pruning, and a cleaner dev diary workflow."
---

A packed session on `gemtop` spanning two releases (`v0.8.4` and `v0.8.5`). What started as a pass over test coverage and Gemini 4 defaults turned into a broader round of CI security hardening, a real UX bug caught during local smoke testing, and a clean-up of how I publish these dev notes.

## Test Layout & Binary E2E Coverage

I started by trimming some self-inflicted friction from the repo layout. For a solo, zero-dependency Go project, keeping a `Makefile` around was mostly root clutter — local shell aliases handle my shortcuts, and Go's native toolchain keeps CI simple. I also renamed `test/` to `tests/integration_test.go` to avoid ambiguity with the Unix `test` built-in and to separate in-process integration tests from full binary end-to-end tests.

Running cross-package coverage (`go test -coverpkg=./...`) highlighted a clear blind spot: the core packages sat between 75% and 100%, while `cmd/gemtop/main.go` was at `0.0%` alongside two production config helpers (`ResolveStateFilePath` and `GetProfileAPIKey`).

To close that gap, I added table-driven unit tests for the config helpers in `internal/config` and a new compiled-binary suite in `tests/e2e_test.go` (`TestBinaryE2E`). The E2E test builds `./cmd/gemtop` into an isolated `t.TempDir()` and exercises the actual CLI end to end:

- `-v` version output and `-plain` stdout logging
- Real loopback HTTP proxying alongside `main.go`'s Host, Origin, and Fetch-Metadata guards
- `-port` collision behaviour vs `-auto-port` fallback
- Graceful `SIGINT` shutdown and state flushing to disk

## Gemini 4 Defaults & Two Quiet Bugs

Next, I moved the embedded defaults (`DefaultConfigJSON`) off Gemini 3.x and onto the Gemini 4 lineup (`gemini-4-hydrogen`, `gemini-4-neon`, `gemini-4-argon`, `gemini-4-carbon`), raising the model drift detector baseline from `3` to `4`. I also updated `README.md` examples to `gemini-4-neon` and linked Google AI Studio's live Rate Limits dashboard.

While working through the config and proxy tests, two subtle bugs turned up:

- **`AllowedHosts` merge bug:** `internal/config/config.go` was running `append(userCfg.AllowedHosts, userCfg.AllowedHosts...)`, overwriting the default loopback hosts while duplicating user entries. Fixed and locked down with a regression test.
- **Latent race & nil-deref in proxy tests:** The reverse-proxy rewrite test wasn't draining `resp.Body` before closing it, making stream-tap metric recording non-deterministic, and lacked a nil guard. Fixed and verified across 10/10 runs with `-race`.

## CI Hardening & v0.8.4

Before tagging a release, I brought `ci.yml` and `release.yml` up to scratch:

- Moved off EOL Go `1.23` to `go-version: 'stable'` so compiled proxy binaries pick up current Go stdlib HTTP/TLS security fixes.
- Enabled CI on `development` pushes and PRs, added `concurrency` cancellation (helpful on private repos where macOS Actions minutes burn at 10×), tightened workflow `permissions`, and added both a `gofmt` check and a tag-vs-binary version guard in `release.yml`.

With that in place, I bumped SemVer to `v0.8.4`, merged `development` into `main`, and pushed the signed release tag. Smoke-testing the compiled binary immediately paid off — it surfaced a real UX annoyance where models from previous days' `~/.gemtop-state.json` were reappearing on startup with zero traffic.

## Free CI Security & Spec Sync

GitHub's native CodeQL and Secret Scanning require paid GitHub Advanced Security (GHAS) on private repositories, so I put together a lightweight `ubuntu-latest`-only workflow (`.github/workflows/security.yml`) that runs on pushes, PRs, and a weekly Monday `06:00 UTC` schedule using:

- `govulncheck` for known Go vulnerabilities
- `gosec` for static security analysis
- `gitleaks` (canonical `zricethezav/gitleaks/v8`) for secret detection

At the same time, I pinned every GitHub Action across `ci.yml`, `release.yml`, and `security.yml` to immutable 40-character commit SHAs, bumping `softprops/action-gh-release` to `v3.0.4` on Node 24. Auditing the initial scan findings was clean: I added justified inline `// #nosec` annotations (`G103`, `G304`, and `G117` for atomic `0600` local config saves), confirmed that all three historical `gitleaks` hits were a single dummy test string, and scoped `.gitleaksignore` strictly to those three fingerprints.

I also did an end-to-end pass over `SPECIFICATION.md` so Sections 2, 4, 6, 8, and 16.5 match the `internal/` layout and v4 baseline, and updated a leftover defensive `3` → `4` fallback in `internal/tui/tui.go`.

## Pruning Stale Models in v0.8.5

With CI security green, I came back to the stale-model issue spotted after `v0.8.4`. On startup, `stats.LoadDiskState` repopulated every model key saved in `~/.gemtop-state.json` even from earlier days, and pressing `[c]` in the TUI only cleared the rolling windows without dropping models or writing the cleared state back to disk.

The fix touched three places:

- Updated `stats.LoadDiskState` to start with a clean daily slate whenever the saved Pacific Time date differs from today, and to skip zero-RPD ghost entries on same-day loads.
- Added `GatewayState.Reset()` and wired an `OnClear` callback in `cmd/gemtop/main.go` so pressing `[c]` clears tracked models, rolling windows, and status totals and immediately flushes the clean state to disk.
- Added a `-reset-state` CLI flag to launch fresh and overwrite any persisted state file.

After adding unit tests and an E2E check for `-reset-state`, I bumped to `v0.8.5`, verified CI and Security green, merged `development` → `main` (closing Issue `#4`), and pushed signed tag `v0.8.5`.

## Dependabot, Verified Commits & Docs

Once `v0.8.5` was out, I tidied up the surrounding release and repo hygiene:

- **Standalone `gh` CLI:** Installed the standalone macOS `arm64` Go binary (no external dependencies), verified the `v0.8.5` release assets (four cross-compiled binaries plus `checksums.txt`), and opened the next batch of roadmap issues (`#5`–`#9`).
- **Dependabot for SHA-pinned actions:** Added `.github/dependabot.yml` targeting `development` weekly on Mondays at `09:00` (`Europe/London` so GMT/BST shifts happen automatically) with grouped `github-actions` updates, then merged to `main` to activate it and close Issue `#9`.
- **Green Verified badges:** Checked via `gh api` why my SSH-signed commits showed as `Unverified` (`reason: "unknown_key"`). GitHub keeps Authentication Keys and Signing Keys separate — registering the same SSH public key as a **Signing Key** retroactively flipped every signed commit and release tag to green **Verified**.
- **Quickstart polish:** Cleared out remaining placeholder URLs in `README.md` and `SPECIFICATION.md`, folded pre-compiled binaries and SHA256 verification into the top Quickstart section, and kept plain `curl -fsSLO` as the primary install path so end users never need `gh`.

## Diary Workflow, Git Hygiene & v0.9.0

Finally, I split session notes from long-term writing based on lifespan:

- **`CHECKPOINT.md`** stays local and unversioned inside the project repo (excluded via `.git/info/exclude` and `~/.gitignore.global`) — it's disposable session glue that can always be rebuilt from `git log` and GitHub Issues.
- **Dev diary entries** now live here in this Hugo (`PaperMod`) repo under `content/diary/`, keeping the `gemtop` codebase clean for open-sourcing while giving long-term notes proper git history and backup.

Because this blog repo is public, my local-first flow is to draft on `development` with `draft: true`, preview with `hugo server -D` (`http://localhost:1313`), and only push and fast-forward merge to `main` once an entry is scrubbed and set to `draft: false`. Along the way I also tightened `~/.gitignore.global` for macOS resource forks (`._*`), `.env.*` (keeping `!.env.example`), extra key/cert formats (`*.p12`, `*.pfx`, `id_ed25519`, `id_rsa`), editor swap files, and `CHECKPOINT.md`, leaving language and build artefacts to per-repo `.gitignore` files.

Looking ahead to `v0.9.0` (Issue `#5`), the plan is to let `gemtop` track live Antigravity usage alongside proxied API traffic via a passive `-watch-antigravity` tailer reading documented `transcript.jsonl` files (`input_tokens`, `cache_read_tokens`, `output_tokens`) across Gemini and Claude sessions — zero dependencies, no process scraping, and no synchronous hook overhead.

## Takeaways

- **Make build tooling earn its keep:** On a solo, zero-dependency Go repo, Go's native toolchain and local shell aliases beat maintaining a root `Makefile`.
- **Always smoke-test the built binary against real state:** Every automated test passed for `v0.8.4`, yet running the real binary immediately exposed stale previous-day models persisting across restarts.
- **Private repos don't need paid GitHub Advanced Security for real security gates:** `govulncheck`, `gosec`, and `gitleaks` on a lightweight `ubuntu-latest` workflow cover vulnerability, static analysis, and secret scanning for free.
- **Clear actions must reset persisted state, not just memory:** A TUI reset (`[c]`) feels broken unless it clears tracked models, flushes to disk immediately, and pairs with clean day-rollover pruning on startup.
- **GitHub keeps SSH Authentication Keys and Signing Keys separate:** SSH-signed commits stay `Unverified` (`unknown_key`) until you also register the key under Signing Keys — doing so retroactively flips past commits and tags to green.
