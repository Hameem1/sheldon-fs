# Documentation Principles

How we write and maintain every doc in SheldonFS, from the Phase 0 planning docs onward.
This file is the single home for these rules; other docs and AI configs link here rather than
restating them.

## Why

Docs get re-read and edited constantly. When the same information lives in more than one place,
the copies drift apart, every change means hunting down the others, and you can never be sure an
edit hasn't broken the flow of something elsewhere. Saying each thing once, briefly, in one place
keeps the doc set cheap to read and safe to change.

## Rules

1. **One home per piece of information.** Every fact or decision lives in exactly one doc. To
   change something about the tech stack, you edit `tech-stack.md`, and nothing else.
2. **Link, don't restate.** Elsewhere, point to the home doc instead of summarizing it. Add only
   as much context as the reader needs to decide whether to follow the link.
3. **One concern per doc.** If a doc starts mixing concerns or growing long, raise it and
   consider splitting it. A split is cheap; untangling a doc later is not.
4. **Concise and to the point.** Write for someone re-reading it in six months. Cut background,
   repetition, and filler; prefer lists or tables where they're clearer than prose.
5. **Know which kind of doc it is.**
   - **Source of truth** (vision, tech stack, phases, design docs): describes the current state and is kept up to date.
   - **Snapshot** (analyses, research, decision records): point-in-time and marked as such. It explains *why*; it is never the current *what*.
   - **Work tracker** (checklists, tickets): may name what goes where, but once something is recorded in its home doc, the home doc wins.
6. **Change the home, then fix the links.** When a fact changes, edit its home doc. When a doc is
   renamed, split, or retired, update everything that links to it.

## Applying it

Raise it, whether in a review, a PR, or while writing, whenever:
- the same information appears in two places,
- a doc mixes concerns or is getting long,
- it's unclear which doc something belongs in.

AI assistants follow the same rules: `CLAUDE.md` and other AI configs point to docs instead of
copying them.

_Lives in `memory-bank/reimagining/` during Phase 0; moves into the project docs during the restructure._
