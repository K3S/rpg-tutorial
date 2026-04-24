---
title: "7. Date Math"
parent: "Part 2: Learn the Language"
nav_order: 7
---

# Date Math

RPG has first-class support for dates, times, and timestamps. Unlike languages where dates are a library you have to import and wrestle with, RPG's compiler understands them natively — and, importantly, enforces type safety so you can't accidentally add a raw number to a date.

This matters because business logic is full of date questions. When is this order due? How many days until this product runs out? Did this shipment arrive late? Is this supplier contract still valid?

## The three temporal types

```rpgle
dcl-s orderDate      date;
dcl-s pickupTime     time;
dcl-s orderCreatedAt timestamp;
```

- **`date`** — just a date (year, month, day)
- **`time`** — just a time (hour, minute, second)
- **`timestamp`** — both, plus microseconds

All three are stored internally as numbers, but the compiler treats them as opaque temporal values. You can't read or assign them as integers.

## Getting the current date/time

```rpgle
today  = %date();          // today
now    = %time();          // current time
stamp  = %timestamp();     // right now, full precision
```

These are system calls — they return the current value each time you call them.

## Creating a specific date

```rpgle
dueDate       = %date('2026-06-15' : *iso);
anniversary   = %date('06/15/2026' : *usa);
```

The second argument is a format specifier. `*iso` is the standard `YYYY-MM-DD`. Other common ones: `*usa` (`MM/DD/YYYY`), `*eur` (`DD.MM.YYYY`), `*jis` (`YYYY-MM-DD` like ISO).

**Use `*iso` unless you have a strong reason.** Ambiguous date formats cause bugs.

## Converting between date and string

```rpgle
displayDate = %char(orderDate : *iso);     // '2026-06-15'
orderDate   = %date('2026-06-15' : *iso);  // back to date
```

`%char` with a format specifier produces the string. `%date` with a format specifier parses it.

## Duration BIFs

These create values you can add to or subtract from dates:

```rpgle
nextWeek    = today + %days(7);
nextMonth   = today + %months(1);
nextYear    = today + %years(1);
contractEnd = today + %days(365);
oneHourAgo  = now   - %hours(1);
```

Full list: `%days`, `%weeks`, `%months`, `%years`, `%hours`, `%minutes`, `%seconds`, `%ms`.

You can't skip them. This is illegal:

```rpgle
nextWeek = today + 7;  // compile error
```

The compiler enforces: to add 7 to a date, you must specify what 7 *is*. `%days(7)`? `%weeks(7)`? That's the type safety paying off.

## Calculating differences

`%diff` computes the difference between two dates, times, or timestamps:

```rpgle
daysOld     = %diff(%date()    : hiredDate    : *days);
hoursWorked = %diff(clockOut   : clockIn      : *hours);
ageInMonths = %diff(%date()    : birthDate    : *months);
```

The third argument is the unit. Positive result means the first date is later.

## Practical patterns

### Is this due?

```rpgle
if expectedDate < %date();
  dsply 'Overdue';
endif;
```

### Is this within tolerance?

```rpgle
if %diff(%date() : expectedDate : *days) > toleranceDays;
  dsply 'Past tolerance window';
endif;
```

### Time supply (inventory days remaining)

```rpgle
if avgDailyDemand > 0;
  daysOfStock = PR_QOH / avgDailyDemand;
else;
  daysOfStock = 9999;  // no demand — effectively infinite
endif;
```

### Next scheduled reorder

```rpgle
nextReorder = lastOrderDate + %days(reorderCycleDays);
```

### Age in years (decimal)

```rpgle
ageInDays  = %diff(%date() : birthDate : *days);
ageInYears = ageInDays / 365.25;
```

## Watch out for the 80/20 edge cases

- **Month math is weird.** `%date('2026-01-31') + %months(1)` returns `2026-02-28`, not `2026-02-31` (which doesn't exist). Compiler handles it sensibly, but be aware.
- **Daylight savings.** Time arithmetic across a DST boundary can produce unexpected results. If you're working with exact durations, consider timestamps in UTC.
- **Year 9999.** RPG dates max out at `9999-12-31`. You probably won't hit it, but effectively-infinite sentinel values should avoid dates.
- **Date zero.** The default value for an uninitialized date is `0001-01-01`. Watch for this when reading records where the date column might be NULL or unset — you'll sometimes see `0001-01-01` as an actual stored value.

## Try it

Using your practice tables, run this query to find products that haven't had a price change in over a year:

```sql
SELECT PR_PROD, PR_PRICE, PR_CHGDT
  FROM PRODUCT
 WHERE PR_CHGDT < CURRENT_DATE - 365 DAYS
   AND PR_ACTIVE = 'Y'
 ORDER BY PR_CHGDT;
```

In the sample data, most products have recent change dates (the past couple months). Only the inactive products and a handful with 2023–2024 dates will come back — a realistic pattern for a stale-pricing audit.

---

That's the language. Chapters 1–7 cover what you need to read and write modern RPG fluently. The next three chapters cover the tooling around it in VS Code: compiling in its various forms, the debugger, and Git.

Next: Chapter 8 — Compiling from VS Code (coming soon).
