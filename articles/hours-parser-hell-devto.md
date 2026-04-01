---
title: 'I Wrote 82 Regex Replacements to Parse 6,933 Time Format Variations From a Government Dataset'
published: true
description: 'A Japanese government pharmacy dataset has free-text business hours with 6,933 unique formats. This is the story of parsing them — and knowing when to stop.'
tags: 'javascript, regex, opendata, webdev'
id: 3380538
date: '2026-03-21T14:15:31Z'
---

> **Note:** This article is also available in [Japanese](https://zenn.dev/odakin/articles/zenn-hours-parser/).

## The Setup

Japan's Ministry of Health publishes a list of ~10,000 pharmacies that dispense emergency contraception. I built a [search tool](https://odakin.github.io/mhlw-ec-pharmacy-finder/) for it.

The dataset has a `hours` field. Business hours. How bad could it be?

```
Mon-Fri:9:00-18:00,Sat:9:00-13:00
```

Split on `,`, split on `:`, parse the range. One regex. Done.

**First version coverage: 89.4%.**

Over 10% of entries failed to parse. Here's why.

## The Horror: Free-Text Entry With No Schema

There's no format specification. Each pharmacy across 47 prefectures types whatever they want. Here are real entries that all mean "Monday to Friday, 9:00 to 18:00":

```
月-金:9:00-18:00           ← clean
月～金：９：００～１８：００  ← full-width everything
⽉-⾦:9:00-18:00           ← ...what?
月曜日～金曜日 9時～18時    ← kanji time notation
(月火水木金)9:00-18:00     ← parenthesis grouping
平日:9:00-18:00            ← "weekdays" in Japanese
月から金は9時から18時       ← literal prose
```

**All the same meaning.**

My job: funnel all of these into a single canonical form. The function that does this calls `.replace()` **82 times**.

## Hell #1: Characters That Look Identical But Aren't

```javascript
"月"  // U+6708 — correct
"⽉"  // U+2F47 — CJK Compatibility Ideograph
```

Can you tell them apart? **You can't.** But `/[月火水木金土日]/` only matches the first one.

One entire prefecture's data used CJK compatibility characters for all day-of-week kanji. Every entry looked perfect to the human eye. The parser skipped all of them. Debugging took half a day.

```javascript
.replace(/⽉/g, "月").replace(/⽕/g, "火").replace(/⽔/g, "水")
.replace(/⽊/g, "木").replace(/⾦/g, "金").replace(/⼟/g, "土").replace(/⽇/g, "日")
```

## Hell #2: Five Kinds of Colons

```javascript
"："  // U+FF1A Fullwidth colon (3,587 entries — 36% of all data)
"∶"  // U+2236 RATIO
"︓"  // U+FE13 PRESENTATION FORM
"ː"  // U+02D0 MODIFIER LETTER TRIANGULAR COLON
":"  // U+003A Normal colon
```

All intended as colons. **36% of the entire dataset doesn't use the normal colon.**

Dashes are worse — **nine varieties**:

```javascript
"－" "‐" "−" "–" "—" "ー" "―" "ｰ" "-"
```

That `ー` is a **katakana vowel lengthener**. People use it as a hyphen.

## Hell #3: "FridaySunday"

To normalize days, I strip `曜日` (day-of-week suffix). Simple enough:

```
月曜日-金曜日,日曜日  →  月-金,日
```

But the data contains `金曜日曜` — "Friday" and "Sunday" concatenated without a separator.

```
月曜-金曜日曜9:00-18:00
```

Naive stripping of `曜日?`:

```
月曜-金曜日曜        ← input
  ↓ match 曜日?
月 - 金              ← "金曜日" matched, ate "日" (Sunday)
```

**Sunday vanished.**

Fix: lookahead assertion.

```javascript
.replace(/曜(?:日(?!曜))?/g, "")
```

Don't eat `日` if it's followed by `曜` (meaning it's the start of the next day name).

But now the result is `月-金日` — "Friday" and "Sunday" are glued together. Another fix:

```javascript
.replace(/([月火水木金土日])-([月火水木金土日])([月火水木金土日])/g, "$1-$2・$3")
```

Every bug fix spawns a new bug.

## Hell #4: Commas Mean Three Different Things

```
月-金:9:00-18:00,土:9:00-13:00    ← segment separator
月-金:9:00-13:00,14:00-18:00      ← AM/PM split shift
月,水,金:9:00-18:00               ← day enumeration
```

I started with `.split(",")`. The second pattern splits AM and PM into separate segments — PM has no day, parse fails. The third splits into `月`, `水`, `金:9:00-18:00` — Monday and Wednesday have no hours.

**You have to determine the meaning of each comma from context.**

```javascript
// Heuristics:
// - "day+time" after comma → new segment
// - "time only" after comma → append to previous segment (split shift)
// - "day only" after comma → merge with next element (enumeration)
```

## Hell #5: Seven Ways to Separate Days From Hours

```
月-金:9:00-18:00     ← colon
月-金 9:00-18:00     ← space
月-金9:00-18:00      ← direct concatenation
(月火金)9:00-18:00   ← parentheses
月-金/9:00-18:00     ← slash
[月-金]9:00-18:00    ← brackets
月-金;9:00-18:00     ← semicolon
```

## Hell #6: Typos in Time Values

```
9:00-8:00    ← closing before opening (probably 18:00)
8:30-6:00    ← probably 16:00 or 18:00
80:30-88:50  ← how
```

If you parse a typo and display "closes at 8:00," someone will show up at 4 PM to an open pharmacy and... wait, no, the opposite. Someone will *not* show up because they think it's closed.

Validation rejects anything outside 0:00–29:59, falling back to raw data display.

(29:59 because Japan uses "25:00" to mean 1:00 AM the next day for late-night businesses.)

## The Biggest Design Decision: Wrong Is Worse Than Missing

At 97% coverage, the question becomes: what about the remaining 3%?

The answer: **don't parse them.** Show the raw data.

This site is a **pharmacy search for emergency contraception**. Users are under time pressure. Showing "this pharmacy is open on Saturday" when it isn't could mean someone misses their window. **That's not recoverable.**

Example:

```
Mon-Fri:9:00-18:00,Sat:???,Sun:9:00-14:00
```

If Saturday is unparseable, I have two choices:

1. Skip Saturday, show Mon-Fri + Sun → user might assume "closed on Saturday"
2. Fail the whole entry, show raw text → user reads it themselves

**I chose 2.** No partial parses. All or nothing.

The same logic applies to "nth weekday" patterns:

```
第1・3土曜:9:00-12:00  (= "1st and 3rd Saturday only")
```

The parser outputs weekly schedules (`{day: "Sat", open: "9:00", close: "12:00"}`). There's no field for "which week of the month."

First implementation: display as "every Saturday." But **that's a lie.** Someone showing up on the 2nd Saturday finds a closed pharmacy. For medication access, you don't get to lie.

Fix: skip the entire entry. Saturday info is lost. But a gap is better than a lie.

## Holiday Hell: One More Layer

After hitting 97.1%, I noticed another problem:

```
月-金:9:00-18:00,日祝休み  (= "closed on Sundays and holidays")
```

The parser doesn't know about holidays. It would show "Open" on a holiday Monday.

**I implemented Japan's entire holiday calendar.** ~60 lines, zero dependencies.

- Fixed holidays (New Year's, National Foundation Day, Emperor's Birthday...)
- Happy Monday holidays (Coming of Age Day, Marine Day, Respect for the Aged Day, Sports Day)
- Vernal/Autumnal equinox (**calculated from an astronomical formula**, valid through ~2099)
- Substitute holidays (if a holiday falls on Sunday, Monday becomes a holiday)
- Citizen's holidays (a weekday sandwiched between two holidays)

```javascript
// Vernal equinox formula
const vernal = Math.floor(
  20.8431 + 0.242194 * (year - 1980)
  - Math.floor((year - 1980) / 4)
);
```

## Coverage Over Time

```
89.4% → 95.7% → 96.6% → 96.3%(!) → 96.9% → 97.1%
```

Notice the **drop** from 96.6% to 96.3%. That's where I stopped lying about "every Saturday" and started skipping nth-weekday entries. Coverage went down. Accuracy went up. **Coverage is a quality indicator, not a target.**

## Why I Stopped at 97.1%

The remaining 2.9%:

```
月～土：9:00~20：00、お客様感謝デーを除く日・祝：9:00~18:00
("except Customer Appreciation Days")
```

Natural language conditions.

```
月/水/金曜日第2/4火曜日第1/3/5土曜日9:00-18:00
```

Days, ordinals, and times in an undifferentiated blob.

```
月	09:00	19:00
火	09:00	19:00
```

Copy-pasted from Excel. Tab-separated.

Each edge case means a new `.replace()` that could interfere with the existing 82. Adding one pattern for 3 entries risks regressions across 9,951. Not worth it.

Unparseable pharmacies show their raw data. Users can read Japanese. It's fine.

## By The Numbers

| Metric | Value |
|--------|-------|
| Input records | 9,951 (non-empty hours) |
| Unique formats | 6,933 |
| `.replace()` calls | 82 |
| Regex patterns | ~150 |
| Parser code | ~590 lines |
| Coverage | 97.1% (9,659 entries) |
| Unparseable | 292 (2.9%) |

## Lessons

1. **Free-text fields in government data are hell.** Please just use JSON. `["Mon-Fri:9:00-18:00","Sat:9:00-13:00"]`. Please.

2. **Normalization rules have order dependencies.** Adding one breaks another. Arranging 82 of them in the right order is basically compiler pass optimization.

3. **Coverage is a quality indicator, not a target.** The commit that dropped coverage from 96.6% to 96.3% was the most important fix.

4. **For health-related data, "unknown" is 100x better than "wrong."** If you can't parse it, show the raw data. Partial parses are a lie factory.

5. **The cost of the last 3% is exponential.** 89→95 was character normalization. 95→97 was context-dependent comma parsing and lookahead assertions. 97→99 would mean NLP. Knowing when to stop is a skill.

---

> **Disclosure:** The regex patterns and parser code were generated with [Claude Code](https://claude.ai/claude-code) and refined through iterative testing against the actual dataset. The design decisions, coverage targets, and "when to stop" judgments described in this article are mine.

## Repository

{% github odakin/mhlw-ec-pharmacy-finder %}

Parser code: [`docs/app.js`](https://github.com/odakin/mhlw-ec-pharmacy-finder/blob/main/docs/app.js) | Design doc: [`docs/HOURS_PARSER.md`](https://github.com/odakin/mhlw-ec-pharmacy-finder/blob/main/docs/HOURS_PARSER.md)

Live site: [Emergency Contraception Pharmacy Search](https://odakin.github.io/mhlw-ec-pharmacy-finder/) — searches 10,000+ pharmacies and 2,900+ clinics from Japan's official MHLW data.

> **Sequel:** [How I Misapplied My Own Design Principle and Shipped Wrong Answers](https://dev.to/odakin/how-i-misapplied-my-own-design-principle-and-shipped-wrong-answers-temp-slug) (link updated after publication)
