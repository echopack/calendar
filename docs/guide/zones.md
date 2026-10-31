---
description: Named and fixed zones, the machine's local zone, where the tz data comes from, custom zone sources, and inspecting transitions.
---

# Zones and tz data

A [](api:calendar::Zone) is a set of rules: for any instant, it knows the offset from UTC, the abbreviation people use, and whether summer time is in effect. You rarely look inside one. You hand it to `at`, `to` or a [Calendar](./calendar.md), and it does its job. This page covers how to get the zone you need, where the rules come from, and what to do when you'd like to look at them yourself.

## Getting a zone

### Named zones

Most zones have an IANA name, like `Europe/Zurich` or `America/New_York`. When you write the name in your code, use the labelled constructor:

```echo
$zurich = Zone(named: 'Europe/Zurich');

echo $zurich->id;                // Europe/Zurich
```

A name the system doesn't know stops the program, since a typo in your source code is a bug. When the name comes from a user, a config file, or a database, use `Zone::named` instead. It returns a `result` with a [](api:calendar::ZoneError):

```echo
$typo = Zone::named('Europe/Zurch')->failure();
$sneaky = Zone::named('../etc/passwd')->failure();

echo "{$typo}";                  // no time zone named 'Europe/Zurch'
echo "{$sneaky}";                // '../etc/passwd' is not a time zone name
```

Names are checked before they ever reach the filesystem, so user input can't be used to read arbitrary files. Only relative names made of letters, digits and `/ _ - + .` are accepted.

### UTC and fixed offsets

`Zone::utc()` is UTC. When all you have is an offset, say from an API that sends `+05:30`, you may build a zone that keeps that offset forever:

```echo
$india = Zone(offset: Offset(hours: 5, $minutes: 30));
$brasilia = Zone(offset: Offset(hours: -3));

echo $india->id;                 // +05:30
echo $brasilia->id;              // -03:00
echo "{$india->fixed()}";        // true
```

An offset of zero is the one exception: `Zone(offset: Offset::utc())` is UTC itself, with the id `UTC`, so the same instant in it and in `Zone::utc()` is `==`.

A fixed zone has no DST. It is the right choice for data that only ever carried an offset, and the wrong one for a place, because places change their rules. Whenever you know the place, prefer its name.

An [](api:calendar::Offset) itself may be written as `Offset(3600)` in seconds, `Offset(hours: 1)`, or parsed with `Offset::parse`, which accepts `Z`, `+01:00`, `+0100` and `+01`.

### The machine's zone

`Zone::local()` is the zone this machine runs in. It checks, in order:

1. The `TZ` environment variable, when it is set. It may hold a zone name (`Europe/Zurich`), a path to a zone file, or a POSIX rule string.
2. The zone that `/etc/localtime` points at.
3. UTC, when neither tells it anything.

A `TZ` that names nothing usable also gives UTC, without a word. That keeps `Zone::local()` from ever failing, but it also hides a typo like `TZ=Europe/Zurch`. When you'd like to catch that, ask `Zone::env()`, which reads `TZ` the same way and returns an error instead. It is null when `TZ` isn't set at all:

```echo
Zone? $fromTz = guard Zone::env() else ($e) {
    die("TZ is set, but {$e}");
};

Zone $here = $fromTz ?? Zone::local();
```

`Calendar()` uses `Zone::local()` as its default zone. On a server, you may prefer to name the zone explicitly, so that your program behaves the same wherever it runs.

## Where the rules come from

Named zones are read from the tz database that macOS and Linux already ship, in `/usr/share/zoneinfo`. Set `TZDIR` to read from another directory. The library parses the compiled zone files directly (TZif versions 1 to 4) and keeps each zone in memory after its first use.

That means the rules are exactly as fresh as your system's tz package. When a country changes its DST rules, an OS update is all it takes. Every past transition is there, and for the far future each zone carries a rule that the library evaluates on the fly.

`Zone::system()` is the default source behind `Zone(named:)` and `Zone::named`. It is a [](api:calendar::SystemZones), and you may ask it what it knows:

```echo
$system = Zone::system();

$plausible = SystemZones::valid('Europe/Zurich');

echo $system->dir;               // /usr/share/zoneinfo
echo "{$plausible}";             // true
```

`ids()` lists every zone name, read from the `tzdata.zi` index. It returns an empty list on systems that don't ship that file.

::: tip Containers
Slim container images often leave out the tz database. If `Zone(named: ...)` can't find zones that exist, install your distribution's `tzdata` package, or copy a `zoneinfo` directory into the image and point `TZDIR` at it.
:::

## Using another source

A [](api:calendar::Calendar) loads zones from whichever [](api:calendar::ZoneSource) you give it. The simplest case is a [](api:calendar::SystemZones) for another directory:

```echo
$bundled = SystemZones('/usr/share/zoneinfo');
$cal = Calendar($zone: Zone::utc(), $zones: $bundled);

Zone $tokyo = guard $cal->zone('Asia/Tokyo');

echo $tokyo->id;                 // Asia/Tokyo
```

`$cal->zone(...)` and `$cal->parse(...)` read zone names from that source. `Zoned::parse` takes a source as an optional second argument too.

### Writing your own source

`ZoneSource` is a small interface with two methods, `load` and `ids`. You may implement it to add your own logic in front of the system data. For example, an app that only serves a handful of offices could refuse every other zone:

```echo
public class OfficeZones : ZoneSource
{
    private SystemZones $system;
    private array<string> $offices;

    constructor(array<string> $offices)
    {
        $this->system = SystemZones();
        $this->offices = $offices;
    }

    public function load(const string& $id) : result<Zone, ZoneError>
    {
        foreach ($this->offices as const &$office) {
            if ($office == $id) {
                return $this->system->load($id);
            }
        }

        return .error(.unknown($id));
    }

    public function ids() : array<string>
    {
        return $this->offices;
    }
}

$cal = Calendar($zones: OfficeZones(['Europe/Zurich', 'America/New_York']));
$refused = $cal->zone('Asia/Tokyo')->failure();

echo "{$refused}";               // no time zone named 'Asia/Tokyo'
```

Keep in mind that `Zone(named: ...)` and `Zone::named` always read from the system source, even when a calendar has its own. Code that should respect your source goes through `$cal->zone(...)`.

## Inspecting a zone

Sometimes you need to look at the rules directly, for example to show "CET" next to a time, or to warn users that the clocks change this weekend. A zone answers questions about any instant:

```echo
$zurich = Zone(named: 'Europe/Zurich');
$now = DateTime(2026, .march, 27, 9, 30)->at($zurich);

echo "{$zurich->offset($now->instant)}";        // +01:00
echo $zurich->abbreviation($now->instant);      // CET
echo "{$zurich->dst($now->instant)}";           // false
```

### The next and previous change

`next` and `previous` find the closest transition after or before an instant. They return `null` when there is none, for example in a fixed zone:

```echo
$zurich = Zone(named: 'Europe/Zurich');
$now = DateTime(2026, .march, 27, 9, 30)->at($zurich);

time::timestamp $change = guard $zurich->next($now->instant);
$switch = Zoned($change, $zurich);

echo "{$switch}";                // 2026-03-29T03:00:00+02:00[Europe/Zurich]
echo $switch->format('z');       // CEST
```

That is all you need for a "the clocks go forward this Sunday" banner.

### Every offset for a wall clock

`candidates` returns every offset a wall clock could have in the zone: none in a gap, two in an overlap, and one the rest of the time. [Daylight saving time](./dst.md) puts it to work.
