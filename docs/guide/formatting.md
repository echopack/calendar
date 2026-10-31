---
description: Turning dates and times into text with patterns, compiled formats, the RFC presets, other languages, and string interpolation.
---

# Formatting

Sooner or later, every date ends up in front of a person or in a file: "Friday, 27 March 2026" on an invoice, `27.03.2026` in a Swiss form, `Fri, 27 Mar 2026 08:30:00 GMT` in an HTTP header. This page shows the ways to produce each of those.

## The default text

Every value has one canonical text form, and that is what you get when you put it in a string. It is always ISO 8601 (or RFC 9557 for a [](api:calendar::Zoned)), so it is unambiguous, sorts correctly as text, and can be parsed back exactly:

```echo
$zurich = Zone(named: 'Europe/Zurich');
$launch = Date(2026, .march, 27);
$standup = DateTime(2026, .march, 27, 9, 30)->at($zurich);

echo "{$launch}";                // 2026-03-27
echo "{$standup}";               // 2026-03-27T09:30:00+01:00[Europe/Zurich]
echo "{$standup->offset}";       // +01:00
echo "{$launch->week()}";        // 2026-W13
```

That is the form to use for logs, databases and APIs. For people, you'll want a pattern.

## Patterns

A pattern describes the text you want with letters. `yyyy` is the year, `MM` the month as two digits, `EEEE` the full weekday name, and so on. Every type has a `format` method that takes one:

```echo
$zurich = Zone(named: 'Europe/Zurich');
$standup = DateTime(2026, .march, 27, 9, 30)->at($zurich);

echo $standup->format('EEEE, d MMMM yyyy');        // Friday, 27 March 2026
echo $standup->format('dd.MM.yyyy HH:mm');         // 27.03.2026 09:30
echo $standup->format('h:mm a');                   // 9:30 AM
echo $standup->format('HH:mm z');                  // 09:30 CET
```

The number of letters matters. One `M` is `3`, two are `03`, three are `Mar`, and four are `March`. The full list is on the [Pattern letters](./patterns.md) page, and you don't need to remember it.

Anything that isn't a letter is copied as it is. To include letters as text, wrap them in single quotes, and write `''` for a single quote:

```echo
$launch = Date(2026, .march, 27);

echo $launch->format("'week' w 'of' YYYY");   // week 13 of 2026
echo $launch->format("d MMM ''yy");           // 27 Mar '26
```

### Mistakes are caught early

A few letters look alike and are easy to mix up. `mm` is minutes while `MM` is the month, `YYYY` is the week-based year while `yyyy` is the calendar year, and `hh` is a 12-hour clock that makes no sense without `a`. Rather than print something subtly wrong, the library refuses these patterns and suggests what you probably meant:

```echo
$minutes = Format::of('yyyy-mm-dd')->failure();
$weekYear = Format::of('YYYY-MM-dd')->failure();

echo "{$minutes}";   // 'm' is minutes and the pattern has no hour; did you mean 'M'? at byte 5
echo "{$weekYear}";  // 'Y' is the week-based year and the pattern has no week 'w'; did you mean 'y'? at byte 0
```

The `YYYY` mix-up is a classic. It prints the right year for most of the year and the wrong one for a few days around New Year, which is exactly the kind of bug nobody notices until December.

## Compiling a Format once

Every call to `format` with a string reads the pattern again. That is perfectly fine for an invoice, but in a loop that formats thousands of rows, you may compile the pattern into a [](api:calendar::Format) once and reuse it:

```echo
$swiss = Format('dd.MM.yyyy');
$march = Date(2026, .march, 1)->span(.month);

foreach ($march as const &$day) {
    $line = $swiss->render($day);
    // ...
}
```

`Format('...')` stops the program on a bad pattern, since a pattern in your source code is either right or a bug. When the pattern comes from a user or a config file, `Format::of` returns a `result` with a [](api:calendar::FormatError) instead.

A compiled format may also be passed to `format` on any value, and to a [Calendar](./calendar.md).

### What a Format carries

A `Format` keeps everything it needs with it: the language for names, the week rule for `Y`, `w` and `e`, and the century for two-digit years when parsing. You set them with methods that return a new format:

```echo
$launch = Date(2026, .march, 27);

$german = Format('EEEE, d. MMMM yyyy')->locale(Locale::de());
$usWeeks = Format("'week' w")->weeks(WeekRule(.sunday, 1));

echo $german->render($launch);   // Freitag, 27. März 2026
echo $usWeeks->render($launch);  // week 13
```

Nothing is passed per call, and nothing changes behind your back. A format built for an English API response stays English, whichever calendar it passes through.

## Other languages

For a one-off, you may pass a [](api:calendar::Locale) to `format`:

```echo
$launch = Date(2026, .march, 27);

echo $launch->format('EEEE d MMMM yyyy', Locale::fr());   // vendredi 27 mars 2026
echo $launch->format('EEEE d MMMM yyyy', Locale::it());   // venerdì 27 marzo 2026
echo $launch->format('EEE, dd.MM.yy', Locale::de());      // Fr., 27.03.26
```

When most of your program speaks one language, put it in a [Calendar](./calendar.md). Its `format` fills in the calendar's language for every pattern string you give it. [Locales](./locales.md) covers the built-in languages and how to add your own.

## Presets for protocols

Some formats are defined by standards rather than by taste. [](api:calendar::Format) has a preset for each of the common ones:

```echo
$zurich = Zone(named: 'Europe/Zurich');
$standup = DateTime(2026, .march, 27, 9, 30)->at($zurich);

echo Format::rfc3339()->render($standup);   // 2026-03-27T09:30:00+01:00
echo Format::rfc2822()->render($standup);   // Fri, 27 Mar 2026 09:30:00 +0100
echo Format::http()->render($standup);      // Fri, 27 Mar 2026 08:30:00 GMT
```

- `rfc3339()` is the internet timestamp. It writes a fraction of a second only when there is one, and `Z` for UTC.
- `rfc2822()` is the email `Date:` header.
- `http()` is the form HTTP headers use, like `Last-Modified`. It always writes UTC.

All three are English, as their standards require, whatever locale you set.

## Patterns inside strings

For quick output, you may put a pattern right inside an interpolated string, after a colon:

```echo
$launch = Date(2026, .march, 27);
$standup = Time(9, 30);

echo "Launch on {$launch:dd.MM.yyyy} at {$standup:HH:mm}";   // Launch on 27.03.2026 at 09:30
```

This is handy for logs and messages. It always uses English names, and a bad pattern stops the program, so it is best kept for patterns you write yourself.

## When a value lacks a field

A pattern can only print what the value has. A [](api:calendar::Date) has no hour, and a [](api:calendar::DateTime) has no zone, so formatting `HH` on a date or `z` on a date-time stops the program:

<!-- check: fail -->
```echo
$launch = Date(2026, .march, 27);

echo $launch->format('HH:mm');   // stops: the pattern 'HH:mm' needs a time of day, and a Date has none
```

That is always a bug in the pattern, not in the data. If you need the field, convert first: `$date->at($time)` for a time, or `$dateTime->at($zone)` for a zone.
