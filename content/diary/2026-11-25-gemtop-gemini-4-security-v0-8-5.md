---
title: "gemtop v0.8.4–v0.8.5: Binary E2E Tests, Gemini 4 Defaults, Free CI Security Gates & Stale-Model Pruning"
slug: "gemtop-gemini-4-security-v0-8-5"
date: 2026-11-25T23:03:00Z
draft: true
categories: ["Diary"]
tags: ["gemtop", "golang", "testing", "security", "github-actions", "gemini-4"]
summary: "Adding compiled-binary E2E tests, migrating defaults to Gemini 4, hardening GitHub Actions with free security scanners and SHA pinning, pruning stale models in v0.8.5, and automating workflow updates with Dependabot."
---

**Project:** `gemtop` (`v0.8.4` → `v0.8.5`)  
**Focus:** Test Architecture (`tests/`), Compiled-Binary E2E Suite, Gemini 4 Migration, Hardened CI/Security Pipeline, Stale-Model Pruning & Verified Releases

---

## 1. Pragmatic Engineering Decisions (Solo Repo Lens)

* **Dropped `Makefile`:** Re-evaluated build automation for a solo zero-dependency Go repo — local shell aliases handle shortcuts with zero root clutter while Go's native toolchain keeps CI minimal.
* **Renamed `test/` → `tests/integration_test.go`:** Avoided any Unix `test` built-in ambiguity and accurately distinguished in-process multi-package integration tests from true compiled-binary E2E tests.

---

## 2. Closing Coverage Blind Spots

* **The Discovery:** Cross-package coverage (`go test -coverpkg=./...`) showed strong core engine coverage (75–100%), but `cmd/gemtop/main.go` sat at `0.0%` alongside two production config helpers (`ResolveStateFilePath` and `GetProfileAPIKey`).
* **The Solution:**
  * Added **`tests/e2e_test.go` (`TestBinaryE2E`)** compiling `./cmd/gemtop` into an isolated `t.TempDir()` and exercising `-v`, real loopback HTTP proxying, `main.go`'s Host/Origin/Fetch-Metadata guards, `-port` collision vs `-auto-port` fallback, `-plain` stdout logging, and graceful `SIGINT` disk state flushing.
  * Added table-driven unit tests for `ResolveStateFilePath` and `GetProfileAPIKey` in `internal/config/config_test.go`.

---

## 3. Gemini 4 Migration & Bug Fixes

* **Gemini 4 Lineup:** Updated embedded defaults (`DefaultConfigJSON`) from Gemini 3.x to the current Gemini 4 element codenames (`gemini-4-hydrogen`, `gemini-4-neon`, `gemini-4-argon`, `gemini-4-carbon`), raising the drift detector baseline from `3` → `4`. Updated `README.md` examples to `gemini-4-neon` and linked Google AI Studio's live Rate Limits dashboard.
* **Bug #1 (`AllowedHosts` merge):** Fixed `internal/config/config.go` where `append(userCfg.AllowedHosts, userCfg.AllowedHosts...)` overwrote default loopback hosts and duplicated user entries; added a regression assertion in `TestLoadAppConfigUserFileOverlay`.
* **Bug #2 (Latent test race & nil-deref):** Fixed `TestNewGatewayReverseProxyRewrite` to drain `resp.Body` before closing (so the stream tap deterministically records metrics) and guard against nil dereference; verified 10/10 runs with `-race`.

---

## 4. CI/Release Hardening & `v0.8.4` Release

* Upgraded `ci.yml` and `release.yml` off EOL Go `1.23` to `go-version: 'stable'` so compiled proxy binaries always receive current Go stdlib HTTP/TLS security patches.
* Enabled CI on `development` pushes/PRs, added `concurrency` cancellation (preserving 10× macOS Actions minutes on private repos), least-privilege `permissions`, `gofmt` gate, and a tag-vs-binary version guard in `release.yml`.
* Bumped SemVer to **`v0.8.4`**, merged `development` → `main`, and published signed tag `v0.8.4`.
* Smoke-tested the compiled local binary and spotted a real UX issue: old `~/.gemtop-state.json` models from previous days were reappearing on launch with zero traffic.

---

## 5. Free Private-Repo Security Pipeline & Spec Sync

* **Security Workflow (`.github/workflows/security.yml`):** Because GitHub CodeQL and native Secret Scanning require paid GitHub Advanced Security on private repos, added an `ubuntu-latest`-only workflow (push/PR + weekly Monday 06:00 UTC cron) running `govulncheck`, `gosec`, and `gitleaks` (via canonical `github.com/zricethezav/gitleaks/v8`).
* **Supply-Chain Pinning:** Pinned all GitHub Actions across `ci.yml`, `release.yml`, and `security.yml` to immutable 40-char commit SHAs (upgrading `softprops/action-gh-release` to `v3.0.4` on Node 24).
* **Audited Findings:**
  * Added justified inline `// #nosec` comments (`G103`, `G304`, `G117` for atomic `0600` local `~/.gemtop.json` saves).
  * Verified all 3 historical `gitleaks` matches were a dummy test string (zero real secrets in history) and added `.gitleaksignore` restricted strictly to those 3 fingerprints.
* **Spec & Code Sync:** Audited `SPECIFICATION.md` end-to-end, brought Sections 2, 4, 6, 8, and 16.5 into full alignment with the `internal/` packages and v4 baseline, and updated the defensive fallback in `internal/tui/tui.go` (`3` → `4`).

---

## 6. Stale Model Pruning, Clear Disk Flush & `v0.8.5` Release

* **The Problem:** `stats.LoadDiskState` repopulated every historical model key saved in `~/.gemtop-state.json` on startup even from previous days, and pressing `[c]` in the TUI only cleared sliding windows without removing models or flushing to disk.
* **The Solution:**
  * Updated `stats.LoadDiskState` to start with a clean daily slate whenever `disk.Date != todayStr` (Pacific Time) and skip zero-RPD ghost models on same-day loads.
  * Added `GatewayState.Reset()` + `OnClear func()` callback wired in `cmd/gemtop/main.go` so `[c]` clears tracked models, rolling windows, and status totals and immediately flushes the clean state to disk.
  * Added the `-reset-state` CLI flag to start fresh and overwrite any persisted state file on launch.
  * Added unit tests (`TestLoadDiskStatePrunesPreviousDayModels`, `TestLoadDiskStateSkipsZeroRPDModelsSameDay`, `TestGatewayStateReset`) and compiled-binary E2E coverage (`TestBinaryE2E/reset_state_flag`).
  * Bumped SemVer to **`v0.8.5`**, verified CI + Security green, merged `development` → `main` (auto-closing Issue `#4`), and pushed signed release tag **`v0.8.5`**.

---

## 7. GitHub CLI, Dependabot, Verified Signatures & Quickstart Polish

* **Standalone GitHub CLI (`gh`):** Installed the standalone macOS `arm64` Go `gh` binary (zero external dependencies), verified the `v0.8.5` GitHub Release assets (4 cross-compiled binaries + `checksums.txt`), and created the next roadmap issues (`#5`–`#9`).
* **Dependabot for SHA-Pinned Actions:** Added `.github/dependabot.yml` targeting `development` weekly on Mondays at `09:00` (`Europe/London` IANA timezone for automatic GMT/BST transitions) with grouped `github-actions` updates; merged to `main` to activate Dependabot and auto-close Issue `#9`.
* **Verified Commit & Tag Signatures:** Diagnosed GitHub's `Unverified` commit badge (`reason: "unknown_key"`) via `gh api` — GitHub keeps SSH Authentication Keys and Signing Keys separate. Registering the SSH public key as a GitHub **Signing Key** retroactively flipped all signed commits and release tags to green **Verified**.
* **Docs & Quickstart Consolidation:** Removed remaining placeholder URLs across `README.md` and `SPECIFICATION.md`, merged **Pre-Compiled Binary Releases & SHA256 Verification** into the top Quickstart section, and kept standard zero-dependency `curl -fsSLO` as the primary download method so end users never need `gh` installed.

---

## 8. Diary Workflow, Global Git Hygiene & `v0.9.0` Direction

* **Separating Ephemeral Checkpoints from Long-Term Diaries:** Decided to treat `CHECKPOINT.md` and developer diaries differently by lifespan:
  * **`CHECKPOINT.md`** stays local inside the project repo (unversioned, excluded via `.git/info/exclude` and `~/.gitignore.global`) — it is disposable session glue easily reconstructed from `git log` and GitHub Issues.
  * **Developer diary entries** move out of `gemtop` into this Hugo (`PaperMod`) repository under `content/diary/`, keeping the code repo clean and ready to open-source while giving long-term notes proper version history and off-site backup.
* **Local-First Hugo Publishing Flow:** Draft and iterate locally in VS Code on a `development` branch with `draft: true` and instant live reload via `hugo server -D` (`http://localhost:1313`). Because the blog repo is public, rough/unscrubbed commits stay local until scrubbed and ready to publish (`draft: false` → push `development` → fast-forward merge to `main` → GitHub Pages).
* **Global `.gitignore` Hardening:** Tightened `~/.gitignore.global` for macOS resource forks (`._*`), `.env.*` (with `!.env.example`), extra cert/key formats (`*.p12`, `*.pfx`, `id_ed25519`, `id_rsa`), editor swap files, and `CHECKPOINT.md`, keeping language and build outputs strictly in per-repo `.gitignore` files.
* **Looking Ahead (`v0.9.0` — Issue `#5`):** Designed a clean, zero-dependency way for `gemtop` to track live Antigravity usage alongside proxied API traffic via a passive `-watch-antigravity` transcript tailer reading documented `transcript.jsonl` files (`input_tokens`, `cache_read_tokens`, `output_tokens`) across Gemini and Claude sessions — no process scraping and no synchronous hook overhead.

