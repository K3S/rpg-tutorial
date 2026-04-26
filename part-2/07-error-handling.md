---
title: "7. Error Handling with MONITOR"
parent: "Part 2: Learn the Language"
nav_order: 7
---

# Error Handling with MONITOR

RPG programs crash. Division by zero, numeric overflow, reading past end-of-file without checking, converting "abc" to a decimal — these are runtime errors, and by default they terminate the program with an unrecoverable message. Users see the awful green-screen halt. Your batch job dies mid-run. You get a 3 AM phone call.

The solution is `monitor` — RPG's equivalent of `try`/`catch` in other languages.

## Basic structure

```rpgle
monitor;
  // code that might fail
on-error;
  // handle any error
endmon;
```

- **`monitor`** starts a protected block
- **`on-error`** introduces an error handler
- **`endmon`** closes the block

If the code inside `monitor` raises an error, execution jumps immediately to the `on-error` handler. If no error occurs, the `on-error` block is skipped.

## A simple example

```rpgle
dcl-proc SafeDivide export;
  dcl-pi *n packed(9:4);
    numerator   packed(9:2) const;
    denominator packed(9:2) const;
  end-pi;

  dcl-s result packed(9:4);

  monitor;
    result = numerator / denominator;
  on-error;
    result = 0;
  endmon;

  return result;
end-proc;
```

If `denominator` is 0, division by zero raises an error; the handler catches it and returns 0 instead of crashing.

## Handling specific errors

You can branch on the specific error code:

```rpgle
monitor;
  result = numerator / denominator;
on-error *zero-divide;
  result = 0;
on-error *out-of-range;
  result = 9999999999.9999;
on-error;
  // everything else
  result = 0;
endmon;
```

A handful of useful named statuses:

- `*zero-divide` — division by zero (status code 00102)
- `*out-of-range` — value out of declared range (00103)
- `*string-error` — string operation failed (00100)
- `*decimal-data` — bad decimal data (00907)

You can also specify numeric ranges directly:

```rpgle
on-error 00100 : 00200;  // any status from 00100 to 00200
```

Order matters. On-error clauses are checked top to bottom; the first match wins.

## Inspecting the error

Inside an `on-error` block, two BIFs give you diagnostic info:

```rpgle
monitor;
  result = numerator / denominator;
on-error;
  dsply ('Error ' + %char(%status) + ': ' + %trim(%error()));
  result = 0;
endmon;
```

- `%status` — the numeric status code
- `%error` — a short text description

Log these when you want to know what actually went wrong. Silently swallowing errors is worse than crashing — you lose the signal.

## SQL errors are different

`exec sql` statements don't raise RPG errors when they fail — they update `sqlstate` and `sqlcode` but keep executing. So `monitor` won't catch SQL errors. You check `sqlstate` explicitly:

```rpgle
exec sql
  update PRODUCT
     set PR_PRICE = :newPrice
   where PR_PROD = :prodCode;

if sqlstate <> '00000';
  exec sql rollback;
  dsply ('SQL update failed: ' + sqlstate);
  return *off;
endif;
```

However, `monitor` *will* catch errors that happen in RPG expressions referenced by an SQL statement — for example, an overflow when you compute a host variable. The rule of thumb: `monitor` for RPG runtime errors, `sqlstate` for SQL errors.

## A realistic pattern: transactional update

```rpgle
dcl-proc UpdatePriceWithRollback export;
  dcl-pi *n ind;
    prodCode char(15)    const;
    newPrice packed(9:2) const;
  end-pi;

  monitor;
    exec sql
      update PRODUCT
         set PR_PRICE = :newPrice,
             PR_CHGDT = current_date
       where PR_PROD = :prodCode;

    if sqlstate <> '00000';
      exec sql rollback;
      return *off;
    endif;

    exec sql commit;
    return *on;

  on-error;
    exec sql rollback;
    return *off;
  endmon;
end-proc;
```

Note the double defense: explicit `sqlstate` check for SQL problems, `monitor` for any RPG-level failures (like a type conversion error computing `newPrice`). Either path rolls back the transaction.

## When to use `monitor`

- Any arithmetic that could divide by zero or overflow (price calculations, quantity conversions)
- Any type conversion from external data (`%dec` on user input, character-to-numeric parsing)
- Any file I/O where a failure shouldn't crash the whole run
- Any procedure that's called from a production batch job

## When not to bother

- Inside a small, pure-math helper where overflow would be a programmer error, not a data error
- When you genuinely want to crash — for programmer assertions during development

"Swallow every possible error" is just as bad as "don't handle any." Aim for `monitor` where real-world data can plausibly be malformed, and let programmer bugs crash loudly during testing.

Next: [Chapter 8: Date Math]({% link part-2/08-date-math.md %}).
