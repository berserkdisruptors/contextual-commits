--
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
^(intent|decision|rejected|constraint|learned|context)\(
```

Group them by commit (preserve chronological order) and by type (for synthesis).

## Step 4: Narrate

This is the critical step. **Do not list or dump the extracted data.** Synthesize it into a conversational briefing that tells the story of where things stand.

### For Scenario A (branch with commits):

Tell the story of this branch: what the user set out to do (from `intent` lines), what approach they took (from `decision` lines), what didn't work (from `rejected` lines), what limits they hit (from `constraint` lines), and what they learned along the way (from `learned` lines).

If there are unstaged changes, describe what's currently in progress.

End with a natural prompt: "How do you want to proceed?" or "What should we tackle next?"

Example tone:
```
Here's where we stand on this branch:

We're extending the auth system to support Google OAuth as the first
social login provider, with GitHub and Apple planned next. We went with
passport.js after ruling out auth0-sdk — it locks you into their session
model which doesn't work with the existing redis store.

The callback routes are working and follow the existing /api/auth/callback/:provider
pattern. One thing to note: passport-google requires an explicit offline_access
scope for refresh tokens, which isn't obvious from their docs.

Currently there are unstaged changes in the test files — looks like the
integration tests for the callback handler are in progress.

What do you want to tackle next?
```

### For Scenario B (branch with no commits):

Briefly describe what's changed (staged/unstaged) and provide recent project context from the default branch history. Acknowledge that there's no branch-specific context yet.

### For Scenario C (default branch, no changes):

Synthesize recent activity from the last merged commits. Tell the story of what happened recently — what was shipped, what decisions were made, what's in motion.

Example tone:
```
Here's what's been happening recently:

The multi-currency support for payments landed last week — we're now
handling EUR and GBP alongside USD, using per-transaction currency
with currency.js. The Stripe integration required some careful handling
since currency gets locked at PaymentIntent creation.

Before that, the auth system got the OAuth provider framework merged.
Google is working, GitHub and Apple are next.

There's an open constraint worth knowing: the redis session store has a
24h TTL, so any token refresh logic needs to stay within that window.

What do you want to work on?
```

### For Scenario D (default branch with uncommitted changes):

Combine the project briefing from Scenario C with a note about the uncommitted changes.

### When there are no contextual commits

If the history has no action lines at all, say so honestly. Fall back to summarizing the conventional commit subjects for recent activity. Suggest that using the `contextual-commit` skill on future commits will make `/recall` much more useful.

## Guidelines

- **Narrate, don't list.** Never output bullet lists of raw action lines. Tell the story.
- **Prioritize the current branch.** On a feature branch, the branch context is what matters. Project-wide context is secondary.
- **Surface rejections prominently.** If something was tried and rejected, that's critical — the user (or the next agent) needs to know before re-exploring it.
- **Be brief.** A few paragraphs, not a report. This is a briefing, not documentation.
- **End with a prompt.** Always close with a natural question about what to do next. This is a session start, not a session end.
- **Don't fabricate.** Only narrate what the action lines and diffs actually show. If there's limited history, keep it short. An honest "there's not much context yet" is better than padding