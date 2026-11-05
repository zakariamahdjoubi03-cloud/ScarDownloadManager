# ADR-0004: Project license

**Status:** Proposed (awaiting owner approval — **decision required**)
**Date:** 2026-11-05
**Deciders:** Project owner

---

## Context

The owner asked for a license such as MIT or GPL-3.0 and has not chosen. The choice affects:

- what others may do with the code (adoption vs. copyleft protection);
- which third-party binaries may be **bundled** later (media extraction, muxing);
- contributor expectations and the project's ability to accept code;
- the ability to publish installers that include third-party components.

Constraints already decided elsewhere:

- The repository must never contain secrets or third-party binaries at rest.
- Optional helpers are isolated processes, not linked libraries ([ADR-0005](0005-video-download-strategy.md)).
- yt-dlp is **Unlicense** (public domain) — bundling is unrestricted.
- ffmpeg builds are **LGPL-2.1+** or **GPL-2.0+/GPL-3.0** depending on configuration; bundling a GPL build would impose GPL on the distributed combination.
- SQLite is public domain; React, Vite, Tauri, and most Rust crates are MIT/Apache-2.0 (permissive).

---

## Options considered

| Criterion | MIT | GPL-3.0 |
|---|---|---|
| Adoption by individuals and companies | ✅ Highest | ⚠️ Lower — copyleft deters some integrators |
| Guarantee that improvements stay open | ❌ None | ✅ Strong |
| Compatibility with bundled GPL ffmpeg | ❌ Not allowed (would require relicense) | ✅ Natural |
| Compatibility with bundled LGPL ffmpeg | ✅ Yes | ✅ Yes |
| App-store / store-policy friction | ✅ Low | ⚠️ Some stores scrutinize copyleft |
| Simplicity for contributors | ✅ Very simple | ⚠️ More obligations to understand |
| Alignment with "open and auditable" promise | ✅ | ✅ (stronger) |

---

## Decision

**Recommended: MIT**, with these conditions:

1. No GPL-licensed binary is ever bundled. If muxing is included, use an **LGPL ffmpeg build** or let the user supply ffmpeg at runtime.
2. A `THIRD_PARTY_NOTICES.md` file lists every bundled component, its version, and its license, updated in the same PR that adds it.
3. Dependencies are license-scanned in CI; a GPL/AGPL dependency requires a new ADR before merge.
4. The `LICENSE` file is added **only after this ADR is approved** — the foundation PR intentionally ships without one.

**If the owner prefers copyleft**, choose **GPL-3.0** instead. The codebase is identical; the consequences are: all distributed derivations must remain GPL, GPL ffmpeg becomes permissible, and the `LICENSE` file changes. This ADR would be superseded by ADR-0007.

---

## Consequences

### If MIT (recommended)

- Maximum adoption and contribution; minimal friction.
- No copyleft guarantee; a competitor may ship a closed fork.
- Bundling choices constrained to permissive/LGPL components.

### If GPL-3.0

- Improvements to distributed forks stay open.
- Some corporate users and stores avoid it; adoption narrows.
- Bundling GPL ffmpeg becomes legal and simple.

---

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| License chosen casually, then bundling conflicts appear | Third-party notice + CI license scan from Phase 1 |
| Contributor submits incompatible code | Contributor checklist in PR template (Phase 1); license review in PR |
| Owner changes mind after 1.0 | Relicensing requires consent of all copyright holders; decide now, early |

---

## Follow-ups

1. Owner approves one option and records it here (Status → Accepted).
2. Add `LICENSE` and `THIRD_PARTY_NOTICES.md` in the Phase 1 PR.
3. Add a license-scanning job to CI when dependencies appear.
