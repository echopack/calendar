---
description: Period for calendar time and duration for exact time, what "one month later" means, and how overflow is handled.
---

# Periods and durations

"Send a reminder one month after the invoice date." "Time out after 30 seconds." Both sentences talk about an amount of time, but they mean very different things. A month is 28 to 31 days depending on where you start. Thirty seconds is always thirty seconds.

This library keeps those two ideas apart:

- A [](api:calendar::Period) is **calendar time**: years, months, weeks, days, hours, minutes, seconds and nanoseconds, each kept exactly as you wrote it.
- A `std::time::duration` is **exact time**: a length in nanoseconds, from Echo's standard library.

Once you know which one you are holding, most of the surprises around dates disappear.

## Creating periods

For a single unit, there is a static per unit. For several, you may name the fields you need:

```echo
$reminder = Period::months(1);
$trial = Period($days: 14, $hours: 12);
$overdue = Period($months: -2);

echo "{$reminder}";              // P1M
echo "{$trial}";                 // P14DT12H
echo "{$overdue}";               // -P2M
```

Periods print as ISO 8601 durations, and `Period::parse` reads them back. When every field points backwards, the sign goes in front, as in `-P2M`. Otherwise each field carries its own sign, so `P1M-3D` (one month forward, then three days back) is a perfectly good period:

```echo
Period $notice = guard Period::parse('P1M-3D');

echo "{$notice}";                // P1M-3D
echo "{$notice->months}";        // 1
echo "{$notice->days}";          // -3
```

### Combining periods

You may add, subtract, negate and multiply periods. Fields are combined one by one:

```echo
$sprint = Period::weeks(2);
$weekend = Period::days(3) + Period::hours(2);
$backwards = -$sprint;

echo "{$sprint * 3}";            // P6W
echo "{$weekend}";               // P3DT2H
echo "{$backwards}";             // -P2W
```

## A day is not 24 hours

Here is the most important thing about periods: **fields are never folded into each other**. `Period::days(1)` and `Period::hours(24)` are different values, and they stay different:

```echo
$day = Period::days(1);

echo "{$day == Period::hours(24)}";   // false
```

That may feel pedantic at first, but it is what makes calendar arithmetic come out right. Let's see why with a meeting in Zurich, on the Saturday before the clocks go forward:

```echo
$zurich = Zone(named: 'Europe/Zurich');
$meeting = DateTime(2026, .march, 28, 9, 0)->at($zurich);

echo "{$meeting + Period::days(1)}";              // 2026-03-29T09:00:00+02:00[Europe/Zurich]
echo "{$meeting + Period::hours(24)}";            // 2026-03-29T10:00:00+02:00[Europe/Zurich]
echo "{$meeting + time::duration::hours(24)}";    // 2026-03-29T10:00:00+02:00[Europe/Zurich]
```

One day later, the meeting is at 09:00 again, which is what a person means by "same time tomorrow". Twenty-four hours later, the clock reads 10:00, because Sunday only had 23 hours. Both answers are correct for their question.

The rule for a [](api:calendar::Zoned) is simple: years, months, weeks and days move the wall clock, and hours and smaller move the instant. [Zoned date-times](./zoned.md#arithmetic) has the details.

## What "one month later" means

Adding months moves the month and keeps the day. When the target month is too short, the day is clamped to its last day:

```echo
$invoice = Date(2026, .january, 31);

echo "{$invoice + Period::months(1)}";   // 2026-02-28
echo "{$invoice + Period::months(2)}";   // 2026-03-31
```

January 31 plus one month is February 28, not March 3. That is how most people, and most accounting rules, read it.

When a period has several fields, they are applied from the largest to the smallest: years and months first, then weeks and days, then the exact clock part. So one month and one day from January 31 is February 28 plus one day:

```echo
$invoice = Date(2026, .january, 31);
$later = Period($months: 1, $days: 1);

echo "{$invoice + $later}";      // 2026-03-01
```

### When clamping is not what you want

Occasionally you may want to know about the short month rather than quietly land on its last day. A recurring payment on the 31st, for example, might need a human to decide. The long form of `add` takes an [](api:calendar::Overflow) rule and returns a `result`:

```echo
$invoice = Date(2026, .january, 31);

$clamped = $invoice->add(Period::months(1), .clamp);
$strict = $invoice->add(Period::months(1), .reject);

echo "{$clamped->unwrap()}";     // 2026-02-28
echo "{$strict->failure()}";     // 2026-02 has no day 31
```

`.clamp` is what `+` does. `.reject` hands you a [](api:calendar::DateError) instead. `sub` takes the same rule.

## Periods on plain dates

A [](api:calendar::Date) has no time of day, so it can only take years, months, weeks and days. Adding a period with hours to a date is a bug, and stops the program. You may check up front with `dated()`:

```echo
$weekend = Period::days(3);
$late = Period($days: 1, $hours: 2);

echo "{$weekend->dated()}";      // true
echo "{$late->dated()}";         // false
```

## Durations

When you need an exact length, reach for `std::time::duration`. It comes from Echo's standard library, so you will need `use std::time;` in the file:

```echo plain
use std::time;

$timeout = time::duration::secs(30);
$shift = time::duration::hours(8);
```

You may add a duration to a [](api:calendar::Time), a [](api:calendar::DateTime) or a [](api:calendar::Zoned). Subtracting two `Zoned` values gives you the exact duration between them:

```echo
$zurich = Zone(named: 'Europe/Zurich');
$sunday = DateTime(2026, .march, 29, 0, 0)->at($zurich);
$monday = DateTime(2026, .march, 30, 0, 0)->at($zurich);

echo "{$monday - $sunday}";      // 23h0m0s
```

### From a period to a duration

`exact()` turns the clock fields of a period (hours, minutes, seconds and nanoseconds) into a duration. Days and longer are left out, because they have no fixed length:

```echo
$shift = Period::hours(36);
$weekend = Period::days(2);

echo "{$shift->exact()}";        // 36h0m0s
echo "{$weekend->exact()}";      // 0s
```

There is no conversion the other way around. A duration of 86400 seconds is not "one day" until you know which day.

## Saying a period out loud

`human` writes a period the way a person would say it, in any of the [built-in languages](./locales.md):

```echo
$age = Period($years: 36, $days: 13);

echo $age->human();                    // 36 years and 13 days
echo $age->human(Locale::de());        // 36 Jahre und 13 Tage
echo $age->human(Locale::fr());        // 36 ans et 13 jours
```

[Differences](./differences.md) shows how to get such a period from two dates in the first place.
