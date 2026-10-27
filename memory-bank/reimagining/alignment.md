# SheldonFS — Team Alignment

The source of truth for *what* we're building and *why*. Detail lives in dedicated docs,
linked from here.

## 1. Vision

**Elevator pitch:** SheldonFS is an AI-powered, cross-platform desktop app that scans,
deduplicates, and reorganizes local files. Using local open-source models, it understands file
content to propose better folder structures and file names, which the user reviews and approves.
It can also combine scans from all of a user's machines into one picture.

**The problem:**
- Files are scattered across machines and operating systems, and it's unclear what lives where.
- Duplicates pile up everywhere.
- Many files have unintuitive names.
- Without an organizing system, the mess keeps growing.

**Target audience:** anyone struggling with digital clutter, especially across several machines
or operating systems, who wants automated, context-aware organization without giving up privacy.

**Supported platforms:** Windows, macOS, and Linux (Ubuntu / Debian-based).

## 2. Principles

- **Privacy-first:** all AI runs locally, and SheldonFS never sends files, their contents, or
  their metadata to any service. The only data that leaves a machine is the database the user
  chooses to move between their own machines.
- **Human in control:** the app proposes; the user approves every file operation before it runs.
- **Useful without AI:** core features work right after install; AI features are opt-in.
- **One core, two interfaces:** the desktop app is the primary interface; the CLI is a
  maintained, first-class interface to the same core.
- **Free and open:** no monetization. Built by an internal team first, then open-sourced and
  developed with the community.

## 3. Scope

**In scope:**
- Simple, single-click installation.
- User-selected directories and fast metadata scanning.
- Multi-factor deduplication: content hashing combined with file name, size, and attribute matching.
- AI-assisted organization through an opt-in **deep scan**: local embeddings for text-rich
  formats (e.g. `.txt`, `.pdf`, `.docx`) power clustering, folder-structure prediction, and
  smart renaming.
- In-app AI model management: download local models (e.g. for embeddings, vision, chat) from
  inside the app, with recommendations based on the machine's hardware.
- Dual-pane UX: a visual file tree showing the proposed before/after structure, next to a
  conversational chat for refining it.
- Guarded execution of moves and renames: per-suggestion accept/reject, plus an operation
  journal that can revert the last batch.
- Cross-machine picture: one portable SQLite database, moved between the user's machines one at
  a time (e.g. through the user's own cloud storage), so scans from each OS build up a single
  combined picture. The exact mechanism is still an open question.

**Out of scope (non-goals):**
- Parsing or embedding non-contextual or system files (e.g. `.exe`, `.deb`, `node_modules`).

**Future (not in initial scope):**
- Cloud storage integrations (Google Drive and other providers), designed as extensible
  integrations. They may be built internally before the open-source release if time allows;
  otherwise they're left to the community.
- An advanced dry-run visualizer and version-control-style rollback.
- More automated (unattended) execution, once the tool has proven stable and reliable.

## 4. Primary User Flow

1. **Install & set up:** install like any desktop app. Optionally enable AI features by
   downloading the recommended local models from within the app.
2. **Select & scan:** choose directories. A normal scan collects metadata and hashes; a deep
   scan also generates embeddings for supported files. Results go into the SQLite database.
3. **Analyze:** find duplicates and, with deep-scan data, group files by content, predict a
   better folder structure, and suggest names for poorly named files.
4. **Review & refine:** inspect the proposal as a before/after file tree, accept or reject each
   suggestion, and refine it through the chat.
5. **Execute:** approved operations are applied; the last batch can be reverted if needed.
6. **Repeat across machines:** take the database to the next machine and scan there, building up
   one combined picture of all the user's files.

## 5. Delivery Phases

_Lives here until `phases.md` exists; then this section becomes a link._

- **Phase 0 — Foundation:** planning, docs, and ticketing; tracked in [`pre-requisites.md`](./pre-requisites.md).
- **Phase 1 — A little bit of everything:** a basic but working version of the whole product,
  including basic AI (embeddings-based organization) and the cross-machine picture working on
  all three OSes. Conversational chat is not included.
- **Phase 2 — Finished feel:** polish and the more advanced features, including conversational
  chat, plus everything needed to get ready for open source.
- **Phase 3 — Open source:** public release, promotion, and managing community contributions.

## 6. Tech Stack

_Lives here until `tech-stack.md` exists; then this section becomes a link._

- **Frontend:** React + TypeScript.
- **Desktop wrapper:** Electron.
- **Core:** TypeScript / Node.js.
- **Database:** SQLite, as a single portable file.
- **AI / ML:** local, open-source models (embeddings first; vision and chat later).
