# Product Requirements Document — ScarDownloadManager

| Field | Value |
|---|---|
| Status | Draft — awaiting approval |
| Version | 0.1 (foundation) |
| Owner | Project owner (zakariamahdjoubi03-cloud) |
| Last updated | 2026-11-05 |
| Implementation status | None. No code exists yet; every requirement below is **untested** unless a test is explicitly linked. |

---

## 1. Summary

ScarDownloadManager is a desktop download manager for Windows, macOS, and Linux. It accelerates transfers by splitting files into byte ranges fetched over parallel connections, resumes interrupted transfers from the last confirmed byte, captures downloadable media from supported sources, organizes finished files into categories, and runs download queues on a schedule.

It is designed for people who download large files on connections that are slow, metered, or unreliable.

---

## 2. Problem

Browsers are poor download managers. They:

- rarely parallelize a single transfer, so one connection caps the speed;
- lose progress when a laptop sleeps, the network drops, or the app is closed;
- offer no queue, no scheduling, no bandwidth limits;
- scatter finished files into a single download folder;
- cannot capture HLS/DASH media streams at all;
- give no visibility into speed, remaining time, retries, or per-file progress.

Commercial tools such as IDM solve this on Windows but are proprietary, Windows-only, and not available on macOS or Linux. ScarDownloadManager targets the same workflow on all three desktops with an open, auditable implementation.

---

## 3. Goals

| ID | Goal |
|---|---|
| G1 | Download a single file measurably faster than a single-stream client when the server supports byte ranges. |
| G2 | Never lose completed bytes to an interruption when the server allows resumption. |
| G3 | Let a user start, pause, resume, retry, and cancel any download without losing data. |
| G4 | Capture downloadable media (direct media, HLS, DASH) that a browser cannot save. |
| G5 | Organize completed files automatically and predictably. |
| G6 | Schedule work: start at a time, stop at a time, limit concurrency, optionally shut down when done. |
| G7 | Ship a UI that is fully translatable, right-to-left correct, and selectable in Settings. |
| G8 | Be trustworthy: no telemetry by default, no hidden network calls, no bundled adware. |

---

## 4. Non-goals

These are deliberate exclusions, not omissions:

| ID | Non-goal |
|---|---|
| NG1 | **No DRM circumvention.** The product will not decrypt, strip, or bypass DRM, encryption, or license checks. |
| NG2 | **No paywall or authentication bypass.** Media behind a login or paywall is out of scope. The user may supply cookies for sites they are entitled to access; the app never harvests credentials. |
| NG3 | **No copyright infringement assistance.** The app downloads what the user points it at; it does not index, search, or recommend pirated content. A first-run notice explains user responsibility. |
| NG4 | Not a browser and not a browser extension in v1. Browser handoff is a later phase. |
| NG5 | Not a BitTorrent client. |
| NG6 | No mobile app in v1 (no iOS/Android). |
| NG7 | No cloud storage sync, no account system, no server-side component. |
| NG8 | No ad-supported installer, no bundling of unrelated software. |
| NG9 | No scraping of sites that forbid it; no circumvention of rate limits. |
| NG10 | No silently auto-updating without a signed artifact and a visible changelog. |

---

## 5. Users and jobs to be done

| Persona | Job | Success looks like |
|---|---|---|
| Gamer | "Download a 60 GB game update overnight without restarting it" | Queue survives sleep and resume; finishes unattended; one notification |
| PC user | "Grab an ISO and a driver package quickly" | Parallel segments saturate the link; files land in the right category folder |
| Video fan | "Save a lecture video that plays in a web page" | HLS/DASH manifest downloaded and merged into a playable file |
| Unstable connection | "Do not restart a 4 GB download after a dropout" | Resume continues at the last confirmed byte, not zero |
| Student | "Download 30 PDFs into folders by course" | Queue with per-file destination and category rules |

---

## 6. Functional requirements

Each requirement carries acceptance criteria. **A requirement is done only when its criteria are covered by automated tests**; otherwise it is marked untested.

### FR-1 — Segmented download

The engine splits a file into byte ranges and downloads them concurrently.

Acceptance criteria:

- AC-1.1 It probes the resource (HEAD, falling back to a `Range: bytes=0-0` GET) and reads `Accept-Ranges`, `Content-Length`, `ETag`, `Last-Modified`.
- AC-1.2 If ranges are supported and the file is large enough, it opens N connections (default 8, user configurable 1–16).
- AC-1.3 Segments are **allocated dynamically** while the transfer runs (a finished segment's connection is reused for the next unclaimed range), not fixed at start.
- AC-1.4 If ranges are not supported, it falls back to a single stream without failing.
- AC-1.5 The merged file is byte-identical to the source when `Content-Length` and hash are available; otherwise size is verified and the limitation is surfaced to the user.
- AC-1.6 Servers returning `206` with a shorter range than requested are handled without corruption.
- AC-1.7 Server errors (`429`, `5xx`) are retried with exponential backoff and jitter; retry counts are user configurable.

### FR-2 — Resume

Acceptance criteria:

- AC-2.1 Partial data and per-segment progress persist in a sidecar file next to the destination, written atomically.
- AC-2.2 After a forced process kill during download, the next run resumes without re-downloading confirmed bytes.
- AC-2.3 Resume validates the remote identity via strong `ETag` first, weak `ETag` second, `Last-Modified` + size third; on mismatch it restarts the file and tells the user why.
- AC-2.4 A resume attempt against a server that no longer honors ranges restarts cleanly instead of corrupting the file.
- AC-2.5 Machine shutdown and sleep are treated as ordinary interruptions; no manual repair step is required.

### FR-3 — Media capture

Acceptance criteria:

- AC-3.1 Direct media URLs (`.mp4`, `.webm`, `.mkv`, `.mp3`, …) download with the normal engine.
- AC-3.2 HLS manifests (`.m3u8`, including master playlists with variant streams) are resolved, segments downloaded, and concatenated into one file.
- AC-3.3 DASH manifests (`.mpd`) are parsed; separate audio and video tracks can be downloaded and muxed if a muxer is available.
- AC-3.4 Optional site extraction runs through an isolated helper process with an explicit allowlist; the helper is optional and clearly labeled.
- AC-3.5 The app refuses DRM-protected streams and explains the refusal (see NG1).
- AC-3.6 Legal notice: the user confirms they have the right to download the content.

### FR-4 — Organization

Acceptance criteria:

- AC-4.1 Every completed download is classified into Videos, Programs, Games, Documents, Audio, Archives, or Other.
- AC-4.2 Classification uses an editable rule set: extension lists plus user rules, first match wins, with a documented default order.
- AC-4.3 The user can override the destination folder per download, per category, and globally.
- AC-4.4 Ambiguous extensions (`.iso`, `.dmg`, `.bin`) follow a documented rule and are not silently guessed.
- AC-4.5 Moving a file preserves the record's history; a failed move reports the reason.

### FR-5 — Scheduling and queue

Acceptance criteria:

- AC-5.1 The user can define: start time, stop time, days of week, and "run once" vs "repeat".
- AC-5.2 The scheduler starts the queue at the configured time and pauses at the stop time without losing progress.
- AC-5.3 Queue supports priorities (move up/down, priority levels) and a global concurrency limit.
- AC-5.4 Optional "shut down the computer when the queue finishes" is off by default and requires explicit confirmation.
- AC-5.5 Clock changes, sleep, and missed windows are handled deterministically and unit-tested with a fake clock.
- AC-5.6 A scheduled window never starts a download the user has paused manually.

### FR-6 — Settings

Acceptance criteria:

- AC-6.1 Settings persist across restarts and survive update.
- AC-6.2 Options include: default download folder, max connections, speed limit, retry policy, proxy, language, theme, notification preferences.
- AC-6.3 Secrets (proxy password) are stored in the OS keychain, never in plain site config.
- AC-6.4 A "reset to defaults" path exists and is tested.

### FR-7 — Internationalization

Acceptance criteria:

- AC-7.1 No user-visible string is hardcoded in the UI; a lint/test fails the build if one is found.
- AC-7.2 First run resolves the language from the OS locale, falls back to English, and persists the choice.
- AC-7.3 Language can be changed in Settings without restarting the app.
- AC-7.4 Arabic renders correctly with `dir="rtl"`, mirrored icons where appropriate, and no clipped layout at 100% and 125% scaling.
- AC-7.5 Pluralization, dates, numbers, and file sizes use locale-aware formatting.
- AC-7.6 Where the installer supports language selection, the choice seeds the first-run locale.
- AC-7.7 Missing translations fall back to English visibly, and a missing-key check runs in CI.

### FR-8 — Updates and distribution

Acceptance criteria:

- AC-8.1 Update packages are signed; the app refuses an unsigned or wrongly signed update.
- AC-8.2 Automatic checks are opt-in; the changelog is shown before install.
- AC-8.3 Release artifacts are produced only by GitHub Actions from a tagged commit.
- AC-8.4 Background/self-update is not required in v1; a manual download-and-install path is acceptable.

---

## 7. Non-functional requirements

| ID | Requirement | Measure |
|---|---|---|
| NFR-1 | Cross-platform parity | The core feature set works on Windows, macOS, and Linux; CI builds and tests all three on every PR |
| NFR-2 | Startup | Cold start to interactive UI under 2 s on a mid-range machine (target, to be measured — **unvalidated**) |
| NFR-3 | Memory | Idle under 250 MB RSS with 100 completed records; active downloads scale sub-linearly (target — **unvalidated**) |
| NFR-4 | Integrity | No corrupted file may be marked complete; failure must surface, never silently pass |
| NFR-5 | Security | No secrets in the repository; every external input validated; dependency and license scanning in CI |
| NFR-6 | Privacy | Zero telemetry by default; no network calls except the user's downloads and opt-in update checks |
| NFR-7 | Accessibility | Keyboard navigation for every action; screen-reader labels on controls; WCAG 2.1 AA contrast target |
| NFR-8 | Reliability | Resume survives kill -9 in tests; scheduler survives clock jumps |
| NFR-9 | Installer | Install and uninstall leave no orphan services; user data removal is explicit |
| NFR-10 | Localization | Adding a language requires no code changes — only a locale file |

---

## 8. Constraints, legal, and ethics

1. **Copyright.** The tool downloads files the user selects. It does not host, index, or promote content. The first-run flow carries a plain-language notice: the user is responsible for having the right to download what they request.
2. **DRM and access control.** Refusing DRM-protected content is a product rule, not a limitation to be worked around later.
3. **Terms of service.** Optional site extraction must respect a site's terms; any allowlist is documented and reviewable, and disabled by default.
4. **Rate limiting.** The app must expose a bandwidth limit and default retry behavior that does not hammer a server.
5. **Licensing.** Bundled third-party binaries require a recorded license decision before merge (see [ADR-0004](adr/0004-license.md), [ADR-0005](adr/0005-video-download-strategy.md)).
6. **Trademark.** "IDM" and "Internet Download Manager" belong to Tonec FZE. They are referenced only as inspiration; no IDM assets, icons, or code are used.

---

## 9. Success metrics

Targets for the first public release. All are **unvalidated** until measured; none may be claimed in docs before a test or measurement exists.

| Metric | Target | How measured |
|---|---|---|
| Resume success rate | ≥ 95% of interrupted downloads resume without restart | Integration test suite + opt-in local telemetry (none collected by default) |
| Segmented speedup | Median ≥ 1.5× single-stream on a test server with 8 segments | Automated benchmark against a local range-capable server (**must be measured, not asserted**) |
| Crash-free sessions | ≥ 99% | Local crash logs only; no upload without consent |
| UI coverage | 100% of user-visible strings translated in English and Arabic | CI missing-key check |
| Platform builds | Green on Windows, macOS, Linux for every PR | CI |

---

## 10. Out of scope for v1

- Mobile apps.
- Browser extension and browser download interception.
- Site mirroring / site grabber.
- FTP, SFTP, Metalink.
- Torrent.
- Cloud accounts, sync, sharing.
- Plugin marketplace.
- Command-line interface (may come later; the engine is designed to allow it).

---

## 11. Open questions requiring owner approval

| # | Question | Default if approved silently |
|---|---|---|
| Q1 | Tech stack: Tauri 2 + Rust/React, or Electron, or Flutter, or Avalonia? | Tauri 2 (see ADR-0001) |
| Q2 | License: MIT or GPL-3.0? | MIT (see ADR-0004) |
| Q3 | Repository visibility: private until 1.0, or public now? | Private until foundation approved |
| Q4 | Docs language: English only, or bilingual English + Arabic? | English with an Arabic overview in README |
| Q5 | Is yt-dlp acceptable as an optional helper, or must media capture be native-only? | Optional, off by default (see ADR-0005) |
| Q6 | Should the installer offer language selection in v1, or is first-run detection enough? | Both where the installer supports it; detection elsewhere |

---

## 12. Glossary

| Term | Meaning |
|---|---|
| Segment | A contiguous byte range of a file downloaded over one connection |
| Dynamic segmentation | Allocating new ranges to connections as they free up, during the transfer |
| Resume | Continuing from the last confirmed byte instead of restarting |
| Sidecar | A small metadata file stored beside the partial download |
| HLS / DASH | HTTP-based streaming formats using playlists/manifests of segments |
| RTL | Right-to-left script direction (Arabic, Hebrew, …) |
| Untested | Documented but not yet proven by an automated test |
