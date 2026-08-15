---
name: session-wrapup
description: Complete end-of-session workflow for SheldonFS project. Handles code commits, quality checks, documentation updates, and merge process. Use when finishing a development session or feature branch.
allowed-tools: Bash, Read, Edit, Write, Grep, Glob
---

# SheldonFS Session Wrap-Up

End-to-end workflow for wrapping up development sessions in the SheldonFS project. Handles both code repository (SheldonFS/) and documentation repository (sheldon-fs/) with proper quality gates and merge workflows.

## When to Use This Skill

- User says "wrap up", "finish session", or "end session"
- Completing work on a feature branch
- Before switching to a different task or ending development work

**Important:** Always run Phase 1 (code repository check) unless user explicitly states there were no code changes in this session.

## Project Structure

**Two separate Git repositories with different workflows:**

```
sheldon-fs/                    # Documentation repository
├── .git/                      # Git repo for docs/planning
├── CLAUDE.md
└── SheldonFS/                 # Code repository (nested)
    ├── .git/                  # Separate git repo
    ├── src/
    └── package.json
```

## Commit Message Reference

### Code Repository (SheldonFS/)

**Format:** Semantic commits
```bash
<type>: Brief description

- Optional bullet points
- For additional context
```

**Types:** `feat`, `fix`, `refactor`, `docs`, `test`, `chore`

**Examples:**
```
feat: Add database layer with Drizzle ORM
```
```
fix: Resolve linting errors in metadataExtractor
```

### Documentation Repository (sheldon-fs/)

**On feature branch:** Simple, brief checkpoint messages
```bash
git commit -m "docs: Update CLAUDE.md with database design"
```

**When merging to main:** Comprehensive session summary (squash merge)
```bash
git commit -m "session-X: Brief description of session

- Detailed accomplishment 1
- Detailed accomplishment 2
- Key decisions made"
```

**Session number:** Increment from last "session-X" commit in git log.

## Main Workflow

### Phase 1: Code Repository (SheldonFS/)

**Skip this phase ONLY if user explicitly stated no code changes occurred.**

#### 1.1 Navigate and Verify

```bash
cd SheldonFS
pwd               # Confirm: /path/to/sheldon-fs/SheldonFS
git status        # Check for uncommitted changes
git branch        # Identify current branch
```

#### 1.2 Commit Any Uncommitted Changes

If `git status` shows changes:

```bash
git diff          # Review changes
git add .
git commit -m "feat/fix/refactor: Brief description

- Additional context if needed"
```

Verify clean state:
```bash
git status        # Should show "nothing to commit"
```

#### 1.3 Quality Check: Linting

```bash
npm run lint
```

**If errors found:**
```bash
npm run lint:fix  # Auto-fix what's possible
npm run lint      # Verify all fixed
```

**If manual fixes needed:**
- Fix the issues
- Verify with `npm run lint`

#### 1.4 Commit Lint Fixes (if any were made)

```bash
git add .
git commit -m "fix: Resolve linting errors"
```

#### 1.5 Quality Check: Type Checking

```bash
npm run typecheck
```

**If errors found:**
- Fix the type errors
- Verify with `npm run typecheck`

#### 1.6 Commit Typecheck Fixes (if any were made)

```bash
git add .
git commit -m "fix: Resolve type errors"
```

#### 1.7 Quality Check: Tests

```bash
npm test
```

**If tests FAIL:**
- **DO NOT attempt to fix them**
- Make note of failures for Phase 2 documentation
- This will prevent merge in step 1.9

#### 1.8 Verify Clean State

```bash
git status        # Must show "nothing to commit"
```

**Critical:** If uncommitted changes exist, diagnose why and commit them before proceeding.

#### 1.9 Review Overall Changes

Understand what changed in this feature branch:

```bash
git log --oneline main..HEAD       # List commits on feature branch
git diff main...HEAD --stat        # Summary of file changes
git diff main...HEAD               # Full diff (review key changes)
```

**Purpose:** This context is essential for updating CLAUDE.md accurately in Phase 2.

#### 1.10 Conditional Merge to Main

**Decision point:** Did tests pass in step 1.7?

**✅ If all quality checks passed (including tests):**

```bash
git checkout main
git merge <feature-branch-name>    # Normal merge (NOT squashed)
git branch                          # Verify both branches exist
```

**Current branch:** main (feature branch still exists but not checked out)

**❌ If tests failed:**

**Do not merge.** Keep feature branch checked out for fixes in next session.

**Current branch:** Feature branch (stays checked out)

---

### Phase 2: Documentation Repository (sheldon-fs/)

#### 2.1 Navigate and Verify

```bash
cd ..             # Move to parent directory
pwd               # Confirm: /path/to/sheldon-fs
git status
git branch        # Verify current branch
```

#### 2.2 Commit Any Uncommitted Changes (if applicable)

If `git status` shows uncommitted changes:

```bash
git add .
git commit -m "docs: Brief description of changes"
```

#### 2.3 Update CLAUDE.md

Based on code changes from Phase 1 (step 1.9) and conversation context:

**Current Status Section:**
- Move completed items from "🚧 Next Steps" to "✅ Completed"
- Add new accomplishments with details
- Update "🚧 Next Steps" with remaining work

**CRITICAL - If tests failed in Phase 1:**
Add as the **FIRST item** in "Immediate Next Steps":
```markdown
## Immediate Next Steps (Start of Next Session)

### 1. Fix Failing Tests
Address test failures from previous session:
- [Describe which tests failed]
- [Any context about why they might be failing]

### 2. [Previous first priority moves to second]
...
```

**Tech Stack Section:**
- Add any new dependencies installed
- Update phase assignments if libraries moved from planned to implemented

**Other Sections:**
- Update any other relevant sections based on work done

#### 2.4 Commit Documentation Updates

```bash
git status
git add CLAUDE.md [other-changed-docs]
git commit -m "docs: Update CLAUDE.md with session accomplishments"
```

Use simple messages - comprehensive session message comes in Phase 3.

#### 2.5 Verify Clean State

```bash
git status        # Must show "nothing to commit"
```

---

### Phase 3: Merge Documentation to Main

#### 3.1 Determine Session Number

```bash
git checkout main
git log --oneline --grep="session-" -3    # Find last session-X commit
```

**New session number** = last session number + 1

#### 3.2 Squash Merge Feature Branch

```bash
git merge --squash <feature-branch-name>
```

This combines all feature branch commits into staged changes ready for one comprehensive commit.

#### 3.3 Create Session Commit

```bash
git commit -m "session-X: Brief description of session accomplishments

- Detailed bullet point 1
- Detailed bullet point 2
- Key decisions made
- Technologies added
- Documentation created"
```

**Guidelines for session message:**
- First line: Concise summary of the session's main achievement
- Bullets: Specific accomplishments, decisions, additions
- Focus on "what" not "how" (high-level outcomes)

**Example:**
```
session-8: Implement database layer with Drizzle ORM

- Created database schema with 5 core tables
- Implemented repository pattern for data access
- Added database integration tests (67 tests, 74% coverage)
- Configured better-sqlite3 with type-safe Drizzle ORM
- Updated CLAUDE.md with database implementation status
```

#### 3.4 Keep Feature Branch Intact

```bash
git branch        # Verify both main and feature branch exist
```

**Do NOT delete the feature branch.** It remains for potential corrections or reference.

#### 3.5 Verify Main Branch State

```bash
git log --oneline -2     # Confirm session commit is latest
git status               # Should be clean
```

---

### Phase 4: Summary Report

Provide structured summary to the user:

```markdown
## Session Wrap-Up Complete

### Code Repository (SheldonFS/)
- **Branch:** <feature-branch-name>
- **Status:** [Merged to main | Kept open for test fixes]
- **Quality checks:**
  - Linting: ✅ Pass [✅/❌ Fixed]
  - Type checking: ✅ Pass [✅/❌ Fixed]
  - Tests: ✅ Pass [❌ FAILED - noted for next session]
- **Commits added:** [List semantic commits]

### Documentation Repository (sheldon-fs/)
- **Branch:** <feature-branch-name>
- **Session commit:** session-X: <title>
- **Status:** Merged to main (feature branch kept intact)
- **Updates:**
  - CLAUDE.md sections updated: [list sections]
  - [Any other documentation changes]
  - [Test failures added as first priority in Next Steps - if applicable]

### Next Session Priority
[If tests failed: First priority is fixing failing tests]
[Otherwise: Next logical task from Immediate Next Steps]

### Notes
[Any issues encountered during wrap-up]
[Any follow-up actions needed]
```

---

## Examples

### Example 1: Successful Session with All Checks Passing

**User:** "Wrap up the feat/add-database-layer branch"

**Process:**
1. Navigate to SheldonFS/ → On feat/add-database-layer
2. Uncommitted changes found → Commit with semantic message
3. npm run lint → Passes
4. npm run typecheck → Passes
5. npm test → ✅ All pass
6. Review changes: `git diff main...HEAD`
7. Merge to main (tests passed)
8. Navigate to sheldon-fs/ → On feat/database-integration
9. Update CLAUDE.md (move "Build database layer" to Completed, update Next Steps)
10. Commit: "docs: Update CLAUDE.md with database implementation"
11. Determine session number: 7 → Next is 8
12. Squash merge feature branch
13. Create session commit: "session-8: Implement database layer with Drizzle ORM..."
14. Provide summary to user

**Outcome:** Both repos on main branch, feature branches kept intact, all quality checks passed.

---

### Example 2: Session with Test Failures

**User:** "Wrap up feat/duplicate-detection"

**Process:**
1. Navigate to SheldonFS/ → On feat/duplicate-detection
2. Uncommitted changes → Commit with semantic message
3. npm run lint → ❌ 3 errors found
4. npm run lint:fix → Fixes applied
5. Verify → npm run lint passes
6. Commit: "fix: Resolve linting errors"
7. npm run typecheck → ❌ 2 type errors
8. Fix type errors manually
9. Verify → npm run typecheck passes
10. Commit: "fix: Resolve type errors"
11. npm test → ❌ 2 tests fail
12. **Do not fix** - Note for documentation
13. Review changes: `git diff main...HEAD`
14. **Do not merge** (tests failed) - Stay on feature branch
15. Navigate to sheldon-fs/ → On feat/duplicate-detection-docs
16. Update CLAUDE.md:
    - Add completed work to "✅ Completed"
    - **Add test failures as FIRST item in "Immediate Next Steps":**
      ```markdown
      ### 1. Fix Failing Tests
      Address test failures in duplicate detection:
      - duplicateDetector.test.ts: 2 tests failing
      - Tests related to hard link vs duplicate distinction
      ```
17. Commit: "docs: Update CLAUDE.md with duplicate detection progress"
18. Session 8 → Create session-9 commit
19. Squash merge docs feature branch
20. Create session commit: "session-9: Implement duplicate detection with pending test fixes..."
21. Provide summary noting test failures as first priority

**Outcome:**
- SheldonFS/ still on feature branch (needs test fixes)
- sheldon-fs/ merged to main
- Test failures documented as first priority for next session

---

## Quick Reference

### Success Criteria
- ✅ All code changes committed with semantic messages (SheldonFS/)
- ✅ Linting passes or fixes committed separately
- ✅ Type checking passes or fixes committed separately
- ✅ Tests run (failures documented, not fixed)
- ✅ Code changes reviewed with git diff
- ✅ Code merged to main if all checks passed, otherwise kept on feature branch
- ✅ CLAUDE.md accurately updated with session accomplishments
- ✅ Test failures (if any) added as FIRST item in Immediate Next Steps
- ✅ Documentation squash-merged to main with session-X commit
- ✅ Both feature branches kept intact (never deleted)
- ✅ Clear summary provided to user

### Key Reminders
- **Two repositories, two workflows:** SheldonFS/ uses normal merge, sheldon-fs/ uses squash merge
- **Separate commits for fixes:** Lint fixes and typecheck fixes get individual commits
- **Test failures are not fixed:** Document them as first priority instead
- **Never delete feature branches:** Keep them intact for corrections
- **Session numbers are sequential:** Always verify last session number
- **Git diff before Phase 2:** Essential context for documentation updates
- **Phase 1 runs by default:** Only skip if user explicitly stated no code changes

### Anti-Patterns
- ❌ Using "session-X" message on feature branch (only on main after squash)
- ❌ Deleting feature branches after merge
- ❌ Normal merge instead of squash for sheldon-fs/
- ❌ Squash merge instead of normal for SheldonFS/
- ❌ Combining lint and typecheck fixes into one commit
- ❌ Attempting to fix test failures during wrap-up
- ❌ Skipping git diff in Phase 1
- ❌ Forgetting to push test failures to top of Next Steps

---

## Troubleshooting

**Problem:** Can't find feature branch
**Solution:** `git branch -a` to list all branches, ask user to confirm branch name

**Problem:** Merge conflicts during merge
**Solution:** `git merge --abort`, notify user, ask for guidance

**Problem:** No package.json found
**Solution:** Verify in SheldonFS/ directory with `pwd`

**Problem:** Large uncommitted changes with unclear purpose
**Solution:** Ask user for guidance on commit message before committing

**Problem:** Unsure which CLAUDE.md sections need updates
**Solution:** Focus on "Current Status" and "Immediate Next Steps" at minimum, review conversation for other impacts

**Problem:** Can't determine session number
**Solution:** `git log --oneline | grep session-` to find all session commits, increment from highest

---

## Related Documentation

- Full commit guidelines: `CLAUDE.md` > "Commit Message Guidelines"
- Project phases: `CLAUDE.md` > "Development Phases"
- Current status: `CLAUDE.md` > "Current Status"
