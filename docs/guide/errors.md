---
description: The four error enums, every case and what returns it, and the list of calls that stop the program instead.
---

# Errors

Things go wrong with dates in a small number of ways: a day that doesn't exist, text that doesn't fit, a zone that can't be found, a pattern with a typo. Each kind has its own enum, so you may match on exactly the case you care about, and each one prints as a readable message, so you don't have to.

## Two kinds of failure

Before the list, a quick reminder of the rule from [Choosing a type](./types.md#literals-and-runtime-values):

- **Values that arrive at runtime** go through `::of`, `::parse`, `Zone::named`, `Format::of`, or a long form like `add($period, .reject)`. These return a `result` with one of the errors below.
- **Values you write in your code** go through the plain constructors and short forms. A bad one is a bug, so it stops the program with a clear message.

```echo
$fromForm = Date::of(2026, 2, 30);

Date $due = guard $fromForm else ($e) {
    echo "{$e}";                 // 2026-02 has no day 30
    return;
};
```

Every error prints as a sentence through string interpolation, as you see above. The positions in parse and format errors are byte offsets into the text.

## DateError

A [](api:calendar::DateError) means a date or time could not be built or moved.

#### `outOfRange(Field $field, int64 $value)`

A component is outside its fixed range: month 13, hour 25, minute 60. Prints `month 13 is out of range`.

#### `noDay(int64 $year, int64 $month, int64 $day)`

The month exists but doesn't have that day: February 30, or January 31 plus one month under `.reject`. Prints `2026-02 has no day 30`.

#### `overflow`

The result would land outside the years -9999 to 9999.

#### `gap(string $zone)`

The wall-clock time doesn't happen in the zone, because the clocks jumped past it, and the rule was `.reject`. Prints `that wall-clock time is skipped in Europe/Zurich`.

#### `overlap(string $zone)`

The wall-clock time happens twice in the zone, and the rule was `.reject`.

**Returned by:** `Date::of`, `Time::of`, `DateTime::of`, `Zoned::of`, and the long forms `add` and `sub` that take an [](api:calendar::Overflow), and `DateTime->resolve`. [Daylight saving time](./dst.md) covers `gap` and `overlap`.

## ParseError

A [](api:calendar::ParseError) means text didn't parse.

#### `expected(usize $at, string $what)`

Something else was needed at that byte: a digit, a `-`, a month name. Prints `expected '-' at byte 4`.

#### `invalid(Field $field, int64 $value, usize $at)`

A field was read, but its value is impossible on its own, like month 13. Prints `month 13 at byte 5 is out of range`.

#### `trailing(usize $at)`

The value was read completely, and there was text left over. Prints `unexpected text at byte 10`.

#### `missing(Field $field)`

The pattern never supplies a field the result needs, like a date without a day. Prints `the text has no day`.

#### `impossible(DateError $reason)`

Each field is fine, but together they don't make a valid value, like February 31. Holds the [DateError](#dateerror) and prints its message.

#### `unknownZone(string $id, usize $at)`

A zone name in the text isn't known to the zone source.

#### `conflict(usize $at)`

The offset in the text disagrees with what the named zone says for that time, as in `+05:00[Europe/Zurich]`.

#### `mismatch(Field $field)`

Two fields disagree, like a weekday that isn't the date's weekday, or PM with hour 9 on a 24-hour clock. Prints `weekday does not match the rest of the text`.

**Returned by:** every `parse`: on `Date`, `Time`, `DateTime`, `Zoned`, `Period`, `Offset`, and `Calendar`. [Parsing](./parsing.md) has examples.

## ZoneError

A [](api:calendar::ZoneError) means a time zone could not be loaded.

#### `unknown(string $id)`

No zone with that name in the source. Prints `no time zone named 'Europe/Zurch'`.

#### `invalid(string $id)`

The name isn't a zone name at all: empty, absolute, or trying to leave the zone directory with `..`. Names are checked before they reach the filesystem.

#### `corrupt(string $id)`

The zone file exists, but isn't valid zone data.

#### `unreadable(string $id, string $reason)`

The zone file couldn't be read, for example because of its permissions.

**Returned by:** `Zone::named`, `Calendar->zone`, and `load` on any [](api:calendar::ZoneSource). See [Zones and tz data](./zones.md).

## FormatError

A [](api:calendar::FormatError) means a pattern was refused. Several of these aren't syntax errors but classic mix-ups, and their messages suggest the likely fix.

#### `empty`

The pattern is empty.

#### `unknownLetter(string $letter, usize $at)`

An ASCII letter that isn't a pattern letter, like a bare `T`. Quote literal text: `'T'`.

#### `tooLong(string $letter, usize $at)`

Too many of one letter in a row, like `MMMMMM`.

#### `unclosedQuote(usize $at)`

A single quote is opened and never closed.

#### `weekYearWithoutWeek(usize $at)`

`Y` (the week-based year) without `w`. You almost certainly meant `y`.

#### `weekWithoutWeekYear(usize $at)`

`w` together with `y`. Week numbers belong with the week-based year `Y`.

#### `dayOfYearWithMonth(usize $at)`

`D` (the day of the year) next to a month. You almost certainly meant `d`.

#### `minuteWithoutHour(usize $at)`

`m` (minutes) in a pattern with no hour. You almost certainly meant `M`.

#### `hourWithoutPeriod(string $letter, usize $at)`

A 12-hour clock without `a`. `$letter` is the one the pattern used, `h` or `K`. You almost certainly meant `H`.

**Returned by:** `Format::of` only. `Format('...')` and pattern strings passed to `format` stop the program instead. See [Formatting](./formatting.md#mistakes-are-caught-early).

## What stops the program

These calls stop the program on bad input, because they are meant for values written in your code. Each has a counterpart that returns a `result`, listed on the right:

| Stops | Returns a `result` instead |
|:--|:--|
| `Date(...)`, `Time(...)`, `DateTime(...)` | `Date::of`, `Time::of`, `DateTime::of` |
| `Zoned($instant, $zone)` out of range | `Zoned::of` |
| `Zone(named: ...)` | `Zone::named`, `Calendar->zone` |
| `Format('...')`, a pattern string in `format` | `Format::of` |
| `$date + $period` past the year 9999 | `add($period, .clamp)` |
| `Offset(...)` past ±25:59:59 | `Offset::parse` |

A few more are always bugs in the calling code, so they have no `result` form:

- Adding a period with hours or smaller to a `Date`. Check with `Period->dated()` first.
- Rounding or taking the start of a unit the type doesn't have, like `.hour` on a `Date`.
- Formatting a field the value doesn't have, like `HH` on a `Date`.
- A `DateRange` or `Interval` whose end is before its start, or `every` with a step that doesn't move forward.
- A `WeekRule` with fewer than 1 or more than 7 minimal days.
- A custom `Locale` whose lists have the wrong length.
