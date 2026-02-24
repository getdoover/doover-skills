---
name: doover-rebase
description: Clean up AI-generated code drift after solving a hard problem. Run this in the same chat where you built your solution — it uses conversation context to understand what to keep and what to clean. Works across single or multiple repos.
user_invocable: true
---

# Doover Rebase

Clean up code that has drifted from convention during an extended AI-assisted debugging or development session. This skill is designed to be run **in the same chat window** where the solution was built, so that the full conversation context (the problem, failed approaches, and final solution) is available.

## When to Use

- You've been going back and forth with Claude to solve a hard problem
- The solution works, but the code has accumulated "AI slop" — unnecessary imports, abandoned approach remnants, inconsistent style, debug artifacts, redundant error handling, over-engineered abstractions
- You want to commit clean, conventional code that preserves the working solution

## Overview

```
Step 1: Context Summary     → Review conversation, summarize problem + solution
Step 2: Repo Discovery       → Find all repos with uncommitted changes
Step 3: Diff Analysis        → Get and categorize every change
Step 4: Cleanup Plan         → Present plan to user for approval
Step 5: Apply Cleanup        → Edit files, preserving the solution
Step 6: Verification         → Confirm nothing is broken
Step 7: Summary              → Report what was cleaned
```

## Execution Instructions

### Step 1: Context Summary

Review the full conversation history to build a mental model of:

1. **The original problem** — what was broken or what feature was being built
2. **The solution** — what specifically fixed it or implemented the feature
3. **Key decisions** — any deliberate architectural or design choices the user confirmed
4. **Failed approaches** — things that were tried and abandoned (these are cleanup targets)

Present a brief summary to the user:

```
Before I clean up, let me confirm I understand the situation:

**Problem:** [1-2 sentences]
**Solution:** [1-2 sentences describing the core fix/feature]
**Key files:** [list the files central to the solution]

Does this look right?
```

Use `AskUserQuestion` to confirm. Do not proceed until the user agrees the summary is accurate. If the user corrects something, update your understanding before continuing.

### Step 2: Repo Discovery

Determine which repositories have uncommitted changes.

**Single repo mode:** If the current working directory is a git repo, check it:

```bash
git status --porcelain
git diff --stat
git diff --cached --stat
```

**Multi-repo mode:** If the user mentions multiple repos, or if the conversation involved work across multiple directories, check each one. Also check for common multi-repo layouts:

```bash
# Check if cwd contains multiple repos
for dir in */; do
  if [ -d "$dir/.git" ]; then
    echo "=== $dir ==="
    git -C "$dir" status --porcelain
  fi
done
```

For each repo with changes, record:
- Repository path
- Branch name
- List of modified/added/deleted files
- Whether there are staged vs unstaged changes

If the user has already committed some changes and the diff is between commits rather than uncommitted, ask the user which commit range to examine. Support comparing against:
- The previous commit (`HEAD~1`)
- A specific commit hash
- A branch point (`main..HEAD`)

### Step 3: Diff Analysis

For each repo with changes, get the full diff:

```bash
# Uncommitted changes
git diff
git diff --cached

# Or if comparing commits
git diff <base>..<head>
```

For every changed file, categorize each change as one of:

| Category | Description | Action |
|----------|-------------|--------|
| **Essential** | Directly implements the solution | Keep as-is |
| **Essential but messy** | Part of the solution but poorly written | Refactor |
| **Abandoned approach** | Leftover from a failed attempt | Revert |
| **Debug artifact** | Print statements, console.logs, temporary comments | Remove |
| **Unnecessary addition** | Extra imports, unused variables, redundant error handling | Remove |
| **Style drift** | Working code that breaks project conventions | Restyle |
| **Over-engineering** | Abstractions or patterns beyond what's needed | Simplify |

**How to identify each category:**

- **Essential**: Changes that, if reverted, would break the solution or remove the feature
- **Essential but messy**: The logic is correct but the code doesn't follow the project's existing patterns (wrong import style, inconsistent naming, non-idiomatic constructs)
- **Abandoned approach**: Code added during a failed attempt that was never fully removed. Look for: commented-out blocks, functions that are defined but not called, imports with no references
- **Debug artifact**: `print()`, `console.log()`, `# DEBUG`, `# TODO: remove`, `breakpoint()`, temporary test values, hardcoded test data
- **Unnecessary addition**: Imports that aren't used, variables assigned but never read, try/except blocks around code that can't fail, type annotations added to untouched code, docstrings added to untouched functions
- **Style drift**: The original codebase uses one pattern (e.g., `from x import y`) but the new code uses another (e.g., `import x; x.y`). Or inconsistent naming conventions, spacing, etc.
- **Over-engineering**: Feature flags for a single use case, abstraction layers with one implementation, configuration for values that won't change, backwards-compatibility shims for code that was just written

### Step 4: Cleanup Plan

Present a file-by-file cleanup plan to the user. For each file:

```
### `path/to/file.py`
- **Keep**: [describe essential changes]
- **Refactor**: [describe what will be cleaned up and how]
- **Remove**: [describe what will be removed and why]
- **Revert**: [describe what will be reverted to original]
```

At the end of the plan, include:

```
**Files to fully revert** (no solution-relevant changes):
- path/to/file1.py — only contained debug logging
- path/to/file2.py — leftover from abandoned approach

**Risk assessment**: [Low/Medium/High] — [brief justification]
```

Use `AskUserQuestion` to get approval:
- "Approve the plan as-is"
- "I want to adjust some items" (then discuss)
- "Skip cleanup for specific files"

Do not proceed until the user approves.

### Step 5: Apply Cleanup

Execute the cleanup plan file by file. For each file:

1. **Read the current file** to get exact content
2. **Apply edits** using the Edit tool — make targeted replacements, not full file rewrites
3. **For files to fully revert**: use `git checkout -- <file>` to restore the original
4. **For deleted files that shouldn't exist**: use `git checkout -- <file>` or `rm <file>` as appropriate

**Cleanup principles:**

- **Match existing conventions exactly.** Look at the unchanged parts of each file and the rest of the codebase for:
  - Import style and ordering
  - Naming conventions (snake_case, camelCase, etc.)
  - Error handling patterns
  - Comment style
  - Indentation and formatting
- **Preserve the solution's logic faithfully.** Never change what the code does, only how it's written
- **Minimize the diff.** The goal is that the final diff (compared to the original commit) should contain only the essential changes for the solution, written cleanly
- **Don't add things.** No new docstrings, type annotations, comments, or abstractions unless they were part of the original solution
- **Don't touch unrelated code.** If a line wasn't part of the solution or the drift, leave it alone

### Step 6: Verification

After cleanup, verify the solution still works.

**Always do:**
```bash
# Check for syntax errors (Python)
python -m py_compile <file>

# Check for syntax errors (JavaScript/TypeScript)
npx --yes acorn --ecma2020 <file>  # or similar

# Check imports resolve
python -c "import <module>"
```

**If the project has tests:**
```bash
# Run relevant tests
pytest <test_file_or_directory>
npm test
```

**If the project has a build step:**
```bash
# Verify it builds
npm run build
uv run export-config
```

**If the user previously ran a specific verification command** during the conversation (check the conversation history), run that same command again.

If verification fails:
1. Report what failed
2. Identify which cleanup edit likely caused it
3. Fix it (or revert that specific edit)
4. Re-verify
5. Do not leave the code in a broken state

### Step 7: Summary

Report to the user:

```
## Rebase Complete

**Repos cleaned:** [count]

### [repo-name]
- **Files modified:** [count]
- **Files reverted:** [count]
- **Lines added (net):** [count]
- **Lines removed (net):** [count]
- **Verification:** [passed/failed with details]

**Changes cleaned up:**
- [Brief list of what was removed/refactored]

**Solution preserved:**
- [Brief list of what was kept, confirming the core solution is intact]
```

Ask the user if they'd like to:
- Review the final diff (`git diff`)
- Commit the changes
- Make additional adjustments

## Multi-Repo Handling

When working across multiple repos:

1. Run Steps 3-4 (analysis + plan) for **all repos first** before applying any changes
2. Present a unified plan covering all repos
3. Apply changes repo by repo in Step 5
4. Verify each repo independently in Step 6
5. Report all repos in the summary

This prevents partial cleanup states where one repo is cleaned but another isn't.

## Edge Cases

**No uncommitted changes:**
If `git status` shows a clean working tree, check if the user has already committed. Look at recent commits:
```bash
git log --oneline -5
```
If the solution was already committed, offer to do an interactive rebase-style cleanup by examining the diff between the solution commit(s) and the branch point. Apply cleanup edits as a new commit on top.

**Staged vs unstaged changes:**
If some changes are staged and others aren't, note this in the plan. Ask the user if the staging is intentional before proceeding.

**New files:**
For entirely new files, apply the same categorization. If a new file is entirely from an abandoned approach, it should be deleted. If it's part of the solution, clean it up like any other file.

**Binary files / generated files:**
Skip these. Note them in the summary but don't attempt to clean them.

## What This Skill Does NOT Do

- **Does not change the solution's behavior.** If the code works, the cleaned version must work identically.
- **Does not optimize or improve.** Performance improvements, better algorithms, or architectural changes are out of scope unless they're fixing drift.
- **Does not rewrite from scratch.** This is targeted cleanup, not a rewrite. Edit surgically.
- **Does not touch files outside the diff.** If a file wasn't changed during the session, it's not a cleanup target.
