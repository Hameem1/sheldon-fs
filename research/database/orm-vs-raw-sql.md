# Research: ORM vs Raw SQL for SheldonFS

**Status:** Decision Made
**Date:** 2025-01-16
**Decision:** Use Drizzle ORM (see DATABASE_DESIGN.md)
**Context:** Evaluating whether to use an ORM (specifically Drizzle) or raw better-sqlite3 for Phase 1 database layer

## Final Decision

**Chosen: Drizzle ORM**

**Rationale:**
- Drizzle is already quite close to raw SQL, allows writing raw SQL when absolutely needed
- Provides superior developer experience with type inference and compile-time safety
- Schema is the source of truth, types always in sync with database
- Migration tooling (drizzle-kit) will be needed in later phases anyway
- Minimal cons (lightweight, no runtime overhead, SQL-like syntax)
- Starting with Drizzle now avoids future refactoring and provides cleaner, more maintainable code from the start

**Trade-offs accepted:**
- Additional dependency (drizzle-orm + drizzle-kit) - acceptable given benefits
- Small learning curve - offset by excellent documentation and SQL-like API

---

## Research Summary

After researching modern TypeScript ORMs (particularly Drizzle, Prisma, TypeORM), here's the analysis for SheldonFS:

**Top ORM Candidates:**
- **Drizzle ORM** - Lightweight (~7.4kb), SQL-first, zero dependencies, TypeScript inference
- **Prisma** - Full-featured, schema-first, great DX, but larger bundle size (~6.5MB)
- **TypeORM** - Mature, supports Active Record/Data Mapper, heavier abstraction

---

## Drizzle ORM Overview

### Key Features

**Installation:**
```bash
npm i drizzle-orm better-sqlite3
npm i -D drizzle-kit @types/better-sqlite3
```

**Setup:**
```typescript
import { drizzle } from 'drizzle-orm/better-sqlite3'
import Database from 'better-sqlite3'

const sqlite = new Database('sqlite.db')
const db = drizzle({ client: sqlite })
```

**Philosophy:**
- "If you know SQL, you know Drizzle ORM"
- SQL-first, minimal abstraction
- TypeScript inference (no code generation)
- Lightweight: ~7.4kb (min+gzip), zero runtime dependencies

**Performance:**
- Can be faster than raw better-sqlite3 by utilizing prepared statements automatically
- "One SQL per query" guarantee (no N+1 problems)
- Excellent for serverless and edge environments (minimal cold start overhead)

---

## Comparison: Drizzle vs Prisma

### Bundle Size & Performance

**Drizzle:**
- ~7.4kb (min+gzip)
- ~1.5 MB bundle size
- No external dependencies
- Zero runtime overhead

**Prisma:**
- ~6.5 MB bundle size
- Uses Rust-based query engine binary
- Some overhead for query translation
- Heavier for serverless environments

### Developer Experience

**Drizzle:**
- TypeScript inference directly from schema
- Changes reflected immediately (no code generation)
- SQL-like query syntax
- Lower abstraction layer (closer to SQL)

**Prisma:**
- Schema-first with Prisma Schema Language
- Generated type-safe client
- Higher abstraction layer
- More "magical" (hides SQL details)

### Type Safety

**Both provide excellent type safety:**
- Drizzle: Types inferred from schema definitions
- Prisma: Types generated from schema files

### Query Philosophy

**Drizzle:**
- Mirrors SQL closely
- Easy to write raw SQL when needed
- Relational queries via `db.query` API

**Prisma:**
- Declarative query API
- Can generate inefficient queries (sub-queries instead of JOINs)
- Raw SQL via `$queryRaw` when needed

---

## Code Comparison: FileRepository.insert()

### Raw better-sqlite3 Approach

```typescript
class FileRepository {
    private insertStmt: Database.Statement

    constructor(private db: Database.Database) {
        this.insertStmt = db.prepare(`
            INSERT INTO files (
                scan_id, path, name, extension, size, hash,
                mime_type, category, source_system,
                created_at, modified_at, accessed_at,
                is_symlink, symlink_target, permissions, owner,
                is_hidden, depth, finder_tags, finder_color,
                inode, hard_link_count, is_executable
            ) VALUES (
                @scan_id, @path, @name, @extension, @size, @hash,
                @mime_type, @category, @source_system,
                @created_at, @modified_at, @accessed_at,
                @is_symlink, @symlink_target, @permissions, @owner,
                @is_hidden, @depth, @finder_tags, @finder_color,
                @inode, @hard_link_count, @is_executable
            )
        `)
    }

    insert(file: Omit<FileRecord, 'id' | 'created_in_db_at'>): number {
        const result = this.insertStmt.run(file)
        return result.lastInsertRowid as number
    }

    findByHash(hash: string): FileRecord[] {
        return this.db.prepare(`
            SELECT * FROM files WHERE hash = ?
        `).all(hash) as FileRecord[]
    }

    countByCategory(scanId: number): Map<FileCategory, number> {
        const rows = this.db.prepare(`
            SELECT category, COUNT(*) as count
            FROM files
            WHERE scan_id = ?
            GROUP BY category
        `).all(scanId) as Array<{ category: FileCategory; count: number }>

        return new Map(rows.map(r => [r.category, r.count]))
    }
}
```

### Drizzle ORM Approach

```typescript
// Schema definition (drizzle/schema.ts)
import { sqliteTable, integer, text } from 'drizzle-orm/sqlite-core'

export const files = sqliteTable('files', {
    id: integer('id').primaryKey({ autoIncrement: true }),
    scan_id: integer('scan_id').notNull().references(() => scanSessions.id, { onDelete: 'cascade' }),
    path: text('path').notNull(),
    name: text('name').notNull(),
    extension: text('extension').notNull(),
    size: integer('size').notNull(),
    hash: text('hash').notNull(),
    mime_type: text('mime_type'),
    category: text('category').notNull(),
    source_system: text('source_system').notNull(),
    created_at: integer('created_at').notNull(),
    modified_at: integer('modified_at').notNull(),
    accessed_at: integer('accessed_at').notNull(),
    is_symlink: integer('is_symlink', { mode: 'boolean' }).notNull().default(false),
    symlink_target: text('symlink_target'),
    permissions: text('permissions'),
    owner: text('owner'),
    is_hidden: integer('is_hidden', { mode: 'boolean' }).notNull().default(false),
    depth: integer('depth').notNull(),
    finder_tags: text('finder_tags'), // JSON
    finder_color: text('finder_color'),
    inode: integer('inode').notNull(),
    hard_link_count: integer('hard_link_count').notNull().default(1),
    is_executable: integer('is_executable', { mode: 'boolean' }).notNull().default(false),
    created_in_db_at: integer('created_in_db_at', { mode: 'timestamp' })
        .notNull()
        .$defaultFn(() => new Date())
})

// Repository implementation
import { drizzle } from 'drizzle-orm/better-sqlite3'
import { eq, sql } from 'drizzle-orm'

class FileRepository {
    constructor(private db: ReturnType<typeof drizzle>) {}

    insert(file: typeof files.$inferInsert): number {
        const result = this.db.insert(files).values(file).run()
        return result.lastInsertRowid as number
    }

    findByHash(hash: string) {
        return this.db.select().from(files).where(eq(files.hash, hash))
    }

    countByCategory(scanId: number) {
        return this.db
            .select({
                category: files.category,
                count: sql<number>`count(*)`
            })
            .from(files)
            .where(eq(files.scan_id, scanId))
            .groupBy(files.category)
    }
}
```

---

## Pros and Cons Analysis

### Raw better-sqlite3

**Pros:**
- ✅ **Full SQL control** - Write exactly the queries you want
- ✅ **Zero abstraction overhead** - No translation layer between code and SQL
- ✅ **Simplicity** - One less dependency, no build steps for schema
- ✅ **Maximum performance** - Direct access to prepared statements
- ✅ **Learning value** - Better understanding of SQL and database internals
- ✅ **No magic** - Explicit about what's happening
- ✅ **Easier debugging** - SQL is visible and inspectable
- ✅ **Smaller dependency tree** - Only better-sqlite3

**Cons:**
- ❌ **Manual type safety** - Must manually maintain TypeScript interfaces matching schema
- ❌ **Verbose SQL strings** - Long INSERT/UPDATE statements get unwieldy
- ❌ **Schema drift risk** - TypeScript types can get out of sync with actual database
- ❌ **No automatic migrations** - Must write all migration SQL manually
- ❌ **More boilerplate** - Preparing statements, handling types explicitly
- ❌ **Type casting required** - Must cast query results to TypeScript types

### Drizzle ORM

**Pros:**
- ✅ **Type inference** - Types derived directly from schema, always in sync
- ✅ **SQL-like syntax** - Familiar to SQL developers, minimal learning curve
- ✅ **Compile-time safety** - TypeScript errors if column/table doesn't exist
- ✅ **Migrations tooling** - `drizzle-kit` generates migrations automatically
- ✅ **Lightweight** - Only 7.4kb, no runtime dependencies
- ✅ **Better DX** - Autocomplete for columns, relationships, queries
- ✅ **Relational queries** - Can use `db.query` for JOIN-heavy operations
- ✅ **Schema as source of truth** - Single place to define database structure
- ✅ **Refactoring safety** - Rename column in schema, TypeScript catches all usages

**Cons:**
- ❌ **Additional dependency** - Another package to maintain (drizzle-orm + drizzle-kit)
- ❌ **Abstraction layer** - One more thing between you and SQL
- ❌ **Learning curve** - Team needs to learn Drizzle API
- ❌ **Build step consideration** - Schema changes require understanding Drizzle conventions
- ❌ **Slightly less control** - Can't always write exotic SQL without `.sql` escape hatch
- ❌ **Potential over-engineering** - May be overkill for simple schemas

---

## Preliminary Recommendation: Raw better-sqlite3 for Phase 1

### Rationale

1. **Phase 1 simplicity** - We have 5 well-defined tables and straightforward queries. The complexity doesn't justify an ORM yet.

2. **No migrations needed yet** - CLAUDE.md explicitly states "No migration system needed yet (local-only, can rescan if schema changes)". Drizzle's main advantage (automatic migrations) isn't valuable in Phase 1.

3. **Learning and control** - Since this is a personal project becoming open-source, having explicit SQL helps contributors understand the database layer without learning Drizzle API.

4. **Performance clarity** - With raw SQL, it's obvious what queries are running. This is valuable for a file scanning tool where performance matters.

5. **Easy migration path** - If needed in Phase 3+ (when migrations matter), we can migrate to Drizzle. The repository pattern we're using makes this straightforward - just swap the implementation.

6. **YAGNI principle** - "You Aren't Gonna Need It" - Don't add abstractions until they're proven necessary.

### When to Reconsider

**Drizzle becomes more attractive when:**
- **Phase 3+** when migration system becomes necessary
- **Phase 4** when web UI needs complex relational queries with many JOINs
- If schema grows beyond ~10 tables and becomes harder to manage manually
- If we find ourselves repeatedly making schema changes that break TypeScript types
- If team grows and contributors struggle with raw SQL

### Compromise Approach (If Desired)

**Incremental adoption:**
- Start with raw better-sqlite3 for Phase 1
- Document the schema well (we have DATABASE_DESIGN.md)
- If complexity grows in Phase 2-3, add Drizzle incrementally
- Drizzle can coexist with raw SQL (use `db.run(sql`...`)` for raw queries)
- Migrate one table at a time if needed

---

## Additional Resources

### Documentation
- [Drizzle ORM - SQLite](https://orm.drizzle.team/docs/get-started-sqlite)
- [Drizzle vs Prisma Comparison](https://www.bytebase.com/blog/drizzle-vs-prisma/)
- [Better Stack: Getting Started with Drizzle](https://betterstack.com/community/guides/scaling-nodejs/drizzle-orm/)

### Benchmarks
- [Drizzle ORM Benchmarks](https://orm.drizzle.team/benchmarks)
- Drizzle can be faster than raw better-sqlite3 due to optimized prepared statement usage

---

## Implementation Notes

See DATABASE_DESIGN.md for:
- Complete Drizzle schema definitions
- Repository implementation examples
- Migration strategy
- Type safety integration
