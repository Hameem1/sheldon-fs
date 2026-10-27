# SheldonFS Reimagining — Phase 0: Foundation Checklist

**Phase 0** of the reimagining: everything we must lock before dev work begins. It takes us
from "mid-planning" to "the project is planned, the docs are clean, and tickets are ready for
the team to pick up." The actual build-out — code and code-related infra (repo restructure,
CI/CD, features) — is **Phase 1 onward**, defined inside `phases.md`. One deliberate
exception: the already-designed database layer is built in Phase 0 as a **pilot PR** that
dry-runs the whole team workflow before Phase 1 starts (#11).

**How to use:** tick items by flipping `[ ]` → `[x]`. Each numbered item is done when all of
its sub-items (1–3) are ticked.

**Scope rule:** the planning block (#1–#7) stays inside `memory-bank/reimagining/` and leaves
the original project files untouched while we cook the plan (aside from the interim
"reimagining in progress" banner already added to `CLAUDE.md`). Once the plan is locked and
reviewed (#7), the later items deliberately reach into the real project and tooling — rewriting
`CLAUDE.md` (#8), cleaning up the existing docs (#9), standing up tickets in Notion (#10), and
shipping the database pilot PR (#11). The sign-off (#12) is the green light to begin Phase 1.

---

## To-Do

1. **Copy the alignment doc into the reimagining dir** — move finessing onto a working copy so the original stays put.
   - [ ] Copy `sheldon-fs-team-alignment.md` → `memory-bank/reimagining/alignment.md`, leaving the root original untouched; all finessing happens on this copy from here on.

2. **Finalize the alignment doc** (`alignment.md`) — fold in every resolved clarification so it reads as the single source of truth for *what & why*.
   - [ ] **Scope & interfaces:** CLI as a permanently-maintained first-class interface; guarded execution (before/after preview, per-suggestion accept/reject, a basic operation journal for "revert last batch"); Google Drive & other storage providers reframed as *future extensibility*, not initial scope.
   - [ ] **Product & tech decisions:** React; completely free / no monetization; in-app model download UX (recommend-by-hardware, baseline vs. opt-in "deep scan").
   - [ ] **MVP intent:** basic AI in Phase 1, conversational chat in Phase 2, cross-OS DB picture working on all three OSes in Phase 1.

3. **Create & seed the decisions log** (`decisions.md`) — a *living* ledger so resolved calls and open threads aren't buried in chat; updated continuously as #4–#6 progress.
   - [ ] **Locked decisions:** the resolved Q&A (first-class CLI, guarded exec + journal, React, free, in-app models, basic-AI-in-P1, chat-in-P2, cross-OS-in-P1, carry forward scanner core + `DATABASE_DESIGN.md`).
   - [ ] **Open spikes / research:** cloud-sync mechanism + SQLite-WAL safety, local-model runtime (bundle vs. sidecar vs. in-process), license options comparison.
   - [ ] **Deferred / out-of-scope:** Google Drive & cloud providers (→ future/community), advanced dry-run & VCS-style rollback, automated unattended execution.

4. **Write the repository & collaboration strategy doc** (`repo-and-collaboration.md`) — my full analysis of your proposed structure + the team workflow.
   - [ ] **Monorepo layout + lightweight tooling:** package/app boundaries (`core`, `cli`, `desktop`, `docs`), and the tooling middle-ground (pnpm/npm workspaces, Turborepo yes/no).
   - [ ] **Internal vs. public repo strategy:** the split, gitignored AI configs, the sync-down approach, and what migrates into the public repo when.
   - [ ] **Team conventions + migration:** branching / PR / review / commit norms and ticket tracking; plus how the current two-repo code (scanner core, DB design, docs) folds into the new structure.

5. **Write `tech-stack.md`** — the *with-what*, tightly coupled to #4 (can be done together).
   - [ ] **Confirm the stack:** Electron + React + Vite + Tailwind + TanStack, better-sqlite3 + Drizzle, vitest, winston — plus the AI additions (model-runtime lib, sqlite-vec).
   - [ ] **Resolve the 6 architecture keystones:** core/IPC boundary; local-model runtime; native-module (better-sqlite3) builds for Electron; long-running jobs; typed data layer; monorepo tooling.
   - [ ] **Mark decidable-now vs. needs-a-Spike**, and link each Spike back into the decisions log (#3).

6. **Write `phases.md`** — the big one; depends on #2–#5 being settled.
   - [ ] **Define the 3 phases + MVP scope:** the "a little bit of everything" thin slice, with explicit in/out per feature area.
   - [ ] **Slot the pinned decisions into phases:** license decision → P1, cloud-sync Spike → P1, repo restructure + CI/CD → P1 kickoff, OSS-readiness → P2, public release + community → P3. (Ticket creation itself is Phase 0 — see #10.)
   - [ ] **Define MVP "definition of done" + dogfooding plan:** your (and your wife's) real, messy file systems, verified across all three OSes.

7. **Consistency review pass** — make sure the finished doc set hangs together before anything real is touched.
   - [ ] Read the full reimagining doc set together; reconcile any contradictions and confirm it's complete and internally consistent before anything in the real project is touched.

8. **Rewrite `CLAUDE.md`** to reflect the new direction (touches an original file; done once the plan above is locked).
   - [ ] **Supersede the old 8-phase solo roadmap**; align overview / status / phases with the finalized reimagining docs, and remove the interim "reimagining in progress" banner.
   - [ ] **Replace solo conventions** (two-repo structure, session-numbered commits, memory-bank workflow) with the team + monorepo workflow.
   - [ ] **Settle its place in the internal/public split:** keep `CLAUDE.md` the AI source-of-truth and reconcile it with the gitignored AI configs.

9. **Documentation hygiene pass** — audit the existing project docs so the foundation is clean (the reimagining set is already vetted in #7; focus here on the legacy docs).
   - [ ] Inventory every `.md` in the project (root, `research/`, `decision-records/`, `SheldonFS/`, READMEs); flag what's outdated, duplicated, or bloated against the locked new direction.
   - [ ] Update or mark-as-superseded any doc describing the old plan/phases; consolidate overlaps and trim bloat.
   - [ ] Confirm the surviving set is accurate, non-redundant, and consistent with the reimagining docs + `CLAUDE.md`.

10. **Set up ticketing + create the backlog** — get the project ready for devs to pick up work; ticketing lives in Notion.
    - [ ] Stand up the ticketing system in Notion (board/structure, statuses, labels, a ticket template).
    - [ ] Derive the backlog from `phases.md` (Phase 1 scope first): epics → tickets with clear acceptance criteria.
    - [ ] Confirm tickets are assignable and the backlog is "ready for dev."

11. **Database layer pilot PR** — implement the already-designed DB layer as the first real ticket, and use it to dry-run the whole team workflow before Phase 1.
    - [ ] Refresh `DATABASE_DESIGN.md` for the new direction before coding (e.g. machine identity / `sourceSystem` namespaces for the cross-OS DB, migrations from day one since DB files now travel between machines, schema room for the operation journal and embeddings, sync vs. async access per `tech-stack.md`), then file it as a Notion ticket.
    - [ ] Implement it tightly scoped to the design: schema, connection management, repositories, `FileMetadata` ↔ record conversions, integration tests. No dedup, sync, or AI features.
    - [ ] Run it end-to-end through the workflow (ticket → branch → PR → teammate review → merge), and fold any friction back into `repo-and-collaboration.md` and the Notion setup.

12. **Foundation sign-off** — the explicit handoff from Phase 0 to Phase 1.
    - [ ] Confirm all Phase 0 items are complete and give the green light to begin Phase 1 (code + code-related infra: repo restructure, CI/CD, then feature work).
