# ScarDownloadManager

**A fast, resumable, cross-platform download manager for Windows, macOS, and Linux.**

> Inspired by the *workflow* of Internet Download Manager (IDM) — segmented acceleration, resume, scheduling, categories, and media capture. Not a clone, not affiliated with Tonec FZE, and not built from IDM code.

---

## Status

**🚧 Foundation — documentation only. There is no runnable application yet.**

This repository currently contains the project's decision record, requirements, architecture, roadmap, contribution rules, and CI scaffolding. Code starts only after the foundation documents are approved. See [`docs/ROADMAP.md`](docs/ROADMAP.md) Phase 1 for the first coding milestone.

**Honesty rule:** this project documents *intended* behavior until a test proves otherwise. Anything not yet covered by an automated test is marked **untested**.

---

## What it does

| Capability | Description | Phase |
|---|---|---|
| ⚡ **Faster downloads** | Splits a file into byte ranges and downloads them over parallel connections; segments are allocated *dynamically* while the transfer runs, the way IDM does it. Falls back to a single stream when the server does not support ranges. | 3 |
| 🔄 **Resume** | Interrupted transfers continue from the last confirmed byte — after a dropped connection, a crash, or a reboot — when the server supports it (validated with `ETag` / `Last-Modified`). | 2 |
| 🎬 **Media capture** | Downloads videos from direct media URLs, HLS (`.m3u8`) and DASH (`.mpd`) manifests; optional extraction for supported sites through a separately licensed helper. | 5 |
| 🗂️ **Organization** | Sorts completed files into categories — Videos, Programs, Games, Documents, Audio, Archives, Other — using extension and rule-based classification. | 4 |
| ⏰ **Scheduling** | Starts a queue at a chosen time, stops it at a deadline, limits concurrent downloads, and optionally shuts the machine down when the queue finishes. | 4 |
| 🌍 **Multilingual UI** | English by default; every language can be added. Language is selectable during installation (where the installer supports it) and later in Settings. Arabic is a first-class language with full RTL layout. | 1 |

---

## Who it is for

| User | Benefit |
|---|---|
| 🎮 Gamers | Large game files and updates that survive flaky connections |
| 💻 PC users | Programs, ISOs, and drivers from direct links |
| 🎬 Video fans | Videos the user is allowed to download (HLS/DASH and direct media) |
| 📶 Slow or unstable internet | Resume instead of restarting from zero |
| 📚 Students | PDFs, lessons, and course material downloaded in batches and organized into folders |

---

## Platforms

Windows 10/11 · macOS 12+ (Intel and Apple Silicon) · Linux (AppImage and `.deb` first; other formats later).

All three platforms are first-class targets in CI from Phase 0.

---

## Languages

- **Primary:** English (source language of the UI).
- **Required:** Arabic (RTL).
- **Planned:** every language, community-contributed through locale files.
- **Detection:** the app follows the operating system locale on first run and remembers the user's choice.
- **Installer:** where the packaging format allows language selection, the choice seeds the app's first-run locale (see [ADR-0002](docs/adr/0002-i18n-strategy.md)).

---

## Repository map

| Path | Purpose |
|---|---|
| [`docs/PRD.md`](docs/PRD.md) | Requirements: what the product does, for whom, and what it explicitly does not do |
| [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) | Components, data flow, security boundaries, failure modes |
| [`docs/adr/`](docs/adr/) | One decision record per major choice — options, trade-offs, consequences |
| [`docs/ROADMAP.md`](docs/ROADMAP.md) | Numbered phases with entry criteria, exit tests, and explicit exclusions |
| [`AGENTS.md`](AGENTS.md) | Operating rules for every agent or human building the project |
| `.github/workflows/ci.yml` | Build and test pipeline (Windows, macOS, Linux) |

---

## How work happens

1. One GitHub **Issue per phase**.
2. One **branch and small PR per change** — no direct pushes to `main`.
3. Every PR must pass **all CI checks on all three operating systems** and receive review before merge.
4. Every feature ships **with its tests**, including edge cases and error paths.
5. Security from day one: no secrets in code, sanitize everything that comes from users or the network.
6. Every PR description includes the prompt that produced it.
7. Releases are built **only** by GitHub Actions, never by hand.

The complete rules are in [`AGENTS.md`](AGENTS.md).

---

## License

**Pending approval.** [ADR-0004](docs/adr/0004-license.md) compares MIT and GPL-3.0 and recommends **MIT**. No `LICENSE` file is added until that decision is approved.

---

## نطرة عامة بالعربية

**ScarDownloadManager** مدير تحميل سريع يعمل على ويندوز وماك ولينكس. يقسّم الملف إلى أجزاء لتحسين السرعة، ويستكمل التحميل بعد انقطاع الإنترنت أو إعادة تشغيل الجهاز، ويكتشف الفيديوهات القابلة للتحميل من بعض المواقع، وينظّم الملفات حسب النوع، ويسمح بجدولة التحميل.

الواجهة بالإنجليزية افتراضيًا ويمكن إضافة كل لغات العالم، والعربية مدعومة بالكامل مع اتجاه النص من اليمين لليسار.

**الحالة:** المرحلة التأسيسية — وثائق فقط، ولا يوجد كود بعد. لا يُبدأ الكود قبل الموافقة على الوثائق والقرارات.

---

## Acknowledgements

- Workflow inspiration: [Internet Download Manager](https://www.internetdownloadmanager.com/) by Tonec FZE. ScarDownloadManager is an independent project with no affiliation, endorsement, or shared code.
- If media extraction is ever bundled, it will be through [yt-dlp](https://github.com/yt-dlp/yt-dlp) (Unlicense / public domain) or a comparable permissive component, always with license review recorded in an ADR.
