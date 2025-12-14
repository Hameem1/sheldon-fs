# Database Design for SheldonFS

## Overview

This document outlines the database architecture for SheldonFS Phase 1, focusing on efficient storage and retrieval of file metadata, scan history, and duplicate relationships.

**Primary Goals:**
- Store comprehensive file metadata (23 fields per file) efficiently
- Enable fast duplicate detection via hash-based queries
- Track scan history and statistics
- Provide type-safe TypeScript integration
- Support future querying for reports and analysis

**Non-Goals (Phase 1):**
- Multi-user support or user authentication
- Database migrations (local-only, can rescan if schema changes)
- Real-time incremental updates (full rescan model for now)
- Network/cloud database support

---

## Database Technology

### Choice: Drizzle ORM + better-sqlite3

**Decision:** Use Drizzle ORM as the data access layer over better-sqlite3.
**Research:** See [research/database/orm-vs-raw-sql.md](./research/database/orm-vs-raw-sql.md)

**Rationale:**
- **SQL-like syntax** - Minimal abstraction, feels like writing SQL
- **Type inference** - Schema is source of truth, types always in sync with database
- **Compile-time safety** - TypeScript errors if column/table doesn't exist
- **Lightweight** - Only ~7.4kb (min+gzip), no runtime dependencies
- **Migration tooling** - drizzle-kit for automatic migrations (needed in later phases)
- **Better DX** - Autocomplete, refactoring safety, less boilerplate than raw SQL
- **Raw SQL fallback** - Can write raw SQL when needed via `.sql` escape hatch
- **Performance** - Can be faster than raw better-sqlite3 via optimized prepared statements

**Underlying Database:** SQLite via better-sqlite3 driver
- **Synchronous API** - Better for CLI tools
- **Single file** - Easy backup, portability, no server setup
- **Transactions** - ACID guarantees for scan integrity
- **Proven scale** - Handles millions of rows efficiently
- **Performance** - 50-130x faster than LibSQL for local file operations (critical for file scanning)

**Alternatives Considered:**

*Database engines:*
- ❌ **PostgreSQL/MySQL** - Overkill for local-only Phase 1, requires server
- ❌ **node-sqlite3** - Async-only API, slower performance
- ❌ **LowDB/JSON files** - Poor performance at scale, no query optimization
- ❌ **LibSQL** - 50-130x slower than better-sqlite3 for local inserts (23s vs 400ms for 10K records); cloud sync features not needed for Phase 1

*ORMs / Data access layers:*
- ❌ **Raw better-sqlite3** - More verbose, manual type safety, schema drift risk
- ❌ **Prisma** - Larger bundle (~6.5MB), heavier abstraction, generates client code
- ❌ **TypeORM** - Heavier, Active Record pattern not ideal for this use case

### Package Installation

```bash
npm install drizzle-orm better-sqlite3
npm install --save-dev drizzle-kit @types/better-sqlite3
```

---

## Schema Design

### Core Tables

#### 1. `files` - File Metadata Storage

**Purpose:** Store all file metadata from scans

```sql
CREATE TABLE files (
    -- Primary key
    id INTEGER PRIMARY KEY AUTOINCREMENT,

    -- Foreign keys
    scan_id INTEGER NOT NULL REFERENCES scan_sessions(id) ON DELETE CASCADE,

    -- Basic file info
    path TEXT NOT NULL,
    name TEXT NOT NULL,
    extension TEXT NOT NULL,  -- Normalized (lowercase, no dot)
    size INTEGER NOT NULL,     -- Bytes
    hash TEXT NOT NULL,        -- SHA256 hex string (64 chars)

    -- Classification
    mime_type TEXT,            -- e.g., 'image/jpeg', NULL if unknown
    category TEXT NOT NULL,    -- FileCategory enum (document, image, video, etc.)
    source_system TEXT NOT NULL,  -- SourceSystem enum (macos, windows, linux, gdrive)

    -- Timestamps (milliseconds since epoch)
    created_at INTEGER NOT NULL,
    modified_at INTEGER NOT NULL,
    accessed_at INTEGER NOT NULL,

    -- Links and permissions
    is_symlink INTEGER NOT NULL DEFAULT 0,  -- Boolean: 0 or 1
    symlink_target TEXT,                     -- NULL if not symlink
    permissions TEXT,                        -- Octal string (e.g., '755'), NULL if unavailable
    owner TEXT,                              -- Username, NULL if unavailable

    -- Organization metadata
    is_hidden INTEGER NOT NULL DEFAULT 0,   -- Boolean: 0 or 1
    depth INTEGER NOT NULL,                 -- Distance from scan root
    finder_tags TEXT,                       -- JSON array (macOS only), NULL otherwise
    finder_color TEXT,                      -- String (macOS only), NULL otherwise

    -- Filesystem metadata
    inode INTEGER NOT NULL,
    hard_link_count INTEGER NOT NULL DEFAULT 1,
    is_executable INTEGER NOT NULL DEFAULT 0,  -- Boolean: 0 or 1

    -- Audit
    created_in_db_at INTEGER NOT NULL DEFAULT (unixepoch('now', 'subsec') * 1000)
);
```

**Indexes:**
```sql
-- Primary queries
CREATE INDEX idx_files_hash ON files(hash);                    -- Duplicate detection
CREATE INDEX idx_files_scan_id ON files(scan_id);              -- Per-scan queries
CREATE INDEX idx_files_category ON files(category);            -- Category reports
CREATE INDEX idx_files_source_system ON files(source_system);  -- System-specific queries
CREATE INDEX idx_files_extension ON files(extension);          -- Extension analysis

-- Composite indexes for common query patterns
CREATE INDEX idx_files_hash_scan ON files(hash, scan_id);      -- Duplicates per scan
CREATE INDEX idx_files_size_hash ON files(size, hash);         -- Quick size + hash lookup

-- Filesystem relationships
CREATE INDEX idx_files_inode ON files(inode, source_system);   -- Hard link detection
CREATE INDEX idx_files_path ON files(path);                    -- Path-based lookups
```

**Notes:**
- `finder_tags` stored as JSON string (e.g., `'["Work","Important"]'` or `NULL`)
- Boolean fields use INTEGER (0/1) for SQLite compatibility
- `created_in_db_at` tracks when record was inserted (vs file creation time)

---

#### 2. `scan_sessions` - Scan Execution History

**Purpose:** Track each scan operation for audit and comparison

```sql
CREATE TABLE scan_sessions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,

    -- Scan configuration
    scan_path TEXT NOT NULL,              -- Root directory scanned
    source_system TEXT NOT NULL,          -- SourceSystem enum

    -- Timestamps
    started_at INTEGER NOT NULL,          -- Milliseconds since epoch
    completed_at INTEGER,                 -- NULL if scan failed/incomplete
    duration_ms INTEGER,                  -- Total scan time (NULL if incomplete)

    -- Results summary
    total_files INTEGER NOT NULL DEFAULT 0,
    total_size INTEGER NOT NULL DEFAULT 0,  -- Bytes
    total_skipped INTEGER NOT NULL DEFAULT 0,
    total_errors INTEGER NOT NULL DEFAULT 0,

    -- Configuration snapshot (JSON)
    options TEXT,                         -- ScanOptions as JSON for reproducibility

    -- Audit
    created_at INTEGER NOT NULL DEFAULT (unixepoch('now', 'subsec') * 1000)
);
```

**Indexes:**
```sql
CREATE INDEX idx_scan_sessions_started_at ON scan_sessions(started_at);
CREATE INDEX idx_scan_sessions_source ON scan_sessions(source_system);
```

**Notes:**
- `options` stores full `ScanOptions` as JSON (e.g., `excludePatterns`, `includeHidden`)
- Incomplete scans have `completed_at = NULL`
- Summary fields updated via transaction after scan completes

---

#### 3. `scan_errors` - Error Tracking

**Purpose:** Store detailed error information from failed file operations

```sql
CREATE TABLE scan_errors (
    id INTEGER PRIMARY KEY AUTOINCREMENT,

    -- Foreign keys
    scan_id INTEGER NOT NULL REFERENCES scan_sessions(id) ON DELETE CASCADE,

    -- Error details
    file_path TEXT NOT NULL,
    error_message TEXT NOT NULL,
    error_code TEXT,                      -- e.g., 'EACCES', 'ENOENT'

    -- Audit
    occurred_at INTEGER NOT NULL DEFAULT (unixepoch('now', 'subsec') * 1000)
);
```

**Indexes:**
```sql
CREATE INDEX idx_scan_errors_scan_id ON scan_errors(scan_id);
CREATE INDEX idx_scan_errors_code ON scan_errors(error_code);
```

---

#### 4. `duplicate_groups` - Duplicate File Clusters

**Purpose:** Group files with identical content for analysis and reporting

```sql
CREATE TABLE duplicate_groups (
    id INTEGER PRIMARY KEY AUTOINCREMENT,

    -- Group identification
    hash TEXT NOT NULL UNIQUE,            -- The shared hash

    -- Metadata
    file_count INTEGER NOT NULL,          -- Number of files with this hash
    total_size INTEGER NOT NULL,          -- size * file_count
    wasted_space INTEGER NOT NULL,        -- size * (file_count - 1)
    is_hard_link_group INTEGER NOT NULL DEFAULT 0,  -- All files share same inode?

    -- First discovered
    first_seen_scan_id INTEGER REFERENCES scan_sessions(id),

    -- Audit
    created_at INTEGER NOT NULL DEFAULT (unixepoch('now', 'subsec') * 1000),
    updated_at INTEGER NOT NULL DEFAULT (unixepoch('now', 'subsec') * 1000)
);
```

**Indexes:**
```sql
CREATE INDEX idx_duplicate_groups_hash ON duplicate_groups(hash);
CREATE INDEX idx_duplicate_groups_wasted_space ON duplicate_groups(wasted_space DESC);
CREATE INDEX idx_duplicate_groups_file_count ON duplicate_groups(file_count DESC);
```

**Notes:**
- `is_hard_link_group = 1` means duplicates are actually hard links (same inode)
- Hard link groups waste no actual space (same physical blocks)
- Updated after each scan to reflect current state

---

#### 5. `statistics` - Cached Aggregate Metrics

**Purpose:** Pre-calculated statistics for fast reporting

```sql
CREATE TABLE statistics (
    id INTEGER PRIMARY KEY AUTOINCREMENT,

    -- Context
    scan_id INTEGER REFERENCES scan_sessions(id) ON DELETE CASCADE,
    metric_type TEXT NOT NULL,            -- e.g., 'category_distribution', 'largest_files'

    -- Data
    metric_name TEXT,                     -- e.g., category name, extension
    metric_value REAL NOT NULL,           -- Count, size, percentage, etc.
    metadata TEXT,                        -- JSON with additional context

    -- Audit
    created_at INTEGER NOT NULL DEFAULT (unixepoch('now', 'subsec') * 1000)
);
```

**Indexes:**
```sql
CREATE INDEX idx_statistics_scan_id ON statistics(scan_id);
CREATE INDEX idx_statistics_type ON statistics(metric_type);
CREATE UNIQUE INDEX idx_statistics_unique ON statistics(scan_id, metric_type, metric_name);
```

**Example Rows:**
```
| scan_id | metric_type            | metric_name | metric_value | metadata                    |
|---------|------------------------|-------------|--------------|------------------------------|
| 1       | category_distribution  | image       | 450          | {"total_size": 12458762}     |
| 1       | category_distribution  | document    | 230          | {"total_size": 5892034}      |
| 1       | largest_files          | NULL        | 524288000    | {"path": "/big.mkv", ...}    |
| 1       | extension_count        | jpg         | 320          | {"avg_size": 2457836}        |
```

---

### Relationships Diagram

```
scan_sessions (1) ──────< (N) files
      │                        │
      │                        │ (hash)
      │                        ↓
      │                  duplicate_groups (1) ──< (N) files [via hash]
      │
      ├──────< (N) scan_errors
      └──────< (N) statistics
```

**Key Relationships:**
- One scan has many files (cascade delete: deleting scan removes its files)
- One scan has many errors (cascade delete)
- One scan has many statistics (cascade delete)
- Files link to duplicate_groups via `hash` (no foreign key, managed by application)
- Duplicate groups reference first scan where duplicates were discovered

---

## Type Safety & TypeScript Integration

### Database Models (TypeScript Interfaces)

```typescript
// Mirrors database schema exactly
export interface FileRecord {
    id: number
    scan_id: number
    path: string
    name: string
    extension: string
    size: number
    hash: string
    mime_type: string | null
    category: FileCategory
    source_system: SourceSystem
    created_at: number
    modified_at: number
    accessed_at: number
    is_symlink: 0 | 1
    symlink_target: string | null
    permissions: string | null
    owner: string | null
    is_hidden: 0 | 1
    depth: number
    finder_tags: string | null  // JSON string or NULL
    finder_color: string | null
    inode: number
    hard_link_count: number
    is_executable: 0 | 1
    created_in_db_at: number
}

export interface ScanSessionRecord {
    id: number
    scan_path: string
    source_system: SourceSystem
    started_at: number
    completed_at: number | null
    duration_ms: number | null
    total_files: number
    total_size: number
    total_skipped: number
    total_errors: number
    options: string | null  // JSON string
    created_at: number
}

export interface ScanErrorRecord {
    id: number
    scan_id: number
    file_path: string
    error_message: string
    error_code: string | null
    occurred_at: number
}

export interface DuplicateGroupRecord {
    id: number
    hash: string
    file_count: number
    total_size: number
    wasted_space: number
    is_hard_link_group: 0 | 1
    first_seen_scan_id: number | null
    created_at: number
    updated_at: number
}

export interface StatisticRecord {
    id: number
    scan_id: number | null
    metric_type: string
    metric_name: string | null
    metric_value: number
    metadata: string | null  // JSON string
    created_at: number
}
```

### Conversion Functions

```typescript
// Convert FileMetadata (in-memory) to FileRecord (database)
function fileMetadataToRecord(
    metadata: FileMetadata,
    scanId: number
): Omit<FileRecord, 'id' | 'created_in_db_at'>

// Convert FileRecord (database) to FileMetadata (in-memory)
function fileRecordToMetadata(record: FileRecord): FileMetadata
```

**JSON Field Handling:**
```typescript
// finder_tags: string[] | undefined → string | null
const tagsJson = metadata.finderTags
    ? JSON.stringify(metadata.finderTags)
    : null

// finder_tags: string | null → string[] | undefined
const tags = record.finder_tags
    ? JSON.parse(record.finder_tags)
    : undefined
```

**Boolean Mapping:**
```typescript
// TypeScript → SQLite
is_symlink: metadata.isSymlink ? 1 : 0

// SQLite → TypeScript
isSymlink: record.is_symlink === 1
```

---

## Data Access Layer Architecture

### Repository Pattern

**Structure:**
```
src/
├── core/
│   ├── database/
│   │   ├── index.ts                    # Exports all database modules
│   │   ├── connection.ts               # Database initialization
│   │   ├── schema.ts                   # CREATE TABLE statements
│   │   ├── repositories/
│   │   │   ├── FileRepository.ts       # CRUD for files table
│   │   │   ├── ScanSessionRepository.ts
│   │   │   ├── DuplicateRepository.ts
│   │   │   └── StatisticsRepository.ts
│   │   ├── models/
│   │   │   └── index.ts                # TypeScript interfaces (FileRecord, etc.)
│   │   └── converters/
│   │       └── index.ts                # Metadata ↔ Record conversions
```

### Database Initialization

```typescript
// src/core/database/connection.ts

import Database from 'better-sqlite3'

export class DatabaseConnection {
    private db: Database.Database

    constructor(filepath: string) {
        this.db = new Database(filepath)
        this.configure()
        this.initializeSchema()
    }

    private configure() {
        // Performance optimizations
        this.db.pragma('journal_mode = WAL')  // Better concurrency
        this.db.pragma('synchronous = NORMAL') // Faster writes
        this.db.pragma('foreign_keys = ON')    // Enforce relationships
    }

    private initializeSchema() {
        // Run all CREATE TABLE and CREATE INDEX statements
        // Idempotent (IF NOT EXISTS)
    }

    getDatabase(): Database.Database {
        return this.db
    }

    close() {
        this.db.close()
    }
}
```

### Repository Example (FileRepository)

```typescript
// src/core/database/repositories/FileRepository.ts

export class FileRepository {
    constructor(private db: Database.Database) {}

    // Insert single file
    insert(file: Omit<FileRecord, 'id' | 'created_in_db_at'>): number

    // Insert multiple files (transaction)
    insertBatch(files: Omit<FileRecord, 'id' | 'created_in_db_at'>[]): void

    // Get by ID
    findById(id: number): FileRecord | null

    // Get by hash (for duplicates)
    findByHash(hash: string): FileRecord[]

    // Get all files for a scan
    findByScanId(scanId: number): FileRecord[]

    // Get by path
    findByPath(path: string): FileRecord | null

    // Count files by category
    countByCategory(scanId: number): Map<FileCategory, number>

    // Delete all files for a scan (cascade handles this, but explicit method for clarity)
    deleteByScanId(scanId: number): number  // Returns rows deleted
}
```

---

## Common Query Patterns

### 1. Duplicate Detection

**Use Case:** Find all files with duplicate content

**Approach:**
```sql
-- Step 1: Find hashes with multiple files
SELECT hash, COUNT(*) as count, SUM(size) as total_size
FROM files
WHERE scan_id = ?
GROUP BY hash
HAVING count > 1
ORDER BY total_size DESC;

-- Step 2: Get all files for a specific duplicate hash
SELECT * FROM files
WHERE hash = ? AND scan_id = ?
ORDER BY path;

-- Step 3: Distinguish hard links from true duplicates
SELECT hash, COUNT(DISTINCT inode) as unique_inodes, COUNT(*) as file_count
FROM files
WHERE hash = ?
GROUP BY hash;
-- If unique_inodes = 1, all files are hard links (same physical file)
```

**Repository Method:**
```typescript
findDuplicates(scanId: number): DuplicateGroup[]
// Returns array of { hash, files[], isHardLinkGroup }
```

---

### 2. Scan Comparison

**Use Case:** Compare two scans to find new/modified/deleted files

**Approach:**
```sql
-- New files (in scan2 but not scan1)
SELECT f2.* FROM files f2
WHERE f2.scan_id = ? -- scan2
AND NOT EXISTS (
    SELECT 1 FROM files f1
    WHERE f1.scan_id = ? -- scan1
    AND f1.path = f2.path
);

-- Deleted files (in scan1 but not scan2)
SELECT f1.* FROM files f1
WHERE f1.scan_id = ? -- scan1
AND NOT EXISTS (
    SELECT 1 FROM files f2
    WHERE f2.scan_id = ? -- scan2
    AND f2.path = f1.path
);

-- Modified files (same path, different hash)
SELECT f1.path, f1.hash as old_hash, f2.hash as new_hash,
       f1.size as old_size, f2.size as new_size
FROM files f1
JOIN files f2 ON f1.path = f2.path
WHERE f1.scan_id = ? -- scan1
  AND f2.scan_id = ? -- scan2
  AND f1.hash != f2.hash;
```

---

### 3. Category Distribution

**Use Case:** Show file counts and sizes by category

**Approach:**
```sql
SELECT
    category,
    COUNT(*) as file_count,
    SUM(size) as total_size,
    AVG(size) as avg_size,
    MIN(size) as min_size,
    MAX(size) as max_size
FROM files
WHERE scan_id = ?
GROUP BY category
ORDER BY total_size DESC;
```

**Cached in `statistics` table after scan:**
```typescript
// Pre-calculate and store
statistics.insert({
    scan_id: 1,
    metric_type: 'category_distribution',
    metric_name: 'image',
    metric_value: 450, // file count
    metadata: JSON.stringify({ total_size: 12458762, avg_size: 27686 })
})
```

---

### 4. Largest Files

**Use Case:** Find top N largest files

**Approach:**
```sql
SELECT path, name, size, category
FROM files
WHERE scan_id = ?
ORDER BY size DESC
LIMIT 50;
```

---

### 5. Extension Analysis

**Use Case:** Find files by extension with statistics

**Approach:**
```sql
SELECT
    extension,
    COUNT(*) as count,
    SUM(size) as total_size
FROM files
WHERE scan_id = ?
GROUP BY extension
ORDER BY count DESC;
```

---

### 6. Find Files by Criteria

**Use Case:** Search files by various filters

**Approach:**
```sql
-- By category
SELECT * FROM files
WHERE scan_id = ? AND category = 'image'
ORDER BY size DESC;

-- By owner
SELECT * FROM files
WHERE scan_id = ? AND owner = 'hameem'
ORDER BY path;

-- Hidden files
SELECT * FROM files
WHERE scan_id = ? AND is_hidden = 1;

-- Executable files
SELECT * FROM files
WHERE scan_id = ? AND is_executable = 1;

-- Files in specific directory (depth-based)
SELECT * FROM files
WHERE scan_id = ?
  AND path LIKE '/Users/hameem/Documents/%'
  AND depth = 3;  -- Exactly 3 levels deep from scan root
```

---

### 7. Scan History

**Use Case:** View all past scans with summaries

**Approach:**
```sql
SELECT
    id,
    scan_path,
    source_system,
    started_at,
    completed_at,
    duration_ms,
    total_files,
    total_size,
    total_errors
FROM scan_sessions
ORDER BY started_at DESC
LIMIT 20;
```

---

### 8. Wasted Space Analysis

**Use Case:** Calculate total wasted space from duplicates

**Approach:**
```sql
-- From duplicate_groups table
SELECT
    SUM(wasted_space) as total_wasted,
    COUNT(*) as duplicate_groups,
    SUM(file_count) as total_duplicate_files
FROM duplicate_groups;

-- Break down by hard links vs true duplicates
SELECT
    CASE WHEN is_hard_link_group = 1 THEN 'Hard Links' ELSE 'True Duplicates' END as type,
    COUNT(*) as groups,
    SUM(wasted_space) as wasted_space
FROM duplicate_groups
GROUP BY is_hard_link_group;
```

---

### 9. macOS Finder Tag Analysis

**Use Case:** Find files by Finder tag (macOS only)

**Approach:**
```sql
-- Find files with specific tag (JSON query - requires JSON extension or LIKE)
SELECT * FROM files
WHERE scan_id = ?
  AND finder_tags LIKE '%"Work"%'  -- Simple substring match
  AND source_system = 'macos';

-- Count files by tag (requires application-level processing)
-- Extract tags from JSON, aggregate counts
```

**Note:** For complex JSON queries, consider enabling JSON1 extension:
```typescript
db.pragma('ENABLE_JSON1')
// Then use: JSON_EACH(finder_tags)
```

---

### 10. Error Analysis

**Use Case:** Review failed files from a scan

**Approach:**
```sql
-- Group errors by type
SELECT
    error_code,
    COUNT(*) as count,
    GROUP_CONCAT(file_path, ', ') as sample_paths
FROM scan_errors
WHERE scan_id = ?
GROUP BY error_code
ORDER BY count DESC;

-- All errors for a scan
SELECT * FROM scan_errors
WHERE scan_id = ?
ORDER BY occurred_at;
```

---

## Data Lifecycle & Management

### Scan Workflow

**1. Start Scan:**
```typescript
const session = scanSessionRepo.create({
    scan_path: '/Users/hameem',
    source_system: SourceSystem.MACOS,
    started_at: Date.now(),
    options: JSON.stringify(scanOptions)
})
const scanId = session.id
```

**2. Process Files (in transaction):**
```typescript
db.transaction(() => {
    for (const metadata of scannedFiles) {
        const record = convertToRecord(metadata, scanId)
        fileRepo.insert(record)
    }
})
```

**3. Complete Scan:**
```typescript
scanSessionRepo.update(scanId, {
    completed_at: Date.now(),
    duration_ms: endTime - startTime,
    total_files: fileCount,
    total_size: totalBytes
})
```

**4. Analyze Duplicates:**
```typescript
const duplicates = fileRepo.findDuplicates(scanId)
for (const dup of duplicates) {
    duplicateRepo.upsert({
        hash: dup.hash,
        file_count: dup.files.length,
        total_size: dup.files[0].size * dup.files.length,
        wasted_space: dup.files[0].size * (dup.files.length - 1),
        is_hard_link_group: dup.isHardLinkGroup ? 1 : 0
    })
}
```

**5. Generate Statistics:**
```typescript
const categoryStats = fileRepo.countByCategory(scanId)
for (const [category, count] of categoryStats) {
    statisticsRepo.insert({ ... })
}
```

---

### Database File Location

**Default Path:**
```
~/.sheldonfs/data.db  (or %USERPROFILE%\.sheldonfs\data.db on Windows)
```

**Custom Path (via CLI):**
```bash
sheldon-scan /path --db ./custom.db
```

---

### Configuration Management

**Approach:** Config file for user preferences, database for runtime state

**Config File Location:**
```
~/.sheldonfs/config.json  (or %USERPROFILE%\.sheldonfs\config.json on Windows)
```

**Structure:**
```json
{
    "database": {
        "path": "~/.sheldonfs/data.db"
    },
    "scanRetention": {
        "keepScans": 5,
        "autoCleanup": true
    },
    "defaults": {
        "excludePatterns": ["node_modules/**", ".git/**"],
        "includeHidden": false
    },
    "logging": {
        "level": "info"
    }
}
```

**Rationale:**
- **Config file for user preferences** - Rarely changing settings, user-editable, version-controllable
- **Database for runtime state** - Frequently changing data, application-managed
- **Solves bootstrap problem** - Need config to know where database is located
- **Common pattern** - Follows CLI tool conventions (.gitconfig, .npmrc, .aws/config)

**What goes where:**

*Config file (`config.json`):*
- Database file path
- Scan retention settings
- Default scan options
- Logging preferences
- Any user-editable preferences

*Database (`app_state` table, Phase 2+):*
- Last scan timestamp
- Scan in progress flag
- Last cleanup timestamp
- Runtime application state

---

### Cleanup & Maintenance

**Delete Old Scans:**
```sql
-- Delete scan and all related data (CASCADE handles files, errors, stats)
DELETE FROM scan_sessions WHERE id = ?;
```

**Vacuum Database (reclaim space):**
```typescript
db.pragma('vacuum')  // Rebuild database file
```

**Analyze Query Performance:**
```sql
EXPLAIN QUERY PLAN SELECT * FROM files WHERE hash = ?;
-- Verify indexes are being used
```

---

## Performance Considerations

### Transaction Batching

**Problem:** Inserting 10,000 files one-by-one is slow (each write flushes to disk)

**Solution:** Wrap inserts in transaction
```typescript
const insert = db.prepare('INSERT INTO files (...) VALUES (...)')

db.transaction((files) => {
    for (const file of files) {
        insert.run(file)
    }
})(filesArray)

// Result: 10-100x faster than individual inserts
```

---

### Prepared Statements

**Reuse compiled queries:**
```typescript
class FileRepository {
    private insertStmt: Database.Statement

    constructor(db: Database.Database) {
        this.insertStmt = db.prepare(`
            INSERT INTO files (scan_id, path, name, ...)
            VALUES (@scan_id, @path, @name, ...)
        `)
    }

    insert(file: FileRecord) {
        return this.insertStmt.run(file)
    }
}
```

---

### Indexing Strategy

**Covered Indexes (avoid table lookups):**
```sql
-- Query: SELECT hash, size FROM files WHERE scan_id = ?
-- Index covers query entirely (no table access needed)
CREATE INDEX idx_files_scan_hash_size ON files(scan_id, hash, size);
```

**Partial Indexes (smaller, faster):**
```sql
-- Only index large files for wasted space analysis
CREATE INDEX idx_files_large_hash ON files(hash)
WHERE size > 1048576;  -- 1MB+
```

---

### WAL Mode Benefits

**Write-Ahead Logging (enabled by default):**
- Readers don't block writers
- Faster writes (sequential instead of random I/O)
- Better concurrency for Phase 4 (web UI)

```typescript
db.pragma('journal_mode = WAL')
```

**Trade-off:** Creates `-wal` and `-shm` files alongside `data.db`

---

## Testing Strategy

### Database Tests

**1. Schema Initialization**
- Tables created successfully
- Indexes exist
- Foreign keys enforced

**2. Repository CRUD**
- Insert, find, update, delete operations
- Batch inserts with transactions
- Constraint violations handled

**3. Query Correctness**
- Duplicate detection logic
- Category aggregation
- Scan comparison

**4. Data Integrity**
- Cascade deletes work correctly
- Foreign key constraints enforced
- JSON fields round-trip correctly

**Test Database:**
```typescript
// Use in-memory database for fast tests
const testDb = new Database(':memory:')
```

---

## Migration Strategy

### Phase 1: `drizzle-kit push` (Development)

**Approach:** Direct schema synchronization without migration files

```bash
# After defining/changing schema in TypeScript
npx drizzle-kit push
```

**Characteristics:**
- No migration files generated
- Schema synced directly from TypeScript definitions to database
- Fast iteration and prototyping
- Recommended for local development and single-user scenarios

**Why this works for Phase 1:**
- You're the only user (can rescan if schema breaks)
- Development phase (rapid schema changes expected)
- No production users to protect
- Faster workflow (no migration file management)

### Phase 3+: `drizzle-kit generate` (Production)

**Approach:** Version-controlled migration files for safe production deployments

```bash
# 1. Change schema in TypeScript
# 2. Test locally with push
npx drizzle-kit push

# 3. If ready for production, generate migration
npx drizzle-kit generate

# 4. Review generated SQL migration file
# 5. Commit migration file to git
# 6. Apply migration in production during deployment
```

**Why switch in Phase 3+:**
- Multiple users/environments need schema versioning
- Can't just "rescan" - must preserve user data
- Team collaboration requires migration history
- Production safety (review SQL before applying)

**Migration tracking:**
Drizzle automatically creates a `__drizzle_migrations` table to track applied migrations.

### Recommended Workflow

**Development (Phase 1-2):**
```bash
npm run db:push    # Alias for drizzle-kit push
```

**Production (Phase 3+):**
```bash
npm run db:generate    # Generate migration files
npm run db:migrate     # Apply migrations
```

---

## Security Considerations

### SQL Injection Prevention

**Always use parameterized queries:**
```typescript
// ✅ SAFE
db.prepare('SELECT * FROM files WHERE hash = ?').get(userHash)

// ❌ DANGEROUS
db.prepare(`SELECT * FROM files WHERE hash = '${userHash}'`).get()
```

**better-sqlite3 uses named/positional parameters automatically**

---

### File Paths

**Handle special characters:**
- SQLite TEXT type supports Unicode
- No escaping needed with parameterized queries
- Store absolute paths to avoid ambiguity

---

## Design Decisions

The following design decisions have been made after thorough research and analysis:

### 1. ORM vs Raw SQL

**Decision:** Use Drizzle ORM
**Research:** [research/database/orm-vs-raw-sql.md](./research/database/orm-vs-raw-sql.md)

**Rationale:**
- SQL-like syntax with minimal abstraction
- Type inference keeps schema and types in sync
- Migration tooling (drizzle-kit) will be needed in later phases
- Better developer experience without significant downsides

### 2. Partial Hash Storage

**Decision:** Compute on-demand for Phase 1
**Research:** [research/database/partial-hash-storage.md](./research/database/partial-hash-storage.md)

**Rationale:**
- Keep schema simple for Phase 1
- No proven performance bottleneck yet
- Easy to add column later if needed
- In development phase, can rebuild database without migrations

**Revisit when:** Phase 5 (Optimization) or if duplicate detection becomes slow (>10-30 seconds)

### 3. Scan Diff Table

**Decision:** Compute on-demand
**Research:** [research/database/scan-diff-table.md](./research/database/scan-diff-table.md)

**Rationale:**
- Complexity far outweighs minor performance benefits
- With proper indexes, diff queries are fast (sub-second for 10K files)
- Scan comparison is not a primary Phase 1 use case
- No data redundancy - files table remains source of truth

**Revisit when:** Phase 4 (Web UI) if dashboard needs instant diff display

### 4. File Deletion Strategy

**Decision:** Hard deletes (no `deleted_at` column)
**Research:** [research/database/soft-deletes.md](./research/database/soft-deletes.md)

**Rationale:**
- Soft deletes add significant complexity (every query needs filtering)
- Each scan is a clean snapshot - simpler mental model
- Better query performance (no filtering, smaller indexes)
- Scan retention policy handles storage management separately

### 5. Scan Retention Policy

**Decision:** Configurable retention with default of 5 scans

**Implementation:**
- Global configuration setting (default: keep last 5 scans)
- User-configurable via config file
- Automatic cleanup of old scans after each new scan completes
- Provides bounded storage without soft delete complexity

**Configuration:**
```typescript
// Global config (~/.sheldonfs/config.json)
{
    "scanRetention": {
        "keepScans": 5,  // Number of scans to keep (0 = keep all)
        "autoCleanup": true
    }
}
```

**Benefits:**
- Bounded database storage (predictable size)
- Recent history available for comparison
- User flexibility (can increase or disable)
- Automatic maintenance (no manual cleanup)

---

## Future Enhancements

### Phase 4 (Web UI)
- Add user session tracking, UI preferences
- Consider scan_diffs table if instant diff display needed
- Dashboard for visualization

### Phase 5 (Optimization)
- Incremental scans with modified_at-based change detection
- Consider partial_hash storage if duplicate detection is slow
- HTML report generation

### Phase 6 (Semantic Intelligence)
- Vector embeddings table (768-dimensional floats)
- sqlite-vec extension integration
- LLM-powered file organization

---

## Summary

**Database:** Drizzle ORM + better-sqlite3 (type-safe, SQL-like, synchronous)

**Core Tables:**
- `files` - 23 metadata fields per file
- `scan_sessions` - Scan execution history
- `scan_errors` - Failed file operations
- `duplicate_groups` - Content-identical file clusters
- `statistics` - Pre-calculated metrics

**Key Features:**
- Type-safe TypeScript integration
- Repository pattern for data access
- Transaction-based batch inserts
- Comprehensive indexing for fast queries
- Cascade deletes for scan cleanup

**Performance:**
- WAL mode for concurrency
- Prepared statements for reuse
- Covered indexes where applicable
- Transaction batching for bulk inserts

**Next Steps:**
1. Implement schema and connection management
2. Build repository layer with CRUD operations
3. Add conversion utilities (Metadata ↔ Record)
4. Write database integration tests
5. Integrate with FileScanner to persist scan results
