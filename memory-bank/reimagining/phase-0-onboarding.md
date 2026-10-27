We're in Phase 0 of reimagining SheldonFS: turning it from a solo CLI project into a
team-built, AI-powered desktop app that will later be open-sourced. Phase 0 is the planning
and foundation work, tracked in a checklist. Get caught up before doing anything else:

1. Read the following files in `memory-bank/reimagining/`:
   - `pre-requisites.md`: the Phase 0 checklist. Ticked items are done; the first item with unticked sub-items is where we are.
   - `documentation-principles.md`: the rules for all doc work. Follow them.
   - `analysis.md`: the initial snapshot of the reimagining. Its questions were answered later; the checklist and newer docs hold those answers and win wherever they disagree.
2. Read `CLAUDE.md` (its top banner says which sections are outdated) and `initial-idea.txt`
   (the project's origin). Consult the legacy docs (`DATABASE_DESIGN.md`,
   `TESTING_STRATEGY.md`, `research/`, `decision-records/`) and the code in `SheldonFS/` only
   when a task touches them, and they are relevant. Otherwise, this just adds bloat to the context.
3. Check recent progress with `git log --oneline -10` and `git status`, in both the repo root
   and `SheldonFS/`.

Then reply with a very concise summary and nothing else:
- **What we're doing:** 2–3 sentences.
- **Where we are:** what's done, and the next unticked item.
- **Flags:** contradictions, gaps, or open questions you noticed (omit if none).

Don't start any work. Wait for me to say what to do next.
