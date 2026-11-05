# ADR-0003: Data storage

**Status:** Proposed (awaiting owner approval)
**Date:** 2026-11-05
**Deciders:** Project owner

---

## Context

The application must persist:

- downloads and their state, including per-segment progress;
- queue order, priorities, and schedules;
- categories and classification rules;
- settings (single source of truth);
- a local event/history log.

Requirements:

- **Crash safety.** A power loss at any moment must not corrupt progress or mark a partial file complete.
- **Concurrency.** The engine writes progress frequently while the UI reads; writes must be atomic and ordered.
- **Scale.** Hundreds to low thousands of download records with segments — small, but queries must stay fast.
- **Migrations.** Schema will evolve; user data must survive upgrades.
- **Portability.** No server, no account, single-file backup/export.

---

## Decision

Use **SQLite** in WAL mode, accessed through **`sqlx`** with compile-time checked queries and versioned migrations.

Rules:

1. Single writer (the Rust store layer); readers may be concurrent.
2. One database file in the platform app-data directory.
3. Migrations are numbered, forward-only, and applied at startup; destructive migrations take a backup first.
4. Settings live in the same database as key/value JSON.
5. Progress writes are batched and transactional; a write is only acknowledged after commit.
6. Export/import of user data is JSON, independent of the schema version.

---

## Options considered

| Option | Crash safety | Query power | Migration story | Concurrency | Verdict |
|---|---|---|---|---|---|
| **SQLite + sqlx** | ✅ WAL, ACID | ✅ Full SQL | ✅ Mature tooling | ✅ WAL readers + one writer | ✅ Chosen |
| JSON files per download | ⚠️ Depends on atomic rename discipline | ❌ Manual scans | ❌ Hand-written | ❌ Locking is ad hoc | Rejected |
| Single JSON state file | ❌ Rewrite risk, large file | ❌ Full scan | ❌ Rewrite on every change | ❌ | Rejected |
| redb / sled (embedded KV) | ✅ | ⚠️ No SQL, manual indexes | ⚠️ Custom | ✅ | Rejected — more code for no benefit |
| Platform APIs (registry/plist) | ⚠️ | ❌ | ❌ | ❌ | Rejected |

JSON was tempting for simplicity, but this app's core promise is *resume after a crash*; that promise is exactly where ad-hoc file writes fail. SQLite gives atomic commits and a real migration path for the cost of one well-known dependency.

---

## Consequences

### Positive

- Atomic progress commits; crash recovery is a documented scan, not a guess.
- SQL for queue ordering, filters, and history; indexes are explicit.
- `sqlx` migrations and compile-time query checks reduce schema drift.
- Single file to back up; easy export path.

### Negative

- A native dependency (SQLite) in the build; cross-compilation needs attention.
- `sqlx` compile-time checks require a prepared database or offline mode in CI.
- Requires discipline: one writer, short transactions, no long-held locks.

---

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| CI lacks a database for `sqlx` compile checks | Use `sqlx` offline mode (`.sqlx` metadata committed) or run migrations before check |
| WAL files left behind on crash | Recovery test: kill during write, reopen, assert consistency |
| Schema migration breaks user data | Backup before destructive migration; migration test with a seeded old schema |
| DB grows with history | Retention policy for events; vacuum on demand |

---

## Follow-ups

1. Phase 1: finalize schema v1 (downloads, segments, categories, schedules, settings, events).
2. Phase 2: kill-during-write recovery test becomes part of the exit gate.
