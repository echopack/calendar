---
description: The start and end of a week, month or year, rounding to an hour or a minute, and the Unit and Rounding enums.
---

# Units and rounding

Reports and dashboards are full of questions like "everything since the start of this month", "the week this date falls in", or "round the arrival to the nearest five minutes". This page covers the methods that answer them, and the two small enums they share: [](api:calendar::Unit) and [](api:calendar::Rounding).

## Units

A [](api:calendar::Unit) names a size of time, from `.year` down to `.nanosecond`:

| Unit | Kind |
|:--|:--|
| `.year`, `.quarter`, `.month` | Calendar units. Their length depends on where they start. |
| `.week`, `.day` | Fixed on a wall clock. A day in a zone can still be 23 or 25 hours. |
| `.hour`, `.minute`, `.second` | Clock units |
| `.millisecond`, `.microsecond`, `.nanosecond` | Clock units below a second |

Each method only accepts the units that make sense for its type. A [](api:calendar::Date) has nothing below a day, and a [](api:calendar::Time) has nothing above an hour. Asking for one anyway is a bug in your code, so it stops the program.

## The start of a unit

`start` moves back to the beginning of the unit the value is in. On a date, that gives you the first day of the week, month, quarter or year:

```echo
$launch = Date(2026, .march, 27);

echo "{$launch->start(.week)}";       // 2026-03-23
echo "{$launch->start(.month)}";      // 2026-03-01
echo "{$launch->start(.quarter)}";    // 2026-01-01
echo "{$launch->start(.year)}";       // 2026-01-01
```

On a [](api:calendar::DateTime) or a [](api:calendar::Zoned), the time is reset too, so the start of a day is its midnight:

```echo
$zurich = Zone(named: 'Europe/Zurich');
$now = DateTime(2026, .march, 27, 9, 30)->at($zurich);

echo "{$now->start(.day)}";      // 2026-03-27T00:00:00+01:00[Europe/Zurich]
echo "{$now->start(.month)}";    // 2026-03-01T00:00:00+01:00[Europe/Zurich]
echo "{$now->start(.hour)}";     // 2026-03-27T09:00:00+01:00[Europe/Zurich]
```

Weeks start on Monday by default. You may pass a [](api:calendar::WeekRule) as the second argument to start them on Sunday instead, as shown in [Dates](./dates.md#other-week-rules).

### The end of a unit

On a `Date`, `end` is the last day of the unit:

```echo
$launch = Date(2026, .march, 27);

echo "{$launch->end(.week)}";    // 2026-03-29
echo "{$launch->end(.month)}";   // 2026-03-31
echo "{$launch->end(.year)}";    // 2026-12-31
```

Times and zoned values deliberately have no `end`. What would the end of a day be, 23:59:59? 23:59:59.999999999? Any answer either misses a sliver of time or overlaps the next day. Use a span instead.

## Spans: the whole unit at once

`span` gives you the whole unit as a half-open range: it includes its start and stops just before the next unit begins. On a `Date`, that is a [](api:calendar::DateRange) you may count or loop over:

```echo
$launch = Date(2026, .march, 27);
$month = $launch->span(.month);

echo "{$month}";                 // [2026-03-01, 2026-04-01)
echo $month->count();            // 31
```

On a `Zoned`, the span is an [](api:calendar::Interval) of real instants. That is how you find out how long a day actually was:

```echo
$zurich = Zone(named: 'Europe/Zurich');
$springForward = DateTime(2026, .march, 29, 12, 0)->at($zurich);
$fallBack = DateTime(2026, .october, 25, 12, 0)->at($zurich);

echo "{$springForward->span(.day)->duration()}";   // 23h0m0s
echo "{$fallBack->span(.day)->duration()}";        // 25h0m0s
```

An interval prints its two ends in UTC. [Ranges and intervals](./ranges.md) covers both kinds of span in detail.

## Rounding

`round` moves to the nearest boundary of a unit. It is available on `Time`, `DateTime` and `Zoned`, and rounds halves up by default:

```echo
$arrival = Time(9, 47, 31);

echo "{$arrival->round(.hour)}";     // 10:00:00
echo "{$arrival->round(.minute)}";   // 09:48:00
echo "{$arrival->round(.second)}";   // 09:47:31
```

On a `DateTime`, rounding carries into the date when it needs to. 22:00 is closer to the next midnight than to the last one:

```echo
$lateShift = DateTime(2026, .march, 27, 22, 0);

echo "{$lateShift->round(.day)}";    // 2026-03-28T00:00:00
```

On a `Zoned`, rounding to a day uses the real length of that day. On a 23-hour day, the halfway point is 11.5 hours after midnight. Since the clocks skipped an hour that morning, a wall clock showing 12:00 is only 11 hours in:

```echo
$zurich = Zone(named: 'Europe/Zurich');
$noon = DateTime(2026, .march, 29, 12, 0)->at($zurich);
$afternoon = DateTime(2026, .march, 29, 12, 45)->at($zurich);

echo "{$noon->round(.day)}";       // 2026-03-29T00:00:00+01:00[Europe/Zurich]
echo "{$afternoon->round(.day)}";  // 2026-03-30T00:00:00+02:00[Europe/Zurich]
```

### Rounding modes

You may pass a [](api:calendar::Rounding) mode as the second argument. They are the same modes [differences](./differences.md#rounding-the-count) use:

```echo
$arrival = Time(9, 47, 31);

echo "{$arrival->round(.hour, .floor)}";      // 09:00:00
echo "{$arrival->round(.hour, .ceil)}";       // 10:00:00
echo "{$arrival->round(.hour, .trunc)}";      // 09:00:00
echo "{$arrival->round(.hour, .halfEven)}";   // 10:00:00
```

`start(.hour)` and `round(.hour, .floor)` give the same result. Use whichever reads better where you write it.
