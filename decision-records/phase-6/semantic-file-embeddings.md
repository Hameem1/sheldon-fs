# Decision Record: Semantic File Organization via Vector Embeddings

**Status:** Approved for Phase 6 implementation
**Date:** 2025-11-08
**Phase:** 6 (Future Feature)

---

## Context & Problem Statement

As SheldonFS grows beyond basic metadata and duplicate detection, users need intelligent, content-aware file organization that goes beyond filename and folder structure. Current limitations:

1. **Naming inconsistencies** - Same content with different filenames can't be grouped
2. **Content-blind organization** - No understanding of what files actually contain
3. **Manual categorization** - Users must manually organize files by topic/project
4. **Limited duplicate detection** - Hash-based only, misses semantic duplicates (e.g., same report in PDF and DOCX)

**Goal:** Enable semantic understanding of file content to provide intelligent organization suggestions, content-based grouping, and natural language queries.

---

## Decision

Implement a **separate, optional embedding enrichment pipeline** that:

1. Generates vector embeddings for file content using locally-run LLMs
2. Stores embeddings in SQLite using sqlite-vec extension
3. Enables semantic similarity search and intelligent auto-categorization
4. Runs as a post-scan enrichment step (not during primary scan)

**Technology Stack:**
- **Embedding Model:** Ollama + nomic-embed-text (768 dimensions, fast, local)
- **Vector Storage:** sqlite-vec extension for SQLite
- **Database:** Continue with SQLite (not PostgreSQL)

---

## Rationale

### Why Separate Pipeline?

**Keeps primary scan fast:**
- Current scan: ~30s for 1,300 files
- With embeddings: Could be 5-10x slower
- Most users won't need embeddings immediately

**Allows incremental processing:**
- Process 100 files at a time
- Resume if interrupted
- Re-run enrichment without re-scanning

**Optional feature:**
- Not all files benefit from embeddings (system files, binaries)
- Users can choose which directories to enrich
- Reduces resource usage for users who don't need it

### Why SQLite (Not PostgreSQL)?

**Deployment simplicity:**
- SQLite: Zero installation, single file, ship with app
- PostgreSQL: Requires separate server installation (dealbreaker for desktop app)

**Performance is sufficient:**
- sqlite-vec handles 100K-1M vectors efficiently (<100ms queries)
- Our expected scale: 1,300-100,000 files
- Perfect fit for the use case

**Modern vector support:**
- sqlite-vec (2024): Fast, SIMD-optimized, MIT licensed
- Supports cosine similarity, binary quantization
- Can handle all our needs

**PostgreSQL only needed for:**
- Multi-user concurrent access (100+ simultaneous users)
- Datasets >1M files
- Complex distributed analytics
- Not relevant for local-first desktop tool

### Why Ollama + nomic-embed-text?

**Local-first:**
- No API keys or cloud dependencies
- Privacy-preserving (files never leave user's machine)
- Works offline

**Performance:**
- nomic-embed-text: 768 dimensions (vs 4096 for llama3)
- Smaller embeddings = faster search, less storage
- ~100-500 files/second embedding generation

**Proven technology:**
- Widely used in RAG systems
- Well-documented, stable
- Active community support

---

## Capabilities Enabled

### 1. Intelligent Content Grouping
Group files by semantic similarity regardless of naming:
```
"Meeting_notes_2024.docx"
"2024_meeting_notes.txt"
"notes-meeting-jan2024.md"
→ Grouped as "Meeting Notes cluster"
```

### 2. Content-Based Duplicate Detection
Find semantic duplicates with different formats:
```
"quarterly_report_v1.pdf"
"Q1_2024_final.docx"
→ Embedding similarity: 0.98 → Likely same content!
```

### 3. Smart Auto-Organization
Suggest organization for new files:
```
New file: "IMG_20240815_beach_vacation.jpg"
→ Finds existing "Travel Photos/2024/Summer" cluster
→ Suggests: "Move to Travel Photos/2024/Summer?"
```

### 4. Natural Language Queries (Future)
```
"Show me all files related to tax documents from last year"
→ Finds: receipts, bank statements, tax forms, W2s, etc.
```

### 5. Topic Clustering
Automatically discover file clusters by content:
```
→ "Project X Documentation" (15 files)
→ "2024 Tax Documents" (23 files)
→ "Family Photos - Summer 2024" (184 files)
```

---

## Implementation Approach

### Phase 6 Architecture

```
┌─────────────────────┐
│  File Scanner       │ → Fast metadata extraction (23 fields)
│  (Phase 1)          │ → Store in SQLite
└─────────────────────┘
          ↓
┌─────────────────────┐
│  Enrichment Pipeline│ → Extract text from files
│  (Phase 6)          │ → Generate embeddings (Ollama)
│                     │ → Store in sqlite-vec
└─────────────────────┘
          ↓
┌─────────────────────┐
│  Semantic Analysis  │ → Similarity search
│                     │ → Smart grouping
│                     │ → Auto-categorization
└─────────────────────┘
```

### CLI Interface

```bash
# Phase 1: Fast metadata scan
sheldonfs scan ~/Documents

# Phase 6: Optional semantic enrichment
sheldonfs enrich --scan-id=<id> --enable-embeddings

# Semantic queries
sheldonfs similar --file=document.pdf --limit=10
sheldonfs cluster --directory=~/Documents
```

### Text Extraction Strategy

**Priority 1 (Easy):**
- Plain text files (.txt, .md, .csv, .json)
- Code files (.js, .py, .ts, .java, etc.)

**Priority 2 (Medium):**
- PDFs (use `pdf-parse` or `pdfjs-dist`)
- Office documents (.docx via `mammoth`, .xlsx)

**Priority 3 (Future):**
- Images (OCR via Tesseract.js)
- Videos/Audio (transcription via Whisper)

**Skip:**
- System files (.DS_Store, thumbs.db)
- Binaries without extractable text
- Files <100 bytes

### Performance Estimates

**Embedding Generation:**
- Speed: ~100-500 files/second (CPU-dependent)
- 1,300 files: ~3-13 seconds
- 100,000 files: ~3-16 minutes

**Storage:**
- 768-dimensional embeddings: ~3KB per file
- 100,000 files: ~300MB of embeddings
- SQLite handles this easily

**Query Performance:**
- <100ms for tens of thousands of vectors
- ~100-200ms for hundreds of thousands
- Acceptable for interactive use

---

## Consequences

### Positive

✅ **Game-changing feature** - Enables truly intelligent file organization
✅ **Privacy-preserving** - All processing happens locally
✅ **Optional** - Doesn't slow down basic usage
✅ **Scalable** - Works for 1K-1M files
✅ **Future-proof** - Can add more LLM capabilities later
✅ **Simple deployment** - SQLite requires no server setup

### Negative

⚠️ **Requires Ollama** - Users must install Ollama separately
⚠️ **CPU/GPU intensive** - Embedding generation uses significant resources
⚠️ **Storage overhead** - ~3KB per file for embeddings
⚠️ **Complexity** - Text extraction, model management adds complexity
⚠️ **Not universal** - Some file types can't be meaningfully embedded

### Risks & Mitigations

**Risk:** Users don't install Ollama
**Mitigation:** Make embeddings optional, provide clear setup instructions, fallback to metadata-only features

**Risk:** Poor quality embeddings for some file types
**Mitigation:** Smart filtering - only embed files that benefit (text, code, PDFs)

**Risk:** Performance issues on older hardware
**Mitigation:** Incremental processing, progress indicators, ability to pause/resume

**Risk:** SQLite performance degrades at scale
**Mitigation:** Monitor performance, can add PostgreSQL backend later if needed (abstraction layer ready)

---

## Migration Path (If PostgreSQL Needed Later)

**Abstraction Layer:**
```typescript
interface VectorStore {
  insert(fileId: string, embedding: number[]): Promise<void>
  search(embedding: number[], limit: number): Promise<SimilarFile[]>
  cluster(threshold: number): Promise<Cluster[]>
}

// Current implementation
class SQLiteVectorStore implements VectorStore { ... }

// Future if needed
class PgVectorStore implements VectorStore { ... }
```

**When to migrate:**
- Dataset >1M files
- Multi-user concurrent access (100+ users)
- Need HNSW indexes for faster approximate search
- Distributed system requirements

**Estimated effort:** 2-3 weeks (with proper abstraction)

---

## Research References

**SQLite Vector Extensions:**
- [sqlite-vec v0.1.0 Release](https://alexgarcia.xyz/blog/2024/sqlite-vec-stable-release/index.html) - 2024 stable release, SIMD-optimized
- [Using SQLite as LLM Vector Database](https://turso.tech/blog/using-sqlite-as-your-llm-vector-database) - Performance benchmarks
- [sqlite-vec GitHub](https://github.com/asg017/sqlite-vec) - Official repository

**Local LLM Embeddings:**
- [Ollama nomic-embed-text](https://ollama.com/library/nomic-embed-text) - Official model page
- [RAG Pipeline with Ollama](https://medium.com/@rahul.dusad/run-rag-pipeline-locally-with-ollama-embedding-model-nomic-embed-text-generate-model-llama3-e7a554a541b3) - Implementation guide

**Performance Benchmarks:**
- sqlite-vec: <100ms queries for 100K vectors
- 1M vectors: ~192ms-8s depending on dimensions
- nomic-embed-text: 768 dimensions, ~100-500 files/sec

---

## Timeline & Dependencies

**Technical Dependencies (from earlier phases):**
- Metadata scanner (Phase 1) - Provides file data to enrich with embeddings
- SQLite database (Phase 1) - Storage backend for sqlite-vec extension
- Tests infrastructure (Phase 2) - Ensures stability before adding complexity
- Duplicate detection (Phase 2) - Related feature for semantic comparison

**Phase 6 Implementation:**
1. Research & prototyping (1 week)
2. sqlite-vec integration (1 week)
3. Text extraction pipeline (2 weeks)
4. Ollama integration (1 week)
5. Semantic search features (2 weeks)
6. Testing & optimization (2 weeks)

**Estimated total:** 9 weeks

---

## Notes

- Keep design flexible with abstraction layers for potential database migration
- Monitor sqlite-vec development for new features (ANN indexes coming)
- Consider user feedback from earlier phases before finalizing Phase 6 design
- May need to adjust based on actual file type distribution in user data

---

## Updates

_This section will track any updates or changes to this decision as implementation progresses._

---

**Last Updated:** 2025-11-09
**Next Review:** After Phase 5 completion
