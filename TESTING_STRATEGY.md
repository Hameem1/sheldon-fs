# Testing Strategy for SheldonFS

## Philosophy

**Goals:**
- Establish robust testing infrastructure that's maintainable as features evolve
- Focus on critical functionality and integration points
- Avoid excessive test overhead that slows down development
- Keep test suite fast (<5 seconds for now)

**Approach: Hybrid Testing**
1. **Unit tests** - Pure functions tested with mocks/stubs (no file I/O)
2. **Integration tests** - Generate temporary files/directories on-the-fly
3. **Minimal fixtures** - Small set of real files (< 10KB total) for specific edge cases

## Implementation Status

**Infrastructure:** ✅ Complete and ready for development

- ✅ Vitest configuration with path aliases and coverage
- ✅ Test helpers for file generation and cleanup
- ✅ Automatic temp file cleanup after each test
- ✅ First test suite: `hashCalculator.test.ts` (16 tests, all passing)
- ✅ Coverage: 96.96% on `hashCalculator.ts`
- ✅ Test execution time: ~52ms

**Bug Found & Fixed:** `calculatePartialHash` crashed on empty files (tried to set `end: -1`). Fixed by checking for empty files and returning hash immediately.

## Test Organization

```
src/
├── core/
│   └── scanner/
│       └── __tests__/
│           ├── hashCalculator.test.ts        # ✅ 16 tests, 96.96% coverage
│           ├── metadataExtractor.test.ts     # Planned: unit + integration
│           └── fileScanner.test.ts           # Planned: integration tests
└── __tests__/
    ├── fixtures/                             # Binary files for MIME testing
    │   └── README.md                         # Documents sources (fixtures TBD)
    └── helpers/
        ├── fileGenerator.ts                  # ✅ Temp file/dir creation
        ├── cleanup.ts                        # ✅ Auto cleanup after tests
        ├── setup.ts                          # ✅ Vitest setup
        └── index.ts                          # ✅ Centralized exports
```

## Priority Test Cases

### Phase 1: Foundation

#### 1. Hash Calculator ✅ COMPLETE
**File:** `src/core/scanner/__tests__/hashCalculator.test.ts` (16 tests, 96.96% coverage)

- Known hash verification, empty/small files, partial hash, quick compare, error handling

#### 2. Metadata Extraction (Next Priority)
**File:** `src/core/scanner/__tests__/metadataExtractor.test.ts`

**Unit tests:** Category detection, permission conversion, depth calculation, hidden file detection, executable detection

**Integration tests:** Basic metadata extraction, timestamps, MIME types, symlinks, owner, macOS Finder tags (conditional)

#### 3. File Scanner
**File:** `src/core/scanner/__tests__/fileScanner.test.ts`

Empty directory, single file, nested structure, hidden files (excluded/included), exclusion patterns (default/custom), progress callback, error handling

### Phase 2: Edge Cases (After Foundation)

- Symlink circular references
- Files with no extension
- Very long file names/paths
- Files with special characters in names
- Permission denied scenarios
- Hard links (same inode, different paths)
- Files modified during scan
- Cross-platform differences (Windows vs Unix paths)

## Test Data Strategy

### 1. Generated Files (Primary Method) ✅
**Implementation:** `src/__tests__/helpers/fileGenerator.ts`

Available utilities:
- `createTempDir()` - Temporary directory with auto-cleanup
- `createTempFile({ content, extension, mode })` - Files with custom content/permissions
- `createFileStructure({ ... })` - Nested directory structures from objects
- `createSymlink()`, `createFileWithSize()`, `createHiddenFile()`, `createExecutableFile()`

All files/directories automatically cleaned up after each test via `cleanup.ts`.

**Benefits:** No git commits, full control, cross-platform, fast and deterministic.

### 2. Minimal Fixtures (To Be Added)
Small binary files for MIME type detection (~4KB total):
- `sample.jpg`, `sample.pdf`, `sample.zip`, `sample.png` (~1KB each)
- Sources: Public domain or generated, documented in `fixtures/README.md`
- Only used when temp files can't replicate scenario

### 3. No Personal Files
❌ Never commit personal files
✅ All test data generated or from public sources

## Vitest Configuration ✅

**File:** `vitest.config.ts`

Key settings:
- Environment: Node.js
- Pattern: `src/**/__tests__/**/*.test.ts`
- Coverage: v8 provider with text/json/html reports
- Setup: Automatic cleanup via `./src/__tests__/helpers/setup.ts`
- Aliases: `@/core`, `@/cli`, `@/types`, `@/enums`, `@/__tests__`

## Running Tests

```bash
# Run all tests
npm test

# Watch mode
npm test -- --watch

# Coverage report
npm test -- --coverage

# Run specific test file
npm test -- hashCalculator.test.ts
```

## Test Maturity Levels

**Level 1 (In Progress):**
- ✅ Test infrastructure setup (complete)
- 🔄 Core unit tests (hashCalculator ✅, metadataExtractor pending, fileScanner pending)
- Target: ~70% code coverage

**Level 2 (After Database Layer):**
- Database CRUD operations, duplicate detection, query correctness
- Target: ~80% code coverage

**Level 3 (Before Open Source):**
- Cross-platform CI (Windows, Mac, Linux), performance benchmarks, edge cases
- Target: ~90% code coverage

## Success Criteria (Phase 1)

1. ✅ Run in < 5 seconds
2. 🔄 Pass on all platforms (macOS ✅, Linux/Windows via CI pending)
3. 🔄 Cover critical paths: hashing ✅, metadata extraction (pending), scanning (pending)
4. ✅ Provide confidence to refactor without breaking functionality
5. ✅ Be easy to maintain and extend

## What NOT to Test (Yet)

- CLI output formatting (low value, high maintenance)
- Performance benchmarks (Phase 5)
- HTML report generation (Phase 5)
- LLM/semantic features (Phase 6)
