---
description: DateRange for spans of days you can loop over, stepping by a period without drift, and Interval for spans of real time.
---

# Ranges and intervals

Plenty of everyday questions are about a stretch of time rather than a single point. Which days does this booking cover? Do these two holidays overlap? What happened during yesterday, from the first instant to the last? This library has two types for that:

- A [](api:calendar::DateRange) is a span of days. You may count it, check it, and loop over it.
- An [](api:calendar::Interval) is a span of real instants, such as one day in Zurich from midnight to midnight.

## Half-open, always

Both types include their start and **exclude** their end. Writing that as `[start, end)` looks a bit mathematical at first, but it saves you from a whole family of off-by-one bugs. March is `[March 1, April 1)`, and April is `[April 1, May 1)`. They fit together with no gap and no overlap, and you never need to write "23:59:59.999" to mean the end of a day.

```echo
$march = Date(2026, .march, 1)->span(.month);

$lastOfMarch = Date(2026, .march, 31);
$firstOfApril = Date(2026, .april, 1);

echo "{$march}";                         // [2026-03-01, 2026-04-01)
echo "{$march->has($lastOfMarch)}";      // true
echo "{$march->has($firstOfApril)}";     // false
```

## Date ranges

### Creating a range

You may build a range from a start and an exclusive end with `until:`, or from a start and the last day you want included with `through:`. A hotel booking reads naturally with `until:` (you leave on the last day), and a holiday with `through:` (the last day is part of it):

```echo
$booking = DateRange(from: Date(2026, .march, 27), until: Date(2026, .march, 30));
$holiday = DateRange(from: Date(2026, .august, 3), through: Date(2026, .august, 14));

echo "{$booking}";               // [2026-03-27, 2026-03-30)
echo $booking->count();          // 3
echo "{$holiday}";               // [2026-08-03, 2026-08-15)
echo $holiday->count();          // 12
```

Both forms store the same thing, a start and an exclusive end. An end before the start is a bug and stops the program. An end equal to the start is an empty range.

`span` on a [](api:calendar::Date) gives you the week, month, quarter or year the date is in, as shown above.

### Looping over days

A `DateRange` is iterable, so `foreach` walks through every day in it:

```echo
$week = Date(2026, .march, 30)->span(.week);

foreach ($week as const &$day) {
    echo $day->format('EEE d MMM');
}
```

That prints `Mon 30 Mar` through `Sun 5 Apr`, seven lines in total.

### Stepping by a period

Sometimes you need every month, or every second week, rather than every day. `every` takes a [](api:calendar::Period) and returns something you may loop over the same way:

```echo
$firstHalf = DateRange(from: Date(2026, .january, 31), until: Date(2026, .june, 1));

foreach ($firstHalf->every(Period::months(1)) as const &$due) {
    echo "{$due}";
}
```

That prints January 31, February 28, March 31, April 30 and May 31.

Notice that the dates don't drift. A naive loop that keeps adding one month to the previous result would get stuck on the 28th after February. Here, every date is computed from the start, so the payment date comes back to the 31st whenever the month allows it.

You may also use a key to number the steps:

```echo
$quarter = DateRange(from: Date(2026, .january, 5), until: Date(2026, .april, 1));

foreach ($quarter->every(Period::weeks(2)) as $sprint => const &$start) {
    echo "sprint {$sprint}: {$start}";
}
```

The step must move forward. A zero or negative period would loop forever, so it stops the program instead. The same goes for building `DateSteps` or a `DateCursor` yourself: the step has to move forward and the end can't come before the start.

### Comparing ranges

`overlaps` and `intersect` tell you whether two ranges share any days, and which ones. That is the check you need before accepting a booking:

```echo
$march = Date(2026, .march, 1)->span(.month);
$easterWeek = Date(2026, .march, 30)->span(.week);
$april = Date(2026, .april, 1)->span(.month);

echo "{$march->overlaps($easterWeek)}";   // true
echo "{$march->overlaps($april)}";        // false

DateRange $shared = guard $march->intersect($easterWeek);
echo "{$shared}";                         // [2026-03-30, 2026-04-01)
```

`intersect` returns `null` when the ranges don't overlap. Since March ends exactly where April starts, they touch but don't overlap. That is half-open ranges doing their job.

`last()` gives you the last day included, or `null` for an empty range, and `empty()` tells you whether there are any days at all.

## Intervals

An [](api:calendar::Interval) is the same idea for real time: a start instant and an exclusive end instant. You will usually get one from `span` on a [](api:calendar::Zoned), which knows exactly when a day, a week or a month begins and ends in its zone:

```echo
$zurich = Zone(named: 'Europe/Zurich');
$friday = DateTime(2026, .march, 27, 9, 30)->at($zurich);

$day = $friday->span(.day);

echo "{$day}";                   // [2026-03-26T23:00:00Z, 2026-03-27T23:00:00Z)
echo "{$day->duration()}";       // 24h0m0s
echo "{$day->has($friday->instant)}";   // true
```

Intervals print in UTC, since they are about instants, not about any particular wall. They are perfect for queries like "every payment that arrived yesterday in Zurich", where the boundaries of the day depend on the zone.

You may also build one directly from two timestamps:

```echo
$zurich = Zone(named: 'Europe/Zurich');
$start = DateTime(2026, .march, 27, 9, 30)->at($zurich);
$end = $start + time::duration::hours(2);

$workshop = Interval(from: $start->instant, until: $end->instant);

echo "{$workshop}";              // [2026-03-27T08:30:00Z, 2026-03-27T10:30:00Z)
```

Like a `DateRange`, an interval has `has`, `empty`, `overlaps` and `intersect`. Its `duration` is the exact length, so on the day the clocks change, it tells you the truth:

```echo
$zurich = Zone(named: 'Europe/Zurich');
$sunday = DateTime(2026, .march, 29, 12, 0)->at($zurich);

echo "{$sunday->span(.day)->duration()}";   // 23h0m0s
```
