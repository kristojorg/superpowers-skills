---
name: Using Git Worktrees
description: Create isolated git worktrees with smart directory selection and safety verification
when_to_use: When starting feature implementation in isolation. When brainstorming transitions to code. When need separate workspace without branch switching. Before executing implementation plans.
version: 1.0.0
---

# Using Git Worktrees

## Overview

Git worktrees create isolated workspaces sharing the same repository, allowing work on multiple branches simultaneously without switching.

**Core principle:** Systematic directory selection + safety verification = reliable isolation.

**Announce at start:** "I'm using the Using Git Worktrees skill to set up an isolated workspace."

## Directory Selection Process

**Default location:** `../<dir-name>.worktrees/`

Where `<dir-name>` is the project directory's name (e.g., `d/radial` → `d/radial.worktrees/`)

### 1. Detect If Already In Worktree

```bash
# Check if current directory matches *.worktrees pattern
if [[ "$PWD" == *".worktrees/"* ]] || [[ "$(basename "$PWD")" == *.worktrees ]]; then
  # Already in worktree - use current directory as worktree_dir
  if [[ "$PWD" == *".worktrees/"* ]]; then
    # In a branch subdirectory like d/radial.worktrees/feature-x
    worktree_dir=$(dirname "$PWD")
  else
    # In the .worktrees directory itself
    worktree_dir="$PWD"
  fi
else
  # In main project - calculate worktree_dir
  project=$(basename "$PWD")
  worktree_dir="../${project}.worktrees"
fi
```

### 2. Summary

**If in main project:** Use `../<project-name>.worktrees/`

**If already in worktree:** Reuse same worktree directory for new branch

## Safety Verification

**Not required** - worktree directory is outside project (sibling to project root).

No .gitignore verification needed.

## Creation Steps

### 1. Detect Worktree Directory Path

```bash
# Check if current directory matches *.worktrees pattern
if [[ "$PWD" == *".worktrees/"* ]] || [[ "$(basename "$PWD")" == *.worktrees ]]; then
  # Already in worktree - use current directory as worktree_dir
  if [[ "$PWD" == *".worktrees/"* ]]; then
    # In a branch subdirectory like d/radial.worktrees/feature-x
    worktree_dir=$(dirname "$PWD")
  else
    # In the .worktrees directory itself
    worktree_dir="$PWD"
  fi
else
  # In main project - calculate worktree_dir
  project=$(basename "$PWD")
  worktree_dir="../${project}.worktrees"
fi
```

### 2. Create Worktree

```bash
# Capture current location before creating worktree
WORKTREE_CREATED_FROM="$PWD"

# Get base repository path (the main repo, not a worktree)
BASE_REPO_PATH=$(git rev-parse --show-toplevel)

# Get current branch name as base branch
BASE_BRANCH=$(git rev-parse --abbrev-ref HEAD)

# Determine full path for this branch
WORKTREE_DIR="${worktree_dir}/${BRANCH_NAME}"
WORKTREE_NAME="$BRANCH_NAME"

# Create worktree with new branch
git worktree add "$WORKTREE_DIR" -b "$BRANCH_NAME"
cd "$WORKTREE_DIR"
```

### 3. Run Custom Setup Script (if exists)

Check for project-specific worktree setup script:

```bash
# Check if custom setup script exists in base repo
if [ -f "${BASE_REPO_PATH}/scripts/worktree-setup.ts" ]; then
  # Run with bun, passing environment variables
  WORKTREE_DIR="$WORKTREE_DIR" \
  WORKTREE_NAME="$WORKTREE_NAME" \
  BASE_BRANCH="$BASE_BRANCH" \
  BASE_REPO_PATH="$BASE_REPO_PATH" \
  WORKTREE_CREATED_FROM="$WORKTREE_CREATED_FROM" \
  bun "${BASE_REPO_PATH}/scripts/setup-worktree.ts"
fi
```

**Available environment variables for script:**
- `WORKTREE_DIR` - Absolute path to new worktree
- `WORKTREE_NAME` - Branch name / worktree name
- `BASE_BRANCH` - Branch the worktree was created from
- `BASE_REPO_PATH` - Absolute path to main repository
- `WORKTREE_CREATED_FROM` - Directory where creation was initiated

### 4. Run Project Setup

Auto-detect and run appropriate setup:

```bash
# Node.js
if [ -f package.json ]; then npm install; fi

# Rust
if [ -f Cargo.toml ]; then cargo build; fi

# Python
if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
if [ -f pyproject.toml ]; then poetry install; fi

# Go
if [ -f go.mod ]; then go mod download; fi
```

### 5. Verify Clean Baseline

Run tests to ensure worktree starts clean:

```bash
# Examples - use project-appropriate command
npm test
cargo test
pytest
go test ./...
```

**If tests fail:** Report failures, ask whether to proceed or investigate.

**If tests pass:** Report ready.

### 6. Report Location

```
Worktree ready at <full-path>
Tests passing (<N> tests, 0 failures)
Ready to implement <feature-name>
```

## Quick Reference

| Situation | Action |
|-----------|--------|
| In main project `d/radial` | Use `d/radial.worktrees/${branch}` |
| In worktree `d/radial.worktrees/feature-x` | Use `d/radial.worktrees/${branch}` |
| In worktree dir `d/radial.worktrees` | Use `d/radial.worktrees/${branch}` |
| Custom setup script exists | Run `scripts/worktree-setup.ts` with env vars |
| Tests fail during baseline | Report failures + ask |
| No package.json/Cargo.toml | Skip dependency install |

## Common Mistakes

**Not detecting worktree context**
- **Problem:** Creates nested or misplaced worktree directories
- **Fix:** Always check if PWD contains `.worktrees` pattern first

**Proceeding with failing tests**
- **Problem:** Can't distinguish new bugs from pre-existing issues
- **Fix:** Report failures, get explicit permission to proceed

**Hardcoding setup commands**
- **Problem:** Breaks on projects using different tools
- **Fix:** Auto-detect from project files (package.json, etc.)

## Example Workflows

### From Main Project

```
You: I'm using the Using Git Worktrees skill to set up an isolated workspace.

[Detect: in main project d/radial]
[Calculate worktree_dir: d/radial.worktrees]
[Create worktree: git worktree add ../radial.worktrees/auth -b feature/auth]
[Run scripts/worktree-setup.ts with environment variables]
[Run npm install]
[Run npm test - 47 passing]

Worktree ready at /Users/kristo/d/radial.worktrees/auth
Tests passing (47 tests, 0 failures)
Ready to implement auth feature
```

### From Existing Worktree

```
You: I'm using the Using Git Worktrees skill to set up an isolated workspace.

[Detect: already in worktree d/radial.worktrees/feature-x]
[Use same worktree_dir: d/radial.worktrees]
[Create worktree: git worktree add ../radial.worktrees/bugfix -b bugfix/auth]
[Run scripts/worktree-setup.ts with environment variables]
[Run npm install]
[Run npm test - 47 passing]

Worktree ready at /Users/kristo/d/radial.worktrees/bugfix
Tests passing (47 tests, 0 failures)
Ready to implement bugfix
```

## Red Flags

**Never:**
- Skip worktree context detection (check for `.worktrees` in PWD)
- Skip baseline test verification
- Proceed with failing tests without asking
- Create worktrees inside other worktrees

**Always:**
- Detect if already in worktree directory first
- Use `../${project}.worktrees/` pattern consistently
- Auto-detect and run project setup
- Verify clean test baseline

## Integration

**Called by:**
- skills/collaboration/brainstorming (Phase 4)
- Any skill needing isolated workspace

**Pairs with:**
- skills/collaboration/finishing-a-development-branch (cleanup)
- skills/collaboration/executing-plans (work happens here)
