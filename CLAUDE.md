# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

SheldonFS is a cross-platform file organization and management system designed to help users analyze, organize, and manage files across multiple storage systems (Windows, Ubuntu, macOS, and Google Drive). The goal is to identify duplicates, create a robust naming convention, and establish a scalable file organization strategy.

## Current Status (Phase 1 - In Progress)

✅ **Completed:**
- File scanner module with hash calculation, metadata extraction, MIME type detection
- CLI interface for testing scans
- TypeScript build setup with ESM modules
- ESLint + Prettier integration
- Production bundling with tsdown
- Path aliases for clean imports

🚧 **Next Steps:**
1. **Test on real directories** (Documents, Downloads folders)
2. **Add basic tests** with vitest to ensure reliability
3. **Build database layer** for storing scan results
4. **Implement duplicate detection** using hash comparisons

## Development Phases

### Phase 1: Discovery Tool (Current)
Build TypeScript/Node.js file system scanner with:
- Recursive directory traversal across local file systems
- Google Drive API integration
- File metadata extraction (size, dates, permissions, MIME types)
- Hash-based duplicate detection (MD5/SHA256)
- SQLite database for file inventory
- Progress tracking and logging
- Report generation (console, JSON, CSV, HTML)
- CLI interface for scanning and reporting

### Phase 2: Organization Strategy
Develop naming conventions and folder hierarchy based on Phase 1 analysis.

### Phase 3: Web UI & API
- Express backend API
- React + TypeScript frontend
- Localhost web interface for reviewing duplicates and managing files
- Reuses core logic from Phase 1

### Phase 4: Desktop Application
- Electron-based desktop app
- Polished user experience for non-technical users

## Immediate Next Steps (Start of Next Session)

### 1. Real-World Testing
Test the file scanner on actual directories:
- Run scans on Documents folder
- Run scans on Downloads folder
- Monitor performance, memory usage, and errors
- Verify hash calculation accuracy
- Test with various file types (images, videos, documents, etc.)

### 2. Add Basic Tests
Set up vitest and create initial test suite:
- Unit tests for hash calculator
- Unit tests for metadata extractor
- Integration tests for file scanner
- Test with mock file structures
- Test error handling scenarios

### 3. After Testing is Stable
- Begin database layer implementation (better-sqlite3)
- Design and implement schema for storing file metadata
- Build duplicate detection using hash comparisons
- Add reporting functionality

## Future Database Schema (To Be Implemented)

When the database layer is built, it will use better-sqlite3 with the following tables:

```sql
-- Core file information
files: id, path, name, extension, hash, size, mime_type, category,
       source_system, created_at, modified_at, accessed_at,
       is_symlink, symlink_target, permissions, owner

-- Duplicate relationships
duplicates: id, hash, file_count, total_size, wasted_space

-- Scan history
scan_sessions: id, path, source_system, started_at, completed_at,
               total_files, total_size, errors_count, duration_ms

-- Cached statistics
statistics: id, session_id, metric_type, value, metadata
```

### Tech Stack

**Core Dependencies:**
- **TypeScript** - Type-safe development
- **Node.js** - Runtime environment
- **fast-glob** - High-performance file pattern matching
- **file-type** - MIME type detection

**Future (Phase 1+):**
- **better-sqlite3** - Fast synchronous SQLite database (to be added)
- **@googleapis/drive** - Official Google Drive API client (to be added)
- **winston** - Structured logging (to be added)

**Phase 3+:**
- **Express** - Web server for API and serving React app
- **React** - Frontend UI
- **Vite** - Frontend build tool
- **TanStack Query** - Data fetching
- **Tailwind CSS** - Styling

**Development Tools:**
- **tsx** - TypeScript execution for development
- **tsdown** - Production bundling (Rolldown-based, replaces deprecated tsup)
- **vitest** - Testing framework
- **eslint** - Linting with flat config
- **prettier** - Code formatting (integrated via eslint-plugin-prettier)

## Development Commands

```bash
# Development
npm run dev [path]          # Run CLI with tsx (fast, no build)

# Production
npm run bundle              # Bundle with tsdown
npm start [path]            # Bundle and run with node

# Code Quality
npm run lint                # Lint and check formatting
npm run lint:fix            # Fix linting and formatting issues
npm run build               # TypeScript type checking only

# Testing
npm test                    # Run tests with vitest
```

## Key Technical Decisions

### Language & Build System
- **TypeScript over Python** - Better for full-stack (React + Node.js), single language, excellent tooling
- **ESM Modules** - `"type": "module"` in package.json
- **moduleResolution: "bundler"** - Clean imports without `.js` extensions or `/index`
- **tsdown for bundling** - Modern Rolldown-based bundler (tsup successor)
- **tsx for development** - Fast TypeScript execution without build step

### Code Organization
- **Separation of enums and types** - `enums.ts` (runtime) vs `types.ts` (compile-time)
- **Path aliases** - `@/core/*`, `@/types`, `@/enums` for cleaner imports
- **ES6 arrow functions** - Preferred for all standalone functions
- **Interface-agnostic core** - Business logic separated from CLI/API layers

### TypeScript Configuration
- `allowJs: false` - Explicit TypeScript-only codebase
- `strict: true` - Maximum type safety
- `noUncheckedIndexedAccess: true` - Extra safety for array/object access
- `verbatimModuleSyntax: true` - Predictable import/export behavior

### Linting & Formatting
- **ESLint flat config** - Modern ESLint 9 configuration
- **eslint-plugin-prettier** - Prettier runs through ESLint (unified interface)
- Single command workflow: `npm run lint:fix` handles both

### File Structure
```
src/
├── core/
│   ├── enums.ts              # Runtime enums (SourceSystem, FileCategory)
│   ├── types.ts              # TypeScript types and interfaces
│   └── scanner/
│       ├── index.ts          # Module exports
│       ├── fileScanner.ts    # Main scanner class
│       ├── metadataExtractor.ts  # File metadata extraction
│       └── hashCalculator.ts     # SHA256 hashing with optimization
├── cli/
│   └── index.ts              # CLI interface
└── (future: database/, duplicates/, googleDrive/, reports/)
```

## Design Principles

- **Cross-platform compatibility** - Handle Windows, Unix, and cloud paths uniformly
- **Performance optimization** - Partial hashing for large files, batch processing, memory management
- **Safety first** - Never modify files in Phase 1, extensive logging, error recovery
- **Scalability** - Handle large file systems efficiently with indexing and caching
- **User experience** - Clear progress indication, human-readable output

## Implementation Details

### File Scanner (`src/core/scanner/`)

**Hash Calculator** (`hashCalculator.ts`):
- `calculateFileHash()` - Full SHA256 hash using streams
- `calculatePartialHash()` - First 100KB for quick comparison
- `quickCompare()` - Size + partial hash check before full hash

**Metadata Extractor** (`metadataExtractor.ts`):
- File properties: size, dates (created/modified/accessed), permissions
- MIME type detection via `file-type` library
- Automatic categorization (document, image, video, etc.)
- Symlink detection and target resolution
- 100+ file extensions mapped to categories

**File Scanner** (`fileScanner.ts`):
- Recursive traversal using `fast-glob`
- Default exclusions: node_modules, .git, caches, temp folders
- Configurable: hidden files, symlink following, max hash size
- Progress callback support
- Comprehensive error handling with detailed error reporting
- Scan statistics and summaries
