# Research Directory

This directory contains research documents for exploring design decisions, trade-offs, and implementation approaches for SheldonFS.

## Purpose

Research documents serve as:
- **Exploration space** - Deep dive into pros/cons before making decisions
- **Trade-off analysis** - Compare multiple approaches with real-world examples
- **Knowledge preservation** - Keep valuable research even after decisions are made
- **Future reference** - Revisit alternatives if circumstances change

## Structure

```
research/
├── database/           # Database design research
├── [future-topics]/    # Other research areas as needed
└── README.md           # This file
```

## How Research Works

### 1. Research Phase
When facing a design decision with multiple viable approaches:
1. Create a research document in the appropriate subdirectory
2. Explore all options with detailed pros/cons
3. Include code examples, performance analysis, real-world impact
4. Provide preliminary recommendations (but **no final decisions**)

### 2. Decision Phase
After reviewing research and discussion:
1. Make final decision based on research findings
2. Document the decision in the appropriate design document (e.g., DATABASE_DESIGN.md)
3. Keep research document for historical context
4. Update research document status to "Decision Made - See [link]"

### 3. Long-term Value
Research documents remain valuable:
- Explain why we chose one approach over alternatives
- Help onboard new contributors ("Why didn't we use an ORM?")
- Allow revisiting decisions if context changes (Phase 4+)
- Preserve analysis for similar future decisions

## Research Document Format

Each research document should include:

### Header
```markdown
# Research: [Topic]

**Status:** Research | Decision Made
**Date:** YYYY-MM-DD
**Context:** Brief explanation of the decision being explored
**Decision:** [If decided] Link to final decision location
```

### Core Sections
- **Background** - Context and problem statement
- **Option A, B, C...** - Each approach with detailed analysis
  - Schema/implementation examples
  - Pros (with ✅)
  - Cons (with ❌)
  - Real-world impact
- **Comparison** - Side-by-side table or performance analysis
- **Preliminary Recommendation** - Initial suggestion with rationale
- **When to Reconsider** - Conditions that would change the decision

### Footer
```markdown
---

## Decision Pending

This research document presents findings only. Final decision will be documented in [DESIGN_DOC.md] after discussion and agreement.
```

## Guidelines

### When to Create Research Documents

**DO create research for:**
- Architectural decisions with multiple viable approaches
- Features with significant trade-offs
- Technology choices (libraries, patterns, algorithms)
- Performance-critical decisions
- Decisions that affect future extensibility

**DON'T create research for:**
- Obvious or trivial choices
- Standard coding practices
- Implementation details (those go in code comments)
- Decisions with only one reasonable option

### Updating Research After Decisions

When a decision is made:
1. Add "Decision Made" to the status
2. Link to the design document with the final decision
3. Keep the research document intact (don't delete analysis)
4. Optionally add a brief summary of why the decision was made

Example:
```markdown
# Research: Partial Hash Storage

**Status:** Decision Made (2025-01-20)
**Decision:** [DATABASE_DESIGN.md - Section X](#)
**Outcome:** Decided to compute on-demand for Phase 1, revisit in Phase 5

[Original research content remains below...]
```

---

**Note:** Research documents are living documents during active exploration. Once decisions are made, they become historical reference materials.
