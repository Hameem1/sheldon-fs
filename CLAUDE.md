# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

SheldonFS is a cross-platform file organization and management system designed to help users analyze, organize, and manage files across multiple storage systems (Windows, Ubuntu, macOS, and Google Drive). The goal is to identify duplicates, create a robust naming convention, and establish a scalable file organization strategy.

## Current Status (Phase 1 - In Progress)

✅ **Completed:**
- File scanner module with SHA256 hash calculation
- Comprehensive metadata extraction (23 fields per file):
  - Basic: path, name, extension, size, hash, mimeType, category, sourceSystem
  - Timestamps: createdAt, modifiedAt, accessedAt
  - Permissions: permissions, owner (with UID→username caching), isSymlink, symlinkTarget
  - Organization: isHidden, depth, finderTags, finderColor
  - Advanced: inode, hardLinkCount, isExecutable
- MIME type detection with binary + extension fallback (file-type + mime packages)
- CLI interface for testing scans
- TypeScript build setup with ESM modules
- ESLint + Prettier integration
- Production bundling with tsdown (output to build/)
- Path aliases for clean imports
- Real-world testing (1,300+ files, 37GB, ~30s scan time)

🚧 **Next Steps:**
1. **Add basic tests** with vitest to ensure reliability
2. **Build database layer** for storing scan results (better-sqlite3)
3. **Implement duplicate detection** using hash comparisons
4. **Add reporting functionality** (JSON, CSV output)

## Development Phases

### Phase 1: Discovery Tool (Current)
Build TypeScript/Node.js file system scanner with:
- **Recursive directory traversal** across local file systems (fast-glob)
- **Comprehensive metadata extraction** (23 fields: size, dates, permissions, MIME types, ownership, inodes, etc.)
- **Hash-based file identification** (SHA256 for content-addressable storage)
- **Test infrastructure setup** with vitest and comprehensive test coverage
- **SQLite database** for file inventory and scan results
- **Duplicate detection** using hash comparisons and inode tracking
- **Basic reporting** (console output, JSON, CSV exports)
- **CLI interface** for scanning and reporting
- **Cross-platform compatibility** verified on Windows, macOS, and Linux
- **Google Drive API integration** (future - for cloud file scanning)

### Phase 2: Organization Strategy & Analysis
Analyze scan results from Phase 1 to develop intelligent organization strategies:
- **Pattern analysis** - Identify common naming patterns, folder structures, and file organization habits
- **Naming convention design** - Create robust, consistent naming rules based on discovered patterns
- **Folder hierarchy design** - Establish scalable directory structure recommendations
- **Category optimization** - Refine auto-categorization rules based on real-world data
- **Duplicate analysis** - Study duplicate patterns to inform cleanup strategies
- **Documentation** - Create organization guidelines and best practices for users

### Phase 3: Open-Source Preparation
Prepare the project for open-source development without adding new functionality:
- **Repository consolidation** - Refactor from two-repo structure to single monorepo with clean organization
- **CI/CD pipelines** - GitHub Actions for automated testing, linting, and builds
- **Installation & distribution** - npm package publishing, easy installation scripts
- **Developer documentation** - CONTRIBUTING.md, architecture documentation, setup guides
- **Code quality** - Enhanced error handling, comprehensive logging, code coverage targets
- **Changelog management** - Automated changelog generation and versioning strategy
- **Cross-platform testing** - Verify compatibility on Windows, macOS, and Linux
- **License selection** - Choose and apply appropriate open-source license

### Phase 4: Web UI & API
- Express backend API
- React + TypeScript frontend
- Localhost web interface for reviewing duplicates and managing files
- Reuses core logic from Phase 1

### Phase 5: Polish & Optimization
Post-web-app enhancements and production-readiness improvements:
- **HTML report generation** for visual, shareable analysis
- **Database migration system** for schema evolution (when multi-user deployment needed)
- **Incremental scans** - Only scan changed/new files instead of full rescans
- **Performance optimization** - Improve scan speed, memory usage, query performance
- **Enhanced CLI features** - Interactive mode, better progress indicators, configuration management

### Phase 6: Semantic Intelligence
**LLM-powered content understanding for intelligent file organization** - [Full Decision Record](./decision-records/phase-6/semantic-file-embeddings.md)

- **Local embedding generation** via Ollama + nomic-embed-text (768-dimensional vectors)
- **Vector storage** using sqlite-vec extension (fast, local, privacy-preserving)
- **Content-based grouping** - Cluster files by semantic similarity regardless of naming
- **Semantic duplicate detection** - Find files with same content in different formats
- **Natural language queries** - "Find all tax documents from last year"
- **Smart auto-categorization** - Understand file content for better organization
- **Organization suggestions** - Recommend where new files should go based on content

### Phase 7: Open-Source Release
Actually release the project as open-source and establish community foundations:
- **Public repository** - Move from private to public GitHub repository
- **Community guidelines** - CODE_OF_CONDUCT.md, issue templates, pull request templates
- **Public documentation** - Comprehensive README, user guides, FAQ
- **Release announcement** - Blog post, social media, relevant communities (Reddit, HackerNews)
- **Contributor onboarding** - First-time contributor guides, good first issues
- **Community channels** - Discord/Slack community, GitHub Discussions
- **Project governance** - Decision-making process, roadmap transparency
- **Support infrastructure** - Issue triage workflow, response time expectations

### Phase 8: Desktop Application
- **Electron-based desktop app** for non-technical users
- **Polished GUI** with drag-and-drop, visual file management
- **System tray integration** for background monitoring
- **Cross-platform installers** (Windows .exe, macOS .dmg, Linux .AppImage)
- **Auto-update system** for seamless version upgrades

## Immediate Next Steps (Start of Next Session)

### 1. Add Basic Tests
Set up vitest and create initial test suite:
- Unit tests for hash calculator
- Unit tests for metadata extractor
- Integration tests for file scanner
- Test with mock file structures
- Test error handling scenarios

### 2. Database Layer Implementation
- Install and configure better-sqlite3
- Design and implement schema for storing file metadata (all 23 fields)
- Build data access layer with type-safe queries
- Note: No migration system needed yet (local-only, can rescan if schema changes)

### 3. Duplicate Detection
- Query files by hash to identify duplicates
- Distinguish between true duplicates vs. hard links (same inode)
- Calculate wasted space from duplicates
- Group duplicates for user review

### 4. Basic Reporting
- JSON export for programmatic access
- CSV export for spreadsheet analysis
- Summary statistics (file counts by category, largest files, duplicate groups)

## Future Database Schema (To Be Implemented)

When the database layer is built, it will use better-sqlite3 with the following tables:

```sql
-- Core file information (all 23 metadata fields)
files: id, path, name, extension, hash, size, mime_type, category,
       source_system, created_at, modified_at, accessed_at,
       is_symlink, symlink_target, permissions, owner,
       is_hidden, depth, finder_tags, finder_color,
       inode, hard_link_count, is_executable

-- Duplicate relationships
duplicates: id, hash, file_count, total_size, wasted_space,
            is_hard_link_group

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

**Phase 1-2:**
- **fast-glob** - High-performance file pattern matching
- **file-type** - Binary MIME type detection (magic numbers)
- **mime** - Extension-based MIME type fallback (800+ types)
- **better-sqlite3** - Fast synchronous SQLite database
- **@googleapis/drive** - Official Google Drive API client (cloud file scanning)
- **winston** - Structured logging for CLI

**Phase 3:**
- **GitHub Actions** - CI/CD automation
- **semantic-release** - Automated versioning and changelog
- **pkg** or **@vercel/ncc** - Standalone executable bundling
- **cross-env** - Cross-platform environment variables

**Phase 4:**
- **Express** - Web server for API and serving React app
- **React** - Frontend UI
- **Vite** - Frontend build tool
- **TanStack Query** - Data fetching
- **Tailwind CSS** - Styling

**Phase 6:**
- **Ollama** - Local LLM runtime for embedding generation
- **nomic-embed-text** - 768-dimensional text embedding model
- **sqlite-vec** - SQLite extension for vector storage and similarity search

**Phase 8:**
- **Electron** - Cross-platform desktop application framework
- **electron-builder** - Package and distribute native installers

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
npm run bundle              # Bundle with tsdown to build/
npm start [path]            # Bundle and run with node

# Code Quality
npm run typecheck           # Fast type checking (no emit)
npm run build               # Build TypeScript to build/ (with declarations)
npm run lint                # Lint and check formatting
npm run lint:fix            # Fix linting and formatting issues

# Testing
npm test                    # Run tests with vitest

# Maintenance
npm run nuke                # Remove node_modules, build, package-lock.json
npm run clean-install       # Nuke + fresh npm install
```

## Commit Message Guidelines

### For SheldonFS/ (Code Repository)

Follow conventional commits format:

```
<type>: Brief description of what the changes do

[Optional: Additional bullet points to elaborate the changes]
```

**Types:**
- `feat`: New feature or functionality
- `fix`: Bug fix
- `refactor`: Code restructuring without changing behavior
- `docs`: Documentation changes only
- `test`: Adding or updating tests
- `chore`: Build process, tooling, dependencies

**Examples:**
```
feat: Add owner field extraction with UID caching

- Implement getOwnerName() function with username lookup
- Add UID→username cache for performance (300-600x speedup)
- Update metadata object to include owner field
```

```
fix: Correct eslint ignore pattern for build directory

- Changed dist/** to build/** in eslint.config.mjs
```

### For sheldon-fs/ (Parent Repository)

Session-based commits for documentation and project-level changes:

```
session-{n}: Brief description of what was done in this session

- Bullet point elaborating changes
- Another bullet point
- Additional context
```

**Session number:** Increment from the last session commit in git history.

**Example:**
```
session-2: Enhanced metadata extraction and project documentation

- Expanded file metadata to 23 comprehensive fields
- Added MIME type extension fallback using mime package
- Restructured development phases in CLAUDE.md
- Established decision-records system
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
Extracts **23 comprehensive metadata fields** per file:

- **Basic Info**: path, name, normalized extension, size, SHA256 hash
- **MIME & Category**: Binary detection (file-type) with extension fallback (mime), auto-categorization (100+ extensions)
- **Timestamps**: created, modified, accessed (milliseconds since epoch)
- **Permissions & Ownership**: Octal permissions, owner username (with UID→username caching for performance)
- **Links**: Symlink detection with target resolution, inode tracking, hard link count
- **Organization**: Hidden file detection (dot-files on Unix, TODO: Windows attributes), depth from scan root
- **macOS Features**: Finder tags, Finder color labels (via xattr)
- **Security**: Executable detection (permission bits on Unix, extensions on Windows)

**Performance optimizations:**
- Owner lookup caching (300-600x speedup on large scans)
- Early bailout for non-macOS systems (Finder tags)
- Streaming hash calculation for large files

**File Scanner** (`fileScanner.ts`):
- Recursive traversal using `fast-glob`
- Default exclusions: node_modules, .git, caches, temp folders
- Configurable: hidden files, symlink following, max hash size
- Progress callback support
- Comprehensive error handling with detailed error reporting
- Scan statistics and summaries

## Documentation & Resources

### Decision Records

Important architectural and feature decisions are documented in `/decision-records/` organized by phase. These records provide deep-dive context, research, and rationale for major design choices.

**See:** [Decision Records README](./decision-records/README.md) for the complete list and documentation structure.
