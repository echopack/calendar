---
description: The built-in English, German, French and Italian locales, picking one from a language tag, week rules, plurals, and building a locale of your own.
---

# Locales

A team in Zurich writes in German, French and Italian, and their clients often in English. The month names, the weekday names, "3 days ago" and "yesterday" are all different in each language. A [](api:calendar::Locale) holds those words for one language, and you hand it to whatever prints text.

## The built-in locales

Four locales come with the library:

```echo
$launch = Date(2026, .march, 27);
$pattern = 'EEEE d MMMM yyyy';

echo $launch->format($pattern, Locale::en());   // Friday 27 March 2026
echo $launch->format($pattern, Locale::de());   // Freitag 27 März 2026
echo $launch->format($pattern, Locale::fr());   // vendredi 27 mars 2026
echo $launch->format($pattern, Locale::it());   // venerdì 27 marzo 2026
```

The German, French and Italian locales are written to be correct in Germany, France and Italy, and in Switzerland too. The patterns themselves are yours, so `d. MMMM` for German or `d MMMM` for French is something you choose per language.

### Picking a locale from a tag

When the language comes from a user setting or an `Accept-Language` header, `Locale::of` takes a tag like `de`, `de-CH`, `fr_CH` or `en-GB`. It returns `null` for a language that isn't built in, so you may fall back with `??`:

```echo
$swissGerman = Locale::of('de-CH') ?? Locale::en();
$spanish = Locale::of('es') ?? Locale::en();

echo $swissGerman->id;           // de-CH
echo $spanish->id;               // en
```

The region only matters for English, where it decides the week rule, described below. Everything else about a language is the same in every region. The id still keeps the region, written the standard way (`en_gb` becomes `en-GB`), so two locales that count weeks differently never share an id.

## Where a locale is used

A locale is only ever used where you pass it. There are three places to do that:

- A `format` call, for a one-off: `$date->format('EEEE', Locale::fr())`.
- A compiled [](api:calendar::Format), with `->locale(...)`. The format keeps it for every render.
- A [Calendar](./calendar.md), as `$locale:`. Its `format`, `parse` and `human` use it for pattern strings.

Without one, everything is English. That keeps a program's output from changing just because it runs on a machine with different settings.

## Asking a locale for words

Occasionally you may need a name on its own, say for a dropdown of months or a weekday column header. You may ask the locale directly:

```echo
$german = Locale::de();

echo $german->month(.march);                  // März
echo $german->weekday(.friday);               // Freitag
echo $german->weekday(.friday, true);         // Fr.
echo $german->count(3, .week);                // 3 Wochen
echo $german->relative(-3, .day);             // vor 3 Tagen
```

`count` and `relative` pick the singular or plural form for you. For full "3 days ago" output from two dates, see [Human-readable output](./human.md).

## Week rules

People in different places count weeks differently. In the US, weeks start on Sunday. In most of Europe, they start on Monday and follow ISO 8601. Each locale carries the rule its speakers usually use:

```echo
$us = Locale::en()->weekRule;
$uk = (Locale::of('en-GB') ?? Locale::en())->weekRule;

echo $us->firstDay->number();    // 7
echo $uk->firstDay->number();    // 1
```

Here is the important part: **a locale's week rule is never applied on its own.** Passing `Locale::en()` to a calendar does not make its weeks start on Sunday. You pass the rule explicitly where you want it:

```echo
$english = Locale::en();
$cal = Calendar($zone: Zone::utc(), $locale: $english, $weekRule: $english->weekRule);

$launch = Date(2026, .march, 27);

echo "{$launch->start(.week, $english->weekRule)}";   // 2026-03-22
echo "{$cal->week($launch)}";                         // 2026-W13
```

It may feel like an extra step, but it means that switching a page to English changes its words and nothing else. Week numbers on a report don't shift just because someone changed their language. [Dates](./dates.md#weeks) has more about week rules.

## Plurals

Languages disagree about which numbers take the singular. English says "0 days", French says "0 jour". A locale's [](api:calendar::Plural) rule decides:

```echo
echo Locale::en()->count(0, .day);   // 0 days
echo Locale::fr()->count(0, .day);   // 0 jour
echo Locale::fr()->count(2, .day);   // 2 jours
```

`.one` uses the singular only for 1, and `.zeroAndOne` uses it for 0 and 1, as French does.

## Building your own locale

A locale is plain data, so you may build one for any language with its field names. Arrays run January to December and Monday to Sunday. The unit words come in pairs, singular then plural, for year, month, week, day, hour, minute and second. The relative phrases come in fours per unit, with `#` where the number goes: future singular, future plural, past singular, past plural.

Here is a complete Spanish locale. It looks long, but it's just lists of words. Each list goes into a typed variable first. That keeps it readable, and it is also required for now: echoc 0.11.5 refuses a list written directly in the `Locale(...)` call. The built-in locales are written the same way:

```echo
array<string> $months = ['enero', 'febrero', 'marzo', 'abril', 'mayo', 'junio', 'julio', 'agosto', 'septiembre', 'octubre', 'noviembre', 'diciembre'];
array<string> $monthsShort = ['ene', 'feb', 'mar', 'abr', 'may', 'jun', 'jul', 'ago', 'sept', 'oct', 'nov', 'dic'];
array<string> $monthsNarrow = ['E', 'F', 'M', 'A', 'M', 'J', 'J', 'A', 'S', 'O', 'N', 'D'];
array<string> $weekdays = ['lunes', 'martes', 'miércoles', 'jueves', 'viernes', 'sábado', 'domingo'];
array<string> $weekdaysShort = ['lun', 'mar', 'mié', 'jue', 'vie', 'sáb', 'dom'];
array<string> $weekdaysNarrow = ['L', 'M', 'X', 'J', 'V', 'S', 'D'];
array<string> $quarters = ['1.er trimestre', '2.º trimestre', '3.er trimestre', '4.º trimestre'];
array<string> $quartersShort = ['T1', 'T2', 'T3', 'T4'];
array<string> $eras = ['a. C.', 'd. C.'];
array<string> $erasWide = ['antes de Cristo', 'después de Cristo'];
array<string> $dayPeriods = ['a. m.', 'p. m.'];
array<string> $units = ['año', 'años', 'mes', 'meses', 'semana', 'semanas', 'día', 'días', 'hora', 'horas', 'minuto', 'minutos', 'segundo', 'segundos'];
array<string> $relative = [
    'dentro de # año', 'dentro de # años', 'hace # año', 'hace # años',
    'dentro de # mes', 'dentro de # meses', 'hace # mes', 'hace # meses',
    'dentro de # semana', 'dentro de # semanas', 'hace # semana', 'hace # semanas',
    'dentro de # día', 'dentro de # días', 'hace # día', 'hace # días',
    'dentro de # hora', 'dentro de # horas', 'hace # hora', 'hace # horas',
    'dentro de # minuto', 'dentro de # minutos', 'hace # minuto', 'hace # minutos',
    'dentro de # segundo', 'dentro de # segundos', 'hace # segundo', 'hace # segundos',
];
array<string> $nearDays = ['ayer', 'hoy', 'mañana'];

$spanish = Locale(
    $id: 'es',
    $months: $months,
    $monthsShort: $monthsShort,
    $monthsStandaloneShort: $monthsShort,
    $monthsNarrow: $monthsNarrow,
    $weekdays: $weekdays,
    $weekdaysShort: $weekdaysShort,
    $weekdaysNarrow: $weekdaysNarrow,
    $quarters: $quarters,
    $quartersShort: $quartersShort,
    $eras: $eras,
    $erasWide: $erasWide,
    $dayPeriods: $dayPeriods,
    $weekRule: WeekRule::iso(),
    $plural: .one,
    $units: $units,
    $relative: $relative,
    $nearDays: $nearDays,
    $now: 'ahora',
    $and: 'y',
);

$launch = Date(2026, .march, 27);

echo $launch->format("EEEE, d 'de' MMMM 'de' yyyy", $spanish);   // viernes, 27 de marzo de 2026
echo $spanish->relative(3, .day);                                // dentro de 3 días
```

A list of the wrong length is caught as soon as the locale is built, so a missing month can't turn into a crash months later. Since a locale is `#[atomic]` and read-only, you may build it once and share it between threads.
