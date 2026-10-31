---
description: Saying how far away a moment or a day is, "in 3 days", "yesterday", "2 hours ago", and tuning the wording with Relative.
---

# Human-readable output

People don't think in timestamps. "Your trial ends in 3 days" or "last seen 2 hours ago" is much friendlier than `2026-03-30T09:30:00+02:00`. This page covers the two ways the library produces that kind of text: relative phrases for moments and days, and spoken periods like "2 years, 3 months and 1 day".

## Relative to now

The easiest way is through a [Calendar](./calendar.md), since it already knows "now" and the language. For the examples, let's fix the clock at Friday, March 27, 2026, 09:30 in Zurich:

```echo
$zurich = Zone(named: 'Europe/Zurich');
$now = DateTime(2026, .march, 27, 9, 30)->at($zurich);
$cal = Calendar($zone: $zurich, $clock: FixedClock($now->instant));

echo $cal->human($now + Period::days(3));             // in 3 days
echo $cal->human($now - Period::days(1));             // yesterday
echo $cal->human($now + time::duration::secs(20));    // now
echo $cal->human($now - time::duration::mins(90));    // 1 hour ago
echo $cal->human($now + Period::months(5));           // in 5 months
```

`human` also takes a [](api:calendar::Date), and then compares it with today:

<!-- check: continue -->
```echo
echo $cal->human(Date(2026, .march, 26));     // yesterday
echo $cal->human(Date(2026, .march, 27));     // today
echo $cal->human(Date(2026, .april, 10));     // in 2 weeks
```

### How the distance is measured

The distance is counted on the calendar, the same way [differences](./differences.md) are. From January 31 to March 1 is "1 month", and from Friday 09:30 to Monday 09:30 is "3 days", even across a DST change that made the weekend an hour shorter. Nobody calls that "2 days and 23 hours".

The largest unit with a count of at least one wins, and the rest is cut off. 45 days is "in 1 month", and 90 minutes is "1 hour ago". Anything closer than 45 seconds is "now".

## Tuning the wording

When the defaults don't quite fit, you may pass a [](api:calendar::Relative) with the options you'd like to change. Each option has a sensible default, so you only name the ones you need.

### Always use numbers

"Tomorrow" and "now" read nicely in a sentence, but in a table column you may prefer the numbers. Set `$numeric`:

<!-- check: continue -->
```echo
$numbers = Relative($numeric: true);

echo $cal->human($now + Period::days(1), $numbers);            // in 1 day
echo $cal->human($now + time::duration::secs(20), $numbers);   // in 20 seconds
```

### A fixed unit

Sometimes you'd like every row in the same unit, say weeks for a project plan. Set `$unit`:

<!-- check: continue -->
```echo
$inWeeks = Relative($unit: .week);

echo $cal->human($now + Period::days(45));             // in 1 month
echo $cal->human($now + Period::days(45), $inWeeks);   // in 6 weeks
```

### Rounding instead of cutting off

By default, 90 minutes is "1 hour ago", since the extra half hour is cut off. Pass a [](api:calendar::Rounding) mode to round instead:

<!-- check: continue -->
```echo
$rounded = Relative($rounding: .halfExpand);

echo $cal->human($now - time::duration::mins(90), $rounded);   // 2 hours ago
```

### The "now" window

`$nowWithin` is the number of seconds that still count as "now". The default is 45. Set it to 0 if every second should be counted.

## Other languages and other reference points

`human` uses the calendar's language. Outside a calendar, `Relative->render` takes the moment, the reference point, and a [locale](./locales.md):

```echo
$zurich = Zone(named: 'Europe/Zurich');
$now = DateTime(2026, .march, 27, 9, 30)->at($zurich);
$style = Relative();

echo $style->render($now + Period::days(3), $now, Locale::de());    // in 3 Tagen
echo $style->render($now - Period::days(1), $now, Locale::fr());    // hier
echo $style->render($now + Period::weeks(2), $now, Locale::fr());   // dans 2 semaines
echo $style->render($now + Period::days(1), $now, Locale::it());    // domani
```

Passing the reference point yourself is also how you say "3 days before the deadline" instead of "3 days ago": use the deadline as the second argument.

## Spoken periods

When you already have a [](api:calendar::Period), say someone's age or the length of a contract, `human` writes it out in full:

```echo
$birthday = Date(1990, .march, 14);
$age = $birthday->until(Date(2026, .march, 27));

echo $age->human();                    // 36 years and 13 days
echo $age->human(Locale::de());        // 36 Jahre und 13 Tage

$contract = Period($years: 2, $months: 3, $days: 1);

echo $contract->human(Locale::fr());   // 2 ans, 3 mois et 1 jour
echo $contract->human(Locale::it());   // 2 anni, 3 mesi e 1 giorno
```

Every non-zero field is listed, joined with commas and the locale's word for "and". An empty period is "0 seconds". Nanoseconds are left out, since nobody says them out loud.
