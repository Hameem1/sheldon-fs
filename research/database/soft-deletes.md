# Research: Soft Deletes vs Hard Deletes with Scan History

**Status:** Decision Made
**Date:** 2025-01-16
**Decision:** Hard deletes with configurable scan retention (see DATABASE_DESIGN.md)
**Context:** Should we mark files as deleted instead of removing them (soft deletes), or rely on scan history (hard deletes)?

## Final Decision

**Chosen: Hard Deletes (Option B) + Scan Retention Policy (separate feature)**

**Rationale:**
- Soft deletes add significant complexity (every query needs `WHERE deleted_at IS NULL`)
- Each scan is a clean snapshot - simpler mental model and implementation
- Better query performance (no filtering deleted records, smaller indexes)
- Scan retention policy is a separate concern from file deletion handling
- Configurable retention provides flexibility without schema complexity

**Scan Retention Policy (separate feature):**
- Default: Keep last 5 scans
- User-configurable via global config (can set to any number)
- Provides bounded storage without soft delete complexity
- Automatic cleanup of old scans

**Trade-offs accepted:**
- No file-level history tracking (acceptable for Phase 1 use case)
- Cannot undelete individual files (can restore from scan snapshots)
- Limited to recent scan comparisons (sufficient for active development)

---

## Background

When a new scan runs and a file is no longer present, we have three options:
1. **Soft delete** - Mark file as `deleted_at = timestamp`, keep record forever
2. **Hard delete** - Remove file record, rely on previous scan for history
3. **Hybrid** - Hard delete old scans, keep N recent scans for comparison

---

## Option A: Soft Deletes (Mark as Deleted)

### Schema Changes

```sql
ALTER TABLE files ADD COLUMN deleted_at INTEGER;  -- NULL = active, timestamp = deleted
CREATE INDEX idx_files_deleted ON files(deleted_at);

-- Queries always filter deleted files
SELECT * FROM files WHERE deleted_at IS NULL AND scan_id = ?;

-- Find deleted files
SELECT * FROM files WHERE deleted_at IS NOT NULL;
```

### Pros

✅ **File history tracking** - See when files were deleted, full lifecycle
```sql
-- "When did report.pdf get deleted?"
SELECT path, created_at, deleted_at
FROM files
WHERE path = '/Documents/report.pdf'
ORDER BY created_at DESC;
```

✅ **Undelete capability** - Can restore deleted records if needed
```sql
-- "Restore file that was accidentally deleted"
UPDATE files SET deleted_at = NULL WHERE id = ?;
```

✅ **Forensic value** - Answer questions like "When did this file disappear?"
- Audit trail for compliance
- Investigate data loss incidents
- Track file lifecycle

✅ **Detailed history** - See complete timeline of file events
```sql
-- "Show all versions of this file"
SELECT created_at, modified_at, deleted_at, size, hash
FROM files
WHERE path = '/Documents/contract.pdf'
ORDER BY created_at;
```

✅ **No data loss** - Never permanently delete file metadata
- Safe for important file tracking
- Can always go back in time

### Cons

❌ **Database bloat** - Files accumulate forever, never actually deleted
- Scan 1: 10,000 files
- Scan 2: 9,500 files (500 deleted, but still in DB)
- Scan 3: 9,000 files (1,000 deleted total, still in DB)
- **Result:** Database has 11,000 records, only 9,000 active

❌ **Query complexity** - Every query needs `WHERE deleted_at IS NULL`
```typescript
// Every query must filter
findByScanId(scanId: number) {
    return db.query(`
        SELECT * FROM files
        WHERE scan_id = ? AND deleted_at IS NULL  -- Easy to forget!
    `).all(scanId)
}

// Statistics become complex
countByCategory(scanId: number) {
    return db.query(`
        SELECT category, COUNT(*)
        FROM files
        WHERE scan_id = ? AND deleted_at IS NULL  -- Must remember every time
        GROUP BY category
    `).all(scanId)
}
```

❌ **Confusion** - Mixing active and deleted records complicates logic
- Developers must remember to filter in every query
- Easy to introduce bugs (forgetting `deleted_at IS NULL`)
- Code becomes more verbose and error-prone

❌ **Storage waste** - For a local CLI tool, deleted files aren't valuable long-term
- 100,000 files scanned over time
- 50,000 deleted
- **Result:** 150,000 records, but only 100,000 useful
- **Impact:** 50% wasted storage

❌ **Index bloat** - Indexes grow with deleted records, slowing queries
- `idx_files_hash` includes deleted files (irrelevant for duplicates)
- `idx_files_category` includes deleted files (irrelevant for reports)
- **Impact:** Larger indexes = slower queries, more memory

❌ **Performance degradation** - Queries get slower as deleted records accumulate
```sql
-- Find duplicates (should only consider active files)
SELECT hash, COUNT(*) FROM files
WHERE scan_id = ? AND deleted_at IS NULL  -- Scans past deleted records
GROUP BY hash;

-- With 50% deleted records, query scans 2x more data than necessary
```

❌ **Cleanup complexity** - When to actually delete old deleted records?
- Never? Database grows forever
- After 1 year? Need scheduled cleanup jobs
- **Added complexity** with no clear benefit for Phase 1

### Real-World Impact

**Scenario: 10 scans over 6 months**

| Scan | Active Files | Deleted (Cumulative) | Total in DB | Active % |
|------|--------------|----------------------|-------------|----------|
| 1    | 10,000       | 0                    | 10,000      | 100%     |
| 2    | 9,500        | 500                  | 10,500      | 90%      |
| 3    | 9,000        | 1,000                | 11,000      | 82%      |
| 5    | 8,000        | 2,000                | 12,000      | 67%      |
| 10   | 7,000        | 3,000                | 13,000      | 54%      |

**After 10 scans:**
- Database: 13,000 records
- Active: 7,000 files (54%)
- Deleted: 6,000 files (46% wasted)

**After 50 scans (2 years):**
- Database: Potentially 100,000+ records
- Active: ~10,000 files
- Deleted: ~90,000 files (90% wasted!)

---

## Option B: Hard Deletes with Scan History

### Approach

Delete files from old scans when new scan completes, rely on `scan_sessions` for history.

**Model:** Each scan is a complete snapshot in time.

```typescript
// After scan completes
async completeScan(scanId: number) {
    // Update scan session
    await scanSessionRepo.update(scanId, { completed_at: Date.now() })

    // Old scans remain intact (each is a snapshot)
    // User can compare scans to see what changed
}
```

### Pros

✅ **Clean schema** - Only current scan data in database
- No `deleted_at` column
- No filtering logic in queries
- Simple and straightforward

✅ **Simple queries** - No need to filter deleted records
```typescript
findByScanId(scanId: number) {
    return db.query(`SELECT * FROM files WHERE scan_id = ?`).all(scanId)
    // That's it! No deleted_at filtering
}
```

✅ **Better performance** - Smaller tables, faster queries, smaller indexes
- Only active data in indexes
- No wasted storage
- Optimal query performance

✅ **Easy to understand** - One scan = one snapshot in time
- Clear mental model
- No confusion about deleted vs active
- Simpler for contributors to understand

✅ **Natural lifecycle** - Old scans deleted entirely with CASCADE
```sql
DELETE FROM scan_sessions WHERE id = 1;  -- Removes all files for that scan
```

✅ **Scan history provides comparison** - Can see changes between scans
```bash
$ sheldon-scan --compare 1 2
- 500 files deleted
+ 300 files added
```

### Cons

❌ **No file-level history** - Can't track individual file over time
- Cannot answer: "When did report.pdf get deleted?"
- Must compare scans manually to find when file disappeared

❌ **Scan-to-scan comparison required** - Must compare two full scans to see changes
- Not instant lookup like soft deletes
- Requires computing diff (though fast with indexes)

❌ **Lost detail** - Once old scan deleted, file history gone
- If keeping 5 scans, can only see last 5 snapshots
- Cannot see file that existed in scan 1 but deleted before scan 6

❌ **No undelete** - Can't restore accidentally deleted files
- If file deleted from filesystem, cannot bring back metadata
- **Mitigation:** User should restore from backup if needed

### Storage Impact

**Scenario: 10 scans, keeping all**
- Each scan: ~10,000 files
- Total: ~100,000 records
- **BUT:** Each scan is independent, so 10× storage
- If files mostly unchanged: ~90% redundant data

**Problem:** Keeping all scans leads to storage explosion too!

---

## Option C: Hybrid (Keep N Recent Scans)

### Approach

Hard deletes, but keep last N scans (e.g., 5 most recent). Delete old scans automatically.

```typescript
async cleanupOldScans(keepCount: number = 5): Promise<number> {
    const allScans = await this.findAll({ orderBy: 'started_at DESC' })

    if (allScans.length <= keepCount) {
        return 0  // Nothing to delete
    }

    const toDelete = allScans.slice(keepCount)  // Keep first N, delete rest
    let deleted = 0

    for (const scan of toDelete) {
        this.delete(scan.id)  // CASCADE removes files, errors, stats
        deleted++
    }

    return deleted
}

// Call after each scan
await scanSession.complete()
await scanSessionRepo.cleanupOldScans(5)  // Keep 5 most recent
```

### Pros

✅ **Bounded storage** - Database size stays manageable
- Max 5 scans × 10,000 files = 50,000 records
- Predictable, controlled growth

✅ **Recent history available** - Can compare last 5 scans
- "What changed in the last month?"
- "Show recent duplicate trends"

✅ **Simple schema** - No `deleted_at` complexity
- Standard hard deletes
- Clean queries

✅ **Predictable performance** - Max 5 scans worth of data
- Queries stay fast
- Indexes stay small

✅ **Configurable** - User can set how many scans to keep
```bash
$ sheldon-scan /path --keep-scans 10   # Keep last 10
$ sheldon-scan /path --keep-all        # Never delete
```

✅ **Automatic maintenance** - No manual cleanup needed
- Old scans pruned automatically
- User doesn't think about it

### Cons

❌ **Still no file-level tracking** - Can't see full file lifecycle
- Limited to last N scans
- Cannot track file over months/years

❌ **Arbitrary limit** - Why 5? Why not 10?
- Requires choosing a default
- Trade-off between history and storage

❌ **Data loss for old scans** - Historical scans are deleted
- Cannot compare to scans older than N
- **Mitigation:** Make it configurable, allow `--keep-all`

### Storage Impact

**Scenario: Keep last 5 scans**
- Each scan: ~10,000 files
- Total: ~50,000 records max
- **Bounded, predictable, manageable**

**Scenario: Keep last 10 scans**
- Total: ~100,000 records max
- Still reasonable

---

## Comparison Table

| Feature                          | Soft Deletes | Hard Deletes (All Scans) | Hybrid (N Scans) |
|----------------------------------|--------------|--------------------------|------------------|
| **Storage growth**               | Unbounded    | Unbounded (N× scans)     | Bounded          |
| **Query complexity**             | High         | Low                      | Low              |
| **File-level history**           | ✅ Full      | ❌ No                    | ⚠️ Recent only   |
| **Performance**                  | ❌ Degrades  | ✅ Good                  | ✅ Excellent     |
| **Scan comparison**              | ✅ Built-in  | ✅ On-demand             | ✅ On-demand     |
| **Undelete capability**          | ✅ Yes       | ❌ No                    | ❌ No            |
| **Schema simplicity**            | ❌ Complex   | ✅ Simple                | ✅ Simple        |
| **Suitable for CLI**             | ❌ Overkill  | ⚠️ Storage grows         | ✅ Perfect       |
| **Suitable for Web UI (Phase 4)**| ✅ Maybe     | ⚠️ Depends               | ✅ Yes           |

---

## Preliminary Recommendation: **Option C (Hybrid - Keep N Recent Scans) for Phase 1**

### Rationale

1. **Best of both worlds** - Recent history for comparison, but bounded storage
   - Can see "what changed recently" (most common use case)
   - Don't care about scans from 6+ months ago for local file management

2. **Use case alignment** - Users care about "what changed recently", not long-term history
   - "Did I delete that file recently?"
   - "What files were added this week?"
   - NOT: "Show me files from 2 years ago"

3. **Simple implementation** - No schema changes, just delete old scans
   - Standard hard deletes with CASCADE
   - No `deleted_at` column
   - Clean queries

4. **Scan history is enough** - Each scan is a complete snapshot
   - Comparing scans gives you file history
   - File-level tracking not needed for Phase 1

5. **Configurable** - User can set how many scans to keep
   - Default: 5 scans (good for most users)
   - Power users: `--keep-scans 20` or `--keep-all`
   - Flexibility without complexity

6. **Predictable performance** - Database size stays bounded
   - 5 scans × 10,000 files = 50,000 records max
   - Fast queries, small indexes
   - No degradation over time

### Implementation

```typescript
// src/core/database/repositories/ScanSessionRepository.ts

class ScanSessionRepository {
    async cleanupOldScans(keepCount: number): Promise<number> {
        const allScans = await this.findAll({
            orderBy: 'started_at DESC'
        })

        if (allScans.length <= keepCount) {
            console.log(`Only ${allScans.length} scans, nothing to clean up`)
            return 0
        }

        const toDelete = allScans.slice(keepCount)
        console.log(`Deleting ${toDelete.length} old scans (keeping ${keepCount} most recent)`)

        let deleted = 0
        for (const scan of toDelete) {
            this.delete(scan.id)  // CASCADE removes files, errors, stats
            deleted++
        }

        return deleted
    }
}

// Usage in FileScanner
async completeScan(scanId: number, options: { keepScans?: number }) {
    await this.scanSessionRepo.complete(scanId)

    // Cleanup old scans (default: keep 5)
    const keepCount = options.keepScans ?? 5
    if (keepCount > 0) {  // 0 = keep all
        await this.scanSessionRepo.cleanupOldScans(keepCount)
    }
}
```

### CLI Options

```bash
# Default: keep last 5 scans
$ sheldon-scan /path

# Keep last 10 scans
$ sheldon-scan /path --keep-scans 10

# Keep all scans (no cleanup)
$ sheldon-scan /path --keep-all

# Manual cleanup (for advanced users)
$ sheldon-scan --cleanup-scans --keep 5
```

### User Messaging

```bash
$ sheldon-scan /path
Scanning /Users/hameem/Documents...
✓ Scanned 10,253 files (37.2 GB) in 28s
✓ Found 42 duplicate groups (wasting 1.2 GB)

Keeping 5 most recent scans, deleted 3 old scans
(Use --keep-scans N to change, or --keep-all to disable cleanup)
```

---

## When to Reconsider

### Add Soft Deletes When:

**Phase 4 (Web UI)** - If users want full file lifecycle tracking
- "Show timeline of this file over 2 years"
- "When was this file last seen?"
- File audit trail for compliance

**Phase 5 (Optimization)** - If implementing incremental scans
- File-level change tracking becomes more valuable
- Need to know "was this file deleted or just moved?"

**User requests long-term tracking**
- "Show files deleted in last year"
- "Compliance: track all file changes"

### Current Approach (Phase 1):

- ✅ Keep last **5 scans** by default
- ✅ Make it configurable via CLI flag (`--keep-scans N`)
- ✅ Allow `--keep-all` for indefinite history
- ✅ Document in help: "Older scans are automatically removed to save space"
- ✅ Hard deletes with CASCADE (simple, clean)

---

## Alternative: Export Old Scans

Instead of deleting, export old scans to JSON for archival:

```bash
# Before deleting scan 1
$ sheldon-scan --export-scan 1 > scan-1-archive.json

# Or automatic archival
$ sheldon-scan /path --archive-old-scans
Archived 3 old scans to ~/.sheldonfs/archives/
```

**Pros:**
- ✅ No data loss
- ✅ Database stays clean
- ✅ Can import archived scans later if needed

**Cons:**
- ❌ Added complexity
- ❌ Archived scans not queryable

**Verdict:** Nice-to-have for Phase 5, not Phase 1.

---

## Implementation Notes

**Hard Deletes:**
- No `deleted_at` column in files table
- Each scan is a complete snapshot of files at that moment
- Simple queries without filtering logic

**Scan Retention Policy:**
See DATABASE_DESIGN.md for:
- Global configuration approach
- Automatic cleanup implementation
- User configuration options
- Storage management strategy

**Separation of Concerns:**
- File deletion handling = Hard deletes (schema decision)
- Scan retention = Configuration feature (operational decision)
- These are independent and should be implemented separately
