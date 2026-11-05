# ADR-0006: Distribution, signing, and updates

**Status:** Proposed (awaiting owner approval)
**Date:** 2026-11-05
**Deciders:** Project owner

---

## Context

The app must be installable by non-technical users on three platforms, and updates must not become an attack vector. Constraints:

- Releases must be produced **only** by GitHub Actions (project rule).
- No paid infrastructure is assumed beyond what the owner already has.
- Windows SmartScreen and macOS Gatekeeper will block unsigned or unnotarized apps.
- Auto-update must be signed and opt-in; an unsigned update must be rejected.

---

## Decision

**Distribution:** GitHub Releases as the only channel for v1.

**Formats:**

| OS | Artifact | Notes |
|---|---|---|
| Windows | NSIS `.exe` (per-user install), MSI later | Per-user avoids admin prompt; MSI for enterprise later |
| macOS | Universal `.dmg` (arm64 + x86_64) | Notarized and stapled before upload |
| Linux | `.AppImage` and `.deb` | AppImage first; `.deb` for Debian/Ubuntu users |

**Signing:**

- Windows: code-signing certificate (OV/EV, or Azure Trusted Signing) once acquired; until then, builds are clearly marked unsigned development builds and excluded from the update channel.
- macOS: Developer ID certificate + notarization (requires a paid Apple Developer account — a cost decision for the owner).
- Linux: no central signature; publish SHA-256 checksums and sign the release metadata.

**Updates:**

- Tauri's built-in updater, **opt-in** (off by default), checking a signed manifest over HTTPS.
- Public key pinned in the app; signature verification before install; refusal is a tested path.
- Changelog shown before install; no silent forced updates.
- No delta updates in v1.

**CI:**

- `.github/workflows/release.yml` (Phase 6) triggers on a `v*` tag from `main` only, builds on the three-OS matrix, signs where keys exist, and attaches artifacts to the GitHub Release.
- Signing material lives only in GitHub Actions secrets; never in the repo or local machines.

---

## Options considered

| Option | Reach | Control | Cost | Verdict |
|---|---|---|---|---|
| **GitHub Releases + signed updater** | Medium (developers, advanced users) | Full | Low | ✅ Chosen |
| Microsoft Store / Mac App Store | High consumer reach | Low (store rules, review, sandbox limits) | $99/yr Apple; store fees | Rejected for v1 |
| Package managers (winget, Homebrew, Flathub) | High for technical users | Medium | Low | ⏳ post-1.0, low effort once artifacts exist |
| Website download page only | High | Full | Hosting cost | Complement, not a channel |

---

## Consequences

### Positive

- One pipeline produces reproducible, signed artifacts from a tag.
- No store review delays; full control of the update mechanism.
- Verifiable provenance: artifact ↔ commit ↔ CI run.

### Negative

- Consumers must be pointed to GitHub; discoverability is lower than a store.
- macOS notarization requires a paid account and adds CI time.
- Windows SmartScreen warnings persist until a certificate is purchased.
- Unsigned interim builds must be clearly labeled to avoid training users to ignore warnings.

---

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| Stolen signing key used to push a malicious update | Keys only in CI secrets; hardware-backed where possible; key rotation plan; updater pinned keys |
| Users ignore signature warnings | Never ship unsigned to the update channel; label dev builds visibly |
| Notarization rejected (bundled sidecar issues) | Notarize in Phase 6 with the real bundle; fix entitlements early |
| Linux glibc/WebKit incompatibility | Target oldest supported Ubuntu; document runtime dependencies; test on 22.04 and 24.04 |

---

## Follow-ups

1. Phase 6: owner decides whether to buy a Windows certificate and an Apple Developer account.
2. Phase 6: write `docs/RELEASE.md` with the manual checklist (clean-VM install, update, rollback).
3. Post-1.0: evaluate winget/Homebrew/Flathub submission.
