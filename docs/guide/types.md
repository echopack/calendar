---
description: Date, Time, DateTime, Zoned, Period and duration, what each one means, and how to pick the right one.
---

# Choosing a type

Most bugs with dates come from holding the wrong kind of value. A birthday is not a moment in time. "09:00 every Monday" is not a number of seconds. "One month" is not 30 days. This library gives each of those its own type, so the compiler can catch a mix-up before your users do.

If that sounds like a lot of types, don't worry. You will mostly use two or three of them, and this page will help you pick.

## The short version

Ask yourself one question: **does this value need to know where on Earth it is?**

| You are holding | It knows | Use |
|:--|:--|:--|
| A birthday, a due date, a public holiday | the day | [](api:calendar::Date) |
| Opening hours, an alarm, "lunch at 12:30" | the time of day | [](api:calendar::Time) |
| "The meeting is on March 30 at 09:30", before you say where | the date and the time of day | [](api:calendar::DateTime) |
| The moment a payment arrived, a flight departure, "now" | the instant, and the zone to read it in | [](api:calendar::Zoned) |
| "One month later", "every two weeks", someone's age | an amount of calendar time | [](api:calendar::Period) |
| "This took 1.5 seconds", "time out after 30 seconds" | an exact length | `std::time::duration` |

When you are unsure, `Zoned` is usually the right answer for anything that happened, and `Date` for anything a person would write on a paper calendar.

## Plain values: Date, Time and DateTime

These three have no time zone. They describe what is written on a calendar or shown on a clock, not a particular moment.

```echo
$birthday = Date(1990, .march, 14);
$opening = Time(8, 30);
$meeting = DateTime(2026, .march, 30, 9, 30);

echo "{$birthday}";              // 1990-03-14
echo "{$opening}";               // 08:30:00
echo "{$meeting}";               // 2026-03-30T09:30:00
```

A `DateTime` is simply a `Date` and a `Time` side by side. You may build one from the other two, and take it apart again:

```echo
$meeting = Date(2026, .march, 30)->at(Time(9, 30));

echo "{$meeting->date}";         // 2026-03-30
echo "{$meeting->time}";         // 09:30:00
```

Because these types have no zone, every day is exactly 24 hours long for them. That is the right model for a birthday, and the wrong one for a flight. [Dates](./dates.md) and [Times and date-times](./times.md) cover them in full.

## Zoned: a moment in a place

A [](api:calendar::Zoned) is a real instant together with the [](api:calendar::Zone) you are reading it in. It knows its UTC instant, its offset, and what the wall clock there says:

```echo
$zurich = Zone(named: 'Europe/Zurich');
$standup = DateTime(2026, .march, 30, 9, 30)->at($zurich);

echo "{$standup}";               // 2026-03-30T09:30:00+02:00[Europe/Zurich]
echo "{$standup->offset}";       // +02:00
echo "{$standup->date}";         // 2026-03-30
```

This is the everyday type for anything that actually happens. It is the only one that can tell you that March 29 in Zurich was 23 hours long. [Zoned date-times](./zoned.md) is the place to start.

## Period and duration: amounts of time

There are two ways to say "how much time", and they are not the same:

- A [](api:calendar::Period) is calendar time: years, months, weeks, days, and clock units, each kept as you wrote it. One month is 28 to 31 days depending on where you add it.
- A `std::time::duration` is an exact length, counted in nanoseconds. One hour is always 3600 seconds.

```echo
$zurich = Zone(named: 'Europe/Zurich');
$saturday = DateTime(2026, .march, 28, 9, 0)->at($zurich);

echo "{$saturday + Period::days(1)}";               // 2026-03-29T09:00:00+02:00[Europe/Zurich]
echo "{$saturday + time::duration::hours(24)}";     // 2026-03-29T10:00:00+02:00[Europe/Zurich]
```

The first keeps the meeting at 09:00. The second moves exactly 24 hours forward, which lands at 10:00 because Sunday skipped an hour. Both are right, they just answer different questions. [Periods and durations](./periods.md) explains when to reach for which.

## The supporting cast

A few smaller types show up along the way. You will meet each one on the page where it matters:

| Type | What it is |
|:--|:--|
| [](api:calendar::Month), [](api:calendar::Weekday) | Enums: `.march`, `.friday` |
| [](api:calendar::Week), [](api:calendar::WeekRule) | A numbered week of a year, and the rule that numbers them |
| [](api:calendar::Offset) | A fixed distance from UTC, like `+02:00` |
| [](api:calendar::Zone) | The rules of a time zone, from the tz database |
| [](api:calendar::DateRange), [](api:calendar::Interval) | A span of days, and a span of instants |
| [](api:calendar::Format), [](api:calendar::Locale) | A compiled pattern, and the language it prints in |
| [](api:calendar::Calendar) | The context: zone, language, clock and week rule together |

## Literals and runtime values

Every type that can be built from numbers or text offers two ways to do it, and the difference is worth knowing early.

A value you write in your source code is either right or a bug. So the plain constructors stop the program, with a clear message, when the value is impossible:

<!-- check: fail -->
```echo
$broken = Date(2026, .february, 30);    // stops: 2026-02 has no day 30
```

A value that arrives at runtime, from a form, a file, or a database, deserves better than a crash. For those you may use `::of` and `::parse`, which return a `result` and let you decide what happens:

```echo
$fromForm = Date::of(2026, 2, 30);

if ($fromForm->failed()) {
    echo "{$fromForm->failure()}";
}
// 2026-02 has no day 30

Date $due = guard Date::of(2026, 2, 28);
echo "{$due}";                   // 2026-02-28
```

The same split runs through the whole library: `Time(...)` and `Time::of`, `Zone(named: ...)` and `Zone::named`, `Format('...')` and `Format::of`. Short forms like `+` and `add(Period)` clamp or stop, and their long forms take an explicit rule and return a `result`. [Errors](./errors.md) lists every one.
