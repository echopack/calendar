---
description: What the library doesn't do yet, the limits it works within, and how to run its tests.
---

# What isn't here

This library tries to do the common things with dates really well. A few things next to that are deliberately left out, or not built yet. Here they are in one place, so you don't go looking.

## Not here yet

- **Bundled zone data.** Named zones always come from a directory of compiled zone files, by default your system's `/usr/share/zoneinfo`. You may point [](api:calendar::SystemZones) at another directory, but you can't yet embed the tz database in your binary, or build a `Zone` from data you load yourself.
- **Zone sources for `Zone::named`.** `Zone(named: ...)` and `Zone::named` always read from the system source. A custom [](api:calendar::ZoneSource) is used by `Calendar->zone` and `Calendar->parse`, and by `Zoned::parse` when you pass it in.
- **Other calendars.** Dates are Gregorian, extended backwards to the year -9999 ("proleptic"). There is no Julian, Hebrew, Islamic or Japanese calendar.
- **More languages.** English, German, French and Italian are built in. You may [build your own](./locales.md#building-your-own-locale) for any other language.
- **Printing enums directly.** [](api:calendar::Month), [](api:calendar::Weekday) and [](api:calendar::Zone) don't interpolate into strings. Use `->number()`, a [locale](./locales.md#asking-a-locale-for-words), or `$zone->id`.
- **Parsing a week.** A [](api:calendar::Week) prints as `2026-W13`, but there is no `Week::parse`. Parse a date with a `YYYY-'W'ww-e` pattern instead.
- **Localized interpolation.** `"{$date:dd MMMM}"` always uses English names. Use `format` with a locale when the language matters.

## Limits

- **Years -9999 to 9999.** Anything outside fails with `DateError::overflow` or stops the program, depending on the call.
- **Offsets up to ±25:59:59.** That covers every offset the tz database has ever used.
- **No leap seconds.** Time is counted the way Unix counts it. A seconds field of `60` in text is read as `59`, so real-world timestamps with a leap second still parse.
- **Counting in small units.** `until` in milliseconds, microseconds or nanoseconds fits in an `int64`, which is about 292 years at nanosecond precision.

## Running the tests

From the repository root:

```bash
echoc test
```

The zone tests read real zone files, so they need a tz database at `/usr/share/zoneinfo` or wherever `TZDIR` points. They expect the 2026 rules for Zurich, New York, Lord Howe Island, Apia and Casablanca. CI runs the suite on macOS 15 and Ubuntu 24.04.

The [tour](https://github.com/echopack/calendar/tree/main/examples/tour) walks through most of the library in one short program:

```bash
cd examples/tour
echoc run
```
