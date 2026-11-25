---
title: "gemtop v0.8.3: Monolith Decomposition, Go 1.22 ReverseProxy & Standard Go Layout"
slug: "gemtop-modularization-v0-8-3"
date: 2026-10-21T04:22:00+01:00
draft: true
categories: ["Diary"]
tags: ["gemtop", "golang", "architecture", "ci", "reverse-proxy"]
summary: "Decomposing a 2,240-line Go monolith into Standard Go Project Layout, modernising to Go 1.22 ProxyRequest.Rewrite, adding fsync durability, and releasing v0.8.3."
---

**Project:** `gemtop` (`v0.8.3`)  
**Focus:** Monolith Decomposition, Reverse Proxy Modernisation, Standard Go Layout (`cmd/`, `internal/`, `test/`), & SemVer `v0.8.3` Release

---

## 1. Storage Durability & Extended Timeout Handling

* **The Problem:** On macOS (APFS) and Linux (ext4), sudden process termination or power interruptions during atomic `os.Rename` operations risked leaving zero-byte or corrupted state files on disk. Additionally, deep-thinking Gemini models occasionally timed out on default 30-second response deadlines.
* **The Solution:**
  * Added `tmp.Sync()` before `os.Rename` across all configuration and state persistence routines (`SaveAppConfig` and `SaveDiskState`) to guarantee physical disk writes.
  * Increased default upstream response header timeout from 30s to 60s, added a configurable `timeout_seconds` setting in `~/.gemtop.json`, and exposed the `-timeout` CLI flag.

---

## 2. Decomposing the Monolithic Composition Root

* **The Problem:** `main.go` had grown into a 2,240-line monolithic file coupling CLI parsing, HTTP reverse proxying, sliding-window rate tracking, POSIX terminal manipulation, and ANSI TUI rendering into a single compilation unit.
* **The Solution:**
  * Decomposed the monolith into cohesive domain packages:
    * **`config`** — Embedded defaults, JSON user overlay (`~/.gemtop.json`), model drift detector, and secure POSIX permission enforcement (`0600`).
    * **`proxy`** — Modern reverse proxy engine, circular 64 KB tail tap for trailing `usageMetadata`, route extractor, and origin/host CSRF guards.
    * **`stats`** — Rolling 60s windows (RPM/TPM), 5m churn metrics, Pacific midnight rollover, and atomic disk persistence with `fsync`.
    * **`terminal`** — POSIX `TIOCGWINSZ` terminal sizing, alternate screen buffer controls, ANSI/C1/Bidi injection shielding, and panic recovery hooks.
    * **`tui`** — Monospace box drawing maths, WCAG accessible status badges, Unicode sparklines, and interactive keypress navigation.
    * **`cmd/gemtop`** — Minimal composition root wiring CLI flags, dual-stack loopback listen, signal handling, and server lifecycle.

---

## 3. Modernising to Go 1.22 `httputil.ReverseProxy.Rewrite`

* **The Problem:** The legacy `Director` pattern in Go's `httputil.ReverseProxy` is deprecated, susceptible to header spoofing, and awkwardly coupled to global variables.
* **The Solution:**
  * Migrated the proxy pipeline to the modern Go 1.22+ `Rewrite: func(r *httputil.ProxyRequest)` API.
  * Preserved upstream request sanitisation (stripping `Accept-Encoding` for raw streaming inspection) and dynamically injected ambient API keys (`GEMINI_API_KEY`) for verified local loopback requests.

---

## 4. CI Workflow Optimisation for Zero-Dependency Builds

* **The Problem:** GitHub Actions runners reported `Restore cache failed: Dependencies file is not found` looking for `go.sum` across macOS and Ubuntu matrix jobs.
* **The Solution:**
  * Because `gemtop` strictly uses 100% Go Standard Library with zero third-party dependencies (`CGO_ENABLED=0`), no `go.sum` file exists or is needed.
  * Added `cache: false` to `actions/setup-go` in both `.github/workflows/ci.yml` and `.github/workflows/release.yml`, eliminating runner warnings and accelerating CI execution.

---

## 5. Standard Go Project Layout Refactoring (`cmd/`, `internal/`, `test/`)

* **The Problem:** The repository root was cluttered with 14 source and test files, making project navigation cumbersome and failing to enforce compiler-level visibility boundaries.
* **The Solution:**
  * Adopted the **Standard Go Project Layout**:
    * `cmd/gemtop/` — Houses the composition root and CLI binary entrypoint (`main.go`, `main_test.go`).
    * `internal/` — Encapsulates domain packages (`config`, `proxy`, `stats`, `terminal`, `tui`) with private access rules enforced by the Go compiler.
    * `test/` — Dedicated home for black-box integration tests (`e2e_test.go`).
  * **Test Architecture Philosophy:** Unit tests remain co-located inside `internal/<pkg>/` to preserve white-box testing fidelity, while higher-level system tests reside outside `internal/`.
  * Anchored `.gitignore` binary patterns (`/gemtop`, `/gemtop-*`) to the root directory to prevent accidental filtering of `cmd/gemtop/`.
  * Updated CI and release workflows to compile `./cmd/gemtop`.

---

## 6. Decoupled Telemetry & Stream Tap Hardening

* **The Problem:** Logging in `-plain` screen-reader mode created an awkward dependency coupling between transport internals and terminal presentation. Furthermore, on keep-alive HTTP connections, response streams did not immediately finalise metrics until the connection closed.
* **The Solution:**
  * Introduced thread-safe `OnCompletion` callback hooks in `stats.GatewayState`, cleanly decoupling presentation from transport.
  * Hardened the 64 KB tail tap in `internal/proxy` to finalise accounting immediately upon `io.EOF`, guaranteeing instant metric updates without waiting for TCP socket teardown.

---

## 7. SemVer Standardisation & Release (`v0.8.3`)

* **The Problem:** Git tag `v0.8.2` had already been published on the prior layout commit. In accordance with SemVer 2.0.0, internal structural refactoring with full backwards compatibility calls for a patch increment.
* **The Solution:**
  * Bumped version to **`v0.8.3`** across code constants, banners (`v0.8.3-local`), documentation, test assertions, and build instructions.
  * Updated `SPECIFICATION.md` (Section 15) documenting the layout migration and architecture.
  * Merged into `development` and `main`, and pushed signed tag `v0.8.3` to trigger automated multi-architecture binary releases.

---

## 8. Verification & Release Status

* **Test Suite:** 100% PASS across all 7 packages with zero race conditions (`go test -v -race ./...`).
* **Quality Gates:** `go vet ./...` & `gofmt -l .` 100% compliant with zero diagnostics.
* **Build Verification:** `go build -ldflags="-s -w" -o gemtop ./cmd/gemtop` compiles cleanly (`gemtop v0.8.3`).
