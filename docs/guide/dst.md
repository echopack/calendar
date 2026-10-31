---
description: What happens to wall-clock times that are skipped or repeated when the clocks change, and how to pick the behavior you want.
---

# Daylight saving time

Daylight saving time has a reputation for causing bugs, and it has earned it. Twice a year, in many zones, the clocks jump. In spring an hour vanishes, in autumn an hour happens twice. Code that assumed every day has 24 hours, or that every wall-clock time exists exactly once, breaks quietly on those two nights.

The good news: once you have seen what actually happens on those nights, there is very little left to be afraid of. The library handles both cases with sensible defaults, and lets you choose differently when you need to. Let's walk through it with Zurich in 2026.

## The two problems

### The gap: a time that never happened

On Sunday, March 29, 2026, the clocks in Zurich jump from 02:00 straight to 03:00. Nobody in Zurich ever saw 02:30 that night. `Zone->candidates` asks the zone for every offset a wall clock could have, and for 02:30 there are none:

```echo
$zurich = Zone(named: 'Europe/Zurich');
$skipped = DateTime(2026, .march, 29, 2, 30);

echo $zurich->candidates($skipped)->count();   // 0
```

### The overlap: a time that happened twice

On Sunday, October 25, 2026, the clocks go back from 03:00 to 02:00. Everything between 02:00 and 03:00 happens twice, once in summer time and once in winter time:

```echo
$zurich = Zone(named: 'Europe/Zurich');
$repeated = DateTime(2026, .october, 25, 2, 30);

echo $zurich->candidates($repeated)->count();  // 2
```

On every other night of the year, a wall clock has exactly one candidate, and nothing on this page applies.

## What `at` does

Most of the time, you place a wall clock in a zone with `at` (or `$cal->at`), and don't think about any of this. `at` resolves both problems the way most calendar apps and the JavaScript Temporal API do. It is called the "compatible" rule:

- **In a gap**, the time moves forward by the length of the gap. 02:30 becomes 03:30, which is the time a person would see on their phone if they set an alarm for 02:30.
- **In an overlap**, the earlier of the two moments wins. 02:30 means the first 02:30, still in summer time.

```echo
$zurich = Zone(named: 'Europe/Zurich');

$spring = DateTime(2026, .march, 29, 2, 30)->at($zurich);
$autumn = DateTime(2026, .october, 25, 2, 30)->at($zurich);

echo "{$spring}";                // 2026-03-29T03:30:00+02:00[Europe/Zurich]
echo "{$autumn}";                // 2026-10-25T02:30:00+02:00[Europe/Zurich]
```

For a recurring meeting, a reminder, or anything a person typed in, that is almost always what you want.

## Choosing another rule

Occasionally you may need something else. A payroll system might want to know about a repeated hour rather than silently pick one, or a scheduler might prefer the later moment. `resolve` takes a [](api:calendar::Disambiguation) rule and returns a `result`:

| Rule | In a gap | In an overlap |
|:--|:--|:--|
| `.compatible` | moves forward by the gap (what `at` does) | the earlier moment |
| `.earlier` | moves back by the gap | the earlier moment |
| `.later` | moves forward by the gap | the later moment |
| `.reject` | fails with `.gap` | fails with `.overlap` |

Here is each rule on the skipped 02:30:

```echo
$zurich = Zone(named: 'Europe/Zurich');
$skipped = DateTime(2026, .march, 29, 2, 30);

$earlier = guard $skipped->resolve($zurich, .earlier);
$later = guard $skipped->resolve($zurich, .later);
$rejected = $skipped->resolve($zurich, .reject)->failure();

echo "{$earlier}";               // 2026-03-29T01:30:00+01:00[Europe/Zurich]
echo "{$later}";                 // 2026-03-29T03:30:00+02:00[Europe/Zurich]
echo "{$rejected}";              // that wall-clock time is skipped in Europe/Zurich
```

And on the repeated 02:30:

```echo
$zurich = Zone(named: 'Europe/Zurich');
$repeated = DateTime(2026, .october, 25, 2, 30);

$first = guard $repeated->resolve($zurich, .earlier);
$second = guard $repeated->resolve($zurich, .later);
$rejected = $repeated->resolve($zurich, .reject)->failure();

echo "{$first}";                 // 2026-10-25T02:30:00+02:00[Europe/Zurich]
echo "{$second}";                // 2026-10-25T02:30:00+01:00[Europe/Zurich]
echo "{$rejected}";              // that wall-clock time happens twice in Europe/Zurich
```

The two 02:30s print with different offsets, `+02:00` and `+01:00`. They are one hour apart in real time.

## Arithmetic across a change

Adding days keeps the wall clock, adding hours keeps the elapsed time. Across a DST change, those two give different answers, and that is exactly how it should be:

```echo
$zurich = Zone(named: 'Europe/Zurich');
$saturday = DateTime(2026, .march, 28, 9, 0)->at($zurich);

echo "{$saturday + Period::days(1)}";             // 2026-03-29T09:00:00+02:00[Europe/Zurich]
echo "{$saturday + time::duration::hours(24)}";   // 2026-03-29T10:00:00+02:00[Europe/Zurich]
```

If adding days lands on a wall clock in a gap or an overlap, it is resolved with the compatible rule, the same as `at`. The long form `add($period, $overflow, $disambiguation)` lets you pick a rule there too, as shown on [Zoned date-times](./zoned.md#overflow-and-dst-explicitly).

## How long was that day?

A day in a zone is not always 24 hours. `span(.day)` gives you the real day as an [](api:calendar::Interval), and its `duration` tells you how long it was:

```echo
$zurich = Zone(named: 'Europe/Zurich');
$spring = DateTime(2026, .march, 29, 12, 0)->at($zurich);
$autumn = DateTime(2026, .october, 25, 12, 0)->at($zurich);
$ordinary = DateTime(2026, .june, 15, 12, 0)->at($zurich);

echo "{$spring->span(.day)->duration()}";     // 23h0m0s
echo "{$autumn->span(.day)->duration()}";     // 25h0m0s
echo "{$ordinary->span(.day)->duration()}";   // 24h0m0s
```

Counting with `until` respects that too. From Saturday 09:00 to Sunday 09:00 is one day and 23 hours, both at once:

```echo
$zurich = Zone(named: 'Europe/Zurich');
$saturday = DateTime(2026, .march, 28, 9, 0)->at($zurich);
$sunday = DateTime(2026, .march, 29, 9, 0)->at($zurich);

echo $saturday->until($sunday, .day);    // 1
echo $saturday->until($sunday, .hour);   // 23
```

## Editing inside a repeated hour

There is one more subtle case, and the library takes care of it for you. Say a value sits in the second, winter-time 02:30 on October 25, and you change its minutes with `with`. The result is another wall clock in the repeated hour, so which of the two should it be?

```echo
$zurich = Zone(named: 'Europe/Zurich');
$second = guard DateTime(2026, .october, 25, 2, 30)->resolve($zurich, .later);

$adjusted = $second->with($minute: 45);

echo "{$adjusted}";              // 2026-10-25T02:45:00+01:00[Europe/Zurich]
```

It stays in the same half of the overlap it started in, with the `+01:00` offset. `start` and `round` behave the same way. A small change to the minutes never makes your value jump an hour.

## Where the rules come from

Every zone's transitions, past and future, come from the tz database on your system. You never hard-code "the last Sunday of March". When a government changes its rules, an updated tz database is all you need. [Zones and tz data](./zones.md) explains where that data lives and how to inspect transitions.
