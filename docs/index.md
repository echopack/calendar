---
layout: home
hero:
  name: calendar
  text: Dates and time zones for Echo
  tagline: Plain dates, wall clocks, zoned moments and the tz database, with formatting and parsing in four languages. Strict where it matters, forgiving where you ask.
  actions:
    - theme: brand
      text: Get started
      link: /guide/
    - text: Choose a type
      link: /guide/types
    - text: API reference
      link: /api/
features:
  - title: No global state
    details: There is no default zone, language or "now" hiding somewhere. You build one Calendar where your program starts and hand it to whatever needs today.
    link: /guide/calendar
  - title: DST without surprises
    details: Adding a day keeps 09:00 at 09:00 across a clock change. Adding 24 hours moves the instant. A time that never happened, or happened twice, is resolved by a rule you can pick.
    link: /guide/dst
  - title: Strict parsing
    details: A pattern reads exactly what it says. Nothing is filled in from today, and a refusal tells you which field and at which byte.
    link: /guide/parsing
  - title: Four languages
    details: English, German, French and Italian month names, weekdays, plurals and "3 days ago". Pick one per Calendar, or per Format.
    link: /guide/locales
---

You have a date to store, a meeting to schedule across a clock change, or a string from a form to read. This library covers all three, and the types tell you which one you are holding.

```echo
use calendar::{Calendar, Date, DateTime, Format, Locale, Period, Zone};

$zurich = Zone(named: 'Europe/Zurich');

// an invoice due at the end of January, paid a month later
$due = Date(2026, .january, 31);
$reminder = $due + Period::months(1);
echo "{$reminder}";                                  // 2026-02-28

// a meeting on the day before the clocks go forward
$meeting = DateTime(2026, .march, 28, 9, 0)->at($zurich);
$nextDay = $meeting + Period::days(1);
echo "{$nextDay}";                                   // 2026-03-29T09:00:00+02:00[Europe/Zurich]

// a date from a Swiss form, said back in German
Date $start = guard Date::parse('01.08.2026', Format('dd.MM.yyyy'));
echo $start->format('EEEE, d. MMMM yyyy', Locale::de());   // Samstag, 1. August 2026
```

January 31 plus one month is February 28, not March 3. The meeting stays at 09:00 even though that Sunday is only 23 hours long. [Getting started](/guide/) builds a small program from here, one step at a time.

```bash
epm add echopack/calendar --git https://github.com/echopack/calendar --range ^0.1
```

It is all Echo. The time zone rules come from the tz database your system already has.
