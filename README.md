# Contextual Commits

**Conventional Commits told you WHAT changed. Contextual Commits tell you WHY.**

A convention for embedding structured reasoning in git commit bodies. Every commit carries not just the code change, but the intent, decisions, rejected alternatives, constraints, and learnings that shaped it.

No new tools. No infrastructure. Just better commits.

## The Problem

AI coding tools have ~90% developer adoption. Trust in AI-generated output sits at 29%. The gap is context.

Every AI coding session produces three outputs:
1. **Code changes** — committed to git. Preserved.
2. **Decisions** — which approach was chosen, what was rejected, what constraints were found. **Lost.**
3. **Understanding** — deeper comprehension of how the system works and why. **Lost.**

Two-thirds of every session's intellectual output evaporates when the conversation window closes. The next session starts from zero. The next developer starts from zero. The next agent starts from zero.

Git already tracks everything about a session — branches track scope, diffs track changes, commit history tracks progression. The one thing it doesn't track is **reasoning**. The commit body has always been available for this. We just never needed it before AI.

## The Convention

A contextual commit uses the standard Conventional Commit subject line and adds **action lines** in the body — typed, scoped entries that capture reasoning the diff cannot show.

```
feat(auth): implement Google OAuth provider

intent(auth): social login starting with Google, then GitHub and Apple
decision(oauth-library): passport.js over auth0-sdk for multi-provider flexibility
rejected(oauth-library): auth0-sdk — locks into their session model, incompatible with redis store
constraint(callback-routes): must follow /api/auth/callback/:provider pattern per existing convention
constraint(session-store): redis 24h TTL means tokens must refresh within that window
learned(passport-google): requires explicit offline_access scope for refresh tokens
```

The subject line tells you **what**. The body tells you **why**.

### Action Types

| Type | Captures | Example |
|------|----------|---------|
| `intent(scope)` | What the user wanted and why | `intent(notifications): batch emails instead of per-event` |
| `decision(scope)` | What was chosen when alternatives existed | `decision(queue): SQS over RabbitMQ for managed scaling` |
| `rejected(scope)` | What was considered and discarded, with reason | `rejected(queue): RabbitMQ — requires self-managed infra` |
| `constraint(scope)` | Hard limits that shaped the approach | `constraint(api): max 5MB payload, 30s timeout` |
| `learned(scope)` | Discoveries that save time next session | `learned(stripe): presentment ≠ settlement currency` |
| `context(scope)` | References to related decisions or prior work | `context(payments): see ADR-007 for background` |

**scope** is a human-readable label — the domain area, module, or concept. Use whatever vocabulary is natural in your project: `auth`, `payment-flow`, `api-contracts`, `session-store`.

### Design Principles

- **Extends, never breaks.** The subject line is a standard Conventional Commit. All existing tooling (commitlint, semantic-release, changelog generators) works unchanged.
- **Additive, not prescriptive.** Use only the action types that apply. A typo fix needs zero action lines. A major refactoring might have ten.
- **Human-readable first.** Anyone running `git log` sees useful context immediately. No tooling required to benefit.
- **Machine-queryable.** `git log --all --grep="rejected(auth"` instantly finds every rejected auth approach. Simple regex extracts all action lines from any commit body.

## Reference Implementation

This repo includes two agent-agnostic files following the [Agent Skills](https://agentskills.io) open standard. They work with any compatible agent — Claude Code, GitHub Copilot, Cursor, Gemini CLI, and [26+ others](https://agentskills.io).

### Quick Start

Copy the two files into your project:

```
your-project/
├── .claude/skills/contextual-commit/SKILL.md    # Claude Code
├── .github/skills/contextual-commit/SKILL.md    # GitHub Copilot
├── .cursor/skills/contextual-commit/SKILL.md    # Cursor
└── ... (wherever your agent loads skills from)
```

Same for the slash command:

```
your-project/
├── .claude/commands/recall.md
├── .github/commands/recall.md
└── ...
```

That's it. Your agent now knows how to write contextual commits and how to reconstruct context from git history.

### What's Included

| File | What It Does |
|------|--------------|
| [`skills/contextual-commit/SKILL.md`](skills/contextual-commit/SKILL.md) | Teaches the agent the contextual commit format. Auto-invoked when committing. Produces structured action lines based on what happened in the session. |
| [`commands/recall.md`](commands/recall.md) | `/recall` — reconstructs development context from the current branch's contextual commit history. Shows accumulated intent, decisions, rejections, constraints, and learnings. |

### Usage

**Writing contextual commits** — just commit normally. The skill activates automatically when the agent writes a commit and produces action lines based on the session's conversation.

**Recalling context** — type `/recall` at the start of a session or when resuming work. The agent reconstructs what's been decided, rejected, discovered, and learned from the branch's commit history.

## Examples

### What's the difference?

**Standard AI commit:**
```
feat(auth): implement Google OAuth provider

Added GoogleAuthProvider class with passport.js integration.
Created callback route handler at /api/auth/callback/google.
Added refresh token logic with offline access scope.
Updated auth middleware to support multiple providers.
```
This describes WHAT the diff contains. The diff already shows that. Zero signal.

**Contextual commit:**
```
feat(auth): implement Google OAuth provider

intent(auth): social login starting with Google, then GitHub and Apple
decision(oauth-library): passport.js over auth0-sdk for multi-provider flexibility
rejected(oauth-library): auth0-sdk — locks into their session model
constraint(callback-routes): must follow /api/auth/callback/:provider pattern
learned(passport-google): requires explicit offline_access scope for refresh tokens
```
This captures what the diff CANNOT show. Every line is signal a future session needs.

### Capturing a mid-implementation pivot

```
refactor(auth): switch from session-based to JWT tokens

intent(auth): original session approach incompatible with redis cluster setup
rejected(auth-sessions): redis cluster doesn't support session stickiness needed by passport
decision(auth-tokens): JWT with short expiry + refresh token pattern
learned(redis-cluster): session affinity requires sticky sessions at LB level — too invasive
```

### Trivial changes — no action lines needed

```
fix(button): correct alignment on mobile viewport
```

```
chore(deps): bump express to 4.21.1
```

The conventional commit subject is sufficient. Don't add noise.

## Why This Matters

### For humans

- **Onboarding** — new engineers read `git log` and understand not just what changed but why
- **Code review** — reviewers see reasoning alongside the diff, reducing back-and-forth
- **Knowledge preservation** — when someone leaves, their decisions stay in the history

### For AI agents

- **No re-proposing rejected approaches** — `rejected` lines prevent agents from exploring dead ends
- **Constraint awareness** — `constraint` lines surface limits before the agent hits them
- **Faster session starts** — `/recall` gives agents accumulated project knowledge immediately
- **Compounding intelligence** — each session's learnings benefit every future session

### For your codebase

- **Self-documenting history** — the repo's own git log becomes its most reliable documentation
- **Zero infrastructure** — no database, no external service, no new workflow. Just git.
- **No merge conflicts** — unlike documentation files, commit bodies are append-only history

## FAQ

**Does this break Conventional Commits?**
No. The subject line IS a conventional commit. Action lines live in the body, which Conventional Commits does not govern. Existing tooling (commitlint, semantic-release, changelog generators) validates the subject line only and is completely unaffected.

**Do I need to use every action type on every commit?**
No. Use only what applies. Most commits need 0-3 action lines. Trivial changes need none.

**What if my agent doesn't support Agent Skills?**
You can still write contextual commits manually. The convention is the value — the skill just automates it. Add the instructions to your CLAUDE.md, .cursorrules, or equivalent project configuration.

**How do I search contextual commits?**
```bash
# Find all rejected approaches related to auth
git log --all --grep="rejected(auth"

# Find all constraints
git log --all --grep="^constraint("

# Find all decisions on a specific branch
git log main..HEAD --grep="^decision("

# Extract all action lines from a branch
git log main..HEAD --format="%b" | grep -E "^(intent|decision|rejected|constraint|learned|context)\("
```

**What should the scope be?**
Whatever is meaningful in your project's vocabulary. Domain concepts (`auth`, `payments`, `notifications`), module names (`api-gateway`, `session-store`), or technical concerns (`database-migration`, `caching`). Be consistent — use the same scope when referring to the same concept across commits.

## What This Is Not

This is a **convention**, not a tool. It works today, manually, with zero installation.

The reference implementation (skill + command) makes it easier to practice the convention with AI agents. But the value is in the commit history itself — readable by any human, parseable by any tool, owned by git.

For codegraph-powered scoping, automated context maintenance, multi-agent orchestration, and intelligent context recall across repositories, see [Engraph](https://github.com/berserkdisruptors/engraph) — where contextual commits become nodes in a queryable knowledge graph.

## License

MIT
