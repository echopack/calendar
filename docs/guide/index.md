---
description: Install calendar, build a Calendar, and write a small program that schedules a meeting across a DST change.
---

# Getting started

Dates look simple until the first month with 30 days, the first clock change, or the first colleague in another country. This library is there so you don't have to think about those things every time. You still decide what should happen, and the library does the counting.

On this page we'll write one small program together: it tells a team in Zurich when their next stand-up is, says it in German, and works out what time that is for a colleague in New York. By the end you will have touched most of the types you need day to day. Don't worry if not every detail clicks right away. Each step links to the page that covers it in full.

## Install it

From your project directory:

```bash
epm add echopack/calendar --git https://github.com/echopack/calendar --range ^0.1
```

That adds a `#[requires:]` line to your `module.eco` and vendors the sources. There is no C to link. The only thing the library reads from your system is the time zone database, which macOS and Linux already ship in `/usr/share/zoneinfo`.

Everything lives in the `calendar` namespace. You may import the types you use with one `use` line:

```echo plain
use calendar::{Calendar, Date, DateTime, Format, Locale, Period, Time, Zone, Zoned};
use std::time;
```

The examples in this guide leave that line out to stay short. Every one of them runs as written once it is there.

## A complete pass

### Step 1: The calendar

Most programs need the same few things whenever they touch a date: which time zone they are in, which language to print in, and what "now" is. Rather than keeping those in global settings, you put them in one [](api:calendar::Calendar) and pass it around:

```echo
$zurich = Zone(named: 'Europe/Zurich');
$cal = Calendar($zone: $zurich, $locale: Locale::de());

echo "{$cal->today()}";
```

That prints whatever today is where you run it. For this guide we'd like the output to stay the same, so let's pin the clock to Friday, March 27, 2026, at 09:30. That is two days before the clocks in Zurich go forward, which will come in handy in a moment:

```echo
$zurich = Zone(named: 'Europe/Zurich');
$clock = FixedClock(DateTime(2026, .march, 27, 9, 30)->at($zurich)->instant);
$cal = Calendar($zone: $zurich, $locale: Locale::de(), $clock: $clock);

echo "{$cal->today()}";          // 2026-03-27
echo "{$cal->now()}";            // 2026-03-27T09:30:00+01:00[Europe/Zurich]
```

In your own program you would leave `$clock` out and get the real one. A fixed clock is also exactly what you want in tests. [Clocks](./clocks.md) has more on that.

### Step 2: Next Monday at 09:30

The stand-up is every Monday at 09:30. [](api:calendar::Date::next) finds the next Monday after a date, and `at` puts a time of day on it:

<!-- check: continue -->
```echo
$monday = $cal->today()->next(.monday);
$wallClock = $monday->at(Time(9, 30));

echo "{$monday}";                // 2026-03-30
echo "{$wallClock}";             // 2026-03-30T09:30:00
```

`$wallClock` is a [](api:calendar::DateTime): a date and a time, but no zone yet. It is what a clock on the wall says, without saying which wall. To turn it into a real moment, we hand it to the calendar, which places it in Zurich:

<!-- check: continue -->
```echo
$standup = $cal->at($wallClock);

echo "{$standup}";               // 2026-03-30T09:30:00+02:00[Europe/Zurich]
```

Notice the `+02:00`. The clocks went forward on Sunday, so Zurich is now on summer time, and the library picked the right offset for you. A [](api:calendar::Zoned) always knows its instant, its zone, and what the clock on the wall reads.

### Step 3: Say it in German

Now we are ready to print it. A pattern spells out the fields you want: `EEEE` is the full weekday, `MMMM` the full month name, and text in single quotes is printed as it is:

<!-- check: continue -->
```echo
echo $cal->format($standup, "EEEE, d. MMMM 'um' HH:mm");   // Montag, 30. März um 09:30
```

The calendar fills in its own language, so the names come out in German. [Formatting](./formatting.md) has every option, and [Pattern letters](./patterns.md) lists every letter.

### Step 4: How far away is it?

People rarely want "2026-03-30T09:30". They want "in 3 days":

<!-- check: continue -->
```echo
echo $cal->human($standup);      // in 3 Tagen
```

From Friday 09:30 to Monday 09:30 is 71 hours, because Sunday was only 23 hours long. The answer is still "3 days", because that is what a person means. [Human-readable output](./human.md) shows how to tune this.

### Step 5: The colleague in New York

One person on the team joins from New York. `to` shows the same moment in another zone:

<!-- check: continue -->
```echo
$newYork = $standup->to(Zone(named: 'America/New_York'));

$same = $newYork->instant == $standup->instant;

echo "{$newYork}";               // 2026-03-30T03:30:00-04:00[America/New_York]
echo "{$same}";                  // true
```

03:30 in the morning. Perhaps that colleague would like a later slot. Both values are the same instant, only read on different walls.

### Step 6: Reading a date someone typed

Finally, let's read the date of the team offsite from a form. Swiss forms write dates as `01.08.2026`. A [](api:calendar::Format) says exactly which shape to expect, and parsing returns a `result`, so you decide what happens when the text doesn't fit:

<!-- check: continue -->
```echo
$swiss = Format('dd.MM.yyyy');

Date $offsite = guard Date::parse('01.08.2026', $swiss) else ($e) {
    die("offsite: {$e}");
};

echo $cal->format($offsite, 'EEEE, d. MMMM');   // Samstag, 1. August
```

So, what happens when the text is wrong? The parser never guesses. Here someone typed a day that February doesn't have, and then a date with a stray space at the end:

<!-- check: continue -->
```echo
$noDay = Date::parse('31.02.2026', $swiss)->failure();
$trailing = Date::parse('01.08.2026 ', $swiss)->failure();

echo "{$noDay}";                 // 2026-02 has no day 31
echo "{$trailing}";              // unexpected text at byte 10
```

Nothing is rounded into March, and nothing is silently trimmed. [Parsing](./parsing.md) covers what is accepted and why.

### A note on literal values

You may have noticed that `Date(2026, .march, 27)` and `Format('dd.MM.yyyy')` don't return a `result`, while `Date::parse` does. That is on purpose. A value you write in your source code is either right or a bug, so a bad one stops the program with a clear message. A value that arrives at runtime goes through `::of` or `::parse`, which hand you the error to deal with. [Choosing a type](./types.md#literals-and-runtime-values) has the full picture.

## Where to go next

- **[Choosing a type](./types.md)** helps you pick between `Date`, `DateTime`, `Zoned` and friends.
- **[Periods and durations](./periods.md)** explains why one day and 24 hours are different things.
- **[Daylight saving time](./dst.md)** shows what happens to a time that is skipped or repeated.
- **[Ranges and intervals](./ranges.md)** walks through every day of a month, or every month of a year.
