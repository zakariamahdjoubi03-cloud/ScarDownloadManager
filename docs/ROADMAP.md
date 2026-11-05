# Roadmap — ScarDownloadManager

Numbered phases. Each phase is one GitHub Issue, one branch, one small PR series, and has a hard exit gate. A phase is complete only when **all** of its success tests pass on Windows, macOS, and Linux.

Legend: ✅ done · 🔄 in progress · ⬜ planned

---

## Phase 0 — Foundation (docs, decisions, CI)

**Status:** 🔄 in progress (this PR)

### Contains

- `README.md`, `docs/PRD.md`, `docs/ARCHITECTURE.md`, `docs/ROADMAP.md`
- ADRs 0001–0006 with proposed decisions
- `AGENTS.md` working rules
- `.github/workflows/ci.yml` scaffolding (docs checks active; build/test jobs activate automatically when code appears)

### Success criteria (exit gate)

- [ ] All required foundation files exist and pass markdown lint in CI.
- [ ] CI workflow is valid and the docs job is green on the foundation PR.
- [ ] Every ADR has status, options, decision, and consequences.
- [ ] Owner approves the decisions listed in the PR summary.
- [ ] No application code exists in this phase.

### Explicitly out of scope

- Any executable code, dependencies, or build artifacts.
- Installers, signing, releases.

---

## Phase 1 — App shell, settings, internationalization

**Status:** ⬜ planned

### Contains

- Tauri 2 application skeleton that launches on all three OSes.
- IPC contract v1 (`scar-ipc`) with create/pause/resume/cancel stubs returning typed errors.
- Settings persistence (SQLite via `scar-store`), including language and theme.
- Locale pipeline: English + Arabic locale files, OS-locale detection, language switcher in Settings, RTL layout.
- App shell UI: sidebar, downloads view (empty state), settings view.
- CI activates Rust, web, and Tauri build jobs.

### Success criteria (exit gate — must be automated tests)

1. `locale_resolution` unit tests: explicit setting → installer seed → OS locale → English fallback, including unknown locale.
2. Settings round-trip test: write, restart, read; corrupt DB recovery path.
3. RTL snapshot test: Arabic renders `dir="rtl"` with no horizontal overflow at 100% and 125%.
4. Missing-key test fails the build when a locale key is absent from English or Arabic.
5. Hardcoded-string lint test fails on a user-visible literal.
6. CI produces a runnable app bundle for Windows, macOS, and Linux (artifact only, not a release).

### Explicitly out of scope

- Any real network download.
- Media capture, scheduling, categories.

---

## Phase 2 — Single-stream download and resume

**Status:** ⬜ planned

### Contains

- `scar-net`: HTTP client with redirects, timeouts, retries, user-agent, proxy hook.
- Probe: HEAD → `Range: bytes=0-0` fallback; read length and validators.
- Single-stream download to `*.scarpart` with atomic `*.scarmeta` progress.
- Pause / resume / cancel / retry semantics and the state machine.
- Verification by size and available hash, then atomic rename.

### Success criteria (exit gate)

1. Integration test against a local server: 200 with body → file byte-identical.
2. No-range server (`Accept-Ranges: none`) → completes as single stream.
3. Kill the process mid-download (SIGKILL equivalent) → restart resumes and completes with correct hash.
4. `ETag` changed between runs → restart from zero, old partial preserved, reason surfaced.
5. `Last-Modified` + size path tested; weak `ETag` path tested for precedence.
6. Timeout and 5xx retry with backoff; retry exhaustion surfaces `FAILED` with a typed error.
7. Cancel leaves no orphan file; completed file appears only after verification.
8. Redirect chain (including cross-host) followed with limits; loop rejected.

### Explicitly out of scope

- Parallel connections / segmentation.
- Scheduler, categories, media.

---

## Phase 3 — Dynamic segmentation

**Status:** ⬜ planned

### Contains

- Range planner with a shared cursor and dynamic claim (not fixed pre-split).
- N connections (default 8, 1–16 configurable), warm-connection reuse.
- Merge of segment ranges into the final file; integrity verification.
- Fallback and degradation strategy when a server misbehaves.
- Progress accounting that reconciles segment bytes.

### Success criteria (exit gate)

1. Planner unit tests: ranges cover `[0, len)` exactly once, no gaps, no overlaps, edge sizes (0, 1 byte, less than N segments, exact multiple).
2. Integration: 8-segment download from a local server equals the source hash.
3. Server that ignores ranges → automatically degrades to single stream, still correct.
4. Server that returns shorter 206 ranges than requested → no corruption.
5. Mid-download connection failure on one segment → that range is retried, others continue.
6. Pause during segmented download → resume continues all segments correctly.
7. Throughput measurement recorded against the local test server (reported as measured, not promised).

### Explicitly out of scope

- Media manifests.
- Advanced congestion control or protocol tricks.

---

## Phase 4 — Queue, categories, scheduling

**Status:** ⬜ planned

### Contains

- Queue with priorities, reordering, global concurrency limit.
- Category engine: default rules, user overrides, destination templates.
- Scheduler: start/stop windows, days, repeat, one-shot; fake clock in tests.
- Optional "shut down when queue completes" (off by default, explicit confirmation).

### Success criteria (exit gate)

1. Fake-clock tests: window start, window stop, missed window, DST jump, clock moved backwards.
2. Priority ordering and preemption test; concurrency cap never exceeded.
3. Category rule tests: extension mapping, user rule precedence, ambiguous extensions, unknown extension → Other.
4. Destination template tests, including invalid characters per OS.
5. Manual pause is never overridden by a scheduled window.
6. Shutdown action is unit-tested behind a trait; never executed in CI.

### Explicitly out of scope

- Site grabber / mirroring.
- Cloud sync.

---

## Phase 5 — Media capture (direct, HLS, DASH; optional helper)

**Status:** ⬜ planned

### Contains

- Direct media URLs through the normal engine.
- HLS: master playlist, variant selection, segment download, concatenation.
- DASH: manifest parsing, separate audio/video tracks, optional mux.
- Optional site extraction through an isolated helper (feature flag, off by default).
- DRM detection and refusal; legal notice acceptance.

### Success criteria (exit gate)

1. Local fixtures: simple m3u8, master m3u8 with variants, fMP4 and TS segments → correct output.
2. Local fixtures: MPD with separate tracks → downloaded; mux only when a muxer is present, otherwise clearly reported.
3. Malformed manifest fuzz/property test: no panic, typed error.
4. DRM-protected fixture → refused with an explanatory error, no bytes written to the final file.
5. Helper adapter tests: argument array is exact (snapshot), output parsing tolerates noise, timeout kills the process.
6. Legal notice cannot be bypassed silently (first-run state test).

### Explicitly out of scope

- Browser extension and download interception.
- Site mirroring / grabber.
- Any DRM circumvention.

---

## Phase 6 — Packaging, signing, updates

**Status:** ⬜ planned

### Contains

- Windows NSIS installer, macOS DMG, Linux AppImage + `.deb`.
- Code signing (Windows), notarization (macOS), checksums (Linux).
- GitHub Actions release workflow on tags; artifact provenance.
- Tauri updater integration, opt-in checks, signature verification.

### Success criteria (exit gate)

1. Release workflow produces installable artifacts from a tag only; no manual uploads.
2. Install/uninstall checklist executed on clean Windows, macOS, and Linux VMs and recorded in the PR.
3. Updater rejects an unsigned or wrongly signed artifact (automated negative test).
4. Update check is opt-in and off by default (tested setting path).
5. Artifacts carry checksums and are verifiable.

### Explicitly out of scope

- App Store / Microsoft Store distribution.
- Delta updates.

---

## Phase 7 — Hardening and 1.0

**Status:** ⬜ planned

### Contains

- Fuzz/property tests on parsers and path handling.
- Accessibility audit and fixes (keyboard, screen reader, contrast).
- Performance measurements replacing all "target" numbers.
- Documentation truth pass: every claim either tested or labeled untested.
- Security review checklist execution.

### Success criteria (exit gate)

1. Fuzz targets run for a fixed iteration budget in CI without panic.
2. Accessibility checks pass for the main flows; manual screen-reader pass recorded.
3. Startup, memory, and segmented-speed numbers measured and documented with methodology.
4. No unlabeled performance or compatibility claim remains.
5. All Phase 1–6 exit tests green on all three platforms.
6. Owner signs off on 1.0 scope.

### Explicitly out of scope

- Post-1.0 features (browser extension, site grabber, CLI).

---

## Phase gates at a glance

| Phase | Theme | Exit gate |
|---|---|---|
| 0 | Docs + CI | Docs green, decisions approved |
| 1 | Shell + i18n | Locale/RTL/settings tests green, 3-OS bundle builds |
| 2 | Download + resume | Kill/resume, validators, retries, integrity |
| 3 | Segmentation | Planner invariants, hash equality, fallback |
| 4 | Queue + schedule | Fake-clock windows, priorities, categories |
| 5 | Media | Fixtures for HLS/DASH, DRM refusal, helper isolation |
| 6 | Ship | Signed artifacts, updater negative test, clean-VM installs |
| 7 | 1.0 | Fuzz, a11y, measured performance, truthful docs |

---

## Rules that apply to every phase

1. Each phase starts with an Issue containing its scope and exit tests, and ends with a PR that satisfies them.
2. A phase does not start until the previous phase's exit gate is green and approved.
3. Tests are written with the feature, in the same PR.
4. Any change to this roadmap requires owner approval and an ADR update if a decision changes.
