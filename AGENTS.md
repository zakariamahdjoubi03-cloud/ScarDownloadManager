# AGENTS.md — Working rules for ScarDownloadManager

This file is binding for every agent (AI or human) that touches this repository. If a rule conflicts with convenience, the rule wins. If a rule is ambiguous, **ask the owner** — do not decide alone.

---

## 0. Prime directive

Ship a small, correct, tested change at a time. Never trade correctness, security, or honesty for speed.

---

## 1. Workflow

1. **One issue per phase.** Each roadmap phase is a GitHub Issue containing scope, out-of-scope, and the exact success tests. Phases are defined in [`docs/ROADMAP.md`](docs/ROADMAP.md).
2. **One branch per change.** Name it `phase-N/<short-slug>` or `feat|fix|docs|chore/<short-slug>`.
3. **One small PR per change.** A PR must do exactly one thing. If the diff needs "and", split it.
4. **Never push directly to `main`.** `main` is protected; changes arrive through PRs only.
5. **No merge without all checks.** Every CI job must be green on **Windows, macOS, and Linux**, and the PR must have review approval.
6. **No skeleton PRs.** A PR that only "prepares" without a working, tested increment is rejected.

### PR description requirements

Every PR description must contain, in this order:

1. What changed and why (2–5 sentences).
2. The **exact prompt** that produced the change (verbatim, in a collapsible block). This is mandatory.
3. Which phase/issue it advances, and which acceptance criteria it satisfies.
4. The tests added and what they prove.
5. What is **not** included and what remains untested.
6. Screenshots for any UI change.

---

## 2. Tests

1. **Every feature ships with tests in the same PR.** No follow-up "tests later" PRs.
2. Tests must cover **edge cases and failure paths**, not only the happy path:
   - empty, missing, and malformed input;
   - network failure, timeout, retry exhaustion;
   - server without range support, redirects, partial content;
   - cancellation mid-flight and process restart;
   - locale fallback and RTL;
   - disk-full and permission-denied behavior where testable.
3. **Bug fixes start with a failing test** that reproduces the bug, then the fix.
4. Tests must be deterministic: no sleeps for synchronization, no dependence on external network, no wall-clock in scheduling tests (use a fake clock).
5. A test that does not fail when the behavior is broken is not a test. Reviewers may challenge it.
6. Coverage is a signal, not a goal; untested branches in critical modules (segmentation, resume, merge, path handling) are release blockers.

---

## 3. Security

1. **No secrets in the repository.** No API keys, passwords, tokens, certificates, cookies, or signing material — not in code, tests, fixtures, examples, or commit history. Use GitHub Actions secrets and OS keychain.
2. **Sanitize everything.** Every value from the user, the network, a filename, a header, or a manifest is untrusted. Validate type, length, range, and encoding before use.
3. **No shell strings.** Execute sidecars with argument arrays. Never build a command line by concatenation.
4. **Paths are hostile.** Reject traversal, absolute paths outside allowed roots, symlink escapes, and reserved Windows names. Canonicalize before writing.
5. **URLs are hostile.** Allow `http`/`https` only; reject unexpected schemes, credentials in URLs, and control characters; bound redirect chains and response sizes.
6. **Logs are public.** Never log tokens, cookies, auth headers, or full URLs containing secrets.
7. **Dependencies need review.** New dependency = license + maintenance check + justification in the PR. Lockfiles are committed.
8. **Report vulnerabilities privately**, never in a public issue.

---

## 4. Documentation truthfulness

1. Docs describe **only** behavior proven by a test or a recorded manual verification.
2. Anything unproven is explicitly labeled **untested**.
3. No performance number, speed claim, or compatibility claim may be published without a repeatable measurement linked in the PR.
4. Decision changes require updating the relevant ADR — not silently editing prose.
5. Every doc states its status (`Draft`, `Proposed`, `Accepted`, `Superseded`).

---

## 5. Releases

1. Releases are produced **only** by GitHub Actions from a tagged commit on `main`.
2. No manual builds, no locally signed installers, no artifacts uploaded by hand.
3. A release requires: green CI on all platforms, all phase exit tests passing, updated changelog, and owner approval.
4. Versioning follows Semantic Versioning; pre-1.0 minor versions may break compatibility but must document it.
5. Signing keys never leave the CI secret store.

---

## 6. Code conventions (apply once code exists)

| Area | Rule |
|---|---|
| Rust | `cargo fmt` enforced; `clippy -D warnings`; no `unsafe` without an ADR and safety comment; errors via typed enums, no panics on user input |
| TypeScript | `strict: true`; no `any` without justification; no floating promises; explicit return types on exported functions |
| Naming | Domain language from `docs/ARCHITECTURE.md`; no synonyms (`download` never becomes `job`) |
| Files | One responsibility per module; no god files over ~400 lines without review |
| Commits | Conventional Commits (`feat:`, `fix:`, `docs:`, `test:`, `chore:`, `refactor:`); imperative, ≤ 72 chars |
| Comments | Explain *why*, never *what*; delete stale comments in the same PR |
| Dead code | Not allowed; no commented-out blocks, no unused flags |

---

## 7. Definition of Done

A change is done only when **all** of these are true:

- [ ] Scope matches the issue; nothing extra.
- [ ] Tests added for happy path, edge cases, and error paths.
- [ ] All CI jobs green on Windows, macOS, and Linux.
- [ ] No secrets, no debug prints, no TODOs left behind.
- [ ] Docs/ADRs updated where behavior or decisions changed.
- [ ] PR description includes the originating prompt.
- [ ] Reviewer approved.
- [ ] Untested items explicitly listed.

---

## 8. When to stop and ask

Stop **before** proceeding and ask the owner when:

- a decision changes architecture, licensing, distribution, or user-visible behavior;
- two requirements conflict;
- a test cannot be written for a claimed behavior;
- a dependency adds a license obligation;
- data migration could destroy user data;
- a security trade-off is involved;
- the roadmap order would change.

**Do not guess. Do not "improve" scope silently. Do not mark something tested when it is not.**

---

## 9. Language

- Code, comments, commit messages, issues, PRs, and docs are written in **English**.
- User-facing strings exist only in locale files; English is the source language and Arabic is required.
- Arabic may be used in discussion with the owner, but not in the codebase's canonical documents unless a bilingual ADR decides otherwise.

---

## 10. Prohibited actions

- Pushing to `main` directly, force-pushing shared branches, or rewriting published history.
- Committing secrets, binaries, or generated artifacts.
- Adding telemetry, analytics, or third-party tracking without an approved ADR.
- Bypassing DRM, paywalls, or authentication (product boundary, not a technical limitation).
- Disabling, skipping, or weakening a failing test to make CI green.
- Changing `docs/PRD.md` scope without an approved decision.
