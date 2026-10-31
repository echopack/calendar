---
description: Creating dates, reading their parts, moving to the next Monday or the end of the month, and working with months, weekdays and weeks.
---

# Dates

A [](api:calendar::Date) is a day on the calendar, with no time and no zone: a birthday, an invoice's due date, the first day of a holiday. It is the simplest type in the library, and a good one to reach for whenever a person would write the value on a paper calendar.

Dates run from the year -9999 to 9999 in the proleptic Gregorian calendar. Year 0 exists and is 1 BC, which keeps the arithmetic simple.

## Creating dates

The most common way is to write the year, the month and the day:

```echo
$launch = Date(2026, .march, 27);

echo "{$launch}";                // 2026-03-27
```

The month is a [](api:calendar::Month) enum, so `.march` reads naturally and you can never mix up the month and the day. A day that the month doesn't have stops the program, since a literal like that is a bug in your code.

### Dates from numbers

When the parts arrive at runtime, say from a database row or three form fields, you may use `Date::of`. It takes plain integers and returns a `result`:

```echo
Date $due = guard Date::of(2026, 4, 30) else ($e) {
    die("due date: {$e}");
};

echo "{$due}";                   // 2026-04-30
```

### Dates from text

`Date::parse` reads the ISO form, `yyyy-MM-dd`. For anything else, hand it a [](api:calendar::Format):

```echo
Date $iso = guard Date::parse('2026-03-27');
Date $swiss = guard Date::parse('27.03.2026', Format('dd.MM.yyyy'));

echo "{$iso == $swiss}";         // true
```

[Parsing](./parsing.md) covers patterns, languages and errors.

### Other ways in

Occasionally you may need to build a date from something other than a year, month and day. Each has a labelled constructor:

```echo
$epoch = Date(epochDay: 0);
$isoWeek = Date(week: Week(2026, 14), $weekday: .monday);

echo "{$epoch}";                 // 1970-01-01
echo "{$isoWeek}";               // 2026-03-30
```

`epochDay:` counts days from 1970-01-01, which is handy when a database stores dates as integers. `week:` takes a [](api:calendar::Week) and a weekday, numbered by ISO rules unless you pass another [](api:calendar::WeekRule).

## Reading a date

The parts are public fields, and the rest are methods:

```echo
$launch = Date(2026, .march, 27);

echo $launch->year;              // 2026
echo $launch->month->number();   // 3
echo $launch->day;               // 27
echo $launch->ordinal();         // 86
echo $launch->quarter();         // 1
echo "{$launch->week()}";        // 2026-W13
echo "{$launch->leap()}";        // false
echo $launch->epochDay();        // 20539
```

`ordinal()` is the day of the year, starting at 1. `week()` is the ISO week by default, and you may pass another rule, as described in [Weeks](#weeks) below.

### Which weekday is it?

`weekday()` returns a [](api:calendar::Weekday). In a string it prints its English name. For another language, ask a [locale](./locales.md), or use a pattern:

```echo
$launch = Date(2026, .march, 27);
$day = $launch->weekday();

echo $day->number();                 // 5
echo "{$day}";                       // Friday
echo Locale::de()->weekday($day);    // Freitag
echo $launch->format('EEEE');        // Friday
```

Weekdays are numbered the ISO way, Monday is 1 and Sunday is 7.

## Moving around

### The next or previous weekday

Sometimes you need "the next Monday" or "last Friday". `next` and `previous` always move at least one day, so asking a Friday for the next Friday gives you the one a week later:

```echo
$friday = Date(2026, .march, 27);

echo "{$friday->next(.monday)}";       // 2026-03-30
echo "{$friday->next(.friday)}";       // 2026-04-03
echo "{$friday->previous(.friday)}";   // 2026-03-20
```

### The start and end of a unit

`start` and `end` give you the first and last day of the week, month, quarter or year a date falls in:

```echo
$launch = Date(2026, .march, 27);

echo "{$launch->start(.month)}";       // 2026-03-01
echo "{$launch->end(.month)}";         // 2026-03-31
echo "{$launch->start(.week)}";        // 2026-03-23
echo "{$launch->start(.quarter)}";     // 2026-01-01
```

When you want the whole unit as a range you can loop over, `span` returns a [](api:calendar::DateRange). [Ranges and intervals](./ranges.md) has more.

### Changing one part

`with` returns a copy with some parts replaced. You may name only the parts you want to change:

```echo
$launch = Date(2026, .march, 27);

$firstOfMonth = $launch->with($day: 1);
$nextSummer = $launch->with($year: 2027, $month: .june);

echo "{$firstOfMonth}";          // 2026-03-01
echo "{$nextSummer}";            // 2027-06-27
```

If the new month is too short for the day, the day is clamped to the month's last day. January 31 with the month set to February becomes February 28 (or 29 in a leap year):

```echo
$endOfJanuary = Date(2026, .january, 31);
$february = $endOfJanuary->with($month: .february);

echo "{$february}";              // 2026-02-28
```

### Adding and subtracting

You may add a [](api:calendar::Period) of years, months, weeks or days with `+` and `-`:

```echo
$launch = Date(2026, .march, 27);

echo "{$launch + Period::weeks(2)}";    // 2026-04-10
echo "{$launch - Period::months(1)}";   // 2026-02-27
```

There is a lot to say about what "one month later" means, so it has its own page: [Periods and durations](./periods.md). Counting the distance between two dates is covered in [Differences](./differences.md).

## Months and weekdays

[](api:calendar::Month) and [](api:calendar::Weekday) are enums with a few helpers of their own. They are handy when you need to do something per month, or cycle through the week. In a string, both print their English name, so `"{$month}"` is `March`:

```echo
echo Month::february->days(2028);          // 29
echo Month::november->quarter();           // 4
echo Month::december->next()->number();    // 1
echo Weekday::friday->until(.monday);      // 3
```

`Month::of` and `Weekday::of` turn a number into the enum, and return `null` when the number is out of range, so you may use `??` or `guard` with them.

## Weeks

Week numbers sound simple until you look at January 1. Is it in week 1 of the new year, or in the last week of the old one? The answer depends on a rule, and this library makes you pick it, defaulting to ISO 8601: weeks start on Monday, and week 1 is the first week with at least four days in the new year.

```echo
$newYear2026 = Date(2026, .january, 1);
$newYear2027 = Date(2027, .january, 1);

echo "{$newYear2026->week()}";   // 2026-W01
echo "{$newYear2027->week()}";   // 2026-W53
```

January 1, 2027 is a Friday, so under ISO rules it still belongs to the last week of 2026. A [](api:calendar::Week) carries both numbers, the week-based year and the week, so it never loses that information.

### Other week rules

The US and some other regions start the week on Sunday and count the week containing January 1 as week 1. You may build any [](api:calendar::WeekRule) from a first day and a minimum number of days, or take the one a [locale](./locales.md) suggests:

```echo
$us = WeekRule(.sunday, 1);
$newYear = Date(2027, .january, 1);
$launch = Date(2026, .march, 27);

echo "{$newYear->week($us)}";            // 2027-W01
echo "{$launch->start(.week, $us)}";     // 2026-03-22
echo WeekRule::iso()->weeks(2026);       // 53
```

Everything that cares about weeks (`week`, `start`, `end`, `span`, and the `Y`, `w` and `e` pattern letters) takes the rule as an argument. Nothing picks one up behind your back, not even from the locale.
