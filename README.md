[![Spec](https://img.shields.io/badge/spec-v0.1.0-blue)](SPEC.md)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)

# Contextual Commits

**Conventional Commits standardised WHAT changed. Contextual Commits add WHY.**

A convention for embedding decision traces in git commit bodies. Every commit carries not just the code change, but the intent and reasoning that shaped it — structured as typed action lines that any tool can parse and any agent can query.

No new tools. No infrastructure. Just better commits.

## The Problem

AI coding tools have ~90% developer adoption. Trust in AI-generated output sits at 29%. The gap is context.

Every AI coding session produces three outputs:
1. **Code changes** — committed to git. Preserved.
2. **Decisions** — which approach was chosen, what was rejected, what constraints were found. **Lost.**
3. **Understanding** — deeper comprehension of how the system works and why. **Lost.**

Two-thirds of every session's intellectual output evaporates when the conversation window closes. This is **session context** (intent and decision traces) — ephemeral by nature, critical by impact. It lives in the agent's conversation window and disappears when the window is compacted or closed. Storing it in local file artifacts (specs, plans, session logs) has been tried; it creates version control friction, context window bloat, and maintenance overhead for information that is fundamentally transient. The problem is not that session context needs a new storage location. It needs to travel with the thing that already persists: the commit.

And there is a second, quieter problem. Commit history is the one context source available to every AI coding tool — durable, portable, queryable out of the box. Yet it's underutilised. Commits in general, and the Conventional Commits spec in particular, were designed to help developers navigate change history — changelogs, semantic versioning, human-readable summaries. Agents are now the most frequent consumers of commit logs, yet the standard AI-generated commit body restates what the diff already shows: "Added GoogleAuthProvider class. Created callback route handler. Updated auth middleware." This is noise. An agent reading the diff gets the same information. What it cannot get from the diff is why passport.js was chosen over auth0-sdk, what constraint forced the callback route pattern, or what the developer was actually trying to achieve. The opportunity is to optimise for agent comprehension.

Git already tracks everything about a session — branches track scope, diffs track changes, commit history tracks progression. The one thing it doesn't track is reasoning. The commit body has always been available for this. We just never needed it before AI.

## The Solution

Contextual commits capture **intent** (what you're building and why) and **historical** context (decisions, rejections, constraints, learnings) — the two context types that accumulate over time and cannot be extracted from code analysis alone.

The format extends a standard Conventional Commit subject line with **action lines** in the body — typed, scoped entries that capture reasoning the diff cannot show.

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
| `learned(scope)` | Discovered facts that prevent future mistakes | `learned(stripe): presentment ≠ settlement currency` |

**Five types. Each captures signal no other type covers. Each is immediately useful to an agent starting a new session.**

- `intent` — what the user is trying to achieve. Without it, the agent reverse-engineers purpose from code.
- `decision` — what approach was chosen. Without it, the agent doesn't know if a pattern is intentional or accidental.
- `rejected` — what was tried and discarded. Without it, the agent re-explores dead ends. The highest-value type.
- `constraint` — hard limits on the implementation. Without it, the agent discovers them by failing.
- `learned` — API quirks, non-obvious behaviors, documentation gaps. Without it, the agent wastes cycles rediscovering gotchas. Distinct from constraints: constraints are boundaries to enforce, learnings are traps to avoid.

**scope** is a human-readable label — the domain area, module, or concept. Use whatever vocabulary is natural in your project: `auth`, `payment-flow`, `api-contracts`, `session-store`.

### Design Principles

- **Extends, never breaks.** The subject line is a standard Conventional Commit. All existing tooling (commitlint, semantic-release, changelog generators) works unchanged.
- **Agent-native, human-readable.** Designed for agents to parse and query programmatically. Humans benefit as a byproduct — `git log` shows useful context immediately, but the format is optimised for machine consumption.
- **Signal the diff can't show.** Every action line must carry information that is not already visible in the code changes. If the diff explains it, don't repeat it. This is the core quality rule.
- **Additive, not prescriptive.** Use only the action types that apply. A typo fix needs zero action lines. A major refactoring might have ten.
- **Zero infrastructure.** No database, no external service, no configuration file, no CI step. The convention lives in git commit bodies — the most universal, portable, and durable storage in software development.
- **Queryable by default.** `git log --all --grep="rejected(auth"` instantly finds every rejected auth approach across the entire history. Simple regex extracts all action lines from any commit range.

For the formal specification with numbered rules and ABNF grammar, see [SPEC.md](SPEC.md).

## Why This Matters

### The context that disappears

The agent proposes an approach you already tried and rejected last session — but the reasoning that ruled it out died with the conversation window. It writes a clean implementation that violates a constraint it has no way of knowing about, and discovers it by failing. Three months later, another session sees a pattern in the code that looks arbitrary — it wasn't, but the reason existed in a conversation that no longer exists.

Same problem, three forms. AI coding sessions produce decisions and understanding alongside code, but only the code survives in git.

### What agents can and cannot recover

Several categories of context shape AI coding quality. Most can be reverse-engineered from the codebase: architecture, code patterns, test strategy, naming conventions. An agent that reads your code can figure these out.

Two categories cannot be reverse-engineered: **what you intended** and **what you already tried**. Intent and historical context — the decisions made, alternatives rejected, constraints discovered, lessons learned — exist only in human memory and disappearing conversations.

Contextual commits capture exactly these two. Not because they're the most interesting, but because they're the ones that would otherwise be permanently lost.

### Compounding

The first contextual commit saves one future re-exploration. The hundredth means an agent starting a fresh session inherits every decision, rejection, constraint, and learning from every previous session — across every contributor.

This is not documentation you maintain. It's append-only history that accumulates as a side effect of committing code. No files to keep current. No wiki pages to update. No merge conflicts. Just git.

## Who Needs to Act

This convention is a spec for AI coding tools to implement natively. 

The goal is for coding agents (Claude Code, OpenCode, Codex, Gemini CLI, and others) and IDEs (Cursor, Windsurf, and others) to follow this convention by default — replacing the noise they currently generate with signal. 

Every developer benefits when agents stop restating diffs and start preserving reasoning. The problem is universal; the solution scales through tooling, not individual discipline. 

The reference implementation below is a bridge for today — agent skills that any developer can install. The real adoption path is tool makers building this in.

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
This describes WHAT the diff contains. The diff already shows that. Zero signal added.

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

## Reference Implementation

This repo includes two agent-agnostic files following the [Agent Skills](https://agentskills.io) open standard. They work with any compatible agent — Claude Code, GitHub Copilot, Cursor, Gemini CLI, and [26+ others](https://agentskills.io).

### Quick Start

```bash
npx skills add berserkdisruptors/contextual-commits
```

Auto-detects your agent (Claude Code, Cursor, Copilot, Codex, Gemini CLI, and [40+ others](https://agentskills.io)), installs the skills to the correct directory. That's it.

### What's Included

| File | What It Does |
|------|--------------|
| [`skills/contextual-commit/SKILL.md`](skills/contextual-commit/SKILL.md) | Teaches the agent the contextual commit format. Auto-invoked when committing. Produces structured action lines based on what happened in the session. |
| [`skills/recall/SKILL.md`](skills/recall/SKILL.md) | `/recall` — reconstructs development context from contextual commit history. Supports three modes: full session briefing, scope queries, and action+scope queries. |

### Usage

**Writing contextual commits** — just commit normally. The skill activates automatically when the agent writes a commit and produces action lines based on the session's conversation.

**Recalling context** — `/recall` supports three modes:

- `/recall` — full session briefing. Shows accumulated intent, decisions, rejections, constraints, and learnings for the current branch.
- `/recall auth` — scope query. Searches the entire repo history for all action lines matching a scope (prefix-matched, so `auth` also finds `auth-tokens`, `auth-library`).
- `/recall rejected(auth)` — action+scope query. Searches for a specific action type within a scope. Useful for checking what's been tried and discarded before proposing an approach.

## FAQ

**What about repos that already use conventional commits but not contextual commits?**

`/recall` still works — it falls back to summarizing recent activity from commit subjects and file change patterns. The output is thinner (WHAT happened, not WHY) but still provides useful orientation. Adopting contextual commits is incremental: new commits carry action lines, old commits remain as they are. The context accumulates forward.

**What if the agent wasn't part of the reasoning (copy-paste, external changes)?**

The skill handles this explicitly. The agent only writes action lines for what it can observe in the diff — typically a `decision` line if a clear technical choice is visible (new dependency, pattern change). It does not fabricate `intent`, `rejected`, or `constraint` lines it has no evidence for. A clean conventional commit with no action lines is always better than invented context.

**What happens with squash merges?**

When squash-merging, git concatenates all commit bodies. The result is a chronological trail of typed, scoped action lines — agents parse and group these without issue. No cleanup needed. Regular merges, rebases, and cherry-picks preserve commit bodies intact.

**What about git worktrees?**

Worktrees share the same `.git` database. All contextual commits are visible and queryable from any worktree — `git log --grep`, `/recall`, and all action line queries work identically regardless of which worktree you're in. No special handling needed.

**What if my agent doesn't support Agent Skills?**

You can still write contextual commits manually. The convention is the value — the skill just automates it. Add the instructions to your CLAUDE.md, .cursorrules, or equivalent project configuration.

## What This Is Not

This is a **convention**, not a tool. It works today, manually, with zero installation.

Contextual commits focus on intent and historical context — the two types that vanish with conversation windows. Other context types — structural (architecture, dependencies), conventional (patterns, code style), verification (test strategy, quality criteria) — can be recovered from the codebase itself and require different capture mechanisms.

The reference implementation (skill + command) makes it easier to practice the convention with AI agents. But the value is in the commit history itself — readable by any human, parseable by any tool, owned by git.

The formal specification is in [SPEC.md](SPEC.md).

For consistent scoping, broader context coverage, automated context maintenance, agent-native tooling, and richer context recall, see [Engraph](https://github.com/berserkdisruptors/engraph) — the context layer for agentic coding, where contextual commits are only the beginning.

## License

MIT
