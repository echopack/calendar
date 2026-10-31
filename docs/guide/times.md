---
description: Time for a time of day that wraps at midnight, and DateTime for a wall clock with no zone.
---

# Times and date-times

Not every value needs a full date. Opening hours, an alarm, or "lunch at 12:30" are just a time of day. And sometimes you know the date and the time, but not yet where: a meeting someone typed into a form before you know which office it is in. This page covers the two types for those situations, [](api:calendar::Time) and [](api:calendar::DateTime).

## Time

A [](api:calendar::Time) is a time of day, from 00:00 up to the last nanosecond before midnight. You may give it hours, minutes, seconds and nanoseconds, and leave out whatever is zero:

```echo
$lunch = Time(12, 30);
$precise = Time(14, 5, 30, 250000000);
$noon = Time::noon();

echo "{$lunch}";                 // 12:30:00
echo "{$precise}";               // 14:05:30.250
echo "{$noon}";                  // 12:00:00
```

For values that arrive at runtime, `Time::of` returns a `result`, and `Time::parse` reads `HH:mm` or `HH:mm:ss` with an optional fraction:

```echo
Time $closing = guard Time::parse('18:45');
$refused = Time::of(25, 0)->failure();

echo "{$closing}";               // 18:45:00
echo "{$refused}";               // hour 25 is out of range
```

### Time wraps around midnight

A time of day has nowhere to go past midnight, so adding to it wraps around, the same way a clock on the wall does:

```echo
$lateShift = Time(23, 30);

echo "{$lateShift + time::duration::hours(2)}";   // 01:30:00
```

Time arithmetic takes a `std::time::duration`, since a time of day only knows about hours and smaller. If you need the date to roll over too, you want a [DateTime](#datetime).

### How long until then?

`until` measures from one time to another on the same day, and returns a signed `duration`. Pass a unit to get a whole number instead:

```echo
$open = Time(9, 0);
$close = Time(17, 30);

echo "{$open->until($close)}";           // 8h30m0s
echo $open->until($close, .hour);        // 8
```

Since a `Time` has no date, `until` never assumes the other time is "tomorrow". From 23:30 to 01:00 is `-22h30m0s`, not 1.5 hours. When the day matters, use a `DateTime`.

### Rounding a time

You may round to any clock unit. Rounding defaults to "half up", and you may pass another [](api:calendar::Rounding) mode:

```echo
$arrival = Time(9, 47, 31);

echo "{$arrival->round(.hour)}";          // 10:00:00
echo "{$arrival->round(.minute)}";        // 09:48:00
echo "{$arrival->round(.hour, .floor)}";  // 09:00:00
echo "{$arrival->start(.minute)}";        // 09:47:00
```

`start` is rounding down to the start of the unit, which reads better when that is what you mean. [Units and rounding](./units.md) covers every mode.

## DateTime

A [](api:calendar::DateTime) is a date and a time of day together, with no zone. It is what a clock on the wall shows, without saying which wall:

```echo
$meeting = DateTime(2026, .march, 30, 9, 30);
$sameMeeting = Date(2026, .march, 30)->at(Time(9, 30));

echo "{$meeting}";                        // 2026-03-30T09:30:00
echo "{$meeting == $sameMeeting}";        // true
```

The parts are public fields, `$meeting->date` and `$meeting->time`. `Date->midnight()` and `Time->on($date)` are convenient shortcuts when you have one half already.

### Parsing a date-time

`DateTime::parse` reads ISO text. The separator may be `T` or a space, and seconds are optional:

```echo
DateTime $fromApi = guard DateTime::parse('2026-03-27T09:30');
DateTime $fromLog = guard DateTime::parse('2026-03-27 09:30:15');

echo "{$fromApi}";               // 2026-03-27T09:30:00
echo "{$fromLog}";               // 2026-03-27T09:30:15
```

Text with an offset like `+01:00` is refused, because a `DateTime` has nowhere to keep it. That text describes a real instant, and [Zoned::parse](./zoned.md#parsing) is the right reader for it.

### Arithmetic carries into the date

Unlike a bare `Time`, a `DateTime` rolls over into the next day. You may add a [](api:calendar::Period), a `duration`, or both:

```echo
$lateShift = DateTime(2026, .march, 27, 22, 0);

echo "{$lateShift + time::duration::hours(3)}";   // 2026-03-28T01:00:00
echo "{$lateShift + Period::days(1)}";            // 2026-03-28T22:00:00
```

### Every day is 24 hours

Here is the one thing to keep in mind about `DateTime`. Since it has no zone, it has no daylight saving time either. For a `DateTime`, every day is exactly 24 hours long, including the Sunday when the clocks in Zurich go forward:

```echo
$saturday = DateTime(2026, .march, 28, 9, 0);
$sunday = DateTime(2026, .march, 29, 9, 0);

echo $saturday->until($sunday, .hour);   // 24
```

In Zurich, that Sunday really was only 23 hours long. That is not a bug in `DateTime`, it simply doesn't know where it is. As soon as the real length matters, place it in a zone with `at` and work with the resulting [](api:calendar::Zoned):

```echo
$zurich = Zone(named: 'Europe/Zurich');
$saturday = DateTime(2026, .march, 28, 9, 0)->at($zurich);
$sunday = DateTime(2026, .march, 29, 9, 0)->at($zurich);

echo $saturday->until($sunday, .hour);   // 23
```

[Zoned date-times](./zoned.md) picks up from here.

### Changing parts

`with` works like it does on a `Date`, and also accepts the time fields:

```echo
$meeting = DateTime(2026, .march, 30, 9, 30);
$evening = $meeting->with($hour: 17, $minute: 0);

echo "{$evening}";               // 2026-03-30T17:00:00
```
