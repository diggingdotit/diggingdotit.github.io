---
title: "gemtop v0.8.3: Splitting the Monolith & Standard Go Layout"
slug: "gemtop-modularization-v0-8-3"
date: 2026-10-21T04:22:00+01:00
draft: false
categories: ["Diary"]
tags: ["gemtop", "golang", "architecture", "ci", "reverse-proxy"]
summary: "Breaking a 2,240-line Go monolith into Standard Go Project Layout, moving to Go 1.22 ProxyRequest.Rewrite, adding fsync durability, and releasing v0.8.3."
---

A major structural refactoring session for `gemtop`, culminating in `v0.8.3`. The goal was to break apart a single 2,240-line `main.go` before it became painful to maintain, modernise the reverse proxy pipeline to Go 1.22+, harden disk writes, and clean up CI for a zero-dependency Go build.

## Disk Durability & Longer Timeouts

Two reliability gaps stood out before touching the architecture. First, atomic `os.Rename` writes on macOS (APFS) and Linux (ext4) can still leave zero-byte or corrupted files after an abrupt crash or power loss if the temporary file isn't flushed to physical storage first. Second, deep-thinking Gemini models occasionally ran past the default 30-second response header deadline.

To tighten both up, I added `tmp.Sync()` before `os.Rename` across `SaveAppConfig` and `SaveDiskState`, raised the default upstream response header timeout from 30s to 60s, and exposed it via `timeout_seconds` in `~/.gemtop.json` and a `-timeout` CLI flag.

## Splitting the 2,240-Line Monolith

`main.go` had grown into a single compilation unit coupling CLI parsing, HTTP proxying, rate tracking, POSIX terminal controls, and ANSI TUI rendering, leaving 14 files sitting flat in the repository root with no compiler-enforced boundaries.

I broke it up using the **Standard Go Project Layout** (`cmd/`, `internal/`, `test/`):

- **`cmd/gemtop/`** — Minimal composition root handling CLI flags, dual-stack loopback listening, signal handling, and server lifecycle.
- **`internal/config`** — Embedded defaults, `~/.gemtop.json` user overlay, model drift detector, and `0600` permission enforcement.
- **`internal/proxy`** — Reverse proxy engine, 64 KB circular tail tap for trailing `usageMetadata`, route extraction, and host/origin CSRF guards.
- **`internal/stats`** — Rolling 60s windows (RPM/TPM), 5m churn metrics, Pacific midnight rollover, and atomic `fsync`-backed disk persistence.
- **`internal/terminal`** — POSIX `TIOCGWINSZ` sizing, alternate screen controls, ANSI/C1/Bidi injection shielding, and panic recovery.
- **`internal/tui`** — Monospace box-drawing layout, WCAG-accessible status badges, Unicode sparklines, and interactive keypress navigation.
- **`test/`** — Black-box integration tests (`e2e_test.go`), while white-box unit tests stay co-located inside each `internal/<pkg>/`.

One quick gotcha during the layout move: `.gitignore` had unanchored `gemtop` binary patterns that would have silently matched `cmd/gemtop/`. Anchoring them to the root (`/gemtop`, `/gemtop-*`) solved it cleanly, and CI and release builds were updated to target `./cmd/gemtop`.

## Modernising to `ProxyRequest.Rewrite`

Go's legacy `httputil.ReverseProxy.Director` hook is deprecated, prone to header spoofing, and awkwardly tied to package globals. I migrated the proxy pipeline to Go 1.22+'s `Rewrite: func(r *httputil.ProxyRequest)` API while keeping upstream request sanitisation intact — stripping `Accept-Encoding` so the stream tap can inspect raw responses, and injecting ambient `GEMINI_API_KEY` credentials only for verified local loopback requests.

## Decoupled Telemetry & Immediate EOF Accounting

Two transport-layer rough edges came out during the split. In `-plain` screen-reader mode, logging created unnecessary coupling between the proxy transport and terminal presentation. Meanwhile, on HTTP keep-alive connections, response metrics weren't finalised until the underlying TCP socket closed.

Introducing a thread-safe `OnCompletion` callback on `stats.GatewayState` cleanly separated presentation from transport, and updating the 64 KB tail tap in `internal/proxy` to finalise accounting as soon as it sees `io.EOF` ensured metrics update immediately without waiting for connection teardown.

## Zero-Dependency CI & SemVer `v0.8.3`

GitHub Actions runners were emitting `Restore cache failed: Dependencies file is not found` warnings because `actions/setup-go` looked for a `go.sum` file by default. Since `gemtop` uses 100% Go standard library with `CGO_ENABLED=0` and zero third-party dependencies, no `go.sum` exists. Setting `cache: false` in both `.github/workflows/ci.yml` and `.github/workflows/release.yml` silenced the warning and sped up runs.

Because tag `v0.8.2` had already been cut on the pre-refactor layout and this restructuring preserved full backwards compatibility, SemVer 2.0.0 called for a patch bump to **`v0.8.3`**. I updated version constants, banners, `SPECIFICATION.md`, and docs, verified all 7 packages passed with `go test -v -race ./...`, `go vet ./...`, `gofmt -l .`, and `go build -ldflags="-s -w" -o gemtop ./cmd/gemtop`, then merged to `main` and pushed signed tag `v0.8.3`.

## Takeaways

- **2,000+ lines in one file is hard to maintain:** Letting `main.go` grow to 2,240 lines made navigation and changes unnecessarily fragile — splitting early into focused packages pays off quickly.
- **Default HTTP timeouts are too tight for reasoning models:** Standard 30-second response header deadlines cut off deep-thinking model calls; raising the default to 60s and exposing a flag and config option avoids silent failures.
- **Co-locate white-box tests, isolate black-box tests:** Keeping unit tests inside `internal/<pkg>/` preserves white-box access while placing integration tests outside `internal/` validates the real boundary.
- **Set `cache: false` in `actions/setup-go` for stdlib-only repos:** Zero-dependency Go projects have no `go.sum`; turning off dependency caching eliminates runner warnings and trims CI time.
