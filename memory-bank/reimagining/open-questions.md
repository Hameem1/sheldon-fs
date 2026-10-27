# Open Questions

Everything still unresolved. No decisions live here: once a question is settled, its answer goes
into the home doc named in the entry, and the entry is deleted.

---

## Cross-machine database: how does it move between machines?

**Answer goes to:** `tech-stack.md` (mechanism) and `DATABASE_DESIGN.md` (schema).
**Decide by:** a Spike early in Phase 1.

Given (`alignment.md` → Scope): one portable SQLite database, moved between the user's machines
one at a time. If each machine writes only its own rows, combining scans is purely additive, with
no merge conflicts.

Open:
- **Transport.** Pointing the app at a live database inside a cloud-synced folder is the simplest
  option, but the riskiest. In WAL mode, SQLite keeps `-wal` / `-shm` companion files, and sync
  clients (Drive, Dropbox, iCloud) can upload a half-written database or get it out of step with
  those files, which corrupts it. The safer option is an explicit push/pull: checkpoint, write a
  consistent snapshot (`VACUUM INTO`), and copy that; the other machine opens it only once the
  download is complete.
- **Concurrent use.** Two machines using the database at once will overwrite each other's work.
  A lock marker, or a warning when the incoming snapshot came from another machine, is probably
  enough for a small set of trusted machines.
- **Machine identity.** Rows need to be keyed per machine, not per OS: `sourceSystem` alone can't
  tell two Macs apart.

## Local model runtime: how does the app run local models?

**Answer goes to:** `tech-stack.md`.
**Decide by:** a Spike early in Phase 1.

Constraints (`alignment.md`): simple install, models downloaded in-app with hardware-based
recommendations, AI is opt-in, all three OSes. Embeddings come first; vision and chat later.

Options:
- **Bundled Ollama sidecar:** the app ships and manages an Ollama binary. Mature model catalogue,
  but adds install size and a background process.
- **User-installed Ollama:** smallest app, but breaks the simple-install goal.
- **In-process inference:** e.g. `node-llama-cpp` (GGUF models) or Transformers.js / ONNX Runtime
  (embeddings). No separate process, but native modules need per-platform builds.

Also open: which embedding model to start with (the pre-reimagining decision record
`decision-records/phase-6/semantic-file-embeddings.md` chose `nomic-embed-text`), how to detect
hardware for recommendations, and whether one runtime can serve embeddings, vision, and chat.

## License: which open-source license?

**Answer goes to:** `alignment.md` (Principles), and a `LICENSE` file at the public release.
**Decide by:** Phase 1; applied at the Phase 3 release.

Deciding early matters because dependencies constrain it: linking a GPL library, for example,
would force the app itself to be GPL.

Shortlist:
- **MIT:** simplest and most permissive; maximizes adoption and contributions.
- **Apache-2.0:** permissive, plus an explicit patent grant.
- **GPL-3.0:** forks and redistributions must stay open source.
