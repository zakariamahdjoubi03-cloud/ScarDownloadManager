# ADR-0002: Internationalization strategy

**Status:** Proposed (awaiting owner approval)
**Date:** 2026-11-05
**Deciders:** Project owner

---

## Context

Requirements:

- English is the source and default language.
- **Every language** must be addable — the catalog is community-contributed.
- **Arabic is required** and must be fully right-to-left correct.
- Language is selectable **during installation where the installer supports it**, and always changeable in Settings.
- On first run the app should follow the **operating system locale**.
- No user-visible string may be hardcoded.
- Dates, numbers, plurals, and file sizes must be locale-aware.
- Adding a language must not require a code change.

IDM's own multilingual interface is a useful benchmark: it ships dozens of language packs and a language menu, with no restart needed for most screens.

---

## Decision

1. **Library:** `i18next` + `react-i18next` in the UI, with **ICU MessageFormat** for plurals, gender, and selection.
2. **Format:** JSON, one file per namespace per locale, under `locales/<locale>/<namespace>.json`.
3. **Source of truth:** English (`en`), reviewed like code.
4. **Resolution order:** explicit user setting → installer-seeded value → OS locale (via Tauri/OS APIs) → `en` fallback.
5. **RTL:** a single `dir` attribute driven by locale metadata; CSS uses logical properties; directional icons mirror.
6. **Installer:** where the format supports languages (NSIS on Windows), show a language selector and persist the choice into first-run configuration; macOS DMG and Linux AppImage fall back to OS detection.
7. **CI enforcement:** missing-key check, unused-key check, and a hardcoded-string lint rule.
8. **Catalog workflow:** locale files are plain JSON in-repo; a translation service may be added later without changing the runtime format.

---

## Options considered

| Option | Strengths | Weaknesses | Verdict |
|---|---|---|---|
| **i18next + ICU JSON** | Mature React support, lazy loading, namespaces, ICU plurals, huge ecosystem | Runtime size (~15 KB), JSON can drift if unreviewed | ✅ Chosen |
| Fluent (Rust/JS) | Excellent plural/gender model, designed for localization | Two implementations (Rust + JS), smaller React ecosystem, less familiar to contributors | Rejected for v1 |
| OS-native resource strings | Zero runtime dependency | No cross-platform unity, no hot language switching, HTML UI cannot consume it | Rejected |
| gettext / PO files | Translator-friendly tooling | Awkward for dynamic web UI and RTL metadata, weaker TypeScript support | Rejected |
| Custom JSON loader | Full control, tiny | Reinvents plurals, formatting, fallback, and tooling | Rejected |

---

## Consequences

### Positive

- Translators can contribute without touching code.
- Hot language switching in Settings, no restart.
- ICU handles Arabic plurals (six forms) correctly.
- CI prevents missing translations from shipping silently.

### Negative

- Locale JSON is another artifact to review; drift is possible without checks (mitigated by CI).
- ICU messages are more complex to author than plain strings; contributors need a short guide.
- RTL correctness requires testing at multiple zoom levels, not only a `dir` flip.

---

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| Arabic layout breaks (clipping, mirrored icons) | Snapshot tests at 100%/125%; manual review checklist per UI change |
| Translator introduces placeholder errors (`{count}` typo) | ICU parse check in CI; placeholder-preservation test |
| Locale detection returns an unsupported tag (`ar-EG` vs `ar`) | Normalize with BCP-47 fallback chain; unit tests |
| Installer language and app language diverge | Installer value is a seed only; Settings wins after first change |

---

## Follow-ups

1. Phase 1: define the namespace layout and the CI missing-key script.
2. Phase 1: decide punctuation and file-size unit localization (decimal separators, `KB` vs `ك.ب`).
3. Post-1.0: evaluate a translation platform (Crowdin/Weblate) — no runtime change required.
