[![Spec](https://img.shields.io/badge/spec-v0.1.0-blue)](SPEC.md)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)

# Contextual Commits

**Conventional Commits standardised WHAT changed. Contextual Commits add WHY.**

A convention for embedding decision traces in git commit bodies. Every commit carries not just the code change, but the intent and reasoning that shaped it — structured as typed action lines that any tool can parse and any agent can query.

No new tools. No infrastructure. Just better commits.

---

**Standard AI commit:**
```
feat(auth): implement Google OAuth provider

Added GoogleAuthProvider class with passport.js integration.
Created callback route handler at /api/auth/callback/google.
Added refresh token logic with offline access scope.
Updated auth middleware to support multiple providers.
```
This restates what the diff already shows. Zero signal added.

**Contextual commit:**
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

---

## Motivation

AI coding tools are everywhere. Trust in their output isn't. The gap is context.

Every AI coding session produces three outputs:
1. **Code changes** — committed to git. Preserved.
2. **Decisions** — which approach was chosen, what was rejected, what constraints were found. **Lost.**
3. **Understanding** — deeper comprehension of how the system works and why. **Lost.**

Two-thirds of every session's value evaporates when the conversation window closes. Commit history is the one context source every AI coding tool can access out of the box — yet the standard AI-generated commit body restates what the diff shows. What agents can't get from the diff is *why* an approach was chosen, what constraints shaped it, or what was tried and rejected.

Git tracks branches, diffs, and history. The one thing it doesn't track is reasoning. The commit body has always been available for this.

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

---

## The Convention

### Format

A contextual commit uses the commit body to carry structured context. The subject line is a standard [Conventional Commit](https://www.conventionalcommits.org/en/v1.0.0/). The body extends it with typed action lines:

```
<type>(<scope>): <description>

<action-type>(<scope>): <content>
<action-type>(<scope>): <content>
```

**scope** is a human-readable label — the domain area, module, or concept. Use whatever vocabulary is natural in your project: `auth`, `payment-flow`, `api-contracts`, `session-store`.

### Action Types

| Type | Captures | Example |
|------|----------|---------|
| `intent(scope)` | What the user wanted and why | `intent(notifications): batch emails instead of per-event` |
| `decision(scope)` | What was chosen when alternatives existed | `decision(queue): SQS over RabbitMQ for managed scaling` |
| `rejected(scope)` | What was considered and discarded, with reason | `rejected(queue): RabbitMQ — requires self-managed infra` |
| `constraint(scope)` | Hard limits that shaped the approach | `constraint(api): max 5MB payload, 30s timeout` |
| `learned(scope)` | Discovered facts that prevent future mistakes | `learned(stripe): presentment ≠ settlement currency` |

Five types. Each captures signal no other type covers. Each is immediately useful to an agent starting a new session.

- `intent` — what the user is trying to achieve. Without it, the agent reverse-engineers purpose from code.
- `decision` — what approach was chosen. Without it, the agent doesn't know if a pattern is intentional or accidental.
- `rejected` — what was tried and discarded. Without it, the agent re-explores dead ends. The highest-value type.
- `constraint` — hard limits on the implementation. Without it, the agent discovers them by failing.
- `learned` — API quirks, non-obvious behaviors, documentation gaps. Without it, the agent wastes cycles rediscovering gotchas. Distinct from constraints: constraints are boundaries to enforce, learnings are traps to avoid.

### Design Principles

- **Extends, never breaks.** The subject line is a standard Conventional Commit. All existing tooling (commitlint, semantic-release, changelog generators) works unchanged.
- **Agent-native, human-readable.** Designed for agents to parse and query programmatically. Humans benefit as a byproduct — `git log` shows useful context immediately, but the format is optimised for machine consumption.
- **Signal the diff can't show.** Every action line must carry information that is not already visible in the code changes. If the diff explains it, don't repeat it. This is the core quality rule.
- **Additive, not prescriptive.** Use only the action types that apply. A typo fix needs zero action lines. A major refactoring might have ten.
- **Zero infrastructure.** No database, no external service, no configuration file, no CI step. The convention lives in git commit bodies — the most universal, portable, and durable storage in software development.
- **Queryable by default.** `git log --all --grep="rejected(auth"` instantly finds every rejected auth approach across the entire history. Simple regex extracts all action lines from any commit range.

For the formal specification with numbered rules and ABNF grammar, see [SPEC.md](SPEC.md).

---

## Examples

### When rejected lines do the work

```
feat(search): add full-text search to product catalog

intent(search): replace LIKE queries — too slow beyond 50k products
decision(search-engine): Postgres full-text search over Elasticsearch
rejected(search-engine): Elasticsearch — operationally too heavy for current scale, revisit at 500k products
rejected(search-engine): Algolia — cost at scale prohibitive, vendor lock-in concern
constraint(search-index): index rebuild takes ~8 min on full catalog, must run off-peak
```

Two future re-explorations prevented. Anyone asking "why not Elasticsearch?" gets the answer without a meeting.

### When learned lines do the work

```
feat(exports): generate PDF reports from dashboard data

decision(pdf-engine): Puppeteer over pdfkit — design team needs pixel-perfect HTML/CSS rendering
learned(puppeteer): requires --no-sandbox in Docker; sandbox mode crashes the container silently
learned(puppeteer): must await networkidle0, not load — times out on pages with deferred JS
constraint(exports): PDF generation blocks for ~3s — must run as background job, never inline
```

Each `learned` line is two hours someone won't spend rediscovering it.

### When one line is enough

```
fix(checkout): prevent duplicate orders on double-click submit

rejected(checkout): debounce — 300ms delay feels broken on slow connections
```

The fix is obvious from the diff. The only thing worth capturing is why the obvious alternative was ruled out.

---

Trivial commits — dependency bumps, typo fixes, formatting — need zero action lines. A clean conventional commit subject is always better than invented context.

---

## The Real Adoption Path

This convention is a spec for AI coding tools to implement natively.

The goal is for coding agents (Claude Code, OpenCode, Codex, Gemini CLI, and others) and IDEs (Cursor, Windsurf, and others) to follow this convention by default — replacing the noise they currently generate with signal. Every developer benefits when agents stop restating diffs and start preserving reasoning. The problem is universal; the solution scales through tooling, not individual discipline.

The reference implementation below is a bridge for today — agent skills that any developer can install while native adoption happens. The real adoption path is tool makers building this in.

Adoption is incremental — new commits carry action lines, old commits remain as they are. The context accumulates forward.

---

## Reference Implementation

This repo includes two agent-agnostic files following the [Agent Skills](https://agentskills.io) open standard. They work with any compatible agent — Claude Code, GitHub Copilot, Cursor, Gemini CLI, and [26+ others](https://agentskills.io).

### Quick Start

```bash
npx skills add berserkdisruptors/contextual-commits
```

Auto-detects your agent, installs the skills to the correct directory. That's it.

### What's Included

| File | What It Does |
|------|--------------|
| [`skills/contextual-commit/SKILL.md`](skills/contextual-commit/SKILL.md) | Teaches the agent the contextual commit format. Auto-invoked when committing. Produces structured action lines based on what happened in the session. |
| [`skills/recall/SKILL.md`](skills/recall/SKILL.md) | `/recall` — reconstructs development context from contextual commit history. |

### Usage

**Writing contextual commits** — just commit normally. The skill activates automatically when the agent writes a commit and produces action lines based on the session's conversation.

**Recalling context:**

```
/recall                         full session briefing for the current branch
/recall <scope>                 all action lines matching scope (prefix-matched)
/recall <action>(<scope>)       specific action type within a scope
```

You can invoke `/recall` explicitly, or let the agent call it on its own — it's a skill, so any agent that reads context before acting will reach for it naturally.



<!-- ---

For consistent scoping, broader context coverage, automated context maintenance, agent-native tooling, and richer context recall, see [Engraph](https://github.com/berserkdisruptors/engraph) — the context layer for agentic coding, where contextual commits are only the beginning. -->

## License

MIT