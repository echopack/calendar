---
description: Zoned, the everyday type for a moment in a place: creating, converting between zones, arithmetic, comparing and parsing.
---

# Zoned date-times

When something actually happens (a payment arrives, a flight departs, a user logs in), you want to remember the exact moment and also be able to say what the clock read where it happened. A [](api:calendar::Zoned) holds both: a real instant, and the [](api:calendar::Zone) to read it in.

If you are unsure which type to use for a moment in time, this is the one. It is the only type that knows how long a day really was.

## What a Zoned holds

A `Zoned` has five public fields. The first two are what it is, and the last three are what the zone says at that moment:

```echo
$zurich = Zone(named: 'Europe/Zurich');
$standup = DateTime(2026, .march, 27, 9, 30)->at($zurich);

echo "{$standup}";               // 2026-03-27T09:30:00+01:00[Europe/Zurich]
echo $standup->zone->id;         // Europe/Zurich
echo "{$standup->offset}";       // +01:00
echo "{$standup->date}";         // 2026-03-27
echo "{$standup->time}";         // 09:30:00
```

`$instant` is a `std::time::timestamp`, the moment itself, independent of any zone. `local()` gives you the wall clock as a [](api:calendar::DateTime) when you need to hand it to something that has no zone.

## Creating a Zoned

There are a few ways in, depending on what you start with.

### From a wall clock

When you know what the clock on the wall should read, write it as a [](api:calendar::DateTime) and place it in a zone with `at`:

```echo
$zurich = Zone(named: 'Europe/Zurich');
$standup = DateTime(2026, .march, 30, 9, 30)->at($zurich);

echo "{$standup}";               // 2026-03-30T09:30:00+02:00[Europe/Zurich]
```

The zone works out the offset. Here it is `+02:00`, since the clocks went forward the day before. Every once in a while a wall clock doesn't exist, or exists twice, because of a daylight saving change. `at` handles that the way most calendar apps do, and [Daylight saving time](./dst.md) shows how to choose differently.

If you have a [Calendar](./calendar.md), `$cal->at($wallClock)` does the same thing in the calendar's zone.

### From now

`Zoned::now` reads the system clock. In application code you will usually prefer `$cal->now()`, so that tests can replace the clock:

```echo
$zurich = Zone(named: 'Europe/Zurich');
$now = Zoned::now($zurich);
```

### From an instant

When you already have a `std::time::timestamp`, say from a database column of Unix seconds, you may pair it with a zone directly:

```echo
$zurich = Zone(named: 'Europe/Zurich');
$paidAt = time::timestamp::from_unix_secs(1774600200);

$payment = Zoned($paidAt, $zurich);

echo "{$payment}";               // 2026-03-27T09:30:00+01:00[Europe/Zurich]
```

An instant whose wall clock would land outside the years -9999 to 9999 stops the program. For instants that arrive at runtime, `Zoned::of` returns a `result` instead.

## Converting between zones

`to` shows the same instant on another wall. Nothing about the moment changes, only how it is read:

```echo
$zurich = Zone(named: 'Europe/Zurich');
$standup = DateTime(2026, .march, 27, 9, 30)->at($zurich);

$newYork = $standup->to(Zone(named: 'America/New_York'));
$utc = $standup->to(Zone::utc());

echo "{$newYork}";               // 2026-03-27T04:30:00-04:00[America/New_York]
echo "{$utc}";                   // 2026-03-27T08:30:00+00:00[UTC]
```

Notice the difference is five hours, not the usual six. New York switched to summer time three weeks before Zurich does. You don't have to know that: the zone rules take care of it.

## Arithmetic

You may add a [](api:calendar::Period), a `duration`, or both. The rule is short:

- **Years, months, weeks and days move the wall clock.** The meeting stays at 09:30.
- **Hours and smaller, and any `duration`, move the instant.** Exactly that much real time passes.

```echo
$zurich = Zone(named: 'Europe/Zurich');
$friday = DateTime(2026, .march, 27, 9, 30)->at($zurich);

echo "{$friday + Period::days(3)}";             // 2026-03-30T09:30:00+02:00[Europe/Zurich]
echo "{$friday + time::duration::hours(72)}";   // 2026-03-30T10:30:00+02:00[Europe/Zurich]
```

Three days later is Monday at 09:30. Seventy-two hours later is Monday at 10:30, because Sunday was an hour short. Both are correct answers, to different questions.

When a period has both parts, the calendar part is applied first, then the exact part. One day and two hours from Friday 09:30 is Saturday 11:30.

### Changing parts

`with` replaces some wall-clock fields and keeps the zone. The result is placed back in the zone, so the offset is always right:

```echo
$zurich = Zone(named: 'Europe/Zurich');
$friday = DateTime(2026, .march, 27, 9, 30)->at($zurich);

$evening = $friday->with($hour: 17);
$endOfMonth = $friday->with($day: 31);

echo "{$evening}";               // 2026-03-27T17:30:00+01:00[Europe/Zurich]
echo "{$endOfMonth}";            // 2026-03-31T09:30:00+02:00[Europe/Zurich]
```

### Overflow and DST, explicitly

`+` clamps a day that doesn't exist (January 31 plus a month) and resolves DST the compatible way. When you need control over either, the long form of `add` takes an [](api:calendar::Overflow) rule and a [](api:calendar::Disambiguation) rule, and returns a `result`:

```echo
$zurich = Zone(named: 'Europe/Zurich');
$lastDay = DateTime(2026, .january, 31, 9, 0)->at($zurich);

$strict = $lastDay->add(Period::months(1), .reject, .compatible);

echo "{$strict->failure()}";     // 2026-02 has no day 31
```

## Comparing

`==` is strict: two `Zoned` values are equal when they are the same instant **in the same zone**. That way, two values that print differently are never "equal". To ask whether two values are the same moment, compare their instants:

```echo
$zurich = Zone(named: 'Europe/Zurich');
$standup = DateTime(2026, .march, 27, 9, 30)->at($zurich);
$inNewYork = $standup->to(Zone(named: 'America/New_York'));

echo "{$standup == $inNewYork}";                     // false
echo "{$standup->instant == $inNewYork->instant}";   // true
```

`<` and `>` order by instant, so sorting a list of `Zoned` values from different zones puts them in the order they happened.

## Parsing

`Zoned::parse` reads the RFC 9557 form that `Zoned` prints, an ISO date-time with an offset and a zone name in brackets:

```echo
Zoned $paid = guard Zoned::parse('2026-03-27T09:30:00+01:00[Europe/Zurich]');

echo "{$paid}";                  // 2026-03-27T09:30:00+01:00[Europe/Zurich]
```

That is the format to use whenever you store or send a `Zoned` as text, since it round-trips exactly. The parser is helpful in a few other cases too:

```echo
Zoned $fromUtc = guard Zoned::parse('2026-03-27T08:30:00Z[Europe/Zurich]');
Zoned $noZone = guard Zoned::parse('2026-03-27T09:30:00+01:00');
Zoned $noOffset = guard Zoned::parse('2026-03-27T09:30:00[Europe/Zurich]');

echo "{$fromUtc}";               // 2026-03-27T09:30:00+01:00[Europe/Zurich]
echo "{$noZone}";                // 2026-03-27T09:30:00+01:00[+01:00]
echo "{$noOffset}";              // 2026-03-27T09:30:00+01:00[Europe/Zurich]
```

- A `Z` with a zone means "this UTC instant, shown in that zone".
- An offset without a zone gives you a fixed-offset zone.
- A zone without an offset places the wall clock in the zone.

So, what happens when the offset and the zone disagree? The text is refused, instead of quietly trusting one of them:

```echo
$conflict = Zoned::parse('2026-03-27T09:30:00+05:00[Europe/Zurich]')->failure();

echo "{$conflict}";              // the offset does not match the zone at byte 19
```

Text with neither an offset nor a zone is refused as well, since it doesn't describe a moment. For text in other shapes, such as `27.03.2026 09:30` or an email header, see [Parsing](./parsing.md).
