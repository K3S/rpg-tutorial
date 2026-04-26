---
title: "8. Error Handling with MONITOR"
parent: "Part 3: Build Real Things"
nav_order: 8
---

# Tutorial 8: Error Handling with MONITOR

Every program you've written so far assumes the inputs are good. Reality is messier. Users type `"abc"` into numeric fields. Calculations overflow. Divisions by zero happen. In production, your program doesn't get to choose its inputs — and the default behavior when something goes wrong is a program crash.

[Part 2 Chapter 7]({% link part-2/07-error-handling.md %}) introduced `monitor` as RPG's try/catch. This tutorial puts it to work on a real batch scenario — processing a list of pricing updates, some of which are bad — and shows the difference between a program that falls over on bad data versus one that handles it and keeps running.

## What you'll build

A program called **PRICEUPD** (Price Update) that takes a single product and a new price as parameters, validates the inputs, and updates the PRODUCT table. The wrinkle: the price is passed as a *character string* (because that's how real callers deliver it — from CSV files, from CL parameters, from screens) and must be converted to a packed decimal. That conversion is where things go wrong.

You'll write it so that bad input fails loudly with a clear message — never crashes, never leaves the database in a weird state.

## What you'll learn

- Using `monitor` / `on-error` / `endmon` to trap runtime errors
- Catching specific error codes (`*zero-divide`, `*out-of-range`) vs. catch-all
- Using `%status` and `%error` to log what went wrong
- The crucial distinction: `monitor` catches *RPG runtime errors*, but `sqlstate` catches *SQL errors* — use both
- The "validate before you act" pattern

## Before you start

- Part 1 complete
- [Part 2 Chapter 6: Error Handling with MONITOR]({% link part-2/06-error-handling.md %}) read
- Tutorials 1–7 completed

## The program

Create a new member in `QRPGLESRC` called **PRICEUPD**, type **SQLRPGLE**. Paste:

```rpgle
**free
ctl-opt dftactgrp(*no) actgrp(*caller)
        option(*nodebugio:*srcstmt)
        main(Main);
// ************************************************************
// * Name: PRICEUPD
// * Type: ILE RPG Program (SQLRPGLE)
// * Desc: Price Update with error handling
// *       Parses a price from a character string, validates,
// *       and updates PRODUCT. Demonstrates MONITOR for
// *       RPG errors and SQLSTATE for SQL errors.
// * Tutorial: Part 3, Tutorial 8
// ************************************************************

dcl-pr ParsePrice packed(9:2);
  priceStr char(20) const;
  ok       ind;
end-pr;

dcl-proc Main;
  dcl-pi *n;
    wk_prodCode char(15) const;
    wk_priceStr char(20) const;
  end-pi;

  dcl-s newPrice packed(9:2);
  dcl-s parseOk  ind;

  exec sql
    set option commit = *none,
               datfmt = *iso,
               closqlcsr = *endactgrp;

  // Step 1 — validate the inputs before touching the database
  if %trim(wk_prodCode) = '';
    dsply 'Error: product code is blank';
    return;
  endif;

  newPrice = ParsePrice(wk_priceStr : parseOk);
  if not parseOk;
    dsply ('Error: "' + %trim(wk_priceStr) + '" is not a valid price');
    return;
  endif;

  if newPrice <= 0;
    dsply ('Error: price must be greater than zero (got ' + %char(newPrice) + ')');
    return;
  endif;

  // Step 2 — attempt the update
  exec sql
    update PRODUCT
       set PR_PRICE = :newPrice,
           PR_CHGDT = current_date
     where PR_PROD = :wk_prodCode
       and PR_ACTIVE = 'Y';

  // Step 3 — handle the SQL outcome
  select;
    when sqlstate = '00000' and sqlerrd(3) = 1;
      dsply ('Updated ' + %trim(wk_prodCode) + ' to ' + %char(newPrice));

    when sqlstate = '00000' and sqlerrd(3) = 0;
      dsply ('No active product found matching ' + %trim(wk_prodCode));

    other;
      dsply ('SQL error: ' + sqlstate);
  endsl;

  return;
end-proc;

// ---- Helper: convert char to packed decimal with error trap -

dcl-proc ParsePrice;
  dcl-pi *n packed(9:2);
    priceStr char(20) const;
    ok       ind;
  end-pi;

  dcl-s result packed(9:2) inz(0);

  ok = *on;

  monitor;

    // %dec on a character value raises a runtime error if the
    // string can't be interpreted as a number in the specified
    // precision. That's the whole point of wrapping this in
    // MONITOR — we turn "program crash" into "parseOk = *off".
    result = %dec(%trim(priceStr) : 9 : 2);

  on-error *decimal-data : *string-error;
    // Specifically a parse failure
    ok = *off;

  on-error;
    // Any other unexpected error — still fail gracefully, but
    // log the status so we can diagnose later.
    dsply ('ParsePrice unexpected error: ' + %char(%status));
    ok = *off;

  endmon;

  return result;
end-proc;
```

## Walkthrough

### The three-step structure

Look at the comments in `Main`:

1. **Validate the inputs** before touching the database
2. **Attempt the update**
3. **Handle the SQL outcome**

This shape is worth internalizing. Every external-interface program benefits from it. Validate early returns you from a lot of bad-state bugs — if the input is broken, you never run the SQL, so you can't corrupt anything.

### MONITOR around a specific risky operation

```rpgle
monitor;
  result = %dec(%trim(priceStr) : 9 : 2);
on-error *decimal-data : *string-error;
  ok = *off;
on-error;
  dsply ('ParsePrice unexpected error: ' + %char(%status));
  ok = *off;
endmon;
```

Notice the monitor block wraps just the one line that might fail — the `%dec` conversion. Everything around it is safe. That's the pattern: **tight monitor blocks around specific operations you know can raise errors**, not "wrap the whole program and hope."

Wide monitor blocks are a smell. They catch too much, hide real bugs, and make it hard to know where an error came from. Narrow ones make the failure point obvious.

### Catching specific error codes

```rpgle
on-error *decimal-data : *string-error;
  ok = *off;

on-error;
  dsply ('ParsePrice unexpected error: ' + %char(%status));
  ok = *off;
```

Two `on-error` clauses:

**The first** handles the expected failures — `*decimal-data` (bad numeric data) and `*string-error` (string operation failed). These are the errors we *expect* when `%dec` can't parse a bad string. When they happen, just set `ok = *off` — this is normal behavior for bad user input, nothing to log.

**The second** is the catch-all. Something unexpected happened — maybe memory, maybe an obscure type issue, something we didn't plan for. We still want to fail gracefully rather than crash, so we flip `ok = *off`. But we also *log* the status, because "an unexpected error" is something a developer might want to investigate.

This is an important pattern. **Separate the known-bad from the unknown-bad.** Treating them the same means you either alert on routine failures (noise) or silently swallow real problems (invisibility). Splitting them gives you signal.

### The output parameter for success status

```rpgle
dcl-pr ParsePrice packed(9:2);
  priceStr char(20) const;
  ok       ind;
end-pr;
```

`ParsePrice` returns the parsed value, *and* sets an output parameter `ok` indicating whether the parse succeeded. Why both?

Because any `packed(9:2)` value — including zero — might be a *legitimate* parse result. If the function only returned the number, the caller can't tell "parse succeeded and result was 0" from "parse failed so I returned 0 as default." Using a separate `ok` flag resolves the ambiguity.

This pattern — return the value, set a status — is all over good RPG code. Once you start noticing it you'll see it constantly.

### sqlerrd(3) — the rows-affected counter

```rpgle
when sqlstate = '00000' and sqlerrd(3) = 1;
  dsply ('Updated ' + %trim(wk_prodCode) + ' to ' + %char(newPrice));

when sqlstate = '00000' and sqlerrd(3) = 0;
  dsply ('No active product found matching ' + %trim(wk_prodCode));
```

`sqlerrd(3)` is an auto-populated SQL status variable that tells you how many rows the last statement affected. For an UPDATE, this is how you distinguish "the update ran successfully and changed one row" from "the update ran successfully but matched zero rows."

Both are `sqlstate = '00000'` — SQL considers "I executed your UPDATE against zero matching rows" a success. But functionally, they're very different outcomes. Always check `sqlerrd(3)` on writes when "zero rows affected" is a meaningful case.

### Why no MONITOR around the SQL

You might wonder why `monitor` isn't wrapping the UPDATE. Answer: **SQL errors don't raise RPG runtime errors.** Failed SQL sets `sqlstate` to something other than `'00000'` and keeps executing. `monitor` wouldn't catch it even if it did wrap the block.

This is the single most important rule in this tutorial. Remember it:

- **`monitor` / `on-error`** — for RPG runtime errors (bad `%dec`, division by zero, array out of range, pointer errors)
- **`sqlstate` checks** — for SQL errors (constraint violation, lock timeout, table doesn't exist, no rows found)

Different machinery, different error types, check both when your program does both.

## Compile and run

Compile with **CRTSQLRPGI**.

Happy path:

```
CALL PRICEUPD PARM('WIDGET-STD-001' '14.99')
```

Should show:

```
DSPLY  Updated WIDGET-STD-001 to 14.99
```

Verify in SQL:

```sql
SELECT PR_PROD, PR_PRICE, PR_CHGDT
  FROM PRODUCT
 WHERE PR_PROD = 'WIDGET-STD-001';
```

You should see the new price and today's date.

Bad price format:

```
CALL PRICEUPD PARM('WIDGET-STD-001' 'fourteen.99')
```

Should show:

```
DSPLY  Error: "fourteen.99" is not a valid price
```

No update happened — verify with the same SQL as above. The price should still be whatever the prior run set it to.

Empty product code:

```
CALL PRICEUPD PARM('' '14.99')
```

Should show:

```
DSPLY  Error: product code is blank
```

Negative price:

```
CALL PRICEUPD PARM('WIDGET-STD-001' '-5.00')
```

Should show:

```
DSPLY  Error: price must be greater than zero (got -5.00)
```

Product that doesn't exist:

```
CALL PRICEUPD PARM('NOPE-DOES-NOT-EXIST' '14.99')
```

Should show:

```
DSPLY  No active product found matching NOPE-DOES-NOT-EXIST
```

Product that exists but is inactive:

```
CALL PRICEUPD PARM('SPROCKET-15T' '14.99')
```

Should also show `No active product found...` — SPROCKET-15T is in the table but marked inactive, and the UPDATE's `WHERE PR_ACTIVE = 'Y'` filters it out. Zero rows affected.

Run through all six cases. Every one produces a clean outcome — no program crash, no cryptic messages. That's the whole point.

## When to use MONITOR — and when not to

**Use it when external data might be malformed** — parsing user input, reading from CSV or XML, converting types where the data isn't fully trusted.

**Use it when arithmetic could overflow or divide by zero** — price calculations, rate conversions, percentage math with user-supplied denominators.

**Use it around file I/O that might fail in recoverable ways** — though remember, RPG native I/O often uses `%found` / `%eof` / `%error` indicators rather than raising errors, so `monitor` may not be the right tool for every I/O case.

**Don't bother** in pure internal logic where failure means a programmer bug. If a variable should never be zero because you just checked it three lines ago, don't wrap the division in `monitor` — that masks bugs you want to fix, not tolerate.

**Don't use wide `monitor` blocks as a replacement for actual handling.** "Catch everything, set a flag, return" is worse than no handler at all if the flag never gets checked. Monitoring without follow-through silently swallows information.

## Try this

**Add an out-of-range catch.** Very large prices like `99999999.99` exceed `packed(9:2)`'s range. Currently this would hit the `*decimal-data : *string-error` branch or the catch-all. Split it out with `on-error *out-of-range;` and return a specific "price out of range" error message.

**Batch mode.** Change the parameters to accept a product code list (e.g., `char(200)` with comma-separated codes) and apply the same price to all of them. Use a loop. What happens when half the codes exist and half don't? Make sure one bad code doesn't skip the rest.

**Loosen the parser.** Accept prices with dollar signs (`"$14.99"`) and thousands separators (`"1,499.00"`). Strip them before the `%dec` call. Decide: is that a good UX improvement, or is it hiding data quality problems you should raise instead?

**Log failures to a table.** Create a PRICEERR table with columns `PE_PROD`, `PE_ATTEMPT`, `PE_REASON`, `PE_LOGTIME`. When validation fails, insert a row instead of (or in addition to) displaying. Now you have an audit log of bad data that a human can review later.

## What's next

You can write a program that handles real-world messiness. Tutorial 9 shifts focus to date math — the arithmetic you'll use for audits, schedules, age calculations, and anything else with a time dimension.

Next: [Tutorial 9: Date Math in Practice]({% link part-3/09-date-math-practice.md %}).
