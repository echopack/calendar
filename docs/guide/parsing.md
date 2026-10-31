---
description: Reading dates and times from text, strictly: ISO and RFC 9557, your own patterns, other languages, email and HTTP dates, and what a refusal tells you.
---

# Parsing

Text from the outside world is messy. Users type `1.8.26`, APIs send three different timestamp formats, and email headers have been creatively formatted since the 1980s. A parser that guesses will sometimes guess wrong, and a wrong date that looks right is far worse than a clear error.

So this parser doesn't guess. It reads exactly the shape you ask for, fills in nothing from today, and when the text doesn't fit, it tells you which field was wrong and at which byte. Every `parse` returns a `result`, so the decision about what to do next stays with you.

## ISO text

Without a pattern, each type reads its own ISO form, the same form it prints:

```echo
Date $launch = guard Date::parse('2026-03-27');
Time $standup = guard Time::parse('09:30');
DateTime $meeting = guard DateTime::parse('2026-03-27T09:30');
Zoned $paid = guard Zoned::parse('2026-03-27T09:30:00+01:00[Europe/Zurich]');
Period $notice = guard Period::parse('P1M');
Offset $india = guard Offset::parse('+05:30');

echo "{$paid}";                  // 2026-03-27T09:30:00+01:00[Europe/Zurich]
```

So anything you have printed, you may read back exactly. `DateTime::parse` accepts a space instead of the `T`, and seconds and fractions are optional for every time.

ISO is strict about its shape. A one-digit month, or a date without dashes, is refused:

```echo
$short = Date::parse('2026-3-27')->failure();
$compact = Date::parse('20260327')->failure();

echo "{$short}";                 // expected a two-digit month at byte 5
echo "{$compact}";               // expected '-' at byte 4
```

For zoned text, [Zoned date-times](./zoned.md#parsing) shows what happens with `Z`, a missing zone, or an offset that disagrees with the zone.

## Your own patterns

When the text has another shape, describe it with a [](api:calendar::Format). The pattern letters are the same ones [formatting](./formatting.md) uses:

```echo
$swiss = Format('dd.MM.yyyy');

Date $offsite = guard Date::parse('01.08.2026', $swiss);
Time $evening = guard Time::parse('9:30 pm', Format('h:mm a'));

echo "{$offsite}";               // 2026-08-01
echo "{$evening}";               // 21:30:00
```

`Date`, `Time`, `DateTime` and `Zoned` all take a format as the second argument to `parse`.

### What strict means

A few rules make patterns predictable. They might feel picky at first, but each one rules out a real class of bugs:

- **Nothing is filled in from today.** A pattern without a day can't produce a date, so `MM.yyyy` is refused for `Date::parse`, rather than quietly picking the 1st.
- **Every digit is accounted for.** `dd` needs two digits, so `1.8.2026` doesn't match `dd.MM.yyyy`.
- **Literal text must match exactly**, and nothing may follow the end of the pattern. A stray space is an error, not something to trim.
- **Everything must agree.** A weekday that doesn't match the date is refused.

```echo
$swiss = Format('dd.MM.yyyy');

$noDay = Date::parse('03.2026', Format('MM.yyyy'))->failure();
$trailing = Date::parse('01.08.2026 ', $swiss)->failure();
$impossible = Date::parse('31.02.2026', $swiss)->failure();
$mismatch = Date::parse('Monday 27.03.2026', Format('EEEE dd.MM.yyyy'))->failure();

echo "{$noDay}";                 // the text has no day
echo "{$trailing}";              // unexpected text at byte 10
echo "{$impossible}";            // 2026-02 has no day 31
echo "{$mismatch}";              // weekday does not match the rest of the text
```

When you want to accept several shapes, say `dd.MM.yyyy` and `d.M.yyyy`, you may try each format in turn and keep the first one that succeeds. That way, every shape you accept is one you chose on purpose.

### Two-digit years

`yy` reads two digits and needs to know which century you mean. By default it is 2000 to 2099, and it never depends on today's date, so the same text always parses the same way. For birthdays and other dates in the past, you may move the window with `pivot`:

```echo
$short = Format('dd/MM/yy');
$birthdays = Format('dd/MM/yy')->pivot(1950);

Date $future = guard Date::parse('14/03/68', $short);
Date $past = guard Date::parse('14/03/68', $birthdays);

echo "{$future}";                // 2068-03-14
echo "{$past}";                  // 1968-03-14
```

`pivot(1950)` reads `00` to `99` as 1950 to 2049.

### Names in other languages

Month and weekday names are read in the format's language, ignoring ASCII case:

```echo
$french = Format('d MMMM yyyy')->locale(Locale::fr());

Date $one = guard Date::parse('27 mars 2026', $french);
Date $two = guard Date::parse('27 MARS 2026', $french);

echo "{$one == $two}";           // true
```

One-letter "narrow" names (`MMMMM`, `EEEEE`) can't be parsed, since `M` could be March or May. Patterns with them are fine for output but refused for input.

### Zones in patterns

To read a zoned value with a pattern, the pattern needs an offset (`XXX`, `Z`, `O`) or a zone id (`VV`):

```echo
$form = Format('dd.MM.yyyy HH:mm VV');

Zoned $meeting = guard Zoned::parse('27.03.2026 09:30 Europe/Zurich', $form);

echo "{$meeting}";               // 2026-03-27T09:30:00+01:00[Europe/Zurich]
```

Zone abbreviations like `CET` (the `z` letter) can't be parsed, because the same abbreviation means different things in different places. When the text has no zone at all, read it with a [Calendar](./calendar.md), which places it in its own zone:

```echo
$zurich = Zone(named: 'Europe/Zurich');
$cal = Calendar($zone: $zurich);

Zoned $offsite = guard $cal->parse('01.08.2026 10:00', 'dd.MM.yyyy HH:mm');

echo "{$offsite}";               // 2026-08-01T10:00:00+02:00[Europe/Zurich]
```

## Email and HTTP dates

The [presets](./formatting.md#presets-for-protocols) parse too. `Format::rfc3339()` reads internet timestamps, with a fraction of any length:

```echo
Zoned $logged = guard Zoned::parse('2026-03-27T09:30:00.5+01:00', Format::rfc3339());

echo "{$logged}";                // 2026-03-27T09:30:00.500+01:00[+01:00]
```

`Format::rfc2822()` is lenient on purpose, because real email headers are all over the place. It accepts a missing weekday, a one-digit day, two-digit years, missing seconds, the old US zone names like `EST` and `PDT`, and a trailing comment:

```echo
Zoned $proper = guard Zoned::parse('Fri, 27 Mar 2026 09:30:00 +0100', Format::rfc2822());
Zoned $sloppy = guard Zoned::parse('27 Mar 26 09:30 GMT', Format::rfc2822());

echo "{$proper}";                // 2026-03-27T09:30:00+01:00[+01:00]
echo "{$sloppy}";                // 2026-03-27T09:30:00+00:00[UTC]
```

`Format::http()` reads the date format of HTTP headers such as `If-Modified-Since`.

These presets give you a fixed-offset zone, since that is all the text says. An offset of zero is `UTC`, the same zone `Zone::utc()` gives you. Call `to` with a named zone when you'd like to show the result somewhere specific.

## Handling refusals

Every parse failure is a [](api:calendar::ParseError). It prints as a readable message, and you may also match on its cases to react differently, for example to highlight the right field in a form:

<!-- check: fail -->
```echo
$swiss = Format('dd.MM.yyyy');

Date $offsite = guard Date::parse('31.02.2026', $swiss) else ($e) {
    $message = match ($e) {
        .impossible($reason) => "That day doesn't exist.",
        .trailing($at) => "Please remove the extra text.",
        else => "Please use the format DD.MM.YYYY.",
    };
    die($message);
};
```

The positions in a parse error are byte offsets into the text. [Errors](./errors.md#parseerror) lists every case.

::: tip Leap seconds
A seconds field of `60` is accepted and read as `59`. Leap seconds do appear in real timestamps now and then, and this way they never break a parse. The library counts time the way Unix does, without leap seconds.
:::
