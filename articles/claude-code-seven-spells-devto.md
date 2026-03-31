---
title: 7 Phrases That Make Claude Code Actually Reliable — Lessons From 20+ Projects
published: true
description: 'Battle-tested prompting patterns and workflow tricks for Claude Code that prevent context loss, catch documentation drift, and stop recurring mistakes.'
tags: 'ai, productivity, devtools, claude'
id: 3380539
date: '2026-03-21T14:15:34Z'
---

> **Japanese version:** [Zenn (Japanese)](https://zenn.dev/odakin/articles/cc-seven-spells)
>
> **Note:** This is a sequel to [Solving Claude Code's Memory Loss](https://dev.to/odakin/solving-claude-codes-memory-loss-multi-project-design-patterns-4f5n). It assumes familiarity with the CLAUDE.md / SESSION.md pattern.

Claude Code is powerful. It's also sloppy if you let it be.

After running 20+ projects with the [CLAUDE.md / SESSION.md](https://dev.to/odakin/solving-claude-codes-memory-loss-multi-project-design-patterns-4f5n) pattern, I've found specific phrases and habits that dramatically improve output quality. Not vague "prompt engineering" — concrete, repeatable patterns born from real incidents.

Here are seven of them.

---

## 1. "Resume project X" — Start Every Session Fresh

**Never continue a long conversation.** Start a new session every time. Say "resume project X" and nothing else.

```
You: "Resume the pharmacy project."
Claude: [reads CLAUDE.md → reads SESSION.md → understands current state → continues work]
```

### Why this works

- **Context compression risk drops to zero** (context never accumulates)
- SESSION.md was updated at the end of the last session, so you always start from the **latest state**
- If the previous session went off the rails, you get a free reset

### Prerequisite

**SESSION.md must be up to date.** If it's stale, "resume" takes you back to an old state. The auto-update protocol (update SESSION.md on task completion, on important decisions, before every push) is mandatory.

### When to start a new session

- Switching projects
- Same project, but the conversation is getting long
- Natural breakpoints (after a commit, after a major feature)

---

## 2. "Check consistency, non-contradiction, and efficiency" — The Pre-Push Incantation

Say this before every `git push`. Claude will cross-check your documentation against reality.

```
You: "Check consistency, non-contradiction, and efficiency. Push."
```

### What it catches

This is not exaggeration — **it finds something almost every time.** Real examples:

| Issue found | Root cause |
|-------------|-----------|
| "4 FAQ items" but actually 6 | Added 2 items, forgot to update the count in SESSION.md |
| 8 repos listed with wrong visibility | Wrote "private" from memory without running `gh repo view` |
| Heading `## User Preferences` appeared twice | Section insertion duplicated the heading |
| A→B→A circular reference | Two files each said "the source of truth is in the other file" |

### Add "safety" for public repos

When working on public repos, add "safety" to the incantation:

```
You: "Check consistency, non-contradiction, efficiency, and safety. Push."
```

Claude will grep for PII, private repo names, email addresses, and credentials.

---

## 3. "Think deeply about this" — Stop Getting Surface-Level Answers

Claude defaults to quick answers. **Explicitly asking it to think deeply** produces trade-off analysis, alternative evaluation, and edge case consideration instead of the first thing that comes to mind.

### Real example: Feature placement

Adding an FAQ section to a site:

**Without "think deeply":**
```
Claude: I'll add it to the FAQ section. [immediately edits code]
```

**With "think deeply":**
```
Claude:
  Option A: FAQ only — pros/cons
  Option B: Header expansion — pros/cons
  Option C: Contextual note near filters — pros/cons
  Option D: Side-by-side comparison card — pros/cons
  Option E: FAQ + (?) icon tooltip — pros/cons
  Recommendation: A is optimal because...
```

The second version leads to dramatically better decisions. Use this for anything where **the answer isn't obvious**: architecture, feature prioritization, UI placement, API design.

---

## 4. Record the WHY With a Date — Not Just the WHAT

When you decide to implement or *not* implement a feature, record the reasoning in SESSION.md. Without this, the next session (or you, three months later) will re-debate the same question.

### Bad

```markdown
- Don't implement "currently open" filter
```

### Good

```markdown
- Don't implement "currently open" filter (2026-03-21)
  — Data source is secondary (parser estimate, 97.1% coverage),
  ~290 pharmacies can't be classified, nearest pharmacy might be
  among the unclassifiable, and showing all entries does no harm
```

### Why include the date?

Circumstances change. Without a date, decisions calcify into unwritten laws that nobody questions. With a date, you can revisit: "we decided this 6 months ago — is the reasoning still valid?"

### When this saved me

"Didn't we decide station search wasn't needed?" With the WHY recorded, I could answer immediately. Without it, we'd have re-analyzed from scratch.

---

## 5. "What are we losing to [competitor]?" — Competitive Analysis That Doesn't Over-Build

Have Claude fetch a competitor's site and analyze the gaps.

```
You: "What are we losing to this site? https://competitor.example.com"
```

### The key insight: "losing" ≠ "should implement"

After the analysis, **evaluate each gap using #3 ("think deeply").**

In a real case, a competitor had 3 features we lacked. After deep evaluation:
- One was unnecessary (existing feature already covered it)
- One was out of scope (not our site's responsibility)
- One needed only an FAQ addition (no real engineering)

The combo of "what are we losing?" + "think deeply about each gap" prevents knee-jerk feature building.

---

## 6. Feedback Memories — Stop Making the Same Mistake Twice

Claude Code has a [memory system](https://docs.anthropic.com/en/docs/claude-code/memory) (`~/.claude/`). Use it to **save lessons from mistakes.** Claude won't repeat the same error across sessions.

### Format

```markdown
---
name: Check existing repos before creating new ones
description: Always check repo list before proposing a new repo
type: feedback
---

Before proposing a new repo, check the repo index and see if
an existing repo can handle it.

**Why:** Created an articles/ directory inside project-X
when a dedicated articles repo already existed.

**How to apply:** Whenever the thought "let's create a new
repo for Y" comes up, check the repo index first.
```

### Critical: Save successes too

If you only save corrections ("don't do X"), Claude becomes overly cautious and hesitant. **Save what worked well, too.**

Examples:
- "Bundling this refactor into one PR was the right call — splitting would have been churn"
- "FAQ addition was sufficient here — no need for a full feature"

Correction-only memory creates an AI that's afraid to act. Balanced memory creates one with good judgment.

---

## 7. Single Source of Truth — Kill Circular References

Pick exactly **one place** for each piece of information. Everything else references it. Never duplicate content.

### A real circular reference I created

```
CONVENTIONS.md: "Repo list source of truth is in MEMORY.md"
MEMORY.md: "Repo list is in CONVENTIONS.md §9"
CONVENTIONS.md §9: Safety rules (not the repo list)
```

**The source of truth was nowhere.** Result: Claude created an `articles/` directory inside a project repo even though a dedicated articles repo already existed. Because the repo list didn't actually exist anywhere.

### After fixing

```
CONVENTIONS.md: "Repo list source of truth is in MEMORY.md,
                section 'Repo Index'"
MEMORY.md → [actual repo table lives here]
```

### Tips

- **Specify the section name**, not just the file ("MEMORY.md" is too vague — which section?)
- Don't repeat information outside the source of truth
- Circular references are automatically caught by #2 (the consistency check)

---

## Summary

| # | Phrase | When to use |
|---|--------|-------------|
| 1 | "Resume project X" | Every session start |
| 2 | "Check consistency, non-contradiction, efficiency" | Before every push |
| 3 | "Think deeply" | Non-obvious decisions |
| 4 | Record the WHY with a date | Feature accept/reject decisions |
| 5 | "What are we losing to...?" | Feature ideation |
| 6 | Feedback memories | After mistakes *and* successes |
| 7 | Single source of truth | Information architecture |

The underlying principle: **don't let Claude wing it — enforce quality structurally.**

SESSION.md is a state machine. The consistency check is a pre-push quality gate. Feedback memories are a learning system. Safety rules are guardrails.

Claude Code is powerful, but left unsupervised it produces sloppy work. These phrases aren't magic — they're **mechanisms** that structurally guarantee quality. With them, Claude Code becomes a reliable pair programmer.

---

## Resources

{% github odakin/claude-config %}

Previous article: [Solving Claude Code's Memory Loss — Multi-Project Design Patterns](https://dev.to/odakin/solving-claude-codes-memory-loss-multi-project-design-patterns-4f5n)
Next article: [Stop Babysitting Your AI — How I Made Claude Code Enforce Its Own Rules](https://dev.to/odakin/stop-babysitting-your-ai-how-i-made-claude-code-enforce-its-own-rules-jmm)
