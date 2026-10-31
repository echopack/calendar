---
description: The Calendar bundles a zone, a language, a clock, a zone source and a week rule, so nothing about dates is global.
---

# The Calendar

Sooner or later, every program that deals with dates asks the same questions: what is today, which time zone are we in, which language should this print in, and does the week start on Monday or Sunday? Many libraries keep the answers in global settings. That works until a test runs on a machine in another zone, or one request needs French while the next needs German.

This library keeps no global state at all. Instead, you build one [](api:calendar::Calendar) where your program starts and hand it to whatever needs today's date. Think of it as the desk calendar for one person: it knows where they are, what language they read, and what the clock on their wall says.

## Creating a calendar

Every argument has a sensible default, so the smallest calendar is just `Calendar()`. It uses this machine's zone, English, the system clock, the system's tz database, and ISO weeks. In practice you will usually say at least where you are:

```echo
$cal = Calendar($zone: Zone(named: 'Europe/Zurich'), $locale: Locale::de());
```

You may pass any of the five arguments by name, and leave out the rest:

| Argument | Default | What it decides |
|:--|:--|:--|
| `$zone` | `Zone::local()` | Where "now" and "today" are read, and where wall clocks are placed |
| `$locale` | `Locale::en()` | The language for pattern strings and human-readable output |
| `$clock` | `SystemClock()` | What "now" is. Swap it out in [tests](./clocks.md) |
| `$zones` | `Zone::system()` | Where `zone()` and `parse()` look up zone names. See [Zones and tz data](./zones.md) |
| `$weekRule` | `WeekRule::iso()` | Which day starts a week, and how weeks are numbered |

A calendar is `#[atomic]`, so you may share one between threads without wrapping it in anything.

## Now and today

For the examples on this page, let's pin the clock to Friday, March 27, 2026 at 09:30 in Zurich, so the output stays the same:

```echo
$zurich = Zone(named: 'Europe/Zurich');
$clock = FixedClock(DateTime(2026, .march, 27, 9, 30)->at($zurich)->instant);
$cal = Calendar($zone: $zurich, $locale: Locale::de(), $clock: $clock);

echo "{$cal->now()}";            // 2026-03-27T09:30:00+01:00[Europe/Zurich]
echo "{$cal->today()}";          // 2026-03-27
echo "{$cal->week($cal->today())}";   // 2026-W13
```

`now()` is a [](api:calendar::Zoned) in the calendar's zone. `today()` is the [](api:calendar::Date) part of it. That distinction matters near midnight: at 00:30 in Zurich it is already tomorrow there, while it is still yesterday in UTC.

`week()` numbers a date with the calendar's week rule, so a calendar built with a Sunday-first rule gives you US-style week numbers instead.

## Placing a wall clock

When someone types "Monday at 09:30", they mean 09:30 where they are. `at` places a [](api:calendar::DateTime) in the calendar's zone:

<!-- check: continue -->
```echo
$standup = $cal->at(DateTime(2026, .march, 30, 9, 30));

echo "{$standup}";               // 2026-03-30T09:30:00+02:00[Europe/Zurich]
```

If the wall clock falls into a daylight saving gap or overlap, `at` resolves it the same way most calendar apps do. [Daylight saving time](./dst.md) explains exactly what that means, and how to choose differently.

## Formatting and parsing in the calendar's language

The calendar's `format` takes a pattern string and fills in its own language and week rule for you:

<!-- check: continue -->
```echo
echo $cal->format($standup, 'EEEE, d. MMMM, HH:mm');   // Montag, 30. März, 09:30
```

`parse` works the other way around. Text with no zone or offset in it is read in the calendar's zone, which is usually what a form field means:

<!-- check: continue -->
```echo
Zoned $offsite = guard $cal->parse('01.08.2026 10:00', 'dd.MM.yyyy HH:mm');

echo "{$offsite}";               // 2026-08-01T10:00:00+02:00[Europe/Zurich]
```

You may also hand either method a compiled [](api:calendar::Format). A compiled format keeps its own language, so the calendar leaves it alone:

<!-- check: continue -->
```echo
$english = Format('EEEE, d MMMM');

echo $cal->format($standup, $english);   // Monday, 30 March
```

That is deliberate. A format you built for an English API response should not suddenly switch to German because the calendar it passed through is German. [Formatting](./formatting.md) and [Parsing](./parsing.md) go into detail.

## Human-readable distances

`human` says how far away a moment or a day is, in the calendar's language and from the calendar's "now":

<!-- check: continue -->
```echo
echo $cal->human($standup);                    // in 3 Tagen
echo $cal->human(Date(2026, .march, 26));      // gestern
```

You may pass a [](api:calendar::Relative) to tune the wording. [Human-readable output](./human.md) shows how.

## Looking up zones

`zone` loads a zone by name from the calendar's zone source, and returns a `result` because the name may not exist:

<!-- check: continue -->
```echo
Zone $tokyo = guard $cal->zone('Asia/Tokyo');

echo "{$standup->to($tokyo)}";   // 2026-03-30T16:30:00+09:00[Asia/Tokyo]
```

This is the method to use for zone names that come from users or configuration. [Zones and tz data](./zones.md) covers where zones come from and how to point a calendar somewhere else.

## Passing the calendar around

Since the calendar is the only place "now" comes from, it is worth passing it explicitly to the code that needs it:

```echo
internal function invoiceDue(Calendar $cal) : Date
{
    return $cal->today() + Period::days(30);
}

$cal = Calendar($zone: Zone(named: 'Europe/Zurich'));
$due = invoiceDue($cal);
```

`Calendar` is a class, so passing it by value hands over a shared reference, not a copy. It feels like a little extra typing at first. The payoff comes the first time you write a test for `invoiceDue`: you hand it a calendar with a [fixed clock](./clocks.md), and the test passes on every machine, on every day of the year.
