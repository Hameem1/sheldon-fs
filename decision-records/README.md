# Decision Records

This directory contains architectural and feature decision records for SheldonFS. These documents preserve the context, rationale, and research behind major design decisions.

## Purpose

Decision records serve as:
- **Historical context** - Why we made certain choices
- **Onboarding material** - Help new contributors understand the architecture
- **Future reference** - Revisit decisions when circumstances change
- **Research archive** - Preserve analysis and benchmarking data

## Structure

```
decision-records/
├── phase-1/          # Foundation: Scanner, Metadata, CLI
├── phase-2/          # Testing, Database, Duplicate Detection
├── phase-3/          # Web UI & API
├── phase-4/          # Semantic Intelligence & Desktop App
└── README.md         # This file
```

## When to Create a Decision Record

Create a record for:
- Major architectural decisions (database choice, API design)
- New features with significant complexity or tradeoffs
- Technology stack choices (libraries, frameworks)
- Performance-critical decisions
- Decisions that affect future extensibility

Don't create records for:
- Minor implementation details
- Obvious or trivial choices
- Standard coding practices

## Document Format

Each decision record should include:

### 1. Header
- **Status:** Proposed / Approved / Implemented / Superseded
- **Date:** When the decision was made
- **Phase:** Which development phase this applies to

### 2. Core Sections
- **Context & Problem Statement:** What challenge are we addressing?
- **Decision:** What did we decide to do?
- **Rationale:** Why this approach? What are the tradeoffs?
- **Consequences:** What are the positive and negative outcomes?

### 3. Supporting Sections (as needed)
- **Implementation Approach:** High-level technical design
- **Performance Estimates:** Benchmarks and scale considerations
- **Migration Path:** How to transition if this decision needs to change
- **Research References:** Links to articles, benchmarks, documentation
- **Timeline & Dependencies:** What needs to happen first?
- **Updates:** Track changes as implementation progresses

## Existing Records

### Phase 4
- [Semantic File Organization via Vector Embeddings](./phase-4/semantic-file-embeddings.md) - LLM-powered intelligent file organization using local embeddings

## Linking to Decision Records

In CLAUDE.md and other documentation:
- Provide brief summary of the feature/decision
- Link to full decision record for details
- Keep high-level docs concise, decision records comprehensive

Example:
```markdown
### Phase 4: Semantic Intelligence
Enable LLM-powered file understanding for intelligent organization.
See [full decision record](./decision-records/phase-4/semantic-file-embeddings.md) for details.
```

## Updating Records

Decision records are living documents during their phase:
- **Before implementation:** Mark as "Proposed" or "Approved"
- **During implementation:** Update with learnings, add to Updates section
- **After implementation:** Mark as "Implemented", document actual outcomes
- **If superseded:** Mark as "Superseded", link to replacement decision

Keep the Updates section current to track how decisions evolved during implementation.

---

**Note:** This pattern is inspired by Architecture Decision Records (ADRs) but adapted for feature-level decisions in SheldonFS.
