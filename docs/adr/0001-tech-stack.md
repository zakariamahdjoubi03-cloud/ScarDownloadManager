# ADR-0001: Desktop technology stack

**Status:** Proposed (awaiting owner approval)
**Date:** 2026-11-05
**Deciders:** Project owner
**Supersedes:** none

---

## Context

ScarDownloadManager must ship on Windows, macOS, and Linux from one codebase. The product needs:

- a fast, concurrent download engine (many sockets, large files, disk I/O);
- reliable resume and crash-safe persistence;
- a modern, translatable UI with right-to-left support;
- native installers, code signing, and signed auto-update;
- optional sidecar processes (media extraction, muxing);
- a small memory and disk footprint — this is a utility people leave running;
- a test suite that runs headlessly in CI on all three operating systems.

The owner is not yet committed to any language or framework.

---

## Decision

Adopt **Tauri 2** with:

- **Rust** workspace for the engine (`crates/`), no UI dependency;
- **React 19 + TypeScript + Vite** for the UI;
- **SQLite** for persistence ([ADR-0003](0003-data-storage.md));
- **i18next + ICU** for localization ([ADR-0002](0002-i18n-strategy.md));
- Tauri's official bundlers and updater for distribution ([ADR-0006](0006-distribution-and-updates.md)).

---

## Options considered

| Criterion | Tauri 2 + Rust | Electron + Node | Flutter | Avalonia (.NET) | Qt / C++ |
|---|---|---|---|---|---|
| Cross-platform desktop | ✅ Win/mac/Linux | ✅ Win/mac/Linux | ✅ Win/mac/Linux | ✅ Win/mac/Linux | ✅ Win/mac/Linux |
| Engine performance (sockets, hashing, IO) | ✅ Excellent (Tokio) | ⚠️ Good, but GC and single-threaded event loop | ⚠️ Dart async is capable, less mature for raw IO | ✅ Good (.NET) | ✅ Excellent |
| Typical installer size | ✅ ~10–20 MB | ❌ 100–200 MB | ⚠️ 20–40 MB | ⚠️ 30–60 MB (self-contained larger) | ✅ Small |
| Idle memory | ✅ Low | ❌ High | ⚠️ Medium | ⚠️ Medium | ✅ Low |
| Modern UI ecosystem | ✅ Web UI (React) | ✅ Best-in-class | ✅ Good, opinionated | ⚠️ XAML, niche ecosystem | ❌ Custom work |
| Native feel / system integration | ⚠️ Good, some gaps | ✅ Mature | ⚠️ Custom | ✅ Strong on Windows | ✅ Strong |
| Security model | ✅ Capability-based, deny by default | ❌ Broad Node access by default | ⚠️ Varies | ⚠️ .NET sandboxing not the point | ⚠️ Manual |
| Signed auto-update | ✅ Built-in updater | ✅ electron-updater | ⚠️ Third-party | ⚠️ Third-party | ❌ Custom |
| CI complexity | ⚠️ Linux needs webkit deps | ✅ Simple | ✅ Simple | ⚠️ .NET + XAML workload | ❌ Heavy |
| Team familiarity (owner) | ⚠️ Rust is new | ✅ JS is common | ⚠️ Dart is new | ⚠️ C# unknown | ❌ Unlikely |
| Risk if ecosystem shifts | ⚠️ Younger | ✅ Very stable | ✅ Stable | ⚠️ Smaller | ⚠️ Legacy |

### Why Tauri wins for this product

1. **The engine is the product.** Segmented downloads are socket- and IO-bound; Rust's async runtime, zero-GC predictability, and cheap concurrency fit better than a JavaScript event loop.
2. **Footprint matters for a background utility.** A 15 MB installer that idles at low memory is a product feature, not a vanity metric.
3. **Security posture.** Tauri denies everything by default and exposes only declared capabilities; this project handles user URLs and filesystem paths, so a narrow IPC surface is valuable.
4. **Web UI gives the best modern-UI and i18n tooling** without shipping a browser engine — the OS webview is reused.
5. **First-class bundlers and a signed updater** cover the release requirements in ADR-0006 with little custom code.

### Why not Electron (the strongest alternative)

Electron is the mature choice and would shorten the UI learning curve, but it roughly doubles installer size, raises idle memory, and gives the renderer far broader system access by default — three costs that hit exactly this product's profile. It remains the **fallback** if Tauri hits a platform blocker.

### Why not Flutter

Flutter's desktop story is real but its native-plugin and process-management ecosystem is thinner, and a download engine would still need platform channels to native code for the parts that matter.

### Why not Avalonia / Qt

Both are credible desktop toolkits, but neither improves the download engine, and both would push the UI toward a smaller ecosystem for modern web-style design and RTL-heavy localization.

---

## Consequences

### Positive

- One codebase, three installers, small artifacts.
- Engine testable headlessly with `cargo test`; no browser needed in CI.
- Capability-based security by construction.
- Rust crates for HTTP, TLS, SQLite, and hashing are mature.

### Negative

- The owner must learn or supervise Rust; this is the main adoption cost.
- Linux builds require WebKitGTK system packages in CI and on contributor machines.
- Fewer ready-made "download manager" examples than the Node ecosystem.
- Webview rendering differs slightly across platforms; visual tests must run per OS.

---

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| Linux WebKitGTK version drift breaks builds | Pin Ubuntu 22.04 in CI; document system dependencies; test on 24.04 before 1.0 |
| Rust development velocity | Keep the engine small and well-bounded; use well-known crates; enforce clippy |
| Tauri API gaps (tray, notifications, single instance) | Verify each integration in Phase 1 with a spike; Electron fallback documented |
| Webview differences across OS | Per-OS visual smoke tests; avoid exotic CSS |
| Unsigned-build friction during development | Watermark dev builds; signing only in Phase 6 |

---

## Follow-ups

1. Spike in Phase 1: tray icon, notifications, single-instance, deep-link handling on all three OSes — recorded in the Phase 1 issue.
2. Revisit only if a Phase 1 spike fails; a stack change requires a new ADR superseding this one.
