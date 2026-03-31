---
title: Stop Babysitting Your AI — How I Made Claude Code Enforce Its Own Rules
published: true
description: 'Hooks that block bad writes, file budgets that prevent bloat, and a 4-axis review that catches what humans miss. All enforced by Claude itself.'
tags: 'ai, productivity, devtools, claude'
id: 3436511
date: '2026-03-31T13:42:10Z'
---

> This is the third in a series. The first two: [Solving Claude Code's Memory Loss](https://dev.to/odakin/solving-claude-codes-memory-loss-multi-project-design-patterns-4f5n) and [7 Phrases That Make Claude Code Actually Reliable](https://dev.to/odakin/7-phrases-that-make-claude-code-actually-reliable-lessons-from-20-projects-52e5). You don't need to read them, but they give context.

Here's something I learned the hard way: writing rules for an AI coding assistant is easy. Getting it to actually follow them is a different sport entirely.

I run 25 repos through Claude Code. I have a shared conventions file, session logs, templates, the whole setup. It works. But within two weeks of scaling it up, I found myself repeatedly fixing the same three problems:

1. Claude writing information in the wrong place
2. Convention files bloating until they defeated their own purpose
3. Documentation silently drifting out of sync with reality

My first instinct was to add more rules. "Don't write project state to memory." "Keep SESSION.md under 80 lines." "Check references before pushing." Rules on top of rules.

It didn't work. Rules get autocompacted. Rules get ignored. Rules require a human to enforce them, and humans forget.

What actually worked was making Claude enforce the rules on itself.

{% github odakin/claude-config %}

---

## The Pattern: Don't Add More Rules. Add Enforcement.

Every problem had the same shape: Claude does something wrong, I notice too late, I fix it, I write a rule, Claude does it again next week.

The fix was also the same shape every time: instead of telling Claude what to do, I built a mechanism that physically prevents the wrong thing or automatically detects it.

| Problem | Rule (didn't work) | Mechanism (works) |
|---------|-------------------|-------------------|
| Writes to wrong location | "Only write preferences to memory" | Hook that blocks memory writes |
| Files get too long | "Keep it concise" | Explicit line budget + auto-housekeeping |
| Docs drift from reality | "Check before pushing" | 4-axis review with `grep` verification |

Let me walk through each one.

---

## 1. Blocking Wrong Writes with Hooks

Claude Code has a memory system (`~/.claude/` files) and I also use `SESSION.md` for tracking work state. The problem: Claude defaults to stuffing everything into memory. Project status, design decisions, task progress — all shoved into memory where it's invisible to other machines (memory doesn't go into Git) and only loaded when Claude feels like it.

SESSION.md is in the repo. It syncs via Git. It gets loaded every time through the "How to Resume" flow. Memory doesn't have any of those properties.

I wrote a rule saying "only write user preferences to memory." Claude followed it for about three days before autocompact wiped the instruction and it went right back to its old habits.

### The fix: a PreToolUse hook

Claude Code supports [hooks](https://docs.anthropic.com/en/docs/claude-code/hooks) — shell scripts that run before or after tool calls. I wrote one that intercepts writes to the memory directory:

```bash
# memory-guard.sh (simplified)
INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')

# Not a memory write? Let it through.
[[ "$FILE_PATH" != *"/.claude/projects/"*"/memory/"* ]] && exit 0
# Index file is fine
[[ "$FILE_PATH" == */MEMORY.md ]] && exit 0

# Block everything else
cat >&2 << 'EOF'
BLOCKED: Writing to memory directory.
Check the destination table in CONVENTIONS.md before proceeding.
EOF
exit 2  # exit 2 = block the tool call
```

When Claude tries to write to memory, it gets blocked and told to consult the decision table. The table looks like this:

| What you're writing | Where it goes |
|---|---|
| User preferences, feedback | Memory |
| Current work state, tasks | SESSION.md |
| Permanent specs, structure | CLAUDE.md |
| Design rationale | DESIGN.md |
| Cross-project rules | CONVENTIONS.md |
| Anything derivable from code/git | Nowhere |

That last row matters more than you'd think. Recording things `grep` can tell you is how files bloat.

Claude reads the table, realizes "oh, this is project state, not a preference," and writes to SESSION.md instead. It's like a compiler error for documentation.

---

## 2. File Budgets Against Bloat

My shared conventions file (CONVENTIONS.md) started at 7 sections. Then I added LaTeX equation safety rules. Then Google Calendar MCP rules. Then shared-repo Git workflow guards. Before I knew it: 13 sections.

The irony: CONVENTIONS.md exists to prevent context loss from autocompact. A bloated CONVENTIONS.md accelerates autocompact. A recursive problem.

### The fix: split by domain

I moved domain-specific rules to separate files:

```
claude-config/
├── CONVENTIONS.md      # universal rules (down to 6 sections)
└── conventions/
    ├── latex.md        # equation safety, compiler settings
    ├── mcp.md          # MCP connector rules
    └── shared-repo.md  # multi-author Git workflow
```

A physics paper repo references `conventions/latex.md`. A web app repo doesn't. Simple, but it cut CONVENTIONS.md in half.

### SESSION.md gets a budget too

SESSION.md is a work log that Claude updates automatically. Without limits, it grows into a 200-line memoir of every completed task and implementation detail. But its only job is to answer one question after autocompact: "Where was I?"

I set a budget of ~80 lines and made Claude do housekeeping on every push:

- Completed tasks: delete (they're in `git log`)
- Implementation details: delete (they're in commit messages)
- Permanent decisions: move to CLAUDE.md, delete from SESSION.md

This happens automatically. I don't think about it.

---

## 3. Catching Documentation Drift with a 4-Axis Review

SESSION.md says "4 data sources." Actual count: 6.
CLAUDE.md says "images go in `img/`." They moved to `assets/` weeks ago.
README references "section 9." It's section 5 now.

Documentation rots the moment you write it. I had a "check before push" rule, but it was too vague. "Looks fine" is not a review.

### The fix: four explicit axes

Before every `git push`, Claude runs a structured review:

| Axis | What to check |
|---|---|
| **Consistency** | Do numbers, references, and terms match across files? Verify with `grep`. |
| **Non-contradiction** | Does the new text conflict with existing rules or templates? |
| **Efficiency** | Any duplicated info? Is SESSION.md within budget? Dead text? |
| **Safety** | Any PII or credentials in a public repo? |

The invocation is just:

> "Check consistency, non-contradiction, and efficiency. Push."

For public repos, add safety.

This catches things every single time. Today, while rewriting the README for claude-config, it found three issues:

1. Step numbering mismatch between CLAUDE.md ("1, 1b, 2-6") and README ("1-7")
2. A missing "external references" entry in a decision table
3. A code example referencing old section "section 9" (now section 5)

All fixed before push. A human reviewer would probably miss at least two of these.

---

## Bonus: LaTeX Unicode Auto-Fix

If you work with collaborators who use Word, you know the pain: curly quotes and em-dashes sneak into `.bib` files and BibTeX chokes. I wrote a pre-commit hook that auto-converts Unicode to LaTeX commands on every commit. `setup.sh` detects LaTeX repos and installs it automatically. One less reason to dread `bibtex` errors at 2 AM.

---

## The Meta-Lesson

Every fix follows the same pattern: I stopped telling Claude what to do and started building mechanisms that make the wrong thing impossible (or at least detectable).

- **Hooks** make wrong writes impossible
- **Budgets** make bloat self-correcting
- **Structured reviews** make drift detectable

None of these require me to remember anything. Claude enforces them on itself, every time, without getting bored or distracted.

The first article in this series was about making Claude manage its own memory. This one is about making Claude manage its own quality. The principle is the same: if you want reliability from an AI tool, don't rely on yourself to enforce it. Build the guardrail into the workflow and let the AI run into it.

---

Repo: [odakin/claude-config](https://github.com/odakin/claude-config)

Previous articles:
- [Solving Claude Code's Memory Loss](https://dev.to/odakin/solving-claude-codes-memory-loss-multi-project-design-patterns-4f5n)
- [7 Phrases That Make Claude Code Actually Reliable](https://dev.to/odakin/7-phrases-that-make-claude-code-actually-reliable-lessons-from-20-projects-52e5)
