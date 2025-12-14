# Research: Partial Hash Storage in Database

**Status:** Decision Made
**Date:** 2025-01-16
**Decision:** Compute on-demand for Phase 1 (see DATABASE_DESIGN.md)
**Context:** Should we store the 100KB partial hashes in the database for quick pre-filtering before full hash comparison?

## Final Decision

**Chosen: Option B - Compute On-Demand**

**Rationale:**
- Phase 1 focuses on getting the foundation right - keep schema simple
- Current scan performance is acceptable (~30s for 1,300 files)
- No proven performance bottleneck yet - avoid premature optimization
- Easy to add `partial_hash` column later if duplicate detection becomes slow
- In development phase with single user, can rebuild database without migrations if needed

**When to revisit:**
- Phase 5 (Optimization) when handling 100K+ files
- If duplicate detection becomes slow (>10-30 seconds)
- If web UI (Phase 4) needs instant duplicate queries
- If profiling shows partial hash calculation is a bottleneck

**Migration path:**
When needed, simply add column and index - straightforward schema evolution.

---

## Background

SheldonFS already calculates partial hashes (first 100KB of files) for the `quickCompare()` function during duplicate detection. The question is whether to **persist** these partial hashes in the database or compute them on-demand.

**Current Implementation:**
- `calculatePartialHash()` - Calculates SHA256 of first 100KB
- `quickCompare()` - Compares size + partial hash before expensive full hash comparison
- Used during scanning but **not stored in database**

---

## Option A: Store Partial Hashes in Database

### Schema Change

```sql
ALTER TABLE files ADD COLUMN partial_hash TEXT;  -- Hash of first 100KB (SHA256 hex, 64 chars)
CREATE INDEX idx_files_partial_hash ON files(partial_hash);
CREATE INDEX idx_files_size_partial ON files(size, partial_hash);  -- Composite for duplicate detection
```

### Pros

✅ **Faster duplicate detection** - Filter by size + partial_hash before expensive full hash comparison

✅ **Pre-filtering efficiency** - Eliminates ~99% of non-duplicates without full hash comparison
- Files with same size but different partial hashes cannot be duplicates
- Only compare full hashes for files with matching size AND partial hash

✅ **Useful for large files** - Movies (4GB+), ISOs (700MB+), large datasets benefit most
- Full hash of 4GB file takes ~10-20 seconds
- Partial hash (100KB) takes <1ms
- Pre-filtering saves significant time

✅ **Query optimization** - Can find "probable duplicates" quickly
```sql
-- Find files that might be duplicates (same size + partial hash)
SELECT size, partial_hash, COUNT(*) as duplicates
FROM files
WHERE scan_id = ?
GROUP BY size, partial_hash
HAVING duplicates > 1;
```

✅ **Already computing it** - We calculate partial hashes during `quickCompare()`, just need to store
- No additional computational cost during scanning
- Storage is the only new cost

### Cons

❌ **Storage overhead** - Additional 64 bytes per file (SHA256 hex string)
- 10,000 files = ~640KB additional storage
- 100,000 files = ~6.4MB additional storage
- **Impact:** Negligible for modern systems

❌ **Computational cost** - Must calculate partial hash for ALL files during scan
- Currently only calculated during `quickCompare()` on-demand
- Would need to calculate upfront for every file
- **Impact:** Adds ~1-2ms per file (minimal)

❌ **Complexity** - More columns to manage, additional index
- Schema has 24 fields instead of 23
- One more index to maintain
- **Impact:** Slight increase in maintenance burden

❌ **Marginal benefit for small files** - Files < 100KB have same partial and full hash
- No benefit for small files (text documents, config files, etc.)
- Real-world data: ~60-70% of files are < 100KB
- **Impact:** Wasted storage for majority of files

❌ **Index size** - Partial hash index grows with file count
- 100,000 files = ~6.4MB index size (plus B-tree overhead)
- **Impact:** Minimal, but worth considering

### Real-World Impact Analysis

**For typical SheldonFS use case (1,300 files, 37GB):**
- Many large files (videos, photos, archives)
- Partial hash filtering would be highly effective
- Storage cost: ~83KB (negligible)
- Performance gain: Potentially 50-80% faster duplicate detection for large files

**For large-scale use case (100,000 files, 1TB):**
- Storage cost: ~6.4MB (negligible)
- Performance gain: Significant for duplicate detection
- Index lookups: Still very fast with B-tree

### Implementation Example

```typescript
// During scan
const metadata = await extractMetadata(filePath)
metadata.hash = await calculateFileHash(filePath)
metadata.partialHash = await calculatePartialHash(filePath)  // NEW

// Store in database
fileRepo.insert({
    ...metadata,
    partial_hash: metadata.partialHash  // NEW
})

// Duplicate detection (optimized with partial hash)
async findDuplicates(scanId: number): Promise<DuplicateGroup[]> {
    // Step 1: Group by size + partial_hash (fast, indexed)
    const candidates = db.prepare(`
        SELECT size, partial_hash, COUNT(*) as count
        FROM files
        WHERE scan_id = ? AND size > 102400  -- Only files > 100KB
        GROUP BY size, partial_hash
        HAVING count > 1
    `).all(scanId)

    // Step 2: Only compare full hashes for probable duplicates
    for (const candidate of candidates) {
        const files = db.prepare(`
            SELECT * FROM files
            WHERE scan_id = ? AND size = ? AND partial_hash = ?
        `).all(scanId, candidate.size, candidate.partial_hash)

        // Group by full hash (now only comparing small subset)
        const duplicates = groupByHash(files)
        // ...
    }
}
```

---

## Option B: Compute Partial Hash On-Demand

### Approach

Calculate partial hash during duplicate detection phase, don't store in database.

### Pros

✅ **Simpler schema** - Fewer columns, fewer indexes
- 23 fields instead of 24
- Easier to understand and maintain

✅ **Less storage** - No additional space per file
- Important for very large file counts (millions of files)

✅ **Flexibility** - Can change hashing strategy without schema migration
- Switch to first + last 50KB instead of first 100KB
- Use different hash algorithm (e.g., xxHash for speed)
- No database changes needed

✅ **No wasted storage for small files** - Only calculate partial hash when needed
- Don't store partial hash for files < 100KB (same as full hash)

✅ **Deferred computation** - Only pay cost when actually detecting duplicates
- If user never runs duplicate detection, never calculate partial hashes

### Cons

❌ **Slower duplicate detection** - Must recalculate partial hashes every time
- Reading first 100KB of file from disk (I/O cost)
- Calculating SHA256 (CPU cost)
- **Impact:** ~1-2ms per file × number of candidates

❌ **Redundant computation** - Calculating same partial hash multiple times
- Scan 1: Calculate partial hashes for duplicate detection
- Scan 2: Calculate partial hashes again for duplicate detection
- **Impact:** Wasted CPU/disk cycles

❌ **Cannot pre-filter via SQL** - Must load all files, then filter in-memory
- Cannot use database indexes for optimization
- More data transferred from database to application
- **Impact:** Slower queries, more memory usage

### Implementation Example

```typescript
async findDuplicates(scanId: number): Promise<DuplicateGroup[]> {
    // Step 1: Group by size (files with different sizes can't be duplicates)
    const sizeGroups = db.prepare(`
        SELECT size, COUNT(*) as count
        FROM files
        WHERE scan_id = ?
        GROUP BY size
        HAVING count > 1
    `).all(scanId)

    const duplicates: DuplicateGroup[] = []

    // Step 2: For each size group with multiple files
    for (const group of sizeGroups) {
        const files = db.prepare(`
            SELECT * FROM files WHERE scan_id = ? AND size = ?
        `).all(scanId, group.size)

        // Calculate partial hashes on-demand (in-memory, no storage)
        const partialHashes = new Map<string, string>()  // path → partial_hash
        for (const file of files) {
            if (file.size > 102400) {  // Only for files > 100KB
                partialHashes.set(file.path, await calculatePartialHash(file.path))
            }
        }

        // Group by partial hash
        const partialHashGroups = new Map<string, FileRecord[]>()
        for (const file of files) {
            const partialHash = partialHashes.get(file.path) || file.hash
            if (!partialHashGroups.has(partialHash)) {
                partialHashGroups.set(partialHash, [])
            }
            partialHashGroups.get(partialHash)!.push(file)
        }

        // Step 3: Only compare full hashes for files with matching partial hashes
        for (const [partialHash, groupFiles] of partialHashGroups) {
            if (groupFiles.length > 1) {
                // These are probable duplicates, group by full hash
                const fullHashGroups = groupBy(groupFiles, f => f.hash)
                for (const [hash, dupFiles] of fullHashGroups) {
                    if (dupFiles.length > 1) {
                        duplicates.push({ hash, files: dupFiles })
                    }
                }
            }
        }
    }

    return duplicates
}
```

---

## Performance Comparison

### Scenario: 10,000 files, 100 duplicate groups

**Option A (Store Partial Hash):**
1. Load candidates from DB: `SELECT size, partial_hash WHERE count > 1` → ~1ms
2. Load probable duplicates: 100 groups × ~2-5 files each → ~5ms
3. Group by full hash (in-memory): ~1ms
4. **Total: ~7ms**

**Option B (Compute On-Demand):**
1. Load all files by size: 100 size groups → ~10ms
2. Calculate partial hashes: 100 groups × 4 files × 2ms = ~800ms (I/O + CPU)
3. Group by partial hash (in-memory): ~2ms
4. Group by full hash (in-memory): ~1ms
5. **Total: ~813ms**

**Speedup: ~116x faster with stored partial hashes**

### Scenario: 100,000 files, 1,000 duplicate groups (typical large scan)

**Option A:** ~70ms
**Option B:** ~8-12 seconds

**Speedup: ~100-170x faster with stored partial hashes**

---

## Hybrid Approach: Conditional Partial Hash Storage

Store partial hash **only for files larger than 100KB**:

```sql
-- Computed column approach (store NULL for small files)
partial_hash TEXT,  -- NULL if size <= 102400, otherwise SHA256 of first 100KB

-- Or use a separate table (normalized)
CREATE TABLE file_partial_hashes (
    file_id INTEGER PRIMARY KEY REFERENCES files(id) ON DELETE CASCADE,
    partial_hash TEXT NOT NULL
);
-- Only insert rows for large files
```

**Pros:**
- Saves storage for small files (~60-70% of files)
- Still gets performance benefit for large files (where it matters)

**Cons:**
- More complexity (conditional logic)
- Harder to query (must handle NULL)

---

## Preliminary Recommendation: **Option B (Compute On-Demand) for Phase 1**

### Rationale

1. **Keep it simple** - Phase 1 focuses on getting the foundation right
   - Fewer columns, simpler schema
   - Easier to understand and maintain

2. **Current scan performance is acceptable** - Test case: 1,300 files in ~30s
   - Duplicate detection is not a bottleneck yet
   - No proven performance problem to solve

3. **Premature optimization** - We haven't benchmarked duplicate detection yet
   - Don't optimize without data
   - May not be a real-world issue

4. **Easy to add later** - If duplicate detection becomes slow, add `partial_hash` column in Phase 2-3
   - Schema migration is straightforward: `ALTER TABLE files ADD COLUMN partial_hash TEXT`
   - One-time backfill: Calculate and populate for existing files
   - Future scans: Store partial hash from then on

5. **Memory-based filtering works** - During duplicate detection, we can calculate partial hashes in-memory
   - Only for files that are size-matched (small subset)
   - Acceptable performance for Phase 1 scale

6. **Storage is cheap, but simplicity is valuable** - For open-source project, simple schema helps onboarding

### When to Reconsider (Add Partial Hash Storage)

**Add `partial_hash` column when:**
- ✅ Duplicate detection on 100K+ files becomes slow (>10-30 seconds)
- ✅ Implementing real-time duplicate checking in web UI (Phase 4)
- ✅ Users request faster duplicate reports
- ✅ Profiling shows partial hash calculation is bottleneck
- ✅ Phase 5 (Optimization) - Natural time to add performance improvements

**Implementation path:**
1. Add column: `ALTER TABLE files ADD COLUMN partial_hash TEXT`
2. Create index: `CREATE INDEX idx_files_partial_hash ON files(partial_hash)`
3. Backfill existing files: One-time script to populate
4. Update `FileScanner` to calculate and store during scans

---

## Alternative: Keep Partial Hash in Memory Only (Middle Ground)

During a scan session, cache partial hashes in-memory for quick comparison:

```typescript
class PartialHashCache {
    private cache = new Map<string, string>()  // path → partial_hash

    async get(path: string, size: number): Promise<string> {
        if (this.cache.has(path)) {
            return this.cache.get(path)!
        }

        const partialHash = await calculatePartialHash(path)
        this.cache.set(path, partialHash)
        return partialHash
    }

    clear() {
        this.cache.clear()
    }
}

// Use during duplicate detection within a single session
const cache = new PartialHashCache()
// ... use cache.get() instead of recalculating
cache.clear()  // After duplicate detection completes
```

**Pros:**
- Fast within a single session
- No database changes
- No persistent storage cost

**Cons:**
- Cache lost between sessions
- Memory overhead during execution
- Doesn't help with historical data

---

## Implementation Notes

For Phase 1, duplicate detection will:
1. Group files by size (different sizes can't be duplicates)
2. Calculate partial hashes on-demand in-memory for size-matched files
3. Group by partial hash to identify probable duplicates
4. Compare full hashes only for files with matching partial hashes

This provides the performance benefit without schema complexity.
