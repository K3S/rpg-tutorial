---
title: "1. Product Lookup and Price Calculation"
parent: "Part 3: Build Real Things"
nav_order: 1
---

# Tutorial 1: Product Lookup and Price Calculation

The first real program. You'll write something that takes parameters from a caller, reads the database, applies business logic, and returns a result. That's the shape of most RPG work you'll ever do — this tutorial gets you fluent in the moves.

## What you'll build

A program called **PRCALC** (Product Price Calculator) that:

1. Accepts a product code and quantity as parameters
2. Looks the product up in the PRODUCT table
3. Checks that it exists and is active
4. Calculates the total price (unit price × quantity)
5. Applies a volume discount — 5% at 50+ units, 10% at 100+ units
6. Displays the result, including the discount if one was applied

If the product doesn't exist or is inactive, the program tells the user and exits cleanly. No crashes, no confusing errors.

## What you'll learn

- Declaring a **program interface** with `dcl-pi` so your program can accept parameters from a caller
- A single-row **embedded SQL lookup** into host variables
- Checking **SQLSTATE** to distinguish "found," "not found," and "error"
- Handling multiple success and failure cases cleanly with `select`/`when`
- Combining database state (the `PR_ACTIVE` flag) with business logic (the discount tiers)

Every concept here was introduced in Part 2. This is where you see them used together.

## Before you start

You should have:

- Completed [Part 1]({% link part-1/index.md %}) — your PRODUCT, SUPPLIER, and REORDCND tables exist on PUB400 with sample data
- Read at least [Part 2 Chapter 5: Embedded SQL]({% link part-2/05-embedded-sql.md %})
- Your library name handy — you'll need it when you compile

## The program

Create a new member in your `QRPGLESRC` source file called **PRCALC**, type **SQLRPGLE** (not `RPGLE` — this program uses embedded SQL, and the compiler needs to know). Paste this in:

```rpgle
**free
ctl-opt dftactgrp(*no) actgrp(*caller) option(*nodebugio:*srcstmt);
// ************************************************************
// * Name: PRCALC
// * Type: ILE RPG Program (SQLRPGLE)
// * Desc: Product Price Calculator
// *       Looks up a product by code, calculates total price
// *       for a given quantity, applies volume discounts.
// * Tutorial: Part 3, Tutorial 1
// ************************************************************

// Program interface — parameters passed in from the caller
dcl-pi *n;
  wk_productCode char(15) const;
  wk_quantity    packed(7:0) const;
end-pi;

// Variables to receive the row from the SQL lookup
dcl-s prod    char(15);
dcl-s price   packed(9:2);
dcl-s active  char(1);

// Variables for the price calculation
dcl-s totalPrice packed(11:2);
dcl-s discount   packed(5:4);
dcl-s moneyOff   packed(11:2);
dcl-s discountP  int(5);

// SQL options — no transaction management, ISO dates
exec sql
  set option commit = *none,
             datfmt = *iso,
             closqlcsr = *endactgrp;

// Look up the product
exec sql
  select PR_PROD, PR_PRICE, PR_ACTIVE
    into :prod, :price, :active
    from PRODUCT
   where PR_PROD = :wk_productCode;

// Handle the result
select;

  when sqlstate = '02000';
    // No row returned — product code doesn't exist
    dsply ('Product ' + %trim(wk_productCode) + ' not found');

  when sqlstate <> '00000';
    // Some other SQL error
    dsply ('SQL error: ' + sqlstate);

  when active <> 'Y';
    // Row found, but the product is marked inactive
    dsply ('Product ' + %trim(wk_productCode) + ' is inactive');

  other;
    // Found and active — calculate and display
    totalPrice = price * wk_quantity;

    dsply ('Product: ' + %trim(prod));
    dsply ('Price:   ' + %char(price));
    dsply ('Qty:     ' + %char(wk_quantity));
    dsply ('Total:   ' + %char(totalPrice));

    // Determine volume discount
    if wk_quantity >= 100;
      discount = 0.10;
    elseif wk_quantity >= 50;
      discount = 0.05;
    else;
      discount = 0;
    endif;

    // Apply it if any
    if discount > 0;
      moneyOff   = totalPrice * discount;
      totalPrice = totalPrice - moneyOff;
      discountP  = discount * 100;

      dsply (%char(discountP) + '% Discount: ' + %char(moneyOff));
      dsply ('New Total: ' + %char(totalPrice));
    endif;

endsl;

*inlr = *on;
return;
```

## Walkthrough

### The control options line

```rpgle
ctl-opt dftactgrp(*no) actgrp(*caller) option(*nodebugio:*srcstmt);
```

`dftactgrp(*no)` opts out of the default activation group — required for any modern RPG program that uses procedures or service programs. `actgrp(*caller)` tells the compiler to run this program in whatever activation group its caller is using, which is the right default for tutorials and utilities. `option(*nodebugio:*srcstmt)` makes the debugger work with your actual source statements rather than an internally-rewritten version — makes breakpoints land where you expect.

### The program interface

```rpgle
dcl-pi *n;
  wk_productCode char(15) const;
  wk_quantity    packed(7:0) const;
end-pi;
```

This is how an RPG program declares what parameters it accepts from a caller. The `*n` means "use the program's own name" — same shorthand you saw for procedure interfaces in [Part 2 Chapter 3]({% link part-2/03-procedures.md %}).

The `wk_` prefix (for "work") is a common convention marking parameters as caller-supplied inputs. `const` means the program won't modify them — the compiler enforces it, which catches real bugs and tells the caller something useful about the contract.

Matching types to the database matters here. `PR_PROD` in PRODUCT is `char(15)`, so `wk_productCode` is `char(15)`. The compiler can't check this automatically across programs, so keep the habit of matching types to the column you'll compare against.

### The SQL options

```rpgle
exec sql
  set option commit = *none,
             datfmt = *iso,
             closqlcsr = *endactgrp;
```

Three options set once at the top of the program. `commit = *none` tells SQL to auto-commit rather than wait for an explicit commit — fine for tutorial programs, not appropriate for multi-table transactions in production. `datfmt = *iso` standardizes date formatting. `closqlcsr = *endactgrp` controls when SQL cursors close — the right default for programs that exit cleanly.

You'll see this three-line header at the top of almost every SQLRPGLE program in the wild. Copy it.

### The lookup

```rpgle
exec sql
  select PR_PROD, PR_PRICE, PR_ACTIVE
    into :prod, :price, :active
    from PRODUCT
   where PR_PROD = :wk_productCode;
```

A single-row `SELECT INTO`. Three columns selected, three host variables to receive them, in matching order. The `:` prefix marks the RPG variables as SQL host variables — [Chapter 5]({% link part-2/05-embedded-sql.md %}) covered this.

The `where` clause uses `:wk_productCode`, which is the caller's input. Using a host variable (rather than concatenating strings) is what prevents SQL injection. This isn't optional — it's how every production RPG program handles user-supplied values in SQL.

### Handling the result

```rpgle
select;
  when sqlstate = '02000';
    ...
  when sqlstate <> '00000';
    ...
  when active <> 'Y';
    ...
  other;
    ...
endsl;
```

Four distinct cases, ordered by specificity:

- `'02000'` means "no row returned." That's the SQL standard code for a non-result from SELECT INTO. Checked first so we can give a specific message.
- `<> '00000'` is "anything else that isn't success" — genuinely unexpected SQL errors, like a permission issue or a broken connection. Listed second so the specific "not found" check wins when applicable.
- `active <> 'Y'` only runs if `sqlstate = '00000'` (we fell through the first two). The product exists, but is it usable? This is pure business logic layered on top of database state.
- `other` is the default — product exists, is active, go compute.

Order matters in a `select` block. The first matching `when` wins, and later clauses don't run.

### The discount logic

```rpgle
if wk_quantity >= 100;
  discount = 0.10;
elseif wk_quantity >= 50;
  discount = 0.05;
else;
  discount = 0;
endif;
```

Notice the order: check the higher threshold first. If you reversed it — `>= 50` first — the `>= 100` branch would never fire, because 150 satisfies both tests and the first match wins. This kind of bug is easy to write and easy to miss, so when you're writing tiered logic, always check the highest tier first.

The split between "determine discount" and "apply discount" lets us skip the output block entirely when there's no discount, keeping the display clean.

## Compile and run

### Compile

In VS Code's IBM i sidebar, find your new PRCALC member, right-click, choose **Compile**. When prompted, pick **CRTSQLRPGI** (Create SQLRPGLE Program) — not plain CRTBNDRPG. The SQL preprocessor has to run first.

Watch the status bar. On success, a new `*PGM` object called **PRCALC** shows up in your library. On failure, the Problems tab lists errors with clickable line numbers.

### Run

From a 5250 session or the VS Code IBM i terminal:

```
CALL PRCALC PARM('WIDGET-STD-001' 10)
```

You should see:

```
DSPLY  Product: WIDGET-STD-001
DSPLY  Price:   12.99
DSPLY  Qty:     10
DSPLY  Total:   129.90
```

Now try a larger quantity:

```
CALL PRCALC PARM('WIDGET-PRO-001' 75)
```

You should see the discount applied:

```
DSPLY  Product: WIDGET-PRO-001
DSPLY  Price:   29.99
DSPLY  Qty:     75
DSPLY  Total:   2249.25
DSPLY  5% Discount: 112.46
DSPLY  New Total:   2136.79
```

And for the error paths:

```
CALL PRCALC PARM('DOES-NOT-EXIST' 10)
```

Should display `Product DOES-NOT-EXIST not found`.

```
CALL PRCALC PARM('SPROCKET-15T' 10)
```

Should display `Product SPROCKET-15T is inactive` — because that product is in PRODUCT with `PR_ACTIVE = 'N'`.

## Try this

Extensions to deepen your understanding. Pick whichever ones interest you.

**Extend the discount tiers.** Add a new tier: 15% off at 500+ units. What order do you add it in? Test with a quantity that should qualify.

**Add a safety-stock warning.** After calculating the total, do a second SQL lookup for `PR_QOH` and `PR_SAFTYSTK`. If `wk_quantity` would take the on-hand stock below safety stock, display a warning message (the order can still proceed — this is a notice, not a rejection).

**Guard against nonsensical input.** What should happen if someone calls `PRCALC PARM('WIDGET-STD-001' 0)` or a negative quantity? Add a check at the top of the program that rejects anything `<= 0` with a clear message.

**Return a value.** Right now the program just displays results. What if the caller wanted the computed total back? Add a third parameter — an output `packed(11:2)` for the final total — and populate it before returning. Remember to drop `const` on the output parameter, since the program will modify it.

## What's next

You've written a complete program that accepts input, reads the database, branches on multiple conditions, and produces useful output. Every future tutorial in Part 3 builds on this foundation.

Next up: Tutorial 2 will introduce native file I/O — the classic RPG way of moving through records one at a time. You'll see how `CHAIN` and `READ` differ from embedded SQL, when each is the right tool, and why most modern RPG shops mix both.

Next: [Tutorial 2: Reading Records with CHAIN]({% link part-3/02-chain-io.md %}).
