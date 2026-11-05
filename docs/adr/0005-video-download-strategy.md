# ADR-0005: Video capture strategy

**Status:** Proposed (awaiting owner approval)
**Date:** 2026-11-05
**Deciders:** Project owner

---

## Context

The product should let users download videos that a browser plays. There are several very different technical paths, with very different legal and security profiles:

1. **Direct media URLs** (`.mp4`, `.webm`, `.mp3`) — trivially downloadable; the normal engine handles them.
2. **HLS / DASH manifests** — the page hands the player a playlist; segments are ordinary HTTP resources. Parsing and assembling them is engineering work, not circumvention.
3. **Site-specific extraction** (e.g., yt-dlp) — resolves page metadata and stream URLs for thousands of sites. Powerful, but concentrates legal and maintenance risk, and some sites' terms forbid it.
4. **Browser interception** (extension + native messaging) — captures what the browser requests. Highest fidelity; requires maintaining extensions for several browsers.
5. **Local proxy / TLS interception** — inspect all traffic. Rejected outright: it is a security and privacy hazard, requires installing a root certificate, and is indistinguishable from malware behavior.

The owner's requirement is "detects downloadable videos from some sites, depending on the site and how the video plays". That maps to paths 1–2 natively, and optionally 3–4 later.

---

## Decision

**Phase 5 (native, always):**

- Direct media through the standard engine.
- HLS: master/variant playlists, segment fetch, concatenation.
- DASH: manifest parsing, separate audio/video tracks, mux only when a suitable permissively licensed muxer is present.

**Optional, feature-flagged, off by default:**

- A **yt-dlp sidecar** adapter for supported sites, isolated as a child process, launched with an argument array, subject to a documented allowlist and a global timeout. Because yt-dlp is Unlicense, bundling is permitted, but the app will prefer a user-provided binary or a clearly labeled optional component.

**Not in v1:**

- Browser extension and native-messaging interception (post-1.0 candidate; own ADR required).
- Site mirroring / grabber.

**Always refused:**

- DRM-protected streams, paywall bypass, and authenticated content the user is not entitled to. The app detects DRM markers and explains the refusal.

---

## Options considered

| Option | Coverage | Maintenance | Legal risk | Security risk | Verdict |
|---|---|---|---|---|---|
| Native direct + HLS/DASH | Medium (most open media) | Low | Low | Low | ✅ v1 |
| yt-dlp sidecar | Very high | Medium (upstream moves fast) | Medium (site terms) | Medium (parsing untrusted output) | ✅ optional |
| Browser extension | High (real session) | High (per-browser stores) | Medium | Medium (native messaging surface) | ⏳ post-1.0 |
| Local proxy / MITM | Highest | High | High | **Unacceptable** | ❌ rejected |

---

## Consequences

### Positive

- v1 ships useful media support without legal gray zones.
- The engine stays generic; media formats are just another source of byte ranges.
- Optional extraction is isolated: a crash or hang cannot take down the app.
- Clear product boundary: no DRM, no bypass.

### Negative

- Users will ask for "site X" support; native support won't cover it.
- yt-dlp integration adds a version-churn dependency and output-parsing risk.
- Muxing quality depends on an external binary being present.

---

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| yt-dlp output changes between versions | Adapter tests pinned to recorded fixtures; version probe; typed parse errors |
| A site blocks or rate-limits extraction | Respect terms, document supported sites, fail gracefully |
| Legal complaint about a specific site | Allowlist is reviewable and removable without a release (config) |
| Malicious manifest or metadata | Bound sizes, validate types, fuzz parsers |
| Muxer license contamination | LGPL build only, or user-supplied binary; recorded in ADR-0004 |

---

## Follow-ups

1. Phase 5: define the DRM-detection heuristics and their false-positive behavior.
2. Phase 5: snapshot tests for yt-dlp arguments and fixture outputs.
3. Post-1.0: separate ADR for browser interception, including store policy review.
