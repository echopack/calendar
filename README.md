# calendar

Dates, times and time zones for Echo. Plain dates, wall clocks, zoned moments and the tz database, with formatting and parsing in English, German, French and Italian.

```echo
use calendar::{Date, DateTime, Period, Zone};

$due = Date(2026, .january, 31) + Period::months(1);
echo "{$due}";        // 2026-02-28

$meeting = DateTime(2026, .march, 28, 9, 0)->at(Zone(named: 'Europe/Zurich'));
$nextDay = $meeting + Period::days(1);
echo "{$nextDay}";    // 2026-03-29T09:00:00+02:00[Europe/Zurich]
```

**Documentation:** https://echopack.github.io/calendar

## Install

```bash
epm add echopack/calendar --git https://github.com/echopack/calendar --range ^0.1
```

## Develop

```bash
echoc test            # run the test suite
ecodoc dev            # preview the docs at http://localhost:3246
```
