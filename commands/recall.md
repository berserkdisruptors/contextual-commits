---
name: recall
description: >-
  Reconstruct and narrate the current development context from contextual
  commits. Run at session start, when resuming work, or when switching
  branches. Produces a brief, conversational summary of where things stand.
---

# Context Recall

Reconstruct the development story from contextual commit history and present it as a natural briefing.

## Step 1: Detect Branch State

Determine the working state:

```bash
CURRENT_BRANCH=$(git branch --show-current)
DEFAULT_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@' || echo "main")
UNSTAGED=$(git diff --stat)
STAGED=$(git diff --cached --stat)
BRANCH_COMMITS=$(git log ${DEFAULT_BRANCH}..HEAD --oneline 2>/dev/null | wc -l | tr -d ' ')
```

This determines which of four scenarios you are in.

## Step 2: Gather Raw Material

### Scenario A — On a feature branch with commits

This is the richest scenario. Gather:

```bash
# Contextual action lines from all branch commits
git log ${DEFAULT_BRANCH}..HEAD --format="%H%n%s%n%b%n---COMMIT_END---"

# Unstaged changes (what's in progress right now)
git diff --stat
git diff  # read the actual diff for key changes

# Staged changes
git diff --cached --stat
```

### Scenario B — On a feature branch with no commits yet

```bash
# Unstaged and staged changes only
git diff --stat
git diff --cached --stat

# Last few commits on the default branch for project context
git log ${DEFAULT_BRANCH} -10 --format="%H%n%s%n%b%n---COMMIT_END---"
```

### Scenario C — On the default branch

```bash
# Recent commit history with contextual action lines
git log -20 --format="%H%n%s%n%b%n---COMMIT_END---"
```

### Scenario D — On the default branch with uncommitted changes

```bash
# Same as C plus uncommitted changes
git log -20 --format="%H%n%s%n%b%n---COMMIT_END---"
git diff --stat
git diff --cached --stat
```

## Step 3: Extract Action Lines

From the gathered commit bodies, extract lines matching:
```
^(intent|decision|rejected|constraint|learned)\(
```

Group them by commit (preserve chronological order) and by type (for synthesis).

## Step 4: Synthesize Output

**Signal density over narrative flow.** The output should be compact, scannable, and grounded entirely in what the commits and diffs show. Every line should be actionable information. No fluff, no conversational padding.

### For Scenario A (branch with commits):

Output the branch state, then synthesize the contextual action lines into a dense briefing organized by what matters most for continuing work.

Example output:
```
Branch: feat/google-oauth (4 commits ahead of main, unstaged changes in tests/)

Active intent: Add Google as first social login provider. GitHub and Apple to follow.
Approach: passport.js with /api/auth/callback/:provider convention.
Rejected: auth0-sdk — session model incompatible with redis store.
Constraints:
  - Redis session TTL 24h, tokens must refresh within window
  - Callback routes must follow existing :provider pattern
Learned: passport-google needs explicit offline_access scope for refresh tokens.
In progress: Integration tests for callback handler (unstaged).
```

Priority order:
1. Active intent (what we're building and why)
2. Current approach (decisions made)
3. Rejected approaches (what NOT to re-explore — critical)
4. Constraints (hard boundaries)
5. Learnings (things that save time)
6. In-progress work (unstaged/staged changes)

If intent evolved during the branch (a pivot), show both the original and current intent to make the pivot visible.

### For Scenario B (branch with no commits):

```
Branch: feat/new-feature (0 commits ahead of main)

No contextual history on this branch yet.
Staged: 2 files (src/auth/provider.ts, src/auth/types.ts)
Unstaged: none

Recent project activity (from main):
  - Auth: OAuth provider framework merged, Google working
  - Payments: Multi-currency support shipped (EUR, GBP alongside USD)
```

### For Scenario C (default branch, no changes):

Synthesize recent merged work from the last 20 commits. Group by area of activity. Surface any active constraints or learnings that apply broadly.

```
Recent project activity:

Auth: OAuth provider framework merged. Google working, GitHub and Apple planned.
  - Rejected auth0-sdk (session model incompatible with redis store)
  - Constraint: redis session TTL 24h, tokens must refresh within window

Payments: Multi-currency support shipped (USD, EUR, GBP).
  - Per-transaction currency, not account-level
  - Constraint: Stripe locks currency at PaymentIntent creation
  - Learned: presentment ≠ settlement currency in Stripe

What do you want to work on?
```

### For Scenario D (default branch with uncommitted changes):

Same as Scenario C, with uncommitted changes noted at the top.

### When there are no contextual commits at all

If the history contains conventional commits but no contextual action lines, **still produce useful output** from what exists:

```
Branch: main (no contextual commits found in recent history)

Recent activity (from commit subjects):
  - auth: 6 commits (OAuth implementation, middleware updates) — 2 days ago
  - payments: 4 commits (currency handling, Stripe integration) — last week
  - tests: 3 commits — last week
  - config: 2 commits

Most recent: feat(auth): implement Google OAuth provider

No explicit intent, decisions, or constraints are captured in commit history.
Using the contextual-commit skill on future commits will surface
the reasoning behind changes that commit subjects alone cannot show.
```

This is honest about the signal quality while still providing useful orientation. The suggestion to adopt the skill is a natural next step, not a sales pitch.

## Guidelines

- **Dense over conversational.** Every line should carry information. No "Here's what's been happening" or "Let me tell you about."
- **Grounded in data.** Only report what the action lines, commit subjects, and diffs actually show. Do not infer, speculate, or fill gaps.
- **Surface rejections prominently.** Rejected approaches are the highest-value signal — they prevent wasted exploration.
- **Group by scope when multiple scopes exist.** On the default branch with broad history, organize by domain area rather than chronologically.
- **End with a prompt.** Close with "What do you want to work on?" or similar. Keep it short.
- **Scale to the data.** If there are 2 contextual commits, the output is 3-4 lines. If there are 20, it's a few grouped paragraphs. Never pad thin data into a long output.
