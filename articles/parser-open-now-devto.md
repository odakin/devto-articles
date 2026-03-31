---
title: Parsing Text Correctly Doesn't Mean Your Answers Are Right
published: false
description: My regex parser hit 98% coverage. Then I tried using the output to answer 'is this pharmacy open RIGHT NOW?' and discovered that correct parsing and correct answers are two different things.
tags: 'javascript, regex, opendata, webdev'
id: 3438185
---

> **Japanese version:** [Zenn (Japanese)](https://zenn.dev/odakin/articles/parser-open-now)
>
> Sequel to: [I Wrote 82 Regex Replacements to Parse 6,933 Time Format Variations](https://dev.to/odakin/i-wrote-82-regex-replacements-to-parse-6933-time-format-variations-from-a-government-dataset-4mfj)

## How It Started

I built a [pharmacy search tool](https://odakin.github.io/mhlw-ec-pharmacy-finder/) for Japan's emergency contraception dataset — 10,000+ pharmacies with free-text business hours, 6,933 unique formats, 82 regex replacements, 97% coverage. [Wrote about it.](https://dev.to/odakin/i-wrote-82-regex-replacements-to-parse-6933-time-format-variations-from-a-government-dataset-4mfj) Shipped it. Felt good.

Then I started actually using the thing.

Within two days I was annoyed. I'm looking at 50 pharmacies near me. It's Saturday afternoon. Most of them are closed. But I'm scrolling through all of them because there's no way to filter by "open right now."

The hours are already parsed into structured data. Checking `if (now >= open && now < close)` should be trivial. I sat down one evening and had it working by midnight.

And that's when things got interesting. Because the filter worked — it just gave wrong answers.

## The Gap Between Parsing and Interpreting

Here's what the parser produces:

```
Input:  月-金:9:00-18:00,土:9:00-13:00,日祝休み
Output: { schedule: [
           {days:["Mon","Tue","Wed","Thu","Fri"], open:"9:00", close:"18:00"},
           {days:["Sat"], open:"9:00", close:"13:00"}
         ],
         holidayClosed: true }
```

This is correct. The parser did its job perfectly.

Now: is this pharmacy open at 10 AM on Monday, January 12?

The schedule says Mon-Fri 9:00-18:00. So yes, it's open. The filter returns true.

Except January 12, 2026 is Coming-of-Age Day — a national holiday. The data says `日祝休み` ("closed Sundays and holidays"). The parser had already extracted `holidayClosed: true`. But my "open now" check was looking at the day-of-week schedule and ignoring the holiday flag entirely.

The parse was correct. The interpretation was wrong. The user sees "Open" and drives to a closed pharmacy.

For a tool that helps people access emergency medication, this is not an acceptable bug.

## Implementing a Holiday Calculator (For a Regex Parser)

Japan has 16 named holidays, but the actual count per year is higher because of substitute holidays (when a holiday falls on Sunday, Monday becomes a day off) and sandwich holidays (a weekday between two holidays becomes a holiday too).

Some holidays are fixed dates. Some follow "second Monday of September" rules. And the equinoxes — Spring and Autumn — are calculated from an astronomical formula:

```javascript
// Vernal equinox
const vernal = Math.floor(
  20.8431 + 0.242194 * (year - 1980)
  - Math.floor((year - 1980) / 4)
);
```

About 60 lines, no dependencies. Returns all 18 holidays for any given year through ~2099. I never expected a regex-based hours parser to need orbital mechanics, but here we are.

## The Cache That Lied

Same evening, another bug. The tool caches parsed results because re-parsing 10,000 entries on every page load is slow. Cache key: the raw hours text. Same text = same parse result.

But `月-金:9:00-18:00,日祝休み` means "open" on a regular Monday and "closed" on a holiday Monday. Same text, different answer depending on the date. The cache was happily serving Monday's "open" result on a holiday because the text hadn't changed.

I found this by accident — I happened to test on a day near a holiday. If I'd only tested on regular weekdays, this bug would have shipped and silently given wrong answers on every holiday.

Fix: evaluate holiday status outside the cache, on every request.

## Closed Days: The Bug That Took a Week to Notice

The "open now" filter shipped that night. It worked well. I used it daily for a week.

Then on a Wednesday, I noticed a pharmacy showing as "open" that I was pretty sure was closed on Wednesdays. I looked at the raw data:

```
月-金:9:00-18:00（水曜定休）
```

"Monday through Friday 9:00-18:00 (closed Wednesdays)."

The parser had correctly extracted the Mon-Fri schedule. Five days, 9 to 18. The `（水曜定休）` part — "closed Wednesdays" — was sitting right there in the text, but the parser wasn't extracting it. On Wednesdays, the filter said "open" because the schedule included Wednesday.

This was a whole new extraction problem. Closed-day declarations come in at least five flavors:

```
定休日：水・日              — explicit list
月-土（水曜を除く）          — "excluding X"
水曜休み                    — "X closed"
（木・土午後休診）            — afternoons only (medical clinics)
月-金（祝日除く）            — excluding holidays
```

I added a pre-normalization phase that extracts closed-day information before the main regex pipeline runs. Then `applyClosedDays()` removes those days from the weekly schedule — but only when the match is unambiguous. If a pharmacy has a separate Wednesday entry with different hours, the "closed Wednesdays" note contradicts the schedule, and I'd rather show both than guess which is right.

This is the design principle the whole project converges on:

> **Correct > Correct with caveat > Unknown > Wrong**

When you can't tell which interpretation is right, don't pick one. Show what you know and flag the uncertainty. For health-access tools, a confident wrong answer is worse than an honest "we're not sure."

## Where It Stands Now

| Metric | Original article | Now |
|--------|-----------------|-----|
| Pharmacy coverage | 97.1% (9,659/9,951) | 98.2% (9,768/11,734) |
| Medical clinics | — | 88.3% (2,739/3,107) |
| Holiday calculator | basic | 18 holidays/year, equinox formula |
| Closed-day extraction | none | 5 patterns |
| "Open now" filter | none | holiday-aware, cache-safe |

The denominator grew (the government keeps adding pharmacies) but the percentage still went up. Medical clinics were added as a second data source — different formatting conventions, different jargon ("morning clinic / afternoon clinic"), a different kind of hell.

## What I Learned

**Eat your own dogfood, immediately.** The "open now" feature wasn't a user request — it was me using my own tool and getting frustrated. If I hadn't used it myself within the first 48 hours, the holiday bug would have shipped quietly and given wrong answers for months.

**Parsing and interpreting are different layers.** The parser converts text to structure. Interpretation evaluates that structure against context (today's date, holiday calendar, closed-day rules). Most of the bugs in this round weren't parse failures. The data was structured correctly — it just answered the wrong question.

**Time-dependent code needs calendar-diverse tests.** "Works on Tuesday" doesn't mean "works on a holiday Monday." If your feature depends on the date, your test suite needs holidays, weekends, year boundaries, and leap years.

**Your cache doesn't know what day it is.** If the cache key doesn't include context that affects the answer, it will serve yesterday's truth today.

---

{% github odakin/mhlw-ec-pharmacy-finder %}

Previous article: [I Wrote 82 Regex Replacements to Parse 6,933 Time Format Variations](https://dev.to/odakin/i-wrote-82-regex-replacements-to-parse-6933-time-format-variations-from-a-government-dataset-4mfj)
