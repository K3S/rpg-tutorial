---
title: "9. Date Math in Practice"
parent: "Part 3: Build Real Things"
nav_order: 9
---

# Tutorial 9: Date Math in Practice

[Part 2 Chapter 7]({% link part-2/07-date-math.md %}) covered the date BIFs — `%date`, `%days`, `%diff`, and friends. This tutorial uses them on a realistic business question: **which products haven't had a price change in too long?**

Stale pricing is a real problem in distribution. A product whose price hasn't been reviewed in 18 months is probably mis-priced relative to current costs. You want a report that surfaces these for the pricing team to look at.

You already have everything needed. The PRODUCT table has `PR_CHGDT`, the last-change date. The sample data has a spread of change dates, so the report will find genuine candidates.

## What you'll build

A program called **STALEPR** (Stale Price audit) that takes an "age threshold" parameter — a number of days — and reports every active product whose `PR_CHGDT` is older than that threshold. For each stale product, it shows:

- Product code and supplier
- Current price and quantity on hand
- Last change date
- How many days old the price is
- An audit category: STALE (over threshold), VERY STALE (over 2× threshold), or CRITICAL (over 3× threshold)

At the end it displays a summary with counts per category.

## What you'll learn

- `%date()` for today's date
- `%diff()` with `*days` for the age of a date
- Date arithmetic with `%days(N)` as a duration
- Using SQL's `CURRENT_DATE` and `DATE - N DAYS` at the database level
- When to do date math in SQL vs RPG — the tradeoffs
- Categorizing by multiplier thresholds — a pattern you'll reuse

## Before you start

- Part 1 complete — your PRODUCT data is populated
- [Part 2 Chapter 7: Date Math]({% link part-2/07-date-math.md %}) read
- Tutorials 1–8 completed

## The program

Create a new member in `QRPGLESRC` called **STALEPR**, type **SQLRPGLE**. Paste:

```rpgle
**free
ctl-opt dftactgrp(*no) actgrp(*caller)
        option(*nodebugio:*srcstmt)
        main(Main);
// ************************************************************
// * Name: STALEPR
// * Type: ILE RPG Program (SQLRPGLE)
// * Desc: Stale Price Audit
// *       Reports every active product whose last price
// *       change is older than a threshold, categorizing
// *       by how stale.
// * Tutorial: Part 3, Tutorial 9
// ************************************************************

dcl-pr AgeCategory varchar(10);
  ageDays    int(10)     const;
  thresholdDays int(10)  const;
end-pr;

dcl-proc Main;
  dcl-pi *n;
    wk_thresholdDays int(10) const;
  end-pi;

  dcl-s prod       char(15);
  dcl-s sup        char(10);
  dcl-s price      packed(9:2);
  dcl-s qoh        packed(9:0);
  dcl-s chgDate    date;
  dcl-s ageDays    int(10);
  dcl-s category   varchar(10);

  dcl-s today      date;

  dcl-s staleCnt    packed(5:0) inz(0);
  dcl-s veryStaleCnt packed(5:0) inz(0);
  dcl-s criticalCnt  packed(5:0) inz(0);

  exec sql
    set option commit = *none,
               datfmt = *iso,
               closqlcsr = *endactgrp;

  // Guard against nonsense input
  if wk_thresholdDays < 1;
    dsply ('Threshold must be a positive number of days (got '
         + %char(wk_thresholdDays) + ')');
    return;
  endif;

  today = %date();

  dsply ('Stale Price Audit — threshold ' + %char(wk_thresholdDays) + ' days');
  dsply ('Running on: ' + %char(today));
  dsply '---------------------------------------------------------------';

  // The WHERE clause does the primary filter at the database level.
  // We still compute the age precisely in RPG for reporting.
  exec sql
    declare staleCursor cursor for
      select PR_PROD, PR_SUPL, PR_PRICE, PR_QOH, PR_CHGDT
        from PRODUCT
       where PR_ACTIVE = 'Y'
         and PR_CHGDT < current_date - :wk_thresholdDays days
       order by PR_CHGDT, PR_PROD;

  exec sql open staleCursor;

  exec sql fetch next from staleCursor
    into :prod, :sup, :price, :qoh, :chgDate;

  dow sqlstate = '00000';

    // Compute the exact age in days at report time
    ageDays = %diff(today : chgDate : *days);

    // Categorize
    category = AgeCategory(ageDays : wk_thresholdDays);

    // Update the appropriate counter
    select;
      when category = 'CRITICAL';
        criticalCnt += 1;
      when category = 'VERY STALE';
        veryStaleCnt += 1;
      other;
        staleCnt += 1;
    endsl;

    dsply (%char(category : *none)
         + '  ' + %trim(prod)
         + '  Sup:' + %trim(sup)
         + '  $' + %char(price)
         + '  QOH:' + %char(qoh)
         + '  Changed:' + %char(chgDate)
         + '  Age:' + %char(ageDays) + 'd');

    exec sql fetch next from staleCursor
      into :prod, :sup, :price, :qoh, :chgDate;

  enddo;

  exec sql close staleCursor;

  dsply '---------------------------------------------------------------';
  dsply ('Stale:      ' + %char(staleCnt));
  dsply ('Very stale: ' + %char(veryStaleCnt));
  dsply ('Critical:   ' + %char(criticalCnt));

  return;
end-proc;

// ---- Helper: classify an age --------------------------------

dcl-proc AgeCategory;
  dcl-pi *n varchar(10);
    ageDays       int(10) const;
    thresholdDays int(10) const;
  end-pi;

  select;
    when ageDays >= thresholdDays * 3;
      return 'CRITICAL';
    when ageDays >= thresholdDays * 2;
      return 'VERY STALE';
    other;
      return 'STALE';
  endsl;
end-proc;
```

## Walkthrough

### Dates in SQL vs dates in RPG

Look carefully at this line:

```rpgle
exec sql ... where PR_CHGDT < current_date - :wk_thresholdDays days;
```

The database does the primary filter. Only rows whose change date is older than the threshold come back. That's efficient — the database can use its query optimizer and (on a bigger dataset) an index to skip irrelevant rows quickly.

Then inside the loop:

```rpgle
ageDays = %diff(today : chgDate : *days);
```

We compute the *exact* age in RPG. The database already filtered — this is for display.

**Why not just compute the age in SQL?** You could. Adding `DAYS(CURRENT_DATE) - DAYS(PR_CHGDT) AS AGE_DAYS` to the SELECT works. So why not?

Two honest reasons:

1. **The filter condition and the display value are the same logical concept** — age in days. Expressing it once in SQL (for the filter) and once in RPG (for display) is redundant but clearly laid out. Expressing it only in SQL means the RPG has to read two columns for what's essentially one concept.
2. **RPG's date BIFs are a more natural fit for the kind of business logic you'll layer on top** — "what day of the week was this?" "is this more than two quarters old?" "what was the same day last year?" These questions are easier in RPG.

When the filter is simple and the logic is complex, push filter to SQL and do logic in RPG. That's the pattern.

### `current_date` in SQL vs `%date()` in RPG

The SQL uses `current_date`. The RPG uses `%date()`. They return the same value: today.

One subtle thing to notice: when a program runs for a long time (batch jobs, especially around midnight), the value "today" can shift mid-run. `current_date` in an SQL statement is evaluated at statement time. `%date()` in RPG is evaluated when the BIF is called.

For a quick audit report, it doesn't matter. For a job that runs for six hours starting at 10 PM, it can. In that case, store `today = %date();` at the top of the program and use `:today` as a host variable in SQL instead of `current_date`. Then your SQL and RPG agree on what "today" means, no matter how long the run takes.

### The audit categorization pattern

```rpgle
dcl-proc AgeCategory;
  ...
  select;
    when ageDays >= thresholdDays * 3;
      return 'CRITICAL';
    when ageDays >= thresholdDays * 2;
      return 'VERY STALE';
    other;
      return 'STALE';
  endsl;
end-proc;
```

Tiered categorization by multipliers of a threshold is extremely common in business RPG. Credit ratings, overdue invoices, delayed shipments, inventory aging — same pattern. The caller passes a base threshold and the procedure returns a label.

Note the order: check *strictest* (highest age) first. Same pattern as Tutorial 1's discount tiers — in `select`/`when`, the first match wins, so ordering matters.

### The varchar return type

```rpgle
dcl-pi *n varchar(10);
```

`AgeCategory` returns `varchar(10)` because the labels have different lengths — `'STALE'` is 5 chars, `'CRITICAL'` is 8, `'VERY STALE'` is 10. A `varchar` gives you natural-length strings without padding, and `%char` in the display handles it cleanly.

Using `char(10)` would work, but then `'STALE'` comes back as `'STALE     '` (padded to 10), and you'd have to `%trim` it everywhere. Less convenient.

### `%char(category : *none)`

```rpgle
dsply (%char(category : *none) + ...);
```

The `*none` format specifier tells `%char` "no special formatting — just the characters as they are." For a varchar, this is identical to using `category` directly. For numeric types, `*none` strips the formatting that `%char` would otherwise apply.

It's included here more as a hint than a necessity. You'll see `*none` in older code and when people are being explicit about formatting intent.

## Compile and run

Compile with **CRTSQLRPGI**.

Start with a reasonable threshold — 60 days. (Sample data was created in February–March 2026, so reality will depend on when you run this.)

```
CALL STALEPR PARM(60)
```

Output will depend on today's date relative to the `PR_CHGDT` values in your sample data. You should see a list of products whose prices haven't changed in over 60 days, sorted by oldest-change first, with categorization:

```
DSPLY  Stale Price Audit — threshold 60 days
DSPLY  Running on: 2026-04-24
DSPLY  ---------------------------------------------------------------
DSPLY  CRITICAL    CLAMP-LEGACY       Sup:GAMA007  $2.00   QOH:12    Changed:2022-08-14  Age:1349d
DSPLY  CRITICAL    SPROCKET-15T       Sup:DLTA004  $4.50   QOH:0     Changed:2023-12-15  Age:861d
DSPLY  (more rows depending on sample data age...)
DSPLY  ---------------------------------------------------------------
DSPLY  Stale:      8
DSPLY  Very stale: 0
DSPLY  Critical:   2
```

Try different thresholds:

```
CALL STALEPR PARM(30)
```

Catches more products — lower threshold, wider net.

```
CALL STALEPR PARM(365)
```

Only the ancient ones survive.

```
CALL STALEPR PARM(0)
```

Caught by the input guard — displays an error and exits cleanly.

## Try this

**Add a summary by supplier.** Right now the program lists individual products. Modify it to output a supplier-rollup at the end: for each supplier, show the count of stale products they have and the oldest change date. Tests your ability to maintain multiple aggregates as you loop.

**Expose the age threshold tiers as parameters.** Currently the tiers are hardcoded at `thresholdDays × 2` and `× 3`. Make them optional parameters (`options(*nopass)`) so a caller can say "critical is 5× threshold." Defaults stay as they are. Uses material from [Part 2 Chapter 3]({% link part-2/03-procedures.md %}) on optional parameters.

**Send to a table instead of display.** Create a STPRHIST table with the audit result and change STALEPR to INSERT into it rather than DSPLY. Now you have historical records of every audit run. The REORDRPT pattern from Tutorial 7 applies — delete previous results or keep them, your call.

**Month-end and year-end filtering.** Add an optional parameter `wk_asOfDate` (a date). If passed, use it instead of today. This lets you run "what was stale as of the end of last quarter?" Very useful for retrospective audits. The trick: don't use `%date()` directly — compute everything against the asOfDate value.

**Day-of-week weighting.** Products whose last change was on a weekend might be suspicious (did a batch run at 2 AM Sunday?). Use `%subdt(chgDate : *d)` or equivalent to extract day-of-week and flag anything that changed on Saturday or Sunday.

## What's next

You've done date arithmetic in a production-shape program. Tutorial 10 wraps up the "operational" wave by covering CL — the control language that glues RPG programs together, handles parameters, and runs in job schedulers.

Next: [Tutorial 10: CL — The Glue Language]({% link part-3/10-cl-glue.md %}).
