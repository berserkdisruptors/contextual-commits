---
name: recall
description: >-
  Show the current development context reconstructed from contextual commits.
  Displays branch state, accumulated intent, decisions, constraints, and
  learnings from the git history of the current branch. Use at session start,
  when resuming work, or when you need to understand the reasoning behind
  the current state of the code.
---

# Context Recall

Reconstruct and present the development context from the current branch's contextual commit history.

## Step 1: Detect Branch State

Run the following to determine the current working state:

```bash
CURRENT_BRANCH=$(git branch --show-current)
DEFAULT_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@' || (git rev-parse --verify refs/heads/main >/dev/null 2>&1 && echo main || git rev-parse --verify refs/heads/master >/dev/null 2>&1 && echo master || echo main))
UNSTAGED=$(git diff --stat)
STAGED=$(git diff --cached --stat)
BRANCH_COMMITS=$(git log ${DEFAULT_BRANCH}..HEAD --oneline 2>/dev/null | wc -l | tr -d ' ')
```

Determine which scenario applies:

- **On default branch, no changes**: No active work context. Report that and suggest starting a new branch.
- **On default branch, with changes**: Uncommitted work without a branch. Note this and show what's changed.
- **On branch, no commits ahead**: Branch exists but no work done yet. Check for uncommitted changes.
- **On branch, with commits**: Active work. This is where contextual recall is most valuable.

## Step 2: Extract Contextual Commits

If there are branch commits ahead of the default branch, extract all contextual action lines:

```bash
git log ${DEFAULT_BRANCH}..HEAD --format="%H%n%s%n%b%n---COMMIT_END---"
```

Parse each commit and collect:
- The conventional commit subject (the WHAT)
- All action lines from the body matching the pattern: `^(intent|decision|rejected|constraint|learned|context)\(`

Group the collected action lines by type across all commits.

## Step 3: Present the Report

Format the output as a clear status report. Adapt based on what was found.

### Report Structure

**Branch status**: `{branch_name}` — `{N}` commits ahead of `{default_branch}`

**Uncommitted changes** (if any): summarize staged and unstaged changes.

**What's being built** (from `intent` lines):
List all intent lines chronologically. If intent changed mid-branch (a pivot), note the progression.

**Decisions made** (from `decision` lines):
List all decisions. Group by scope if there are many.

**Approaches rejected** (from `rejected` lines):
List all rejections with their reasons. This is critical context — it prevents re-exploring dead ends.

**Constraints discovered** (from `constraint` lines):
List all constraints. These are hard boundaries that must be respected.

**Things learned** (from `learned` lines):
List all learnings. These save time in the current session.

**Related context** (from `context` lines):
List any references to related decisions, prior work, or external context.

**Commit progression**: Brief chronological summary of the conventional commit subjects showing how the work progressed.

### When There Are No Contextual Commits

If the branch has commits but none contain action lines, report:
- The branch status and commit subjects
- Note that no contextual information was found in commit bodies
- Suggest using the `contextual-commit` skill for future commits to build up project context

### When On the Default Branch

If the user runs recall while on the default branch, search the recent history of that branch:

```bash
git log -50 --format="%H%n%s%n%b%n---COMMIT_END---"
```

Parse the last 50 commits on the current branch for any contextual action lines. Present a summary of recent project-wide decisions, constraints, and learnings. This gives the user a general project briefing rather than a branch-specific one.

## Tone

Present the report as a concise briefing — not a data dump. Prioritize:
1. Active intent (what's being worked on and why)
2. Rejected approaches (what NOT to re-explore)
3. Constraints (what limits apply)
4. Recent decisions (what's been decided)

Skip sections that have no entries. Don't show empty headers.