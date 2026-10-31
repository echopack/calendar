---
description: Every pattern letter, what each run length prints, quoting rules, and which letters can be parsed.
---

# Pattern letters

This page lists every letter a [](api:calendar::Format) pattern understands. You don't need to learn it by heart. Keep it open while you write a pattern, and the library will catch the common mix-ups for you, as shown in [Formatting](./formatting.md#mistakes-are-caught-early).

All examples on this page format the same moment, Friday, March 27, 2026, at 09:30 in Zurich:

```echo
$zurich = Zone(named: 'Europe/Zurich');
$standup = DateTime(2026, .march, 27, 9, 30)->at($zurich);

echo $standup->format('EEEE, d MMMM yyyy, HH:mm z');   // Friday, 27 March 2026, 09:30 CET
```

## The rules

- A letter is a field. Repeating it changes the width or the form: `M` is `3`, `MM` is `03`, `MMM` is `Mar`, `MMMM` is `March`.
- Text in single quotes is printed as it is: `'week'`. Two single quotes, `''`, are one quote.
- Anything that isn't an ASCII letter (digits, spaces, punctuation) is printed as it is.
- An ASCII letter with no meaning, such as a bare `T`, is an error. Quote it: `yyyy-MM-dd'T'HH:mm`.
- A number field pads with zeros to the number of letters: `dd` is `07`, `yyyy` is `2026`.

## Dates

| Letters | Field | Example |
|:--|:--|:--|
| `G`, `GG`, `GGG` | Era, short | `AD` |
| `GGGG` | Era, long | `Anno Domini` |
| `y`, `yyyy` | Year of the era | `2026` |
| `yy` | Year, last two digits | `26` |
| `u`, `uuuu` | Year with a sign, -9999 to 9999 | `2026`, `-0043` |
| `Q`, `QQ` | Quarter, number | `1` |
| `QQQ` | Quarter, short | `Q1` |
| `QQQQ` | Quarter, long | `1st quarter` |
| `q` ... `qqqq` | Quarter, standalone form | as `Q` |
| `M`, `MM` | Month, number | `3`, `03` |
| `MMM` | Month, short | `Mar` |
| `MMMM` | Month, long | `March` |
| `MMMMM` | Month, narrow | `M` |
| `L` ... `LLLLL` | Month, standalone form, for a month on its own | as `M` |
| `d`, `dd` | Day of the month | `27` |
| `D`, `DD`, `DDD` | Day of the year. Not together with a month | `86` |

```echo
$launch = Date(2026, .march, 27);

echo $launch->format('G yyyy');           // AD 2026
echo $launch->format('QQQQ');             // 1st quarter
echo $launch->format('MMM MMMM');         // Mar March
echo $launch->format('D');                // 86
```

`y` and `u` differ before year 1. Year 0 is 1 BC, so `y` with `G` prints the era's way of counting, while `u` prints the signed year:

```echo
$idesOfMarch = Date(-43, .march, 15);

echo $idesOfMarch->format('d MMMM y G');   // 15 March 44 BC
echo $idesOfMarch->format('uuuu-MM-dd');   // -0043-03-15
```

## Weeks and weekdays

| Letters | Field | Example |
|:--|:--|:--|
| `E`, `EE`, `EEE` | Weekday, short | `Fri` |
| `EEEE` | Weekday, long | `Friday` |
| `EEEEE` | Weekday, narrow | `F` |
| `e`, `ee` | Weekday as a number, counted from the week rule's first day | `5` |
| `eee` ... `eeeee`, `c` ... | Weekday names, as `E` | `Fri` |
| `w`, `ww` | Week of the week-based year. Not together with `y` | `13` |
| `Y`, `YYYY` | Week-based year. Only together with `w` | `2026` |

The week-based year is the year a [](api:calendar::Week) belongs to. It differs from the calendar year for a few days around New Year, which is why `Y` without `w`, and `w` with `y`, are refused. `w`, `Y` and `e` follow the format's week rule, ISO by default, set with `->weeks(...)`.

```echo
$launch = Date(2026, .march, 27);

echo $launch->format("EEEE, 'week' w 'of' YYYY");   // Friday, week 13 of 2026
echo $launch->format('e');                         // 5
```

## Times

| Letters | Field | Example |
|:--|:--|:--|
| `H`, `HH` | Hour, 0 to 23 | `09` |
| `k`, `kk` | Hour, 1 to 24 | `09` |
| `h`, `hh` | Hour, 1 to 12. Only together with `a` | `9` |
| `K`, `KK` | Hour, 0 to 11. Only together with `a` | `9` |
| `a` | AM or PM | `AM` |
| `m`, `mm` | Minute | `30` |
| `s`, `ss` | Second | `00` |
| `S` ... `SSSSSSSSS` | Fraction of a second, one digit per letter | `250` |

```echo
$precise = Time(21, 5, 30, 250000000);

echo $precise->format('HH:mm:ss.SSS');   // 21:05:30.250
echo $precise->format('h:mm a');         // 9:05 PM
```

## Zones and offsets

These need a [](api:calendar::Zoned). A [](api:calendar::DateTime) has no zone to print.

| Letters | Field | Example |
|:--|:--|:--|
| `z`, `zz`, `zzz` | Zone abbreviation | `CET` |
| `zzzz` | Zone id | `Europe/Zurich` |
| `VV` | Zone id | `Europe/Zurich` |
| `Z`, `ZZ`, `ZZZ` | Offset, basic | `+0100` |
| `ZZZZ` | Offset, GMT form | `GMT+01:00` |
| `ZZZZZ` | Offset, extended, `Z` for UTC | `+01:00` |
| `X` | Offset, hours only, `Z` for UTC | `+01` |
| `XX` | Offset, basic, `Z` for UTC | `+0100` |
| `XXX` | Offset, extended, `Z` for UTC | `+01:00` |
| `XXXX` | Offset, basic, with seconds when there are any, `Z` for UTC | `+0100` |
| `XXXXX` | Offset, extended, with seconds when there are any, `Z` for UTC | `+01:00` |
| `x` ... `xxxxx` | Offset, as `X`, but never `Z` | `+0100` |
| `O` | GMT offset, short | `GMT+1` |
| `OOOO` | GMT offset, long | `GMT+01:00` |

```echo
$zurich = Zone(named: 'Europe/Zurich');
$standup = DateTime(2026, .march, 27, 9, 30)->at($zurich);

echo $standup->format('HH:mm z (VV)');      // 09:30 CET (Europe/Zurich)
echo $standup->format('HH:mm XXX');         // 09:30 +01:00
echo $standup->format('HH:mm ZZZZ / O');    // 09:30 GMT+01:00 / GMT+1
```

## Parsing

The same letters read text back. Three of them work a little differently when parsing:

- **Narrow names** (`MMMMM`, `LLLLL`, `EEEEE`) are refused, since `M` could be March or May.
- **Zone abbreviations** (`z`) are refused, since `CST` means different things in different places. Use `VV` or an offset.
- **`yy`** reads a two-digit year in a 100-year window starting at 2000, which you may move with `pivot`. See [Parsing](./parsing.md#two-digit-years).

A number field needs at least as many digits as it has letters, so `dd` doesn't accept `7`. A single letter, like `d` or `M`, accepts one or two digits.
