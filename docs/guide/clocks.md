---
description: The Clock interface, and how FixedClock and ManualClock make code that depends on "now" easy to test.
---

# Clocks

Code that asks for the current time is famously hard to test. A test for "is this invoice overdue?" passes today and fails next month. A test for "show a banner the week before the clocks change" only passes twice a year. You could fake the system clock, but it is much nicer when the code never reads the system clock directly in the first place.

That is why a [Calendar](./calendar.md) gets "now" from a [](api:calendar::Clock), and why you may swap that clock out.

## The Clock interface

A clock is anything with a `now()` method that returns a `std::time::timestamp`. Three are built in:

| Clock | What `now()` returns |
|:--|:--|
| [](api:calendar::SystemClock) | The real time. This is the default. |
| [](api:calendar::FixedClock) | The same instant, every time. |
| [](api:calendar::ManualClock) | An instant you move forward yourself. |

You hand one to a calendar as `$clock:`, and everything that calendar does with "now" (`now()`, `today()`, `human()`) follows it.

## A clock that stands still

A [](api:calendar::FixedClock) is the simplest way to write a test that doesn't depend on the day it runs. The easiest way to get the instant is to write the wall clock you want and place it in a zone:

```echo
$zurich = Zone(named: 'Europe/Zurich');
$friday = DateTime(2026, .march, 27, 9, 30)->at($zurich);

$cal = Calendar($zone: $zurich, $clock: FixedClock($friday->instant));

echo "{$cal->today()}";          // 2026-03-27
```

Let's put that to work. Here is a small function that decides whether an invoice is overdue, and a test for it:

```echo
internal function overdue(Calendar $cal, const Date& $due) : bool
{
    return $cal->today() > $due;
}

test overdue_invoices
{
    $zurich = Zone(named: 'Europe/Zurich');
    $friday = DateTime(2026, .march, 27, 9, 30)->at($zurich);
    $cal = Calendar($zone: $zurich, $clock: FixedClock($friday->instant));

    assert(overdue($cal, Date(2026, .march, 26)));
    assert(!overdue($cal, Date(2026, .march, 27)));
}
```

That test passes on every machine, in every zone, on every day of the year. Run it with `echoc test`.

## A clock you move yourself

Sometimes a test needs time to pass: a session that expires after 30 minutes, or a reminder that fires the next morning. A [](api:calendar::ManualClock) starts at an instant and only moves when you tell it to:

```echo
$zurich = Zone(named: 'Europe/Zurich');
$start = DateTime(2026, .march, 27, 9, 30)->at($zurich);

$clock = ManualClock($start->instant);
$cal = Calendar($zone: $zurich, $clock: $clock);

$clock->advance(time::duration::hours(48));
echo "{$cal->now()}";            // 2026-03-29T10:30:00+02:00[Europe/Zurich]

$christmasEve = DateTime(2026, .december, 24, 18, 0)->at($zurich);
$clock->set($christmasEve->instant);
echo "{$cal->now()}";            // 2026-12-24T18:00:00+01:00[Europe/Zurich]
```

Notice that advancing 48 hours from Friday 09:30 lands on Sunday at 10:30. The clocks went forward in between, and the calendar reads the clock in its zone, just as it would in production.

The calendar holds on to the same clock object, so `advance` and `set` take effect immediately, with no need to build a new calendar.

Threads may share one `ManualClock`: a server under test can run its workers on it while the test moves it. It keeps the instant as a single count of nanoseconds since 1970, so a read never sees half a move, and that count runs out in the year 2262.

## Your own clock

`Clock` is an interface, so you may implement it yourself, for example to read the time from a trusted server or to replay a log at high speed. It needs one method:

```echo
public class OffsetClock : Clock
{
    private time::duration $skew;

    constructor(time::duration $skew)
    {
        $this->skew = $skew;
    }

    public const function now() : time::timestamp
    {
        return time::timestamp::now() + $this->skew;
    }
}

$cal = Calendar($zone: Zone::utc(), $clock: OffsetClock(time::duration::mins(-5)));
```

## Code that doesn't take a calendar

`Zoned::now($zone)` reads the system clock directly. It is handy in scripts and small tools. In code you'd like to test, prefer `$cal->now()`, and pass the calendar in. [The Calendar](./calendar.md#passing-the-calendar-around) shows that pattern.
