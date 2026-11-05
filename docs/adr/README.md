# Architecture Decision Records

One record per decision that is expensive to reverse. Each record states its status, the context, the options considered, the decision, and the consequences. A decision that changes later is superseded — never silently edited.

| ADR | Title | Status |
|---|---|---|
| [0001](0001-tech-stack.md) | Desktop technology stack (Tauri 2 + Rust + React) | Proposed |
| [0002](0002-i18n-strategy.md) | Internationalization strategy (i18next + ICU, RTL, installer seeding) | Proposed |
| [0003](0003-data-storage.md) | Data storage (SQLite + sqlx, WAL, migrations) | Proposed |
| [0004](0004-license.md) | Project license (MIT recommended; GPL-3.0 alternative) | Proposed — **owner decision** |
| [0005](0005-video-download-strategy.md) | Video capture (native HLS/DASH, optional yt-dlp, no DRM) | Proposed |
| [0006](0006-distribution-and-updates.md) | Distribution, signing, updates (GitHub Releases + signed updater) | Proposed |

## How to add a decision

1. Copy the structure of an existing ADR.
2. Number it sequentially; never reuse a number.
3. Keep it short enough to be read in one sitting.
4. Link the PR that implements it.
5. Update this index in the same PR.
