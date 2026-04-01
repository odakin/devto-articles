---
title: My AI Coding Assistant Misapplied the Design Principle I Gave It
published: true
description: 'I told Claude Code: ''missing info is better than wrong info.'' Claude took that principle and used it to justify silently discarding data. I trusted it and didn''t notice for a week.'
tags: 'javascript, regex, opendata, webdev'
id: 3438185
date: '2026-04-01T00:36:02Z'
---

> **Japanese version:** [Zenn (Japanese)](https://zenn.dev/odakin/articles/parser-open-now)
>
> Sequel to: [I Wrote 82 Regex Replacements to Parse 6,933 Time Format Variations](https://dev.to/odakin/i-wrote-82-regex-replacements-to-parse-6933-time-format-variations-from-a-government-dataset-4mfj)

## The Setup

In the [previous article](https://dev.to/odakin/i-wrote-82-regex-replacements-to-parse-6933-time-format-variations-from-a-government-dataset-4mfj), I had [Claude Code](https://docs.anthropic.com/en/docs/claude-code) build a parser for Japan's emergency contraception pharmacy dataset — free-text business hours, 6,933 formats, 82 regex replacements, 97% coverage.

The most important thing that came out of the project wasn't code. It was a design principle that Claude established and I approved:

> **Missing info > Wrong info.**

If the parser can't handle an entry, show the raw text. Don't guess. For a tool that helps people find emergency medication, a wrong answer is worse than no answer.

Claude wrote this into the project's design docs. Claude followed it. And Claude used it to justify something neither of us caught at the time.

## What Claude Did

The parser encountered data like this:

```
月-金:9:00-18:00（除く水曜）
```

"Monday to Friday 9:00-18:00 (excluding Wednesday)."

Claude's normalization pipeline stripped the parenthetical `（除く水曜）` and parsed the rest: Monday through Friday, 9 to 18. Wednesday included in the output.

From Claude's perspective, this was principled. The exclusion data was complex to parse, so it was dropped. The base schedule was preserved. Missing info > wrong info. Move on.

I reviewed the code. The logic seemed reasonable — parsing parenthetical exclusions reliably across dozens of formats is genuinely hard. I trusted the judgment and shipped it.

But here's the thing: the tool now actively displays Wednesday as a working day with hours 9:00-18:00. A user checks the schedule, sees Wednesday is listed, goes on Wednesday. The pharmacy is closed.

**That's not "missing info." That's wrong info.** Showing Mon-Fri including Wednesday when Wednesday is excluded is an incorrect schedule. Claude was generating the exact category of error the principle was supposed to prevent — and using the principle itself as justification.

## How I Caught It

After shipping, I checked the tool on a Saturday afternoon. Search results: 50 pharmacies, most closed but all showing. If someone actually needs emergency contraception, they'd be scrolling through closed pharmacies one by one. So I had Claude build an "Open Now" filter one evening.

The filter made the error impossible to miss. A schedule grid showing Wednesday hours is easy to overlook. A filter declaring "Open" on a Wednesday when the pharmacy is closed on Wednesdays — that's a binary, definitive wrong answer.

Three gaps surfaced:

**1. Closed-day data discarded.** The `（除く水曜）` example. Claude's normalization stripped it, my principle gave it cover. This was the design-level error.

**2. Holiday flag not wired in.** Claude had already extracted `holidayClosed: true` from `日祝休み` ("closed on holidays") but didn't connect it to the filter logic. Data existed, plumbing didn't. I didn't catch this during review either.

**3. Cache ignoring date context.** Same text, different correct answer on holidays vs regular days. The cache keyed on text alone. Another thing I should have caught.

Bugs 2 and 3 were implementation oversights — the kind of thing that happens in any codebase. Bug 1 was different. It was an AI applying a human's design principle in a way the human didn't intend, and the human trusting the AI enough to not notice.

## The Fix: A Principle Claude Can't Misapply

The old principle had two categories: Missing and Wrong. The gap between them was big enough for Claude to drive a truck through. "I dropped information but kept the base structure" fits comfortably in "Missing" if you squint — and Claude squinted.

The revised principle:

> **Correct > Correct with caveat > Unknown > Wrong**

Four levels. The critical addition is "Correct with caveat" — which means: if you have information you can't fully parse, **don't throw it away.** Show the base schedule and attach the unparseable part as a visible note.

Applied to the Wednesday problem:

| Level | What it looks like |
|-------|-------------------|
| **Correct** | Parse the exclusion. Remove Wednesday. Show Mon/Tue/Thu/Fri. |
| **Correct with caveat** | Can't parse the exclusion reliably, but show the schedule with a note: "※Closed Wednesdays" |
| **Unknown** | Can't parse any of it. Show raw text. |
| **Wrong** | Strip the exclusion. Show Mon-Fri as if Wednesday is normal. |

The old principle jumped from "can't parse it perfectly" straight to "drop it" — skipping the middle option entirely. The new principle forces that middle option to exist.

Implementation: a pre-normalization phase extracts closed-day info before the regex pipeline transforms the text. Clear exclusions get applied to the schedule. Ambiguous ones become visible notes (amber strip above the schedule grid). Nothing gets silently discarded.

## The Broader Lesson About AI + Design Principles

Claude Code is good at following rules literally. Give it "missing > wrong" and it will optimize for that — aggressively, consistently, without second-guessing. That's the value of AI coding. It's also the risk.

A human developer, encountering `（除く水曜）`, might have thought: "Wait, I'm showing Wednesday as open. Is that really 'missing' or is that 'wrong'?" Claude didn't have that hesitation. The principle said missing > wrong, the exclusion was hard to parse, so dropping it was the principled thing to do. Correct reasoning, wrong conclusion.

What I should have done:

1. **Written the principle more precisely.** "Missing" should have been defined as "genuinely absent from the output," not "present in the input but dropped during processing."
2. **Reviewed the normalization pipeline output, not just the code.** I read the code and it looked reasonable. I should have looked at specific examples of what the code was *producing* and asked: "Is this output correct?"
3. **Not trusted the AI's application of design principles without spot-checking.** Code review for AI-generated code needs to include output review.

## Numbers

| Metric | Original article | After fixes |
|--------|-----------------|-----|
| Pharmacy coverage | 97.1% (9,659/9,951) | 98.2% (9,768/11,734) |
| Medical clinics | — | 88.3% (2,739/3,107) |
| Closed-day extraction | none | 5 patterns |
| "Open now" filter | none | holiday-aware, cache-safe |

## Takeaways

**AI follows principles literally.** If your principle has a loophole, AI will find it — not maliciously, but because literal interpretation is what it does. Write principles that are hard to misapply.

**"Missing" and "wrong" need a clear boundary.** Data you had and threw away is not "missing." It's information loss that produces incorrect output. Adding "correct with caveat" as a middle tier forces the question: "Am I truly missing this info, or am I choosing to discard it?"

**Review AI output, not just AI code.** I reviewed the normalization logic and it was reasonable. I didn't look at what it produced for edge cases. The gap between "the code looks right" and "the output is right" is where AI errors hide.

**Binary features are accidental audits.** A schedule grid tolerates small inaccuracies because context is visible. A yes/no filter strips all context. If you want to find the holes in your AI-generated data pipeline, build a feature that makes binary decisions from its output.

---

{% github odakin/mhlw-ec-pharmacy-finder %}

Previous article: [I Wrote 82 Regex Replacements to Parse 6,933 Time Format Variations](https://dev.to/odakin/i-wrote-82-regex-replacements-to-parse-6933-time-format-variations-from-a-government-dataset-4mfj)
