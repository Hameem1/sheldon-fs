# Research: Dedicated Scan Diff Table

**Status:** Decision Made
**Date:** 2025-01-16
**Decision:** Compute on-demand (see DATABASE_DESIGN.md)
**Context:** Should we have a dedicated table for storing scan-to-scan changes, or compute differences on-demand?

## Final Decision

**Chosen: Option B - Compute On-Demand**

**Rationale:**
- Complexity far outweighs minor performance benefits
- With proper indexes, diff queries are fast (sub-second for 10K files, 1-2s for 100K files)
- Scan comparison is not a primary use case for Phase 1-2 CLI
- Simpler schema with fewer tables is easier to understand and maintain
- No data redundancy - files table is the source of truth
- Easy to add dedicated table in Phase 4 if web UI proves it's needed

**When to revisit:**
- Phase 4 (Web UI) if dashboard needs instant diff display across many scan pairs
- If performance profiling shows diff queries are too slow (>5-10 seconds)
- If implementing real-time change notifications

**Performance is acceptable:**
- 10K files: ~50-200ms for all diff queries
- 100K files: ~500ms-2s for all diff queries
- Acceptable for CLI use case

---

## Background

Users may want to compare scans to see what changed:
- New files added
- Files deleted
- Files modified (same path, different hash)
- Files moved (same hash, different path)

**Question:** Should we pre-calculate and store these diffs, or compute them on-demand?

---

## Option A: Dedicated `scan_diffs` Table

### Schema

```sql
CREATE TABLE scan_diffs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,

    -- References
    from_scan_id INTEGER NOT NULL REFERENCES scan_sessions(id) ON DELETE CASCADE,
    to_scan_id INTEGER NOT NULL REFERENCES scan_sessions(id) ON DELETE CASCADE,

    -- Summary statistics
    files_added INTEGER NOT NULL,
    files_removed INTEGER NOT NULL,
    files_modified INTEGER NOT NULL,
    files_moved INTEGER NOT NULL,
    files_unchanged INTEGER NOT NULL,

    -- Size changes
    size_added INTEGER NOT NULL,       -- Total bytes of new files
    size_removed INTEGER NOT NULL,     -- Total bytes of deleted files
    size_changed INTEGER NOT NULL,     -- Net change in total size

    -- Detailed changes (JSON array)
    changes TEXT,  -- [{ type: 'added', path: '...', size: ..., hash: ... }, ...]

    -- Metadata
    created_at INTEGER NOT NULL DEFAULT (unixepoch('now', 'subsec') * 1000)
);

CREATE INDEX idx_scan_diffs_from ON scan_diffs(from_scan_id);
CREATE INDEX idx_scan_diffs_to ON scan_diffs(to_scan_id);
CREATE UNIQUE INDEX idx_scan_diffs_pair ON scan_diffs(from_scan_id, to_scan_id);
```

### Pros

✅ **Instant access** - Pre-computed diffs load immediately from database
- No computation needed when user requests diff
- Fast dashboard loading in web UI (Phase 4)

✅ **Historical tracking** - See how filesystem evolved over time
- "How many files were added/removed per month?"
- Trend analysis: "Is storage growing or shrinking?"
- Timeline view: "What changed between scans #1-10?"

✅ **Useful for web UI** - Dashboard can show trends without expensive queries
```typescript
// Quick stats widget
const recentDiffs = await db.query(`
    SELECT from_scan_id, to_scan_id, files_added, files_removed
    FROM scan_diffs
    ORDER BY created_at DESC
    LIMIT 10
`)
```

✅ **Change notifications** - Can implement "what changed since last scan" feature easily
```bash
$ sheldon-scan /path --show-changes
Since last scan:
+ 23 files added (45MB)
- 5 files removed (12MB)
~ 12 files modified (8MB → 10MB)
```

✅ **Complex queries become simple**
```sql
-- Average number of files added per scan (without diffs: complex multi-join)
SELECT AVG(files_added) FROM scan_diffs;

-- Find scans with large changes (without diffs: expensive computation)
SELECT * FROM scan_diffs
WHERE files_added + files_removed > 1000;
```

### Cons

❌ **Storage explosion** - Each scan pair = new record
- N scans → potentially N² diff records (if comparing all pairs)
- More realistically: N-1 records (each scan compared to previous)
- **Example:** 10 scans = 9 diff records (manageable)
- **Example:** 100 scans = 99 diff records (still ok)

❌ **Staleness** - If old scan deleted, diffs become invalid/orphaned
- CASCADE delete helps, but creates broken chains
- **Example:** Scans 1→2→3→4, delete scan 2, now missing 1→2 and 2→3 diffs
- Chain breaks, can't reconstruct timeline

❌ **Complexity** - When to generate diffs?
- After every scan automatically? (adds latency to scan completion)
- On-demand when user requests? (then why store it?)
- For all pairs, or just consecutive scans?
- What about comparing non-consecutive scans (1 vs 5)?

❌ **Maintenance burden** - CASCADE deletes, cleaning up old diffs
- If keeping last 5 scans, must also clean up old diffs
- Foreign key constraints require careful deletion order

❌ **Questionable value for CLI** - CLI users run ad-hoc comparisons
- "Compare scan 1 to scan 3" - not predictable ahead of time
- Pre-computing all possible pairs is wasteful
- Better to compute on-demand when requested

❌ **Data redundancy** - Information already exists in `files` table
- Storing derived data that can be computed from source
- Risk of inconsistency if not maintained properly

### Storage Estimate

**Scenario: 10 scans, comparing each to previous**
- 9 diff records
- Each record: ~200 bytes (summary) + JSON (depends on change count)
- **Total:** ~2-10KB (negligible)

**Scenario: 100 scans, comparing each to previous**
- 99 diff records
- **Total:** ~20-100KB (still small)

**Storage is NOT the problem - complexity is.**

---

## Option B: Compute On-Demand

### Approach

Run diff queries when user requests comparison, using optimized SQL with indexes.

### Pros

✅ **No storage overhead** - Diffs computed from existing `files` table
- Source of truth remains `files` table
- No redundant data

✅ **Always accurate** - Real-time calculation, never stale
- No risk of diffs being out of sync with actual data
- Always reflects current database state

✅ **Simpler schema** - Fewer tables to manage
- 5 core tables instead of 6
- Easier to understand and maintain

✅ **Flexible comparisons** - Compare any two scans, not just consecutive
- "Compare scan 1 to scan 5"
- "Compare current scan to scan from 3 months ago"
- Not limited to pre-computed pairs

✅ **No CASCADE complexity** - Deleting scan doesn't break diff chains
- Each comparison is independent
- No orphaned records

✅ **Better for CLI use case** - Users request specific comparisons ad-hoc
```bash
$ sheldon-scan --compare 1 5
$ sheldon-scan --compare-to-last
```

### Cons

❌ **Slower for large scans** - Must compute diff each time
- **Mitigation:** With proper indexes, should be sub-second even for 100K files
- Only slow if comparing 100K+ file scans frequently

❌ **No historical diff trends** - Can't easily answer "how often do files change?"
- Would need to compare all consecutive scan pairs on-demand
- **Mitigation:** Can add analytics queries for specific questions when needed

❌ **More CPU for repeated requests** - If user compares same scans multiple times
- **Mitigation:** Rare in practice (why compare same scans twice?)

### Performance Analysis

**Query: Find new files (in scan2, not in scan1)**

```sql
-- Using NOT EXISTS (optimized with indexes)
SELECT f2.*
FROM files f2
WHERE f2.scan_id = 2
AND NOT EXISTS (
    SELECT 1 FROM files f1
    WHERE f1.scan_id = 1
    AND f1.path = f2.path
);

-- With idx_files_scan_id and idx_files_path: Fast (index seek)
```

**Query: Find deleted files (in scan1, not in scan2)**

```sql
SELECT f1.*
FROM files f1
WHERE f1.scan_id = 1
AND NOT EXISTS (
    SELECT 1 FROM files f2
    WHERE f2.scan_id = 2
    AND f2.path = f1.path
);
```

**Query: Find modified files (same path, different hash)**

```sql
SELECT
    f1.path,
    f1.hash as old_hash,
    f2.hash as new_hash,
    f1.size as old_size,
    f2.size as new_size,
    f2.size - f1.size as size_change
FROM files f1
JOIN files f2 ON f1.path = f2.path
WHERE f1.scan_id = 1
  AND f2.scan_id = 2
  AND f1.hash != f2.hash;

-- With proper indexes: Fast (index seek + join)
```

**Query: Find moved files (same hash, different path)**

```sql
SELECT
    f1.path as old_path,
    f2.path as new_path,
    f1.hash,
    f1.size
FROM files f1
JOIN files f2 ON f1.hash = f2.hash
WHERE f1.scan_id = 1
  AND f2.scan_id = 2
  AND f1.path != f2.path;

-- With idx_files_hash: Fast (hash-based join)
```

**Expected Performance:**
- 10K files per scan: ~50-200ms for all 4 queries
- 100K files per scan: ~500ms - 2s for all 4 queries
- **Acceptable for CLI use case**

### Implementation Example

```typescript
interface ScanDiff {
    from_scan: number
    to_scan: number
    summary: {
        files_added: number
        files_removed: number
        files_modified: number
        files_moved: number
        size_added: number
        size_removed: number
        size_changed: number
    }
    details: {
        added: FileRecord[]
        removed: FileRecord[]
        modified: Array<{ old: FileRecord; new: FileRecord }>
        moved: Array<{ old: FileRecord; new: FileRecord }>
    }
}

class ScanComparisonService {
    compareScans(scan1Id: number, scan2Id: number): ScanDiff {
        // Run optimized SQL queries (parallel if possible)
        const added = this.findAddedFiles(scan1Id, scan2Id)
        const removed = this.findRemovedFiles(scan1Id, scan2Id)
        const modified = this.findModifiedFiles(scan1Id, scan2Id)
        const moved = this.findMovedFiles(scan1Id, scan2Id)

        return {
            from_scan: scan1Id,
            to_scan: scan2Id,
            summary: {
                files_added: added.length,
                files_removed: removed.length,
                files_modified: modified.length,
                files_moved: moved.length,
                size_added: sumSize(added),
                size_removed: sumSize(removed),
                size_changed: sumSize(added) - sumSize(removed)
            },
            details: { added, removed, modified, moved }
        }
    }

    // Compare to last scan (convenience method)
    compareToLast(currentScanId: number): ScanDiff | null {
        const previousScan = this.scanRepo.findPreviousScan(currentScanId)
        if (!previousScan) return null
        return this.compareScans(previousScan.id, currentScanId)
    }
}
```

### CLI Usage

```bash
# Compare current scan to previous
$ sheldon-scan /path --compare-to-last

# Compare specific scans
$ sheldon-scan --compare 1 5

# Output
Comparing scan #5 (2025-01-16 14:30) to scan #1 (2025-01-10 09:00)

Files:
  + 23 added      (45.2 MB)
  - 5 removed     (12.8 MB)
  ~ 12 modified   (8.1 MB → 10.3 MB)
  ↔ 8 moved       (same content, new location)

Net change: +32.4 MB (+0.8%)

Top changes:
  + /Downloads/video.mp4 (1.2 GB)
  - /Trash/old-project/ (500 MB)
  ~ /Documents/report.pdf (2 MB → 2.1 MB)
```

---

## Option C: Hybrid (Cache Recent Diffs)

Combine both approaches: pre-compute diffs for recent scans, compute on-demand for older comparisons.

### Approach

- Automatically generate diff for new scan vs. previous scan
- Store last N diffs (e.g., 5 most recent)
- For older comparisons, compute on-demand

### Pros

✅ **Fast for common case** - "What changed?" is instant
✅ **Flexible for historical** - Can still compare any two scans

### Cons

❌ **Added complexity** - Two code paths (cached vs on-demand)
❌ **Cache invalidation** - When to regenerate? When to delete?
❌ **Marginal benefit** - On-demand is already fast enough

---

## Preliminary Recommendation: **Option B (Compute On-Demand) for Phase 1-2**

### Rationale

1. **Sufficient performance** - With proper indexes, diff queries are fast
   - Sub-second for 10K files
   - 1-2 seconds for 100K files
   - Acceptable for CLI use case

2. **Simpler is better** - Fewer tables = easier to understand and maintain
   - Open-source contributors can grasp schema quickly
   - Less code to test and maintain

3. **Rare operation** - Scan comparison is not the primary use case
   - Primary: Duplicate detection
   - Secondary: Reporting, statistics
   - Tertiary: Scan comparison

4. **CLI-friendly** - User runs `sheldon-scan --compare 1 2`, we compute and display diff
   - Users don't expect instant results for CLI commands
   - 1-2 second computation is acceptable

5. **Future flexibility** - If web UI (Phase 4) needs historical trends, add caching layer then
   - Can add `scan_diffs` table in Phase 4 if web UI proves it's needed
   - Phase 1-2 CLI doesn't justify the complexity

6. **No data redundancy** - Don't store derived data that can be easily computed
   - Files table is source of truth
   - Diffs are ephemeral views

### When to Reconsider (Add `scan_diffs` Table)

**Add table when:**
- ✅ **Phase 4 (Web UI)** - If dashboard needs instant diff display across many scan pairs
- ✅ **Slow queries** - If performance profiling shows diff queries are too slow (>5-10 seconds)
- ✅ **Historical analytics** - If users need "change trends over time" reports
- ✅ **Notifications** - If implementing real-time change notifications

**Current recommendation for Phase 1-2:**
- Compute on-demand
- Optimize queries with proper indexes
- Document query patterns in code
- Revisit in Phase 4 if web UI needs justify it

---

## Implementation Notes

### Required Indexes for Fast On-Demand Diffs

```sql
-- Already in DATABASE_DESIGN.md
CREATE INDEX idx_files_scan_id ON files(scan_id);      -- For filtering by scan
CREATE INDEX idx_files_path ON files(path);            -- For path-based joins
CREATE INDEX idx_files_hash ON files(hash);            -- For content-based joins (moved files)

-- Composite index for common diff queries
CREATE INDEX idx_files_scan_path ON files(scan_id, path);  -- Optimize NOT EXISTS queries
CREATE INDEX idx_files_scan_hash ON files(scan_id, hash);  -- Optimize moved file detection
```

### Optimization: Use Temporary Tables for Complex Diffs

For very large scans (100K+ files), use temporary tables to stage results:

```typescript
compareScans(scan1Id: number, scan2Id: number): ScanDiff {
    // Create temp tables for intermediate results
    this.db.exec(`
        CREATE TEMP TABLE IF NOT EXISTS scan1_files AS
        SELECT * FROM files WHERE scan_id = ${scan1Id};

        CREATE TEMP TABLE IF NOT EXISTS scan2_files AS
        SELECT * FROM files WHERE scan_id = ${scan2Id};
    `)

    // Run optimized queries on temp tables
    const added = this.db.query(`
        SELECT * FROM scan2_files
        WHERE path NOT IN (SELECT path FROM scan1_files)
    `).all()

    // ... other queries

    // Cleanup
    this.db.exec('DROP TABLE scan1_files; DROP TABLE scan2_files;')

    return { ... }
}
```

---

## Implementation Notes

See DATABASE_DESIGN.md for:
- Required indexes for fast on-demand diffs
- ScanComparisonService implementation
- Query optimization techniques
- CLI usage examples
