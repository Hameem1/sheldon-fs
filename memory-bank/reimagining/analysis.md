# SheldonFS Reimagining — Analysis

Analysis of `sheldon-fs-team-alignment.md` (the new vision draft) against the project's
current state (`initial-idea.txt`, `CLAUDE.md`, existing code, decision records, and DB design).
Purpose: finesse the alignment doc before the team-readiness overhaul begins.

---

## 1. Overview — Current State vs. New Plan

The project began (`initial-idea.txt`) as a **personal utility**: a way for you and your wife to
untangle cluttered files spread across dual-boot Ubuntu/Windows PCs, a shared MacBook, and two
Google Drive accounts. The plan was to *first design an organization/naming protocol*, using a
scanner tool as the instrument for understanding the existing file landscape.

What actually got built (and is documented in `CLAUDE.md`) is a solid, well-tested **CLI-first
discovery engine**: a cross-platform TypeScript/Node scanner extracting 23 metadata fields per
file with SHA256 hashing (67 tests, ~75% coverage). Everything downstream — database, duplicate
detection, reporting, web UI, semantic embeddings, and finally an Electron desktop app — was laid
out as an **8-phase solo roadmap** where the desktop app is *last* (Phase 8) and semantic AI is a
*future, optional* enrichment (Phase 6). The current setup reflects a single developer:
two-repository structure (`sheldon-fs/` docs + `SheldonFS/` code), session-numbered commits, and
a memory-bank workflow.

The new alignment doc describes a **substantially different product and a different way of
working**. The headline shifts:

- **From CLI-first tool → AI-native desktop app.** Electron + a dual-pane chat/file-tree UX is now
  the *core* deliverable, not the final phase. The CLI becomes (at most) an internal/engine layer.
- **From "AI is a future optional add-on" → AI is the center of gravity.** Local LLM embeddings
  now drive the primary user flow (clustering, directory prediction, smart renaming, conversational
  refinement). This was Phase 6 "someday"; it's now table stakes.
- **From "never modify files" → actively execute moves/renames.** Phase 1's core safety principle
  was read-only. The new scope executes file mutations, with dry-run and undo relegated to
  "future iterations."
- **From personal tool → team product, internal-then-open-source.** 3–4 contributors working
  privately for a few months, then a public release, replacing the old Phase 3/Phase 7 split.
- **Scope shift on "cloud."** The original core pain (Google Drive scanning, knowing *which file
  lives on which system*) appears to have dropped out. "Cloud" now means only **syncing the SQLite
  database state** across the user's *own* desktop machines — a different feature entirely.
- **Solo workflow → team workflow.** The two-repo structure, session-commit convention, and
  memory-bank were built for one person and don't fit collaborative development.

Structurally, the good news: the scanner core and the (as-yet-unimplemented) database design are
directly reusable. The reimagining is primarily a **re-sequencing, re-scoping, and
team-enablement** exercise rather than a rewrite of what exists.

---

## 2. Recommendations

1. **Re-sequence the whole roadmap around the desktop app.** The 8-phase order is now inverted:
   Electron + UI (old Phase 8) and semantic AI (old Phase 6) move to the front, while OSS prep
   and release (Phases 3/7) split into "internal-hardening now" and "public-release later." The
   alignment doc should carry a fresh phase/milestone plan, not inherit the old one.

2. **Promote file-mutation safety to a first-class MVP requirement, not "future iterations."**
   The single riskiest thing this app does is move and rename real user files. Shipping execution
   *before* dry-run preview, a transactional/undo layer, and a manifest/journal of operations is
   how you lose a user's data on day one. Reverse the priority: safety scaffolding lands with (or
   before) the first execution feature.

3. **Resolve the "single-click install" vs. "requires a local LLM" tension explicitly.** A
   privacy-first local model (Ollama/nomic-embed-text) is a heavy dependency — install size,
   RAM/CPU, first-run model download. That collides with the "single-click install" goal. Decide:
   bundle the runtime, require a separate install, define a minimum-hardware baseline, and specify
   graceful degradation (does the app still do dedup + manual org with no model present?).

4. **Pull team-engineering foundations forward to day one.** CI/CD, branch/PR strategy, code
   review norms, issue tracking, coding standards, and a definition-of-done were "Phase 3." With
   3–4 people they're prerequisites, not a later phase. This is the actual content of the
   "readiness overhaul" you're about to start.

5. **Decide the repository structure now and retire the solo workflow.** Collapse the two-repo
   `sheldon-fs/` + `SheldonFS/` split into a single monorepo (likely with `apps/` +
   `packages/` for shared engine/desktop/CLI), and replace session-numbered commits + the
   memory-bank convention with conventional commits + PRs. This unblocks everyone else touching
   the code.

6. **Treat SQLite cloud-sync as a major risk deserving its own decision record.** Syncing a live
   SQLite file across machines invites conflicts and corruption; naive "sync the file" via
   Drive/Dropbox will bite you. Note that `DATABASE_DESIGN.md` explicitly *rejected* LibSQL partly
   because sync wasn't needed — that premise has now flipped. Options (LibSQL/Turso embedded
   replicas, CRDT/operation-log sync, or "export/import" rather than live sync) each have big
   tradeoffs and should be researched before committing.

7. **Make an explicit call on Google Drive / cloud-file scanning.** This was the *original* core
   problem ("which file is on which system," two Drive accounts) and it has quietly vanished from
   scope. Dropping it may be the right call for a focused MVP, but it should be a *stated,
   deliberate* non-goal in the doc, not an omission — especially since your wife's cross-device
   use case motivated the project.

8. **Commit to a UI framework and a reference architecture instead of "framework-agnostic."**
   "UI-framework-agnostic" is a red flag for a small team — it defers a decision everyone needs.
   Pick the stack (e.g., Electron + React/Vite + Tailwind, per the existing tech notes), and
   define the process/IPC architecture: where the scanner and DB run (main process), how the
   renderer talks to them, and where LLM inference lives. Ambiguity here multiplies across 4 devs.

9. **Settle open-source timing, license, and contributor/IP terms *before* the team writes code.**
   "Internal for a few months, then open source" has legal and structural consequences: license
   choice affects dependencies and any hosted-sync backend; contributor IP assignment / CLA should
   be agreed while it's just friends, not retrofitted. Also decide whether the repo starts private
   and flips, and whether any monetization (hosted sync?) is intended — that shapes the license.

10. **Scope a concrete first internal milestone and a dogfooding plan.** The vision is large;
    define what the first shippable-internally version *is* (e.g., scan → dedup review → manual
    reorg with dry-run + undo, LLM suggestions stubbed) so the team has a target. Reuse the
    existing scanner + DB design deliberately as the foundation, and plan to dogfood on your own
    (and your wife's) real, messy file systems — the original test case is still the best one.

---

## 3. Questions

1. **MVP scope:** What is the *first internal milestone*? The full vision (scan → embed → AI
   proposal → conversational refine → execute → sync) at once, or a deliberate slice? What must
   the first working version do?

2. **Google Drive / cloud files:** Is scanning cloud storage (the original core pain) in or out
   for the reimagined product? If out, is that permanent or deferred?

3. **"Cloud-sync" definition:** What do you actually envision syncing DB state *through* —
   the user's own cloud storage (Drive/Dropbox), a backend *you* host, or peer-to-peer? Who runs
   and pays for any server component?

4. **Team composition & skills:** Who are the 3–4 people and where do their strengths lie
   (frontend, backend/systems, ML/LLM, DevOps)? This should drive tech and ownership decisions.

5. **Local-LLM requirement vs. install simplicity:** How do you want to reconcile "single-click
   install" with needing a local model? Bundle it, require a separate Ollama install, define
   minimum hardware, and/or degrade gracefully when no model is present?

6. **UI framework:** Any preference or existing team familiarity (React / Vue / Svelte)? Are you
   comfortable locking this in now rather than staying "agnostic"?

7. **Open-source timing & license:** Roughly how many months internal before going public? Which
   license are you leaning toward, and will you accept outside contributions before the public
   release?

8. **Business model / monetization:** Is this purely a free/OSS tool, or is there an intended
   revenue angle (e.g., hosted sync, pro tier)? This materially affects license and architecture.

9. **Existing code & DB design:** Do you want to carry forward the current scanner core and
   `DATABASE_DESIGN.md` as the foundation (recommended), or treat the reimagining as a fresh
   start? Any parts you already know you want to discard?

10. **File-execution risk appetite & platform priority:** How aggressive should automated file
    moves/renames be (fully automated vs. always human-approved), and which OS do you want working
    first for internal dogfooding (given your Ubuntu/Windows/macOS mix)?
