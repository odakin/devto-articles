---
title: "How I Misapplied My Own Design Principle and Shipped Wrong Answers"
published: false
description: "I built a parser with the principle 'missing info is better than wrong info.' Sounds solid. Turns out I was categorizing 'wrong' as 'missing' and didn't notice for a week."
tags: 'javascript, regex, opendata, webdev'
id: 3438185
---

> **Japanese version:** [Zenn (Japanese)](https://zenn.dev/odakin/articles/parser-open-now)
>
> Sequel to: [I Wrote 82 Regex Replacements to Parse 6,933 Time Format Variations](https://dev.to/odakin/i-wrote-82-regex-replacements-to-parse-6933-time-format-variations-from-a-government-dataset-4mfj)

## The Principle That Felt Right

In the [previous article](https://dev.to/odakin/i-wrote-82-regex-replacements-to-parse-6933-time-format-variations-from-a-government-dataset-4mfj), I built a parser for Japan's emergency contraception pharmacy dataset — free-text business hours, 6,933 formats, 82 regex replacements, 97% coverage. I also established a design principle:

> **Missing info > Wrong info.**

If you can't parse it, show the raw text. Don't guess. For a tool that helps people find emergency medication, a wrong answer is worse than no answer.

This felt solid. I even showed how I intentionally dropped coverage from 96.6% to 96.3% by removing nth-weekday guesses — because guessing "every Saturday" when the data said "1st and 3rd Saturday" was wrong, and I'd rather show nothing than a lie.

The principle was documented. The reasoning was sound. And I was misapplying it from day one.

## What I Got Wrong

The parser encountered data like this:

```
月-金:9:00-18:00（除く水曜）
```

"Monday to Friday 9:00-18:00 (excluding Wednesday)."

My normalization pipeline stripped the parenthetical `（除く水曜）` and parsed the rest: Monday through Friday, 9 to 18. Wednesday was included in the output.

I categorized this as "missing info." The exclusion data was lost, sure, but the base schedule was preserved. Missing info > wrong info. Ship it.

Except it's not missing. The tool actively displays Wednesday as a working day with hours 9:00-18:00. Someone checks the schedule, sees Wednesday is listed, goes on Wednesday. The pharmacy is closed.

**That's not "missing info." That's wrong info.** Showing Monday-to-Friday-including-Wednesday when Wednesday is excluded is an incorrect schedule. I was generating the exact category of error my own principle was supposed to prevent, and my principle gave me cover to not notice.

## How I Noticed

After shipping, I checked the tool on a Saturday afternoon. The search results showed 50 pharmacies, most of them obviously closed. If you actually need emergency contraception, you don't want to scroll through closed pharmacies reading hours. So I built an "Open Now" filter one evening — check today's schedule against current time, show or hide.

The filter worked immediately. It also made the hidden bugs visible.

A pharmacy showing as "Open" on a Wednesday when it's closed on Wednesdays is barely noticeable in a schedule grid — users might catch it, might not. But in an "Open Now" filter it's a definitive, binary, wrong answer. The filter was a magnifying glass on every gap in the parser.

There were three:

**1. Closed-day data being discarded.** The example above — `（除く水曜）` stripped, Wednesday displayed as open. This was the philosophical error.

**2. Holiday flag not connected.** The parser already extracted `holidayClosed: true` from `日祝休み` ("closed on holidays"). The filter didn't check it. Holiday Mondays showed as "Open" because the weekday schedule said Mon-Fri. The data was there; the wiring wasn't.

**3. Cache ignoring date context.** Same schedule text returns "open" on a regular Monday, "closed" on a holiday Monday. The cache keyed on text alone, so it served regular-Monday answers on holidays.

Bug 2 and 3 were implementation mistakes. Bug 1 was the design mistake — the one where the principle itself was providing false confidence.

## The Fix: A Better Principle

The old principle had two categories: Missing and Wrong. That's not enough. When I stripped `（除く水曜）` and showed Mon-Fri, I was telling myself it was "missing." I needed a principle that wouldn't let me do that.

The revised hierarchy:

> **Correct > Correct with caveat > Unknown > Wrong**

Four levels instead of two. And the critical distinction: **"Unknown" means genuinely unknown.** Not "I had the information and threw it away." Throwing away information you had and displaying the remainder as if it's complete — that's Wrong, not Unknown.

Applied to the Wednesday problem:

- **Correct:** Parse `月-金（除く水曜）`, remove Wednesday, show Mon/Tue/Thu/Fri.
- **Correct with caveat:** Can't reliably parse the exclusion, but show the schedule with a note: "※水曜を除く"
- **Unknown:** Can't parse any of it, show raw text.
- **Wrong:** Strip the exclusion, show Mon-Fri as if Wednesday is a normal working day. ← what I was doing

The fix added a pre-normalization phase that extracts closed-day information before the regex pipeline transforms the text. Unambiguous exclusions get applied to the schedule. Ambiguous ones become visible notes (amber strip above the schedule grid). Nothing gets silently thrown away.

## The Concrete Changes

Holiday flag: already extracted, just not wired into the filter. Five-minute fix once noticed. The holiday calculator (~60 lines, astronomical equinox formula, the works) was already in place.

Closed-day extraction: five patterns — explicit lists (`定休日：水・日`), exclusion clauses (`水曜を除く`), closed-day statements (`水曜休み`), afternoon closures (`木・土午後休診`), holiday exclusions (`祝日除く`). When the match is ambiguous, show the note instead of guessing.

Cache: evaluate holiday/closed-day status outside the cache, on every request. The schedule structure can be cached; the "is it open RIGHT NOW" judgment can't.

Notes display: an amber strip above the schedule grid showing exclusion and holiday info the filter might not fully capture. When the algorithm can't guarantee correctness, show the user what it's working with.

## Numbers

| Metric | Original article | After fixes |
|--------|-----------------|-----|
| Pharmacy coverage | 97.1% (9,659/9,951) | 98.2% (9,768/11,734) |
| Medical clinics | — | 88.3% (2,739/3,107) |
| Closed-day extraction | none | 5 patterns |
| "Open now" filter | none | holiday-aware, cache-safe |

## What I Learned

**Two-category principles hide errors.** "Missing vs Wrong" sounds clean but creates a gray zone where you can categorize active misinformation as mere absence. Adding "Correct with caveat" as a middle option forced me to ask: "Am I actually showing nothing, or am I showing something incomplete and passing it off as complete?"

**Principles need to survive feature additions.** The original principle was designed for the parser's display mode, where raw-text fallback made "missing" genuinely safe. The "Open Now" filter had no fallback — it was binary. The principle should have been re-examined when the use case changed. It wasn't.

**The "Open Now" filter was an accidental audit.** A schedule grid can tolerate small inaccuracies because users see surrounding context. A binary yes/no filter strips all context. Every gap becomes a wrong answer. If you want to find holes in your data pipeline, build a feature that makes binary decisions from its output.

**Document your design revisions, not just your designs.** The commit that revised the principle includes the retrospective: what the old principle was, how it was misapplied, why the new one is better. Future contributors (including future me) can see not just what the rule is, but what went wrong when it was simpler.

---

{% github odakin/mhlw-ec-pharmacy-finder %}

Previous article: [I Wrote 82 Regex Replacements to Parse 6,933 Time Format Variations](https://dev.to/odakin/i-wrote-82-regex-replacements-to-parse-6933-time-format-variations-from-a-government-dataset-4mfj)
