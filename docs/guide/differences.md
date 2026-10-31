---
description: How far apart two dates or moments are, as a Period or as a count of one unit, and how the count is rounded.
---

# Differences

"How old is she?" "How many days until the invoice is due?" "How long ago was the last login?" Each of these asks for the distance between two values. This library answers in one of two shapes, and you pick the one that fits the question:

- `until` and `since` with no unit give you a [](api:calendar::Period), like "36 years and 13 days".
- `until` and `since` with a [](api:calendar::Unit) give you a single whole number, like "272 days".

Both work the same way on [](api:calendar::Date), [](api:calendar::DateTime) and [](api:calendar::Zoned).

## The distance as a period

`until` counts from this value forward to another one:

```echo
$birthday = Date(1990, .march, 14);
$today = Date(2026, .march, 27);

$age = $birthday->until($today);

echo "{$age}";                   // P36Y13D
echo $age->human();              // 36 years and 13 days
```

`since` is the same question asked the other way around. `$today->since($birthday)` gives you the same period as `$birthday->until($today)`:

```echo
$today = Date(2026, .march, 27);
$newYear = Date(2026, .january, 1);

echo "{$today->since($newYear)}";   // P2M26D
```

### How the period is counted

The period counts whole years and months first, then the days that are left. That has one useful guarantee: adding the result back always lands exactly on the target.

```echo
$start = Date(2026, .january, 31);
$end = Date(2026, .march, 1);

$gap = $start->until($end);

echo "{$gap}";                   // P1M1D
echo "{$start + $gap == $end}";  // true
```

January 31 plus one month is February 28, and one more day is March 1. You never need to worry that a difference and an addition disagree with each other.

Going backwards, every field is negative:

```echo
$start = Date(2026, .march, 1);
$end = Date(2026, .january, 31);

echo "{$start->until($end)}";    // -P1M1D
```

### Differences between zoned values

On a [](api:calendar::Zoned), the date part is counted in calendar days in the zone, and the rest in exact time. A day across a DST change still counts as one day:

```echo
$zurich = Zone(named: 'Europe/Zurich');
$friday = DateTime(2026, .march, 27, 9, 30)->at($zurich);
$monday = DateTime(2026, .march, 30, 11, 0)->at($zurich);

echo "{$friday->until($monday)}";   // P3DT1H30M
```

Three days and an hour and a half, even though the weekend in between was one hour short. If you'd like the exact length instead, subtract the two values to get a `duration`, as shown in [Periods and durations](./periods.md#durations).

## The distance in one unit

Often you only want one number: "272 days until Christmas", "8 full months". Pass the unit you want:

```echo
$today = Date(2026, .march, 27);
$christmas = Date(2026, .december, 24);

echo $today->until($christmas, .day);     // 272
echo $today->until($christmas, .week);    // 38
echo $today->until($christmas, .month);   // 8
```

You may use any [](api:calendar::Unit) the type understands. A `Date` takes years down to days, and `DateTime` and `Zoned` go all the way down to nanoseconds.

### Rounding the count

By default, the count is truncated: 8 months and 27 days is "8 months". When you would rather round, pass a [](api:calendar::Rounding) mode:

```echo
$today = Date(2026, .march, 27);
$christmas = Date(2026, .december, 24);

echo $today->until($christmas, .month, .halfExpand);   // 9
echo $today->until($christmas, .month, .ceil);         // 9
```

The modes are:

| Mode | Rounds | 8.9 becomes | -8.9 becomes |
|:--|:--|:--|:--|
| `.trunc` | toward zero (the default) | 8 | -8 |
| `.floor` | down | 8 | -9 |
| `.ceil` | up | 9 | -8 |
| `.halfExpand` | to the nearest, halves away from zero | 9 | -9 |
| `.halfEven` | to the nearest, halves to the even number | 9 | -9 |

Rounding works on the calendar distance, not on a guessed average month. Eight months and 27 days is close to nine months because it is 27 days into a month of 31 days, not because 270 days divided by 30 is 9.

### Counting hours across a DST change

Here is where the type you are holding really matters. On a `Zoned`, the hours are real hours:

```echo
$zurich = Zone(named: 'Europe/Zurich');
$saturday = DateTime(2026, .march, 28, 9, 0)->at($zurich);
$sunday = DateTime(2026, .march, 29, 9, 0)->at($zurich);

echo $saturday->until($sunday, .day);    // 1
echo $saturday->until($sunday, .hour);   // 23
```

One day, and 23 hours. A [](api:calendar::DateTime) would say 24 hours for the same two wall clocks, because it has no zone to know better. [Times and date-times](./times.md#every-day-is-24-hours) shows that side by side.

## Times of day

A [](api:calendar::Time) has no date, so its `until` stays within one day and returns a `duration`. It may be negative:

```echo
$open = Time(9, 0);
$close = Time(17, 30);

echo "{$open->until($close)}";           // 8h30m0s
echo "{$close->until($open)}";           // -8h30m0s
echo $open->until($close, .minute);      // 510
```

## Saying it to a person

A period or a count is great for code. For a person, you will usually want "in 3 days" or "2 years ago". [Human-readable output](./human.md) covers that, and uses the same calendar counting you have seen on this page.
