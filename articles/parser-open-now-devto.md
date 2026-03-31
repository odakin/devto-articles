---
title: Your 97% Parser Shipped. Then You Wanted 'Open Now'.
published: false
description: A regex parser for messy government data hit 97% coverage and shipped. Then came the 'Open Now' filter — and a whole new category of bugs where correct parsing produced wrong answers.
tags: 'javascript, regex, opendata, webdev'
id: 3438185
---

> **Japanese version:** [Zenn (Japanese)](https://zenn.dev/odakin/articles/parser-open-now)
>
> Sequel to: [I Wrote 82 Regex Replacements to Parse 6,933 Time Format Variations](https://dev.to/odakin/i-wrote-82-regex-replacements-to-parse-6933-time-format-variations-from-a-government-dataset-4mfj)

## Previously

I built a [pharmacy search tool](https://odakin.github.io/mhlw-ec-pharmacy-finder/) for Japan's emergency contraception pharmacy dataset. The business hours field was free-text with 6,933 unique formats. I wrote a parser with 82 regex replacements, hit 97.1% coverage, and called it done.

The article got a nice reception. The tool shipped. And almost immediately, using my own tool, I wanted one more thing:

**"Show me only the pharmacies that are open right now."**

Sounds simple. The hours are already parsed. Just check if `now` falls within today's time range. Ship it in an evening.

The filter itself worked in one night. But it surfaced a class of bugs I hadn't imagined — cases where the parser was **technically correct but practically wrong** — and the fixes kept coming for days after.

## The Problem With "Correct"

The parser converts free text into structured data:

```
月-金:9:00-18:00,土:9:00-13:00,日祝休み
```
becomes:
```javascript
{ schedule: [
    { days: ["月","火","水","木","金"], open: "9:00", close: "18:00" },
    { days: ["土"], open: "9:00", close: "13:00" }
  ],
  holidayClosed: true }
```

This is correct. The parser did its job. But "open right now" means asking: **what does this schedule say about this specific moment?**

And that's where things broke.

## Bug #1: Holidays Are Invisible

Monday, 9:30 AM. The schedule says Mon-Fri 9:00-18:00. The filter says "Open."

Except today is Coming-of-Age Day. A national holiday. The data says `日祝休み` (closed on Sundays and holidays). The parser extracted `holidayClosed: true`. But the "open now" check was only looking at the weekday schedule.

Fix: implement a full Japanese holiday calculator. ~60 lines, zero dependencies. Fixed holidays, Happy Monday rules, equinoxes computed from astronomical formulae (good through ~2099), substitute holidays, sandwich holidays.

```javascript
// Vernal equinox from astronomical formula
const vernal = Math.floor(
  20.8431 + 0.242194 * (year - 1980)
  - Math.floor((year - 1980) / 4)
);
```

Now `isJapaneseHoliday(date)` returns true for all 18 annual holidays. The "open now" filter checks this before declaring a pharmacy open.

## Bug #2: Closed Days Hide Inside the Text

```
月水金:9:00-18:00（火木休み）
```

This says "open Mon/Wed/Fri, closed Tue/Thu." The old parser would parse the Mon/Wed/Fri schedule just fine. On the "open now" filter, Tuesday would show... nothing. No schedule entry for Tuesday. So the pharmacy wouldn't show up in results.

Correct behavior. But misleading for a different reason — what about this:

```
月-金:9:00-18:00（水曜定休）
```

"Mon-Fri 9:00-18:00, closed Wednesdays." The parser sees Mon-Fri and generates entries for all five weekdays. On Wednesday, the filter says "Open." But the pharmacy is actually closed.

The schedule was parsed correctly. The closed-day information was sitting right there in the text. The parser just wasn't extracting it.

### closedDays extraction

I added a pre-parse phase that pulls closed-day info before the main normalization pipeline touches the text:

```javascript
// 5 patterns for closed days:
// 1. 定休日リスト:   「定休日：水・日」
// 2. Xを除く:       「月-土（水曜を除く）」
// 3. 括弧内除外:     「月-金（祝日除く）」
// 4. X休み:         「水曜休み」
// 5. 括弧内休診:     「（木・土午後休診）」
```

Then after parsing, `applyClosedDays()` removes the closed days from the weekly schedule. But only when the match is unambiguous — if a pharmacy says `水曜定休` and the schedule has a separate Wednesday entry with different hours, I leave it alone rather than guess which one is right.

Conservative is the whole point. The design principle I converged on:

> **Correct > Correct with caveat > Unknown > Wrong**

Showing "we don't know" is always better than showing wrong information. For a tool that helps people access emergency medication, "the pharmacy is open" when it's actually closed isn't just a UI bug.

## Bug #3: Cache Meets Holidays

This one was sneaky. The tool caches parsed hours data to avoid re-parsing 10,000 entries on every page load. The cache key includes the raw hours text — if the text hasn't changed, use the cached parse.

But holiday status changes daily. The same schedule text means "open" on a regular Monday and "closed" on a holiday Monday. The cache was returning Monday's regular schedule even on holidays, because the raw text hadn't changed.

Fix: include the current date in the cache context, and re-evaluate holiday status on every load even for cached entries.

## The Coverage Kept Climbing

While building the "Open Now" filter, I kept tripping over edge cases that the original parser didn't handle. Each fix improved the overall parse rate:

| Version | Pharmacy coverage | Notes |
|---------|------------------|-------|
| Original article | 97.1% (9,659/9,951) | 82 replacements |
| Current | 98.2% (9,768/11,734) | Dataset also grew |

The denominator grew too (more pharmacies added to the government dataset), but the percentage still went up. And a new data source — medical clinics — was added at 88.3% coverage (2,739/3,107).

The remaining failures are things like `外来診療時間内` ("during outpatient consultation hours") which is not a time specification but a pointer to information that doesn't exist in the dataset. You can't parse what isn't there.

## What I Learned (This Time)

**Parsing and interpreting are different problems.** The parser converts text to structure. The "Open Now" filter interprets that structure in the context of a specific moment. Most of the bugs in this round weren't parse failures — they were interpretation failures. The data was structured correctly but answered the wrong question.

**"Works on my machine" has a temporal equivalent.** "Works on a regular Tuesday" doesn't mean it works on a holiday Monday. Time-dependent features need to be tested across the full calendar, not just "right now."

**The cache is part of the interpretation layer.** If your cache doesn't know about context (date, holidays, user timezone), it will serve stale truths.

**Conservative defaults save you.** Every ambiguous case I punted on ("show raw text instead of guessing") turned out to be the right call. The cases I tried to be clever about were the ones that bit me.

---

{% github odakin/mhlw-ec-pharmacy-finder %}

Previous article: [I Wrote 82 Regex Replacements to Parse 6,933 Time Format Variations](https://dev.to/odakin/i-wrote-82-regex-replacements-to-parse-6933-time-format-variations-from-a-government-dataset-4mfj)
