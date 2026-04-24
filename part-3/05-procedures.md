---
title: "5. Building Reusable Logic with Procedures"
parent: "Part 3: Build Real Things"
nav_order: 5
---

# Tutorial 5: Building Reusable Logic with Procedures

So far every tutorial has been a single flat program. This one breaks work into **procedures** — named, reusable chunks of logic with their own inputs and outputs. It's how every real RPG codebase you'll encounter is organized.

[Part 2 Chapter 3]({% link part-2/03-procedures.md %}) introduced the syntax. This tutorial puts it to work on a realistic business task and walks through the design decisions — not just "how to write a procedure" but "when and why to extract one."

## What you'll build

A program called **INVVAL** (Inventory Valuation) that computes the total dollar value of a supplier's active inventory — `SUM(PR_QOH × PR_PRICE)` across all their active products.

You could write it as a flat program: open a cursor, loop, multiply, accumulate, close. And in about fifteen lines of code. But we're going to structure it with procedures — one procedure computes a single product's value, another procedure totals a collection of products, and the main program orchestrates. The logic stays testable and reusable; any of the procedures could move to a service program in Tutorial 6 without rewriting.

## What you'll learn

- Designing a program as a main procedure plus helpers instead of one flat flow
- Declaring a procedure prototype with `dcl-pr` so one procedure can call another
- Using data structure templates with `likeds` to keep shapes consistent
- Returning values from procedures, and choosing the right return type
- When to extract a procedure (and when not to)

## Before you start

- Part 1 complete
- [Part 2 Chapter 3: Procedures]({% link part-2/03-procedures.md %}) read
- Tutorials 1–4 completed

## The program

Create a new member in `QRPGLESRC` called **INVVAL**, type **SQLRPGLE**. Paste in:

```rpgle
**free
ctl-opt dftactgrp(*no) actgrp(*caller)
        option(*nodebugio:*srcstmt)
        main(Main);
// ************************************************************
// * Name: INVVAL
// * Type: ILE RPG Program (SQLRPGLE)
// * Desc: Inventory Valuation
// *       Computes the dollar value of a supplier's active
// *       inventory, broken out by product.
// * Tutorial: Part 3, Tutorial 5
// ************************************************************

// ---- Prototypes for procedures defined later ----------------

dcl-pr CalcLineValue packed(13:2);
  qoh   packed(9:0) const;
  price packed(9:2) const;
end-pr;

dcl-pr GetInventoryTotal packed(15:2);
  suplCode char(10) const;
end-pr;

// ---- Data structure template for a product line -------------

dcl-ds lineInfo qualified template;
  prod  char(15);
  qoh   packed(9:0);
  price packed(9:2);
  value packed(13:2);
end-ds;

// ---- Main procedure -----------------------------------------

dcl-proc Main;
  dcl-pi *n;
    wk_suplCode char(10) const;
  end-pi;

  dcl-s total packed(15:2);

  exec sql
    set option commit = *none,
               datfmt = *iso,
               closqlcsr = *endactgrp;

  total = GetInventoryTotal(wk_suplCode);

  dsply ('Supplier:       ' + %trim(wk_suplCode));
  dsply ('Inventory value: $' + %char(total));

  return;
end-proc;

// ---- Helper: compute a single line's value ------------------

dcl-proc CalcLineValue;
  dcl-pi *n packed(13:2);
    qoh   packed(9:0) const;
    price packed(9:2) const;
  end-pi;

  return qoh * price;
end-proc;

// ---- Helper: walk the supplier's products and sum -----------

dcl-proc GetInventoryTotal;
  dcl-pi *n packed(15:2);
    suplCode char(10) const;
  end-pi;

  dcl-ds line likeds(lineInfo);
  dcl-s runningTotal packed(15:2) inz(0);

  exec sql
    declare invCursor cursor for
      select PR_PROD, PR_QOH, PR_PRICE
        from PRODUCT
       where PR_SUPL = :suplCode
         and PR_ACTIVE = 'Y'
       order by PR_PROD;

  exec sql open invCursor;

  exec sql fetch next from invCursor
    into :line.prod, :line.qoh, :line.price;

  dow sqlstate = '00000';

    line.value = CalcLineValue(line.qoh : line.price);
    runningTotal += line.value;

    dsply (%trim(line.prod)
         + '  QOH: ' + %char(line.qoh)
         + '  Price: ' + %char(line.price)
         + '  Value: ' + %char(line.value));

    exec sql fetch next from invCursor
      into :line.prod, :line.qoh, :line.price;

  enddo;

  exec sql close invCursor;

  return runningTotal;
end-proc;
```

## Walkthrough

### The main procedure pattern

```rpgle
ctl-opt ... main(Main);

dcl-proc Main;
  dcl-pi *n;
    wk_suplCode char(10) const;
  end-pi;
  ...
end-proc;
```

When `ctl-opt main(Main)` is set, the program has an explicit main procedure. Parameters to the program are declared on that procedure's interface. Everything lives inside named procedures — no more top-level code.

This is the modern idiom. All four tutorials before this one were written without `main()` for simplicity, but from here on we'll use it. It's how production codebases are organized: each source file is a collection of procedures, one of which is the entry point.

### Prototypes vs. definitions

```rpgle
// At the top:
dcl-pr CalcLineValue packed(13:2);
  qoh   packed(9:0) const;
  price packed(9:2) const;
end-pr;

// At the bottom:
dcl-proc CalcLineValue;
  dcl-pi *n packed(13:2);
    qoh   packed(9:0) const;
    price packed(9:2) const;
  end-pi;

  return qoh * price;
end-proc;
```

A **prototype** (`dcl-pr`) declares a procedure's *shape* — name, return type, parameters — without providing the body. A **definition** (`dcl-proc`) provides the body.

Why two things? Because procedure calls in RPG are resolved top-down. When the compiler sees `CalcLineValue(...)` inside `GetInventoryTotal`, it needs to know what `CalcLineValue` looks like — how many parameters, what types, what it returns. If the definition comes later in the source, the call won't compile.

The solution: put prototypes for all your procedures near the top of the file. The compiler sees the shapes up front and can resolve any call order it wants. The definitions at the bottom fill in the bodies.

Prototype parameters must match the PI exactly. If you change the PI, update the prototype, or the compiler gets confused.

### Template data structures

```rpgle
dcl-ds lineInfo qualified template;
  prod  char(15);
  qoh   packed(9:0);
  price packed(9:2);
  value packed(13:2);
end-ds;
```

`template` means "this DS is a shape, not a variable." No memory is allocated for `lineInfo` itself. But the compiler knows its layout, and other DS declarations can reference it:

```rpgle
dcl-ds line likeds(lineInfo);
```

This creates a real DS named `line` with the same layout as `lineInfo`. `line` gets the memory. `lineInfo` is just the blueprint.

[Part 2 Chapter 2]({% link part-2/02-data-structures.md %}) introduced `likeds` — this tutorial shows why it's useful. A template at the program scope plus `likeds` instances inside procedures means the "shape" is defined in exactly one place. Change the template, every instance updates on recompile.

### Data flow across procedures

Trace one product through the program:

1. `Main` calls `GetInventoryTotal(wk_suplCode)`
2. Inside `GetInventoryTotal`, the cursor fetches a row into `line.prod`, `line.qoh`, `line.price`
3. `GetInventoryTotal` calls `CalcLineValue(line.qoh : line.price)`
4. `CalcLineValue` multiplies and returns
5. Back in `GetInventoryTotal`, the result goes into `line.value`
6. `line.value` adds to `runningTotal`
7. After the loop, `runningTotal` is returned to `Main`
8. `Main` displays it

Each procedure does one job. The work is decomposed. Any single procedure is small enough to reason about in isolation.

### When to extract a procedure

Honest take, because "extract every three lines into a procedure" is a classic over-engineering trap.

Extract when **the logic is reused**. `CalcLineValue` is the clearest case — it's called once per row, in a loop, and could easily be reused by another program later.

Extract when **the logic has a name**. "Multiply quantity by price" is better expressed as `CalcLineValue(qoh, price)` than as `qoh * price` buried in a loop — the name tells the reader what you *meant*, not just what you did. Pricing rules have a habit of growing: today it's a multiplication, in six months someone adds tax. Having the extraction already in place means the change is local.

Extract when **testing becomes easier**. `CalcLineValue` is pure — inputs go in, output comes out, no side effects. That's trivially testable. If it were inline in the cursor loop, you'd have to set up a cursor every time you wanted to test the calculation.

**Don't extract** when the extracted procedure is just a rename. `dcl-proc Trim; return %trim(...); end-proc;` adds indirection without clarity. Don't do it.

**Don't extract** when the two procedures share so much state that you end up passing ten parameters back and forth. That's a sign the two things belong together as one procedure.

## Compile and run

Compile with **CRTSQLRPGI**.

```
CALL INVVAL PARM('BETA002')
```

You should see output like:

```
DSPLY  BOLT-M10-STEEL  QOH: 890  Price: 0.22  Value: 195.80
DSPLY  BOLT-M12-STEEL  QOH: 1500  Price: 0.35  Value: 525.00
DSPLY  BOLT-M8-STEEL  QOH: 50  Price: 0.15  Value: 7.50
DSPLY  NUT-M8-STEEL  QOH: 125  Price: 0.08  Value: 10.00
DSPLY  Supplier:       BETA002
DSPLY  Inventory value: $738.30
```

Try it on other suppliers — ACME001, CTRL003, INDG009 — and compare. Inactive suppliers like DLTA004 return zero because none of their products qualify as active.

## Try this

**Add a minimum-value filter.** Extract a third procedure, `ShouldReport(value)`, that returns `*on` only if `value >= 10.00`. Use it in `GetInventoryTotal` to skip low-value lines in the display — but still sum them into the total. Practice with boolean-returning procedures.

**Return a DS from GetInventoryTotal.** Right now it returns a scalar. Change it to return a DS with `total packed(15:2)` and `count packed(5:0)` fields. Update the prototype, the PI, and the return. `Main` displays both.

**Sort differently.** Change the ORDER BY from `PR_PROD` to `PR_PRICE DESC`. Does the total change? Why not? What's the conceptual difference between what "changes because of order" and what "doesn't"?

**Count things with `%xfoot`.** `%xfoot` is the "cross foot" BIF — it sums every element of a numeric array. Rewrite `GetInventoryTotal` to build an array of line values during the fetch loop, then `%xfoot` them at the end. Same result, different tools.

## What's next

You've decomposed one program into procedures. Tutorial 6 takes the next step — extracting procedures into a **service program** so they're callable from multiple programs. That's how teams build shared business logic.

Next: [Tutorial 6: Service Programs]({% link part-3/06-service-programs.md %}).
