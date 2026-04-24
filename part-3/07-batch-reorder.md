---
title: "7. Batch Reorder Report — The Capstone"
parent: "Part 3: Build Real Things"
nav_order: 7
---

# Tutorial 7: Batch Reorder Report — The Capstone

Every tutorial up to here has taught one concept. This one builds something *real* by combining them.

You're going to write a batch program that every distributor wants: identify which products need to be reordered, and write that list to a separate table so a buyer can act on it. It uses cursors (Tutorial 4), procedures (Tutorial 5), a service program call (Tutorial 6), and inserts new rows into the database — a capability we haven't used yet.

This is the biggest single program in Part 3. Budget an hour or so. When you're done, you've written something a real RPG developer would ship.

## What you'll build

A program called **REORDRPT** (Reorder Report) that:

1. Wipes out any existing rows in REORDCND (we're rebuilding the list fresh)
2. Walks through every **active** product whose supplier is also **active**
3. For each one, checks whether on-hand quantity is at or below reorder point
4. If it is, calculates a suggested reorder quantity and inserts a row into REORDCND
5. Reports summary counts at the end — how many products scanned, how many flagged, how many skipped because their supplier was inactive

The formula for suggested reorder quantity is realistic but simple:

```
suggested_qty = reorder_point + safety_stock - current_qoh
```

This brings stock up to reorder-point plus safety-stock — enough to cover a reorder cycle with a cushion.

## What you'll learn

- `DELETE` and `INSERT` statements — writing back to the database
- Combining SQL filters with program-level logic, and when to choose one over the other
- Calling a service program procedure from inside a cursor loop
- Structuring a batch program: setup, main loop, summary
- A realistic business calculation expressed cleanly in RPG

## Before you start

- Part 1 complete — your REORDCND table exists and is empty
- Tutorials 4, 5, and 6 completed — cursors, procedures, and service programs all show up here
- The INVSVR service program from Tutorial 6 is built (we'll call `IsProductActive` from it)
- Roughly 45–60 minutes of focused time

## The program

Create a new member in `QRPGLESRC` called **REORDRPT**, type **SQLRPGLE**. Paste:

```rpgle
**free
ctl-opt dftactgrp(*no) actgrp(*caller)
        option(*nodebugio:*srcstmt)
        bnddir('INVBND')
        main(Main);
// ************************************************************
// * Name: REORDRPT
// * Type: ILE RPG Program (SQLRPGLE)
// * Desc: Batch Reorder Report
// *       Scans every active product, flags those at or
// *       below reorder point, calculates suggested reorder
// *       quantity, and writes candidates to REORDCND.
// * Tutorial: Part 3, Tutorial 7 (capstone)
// ************************************************************

/copy QCPYLESRC,INVSVR_H

// ---- Prototypes ---------------------------------------------

dcl-pr CalcReorderQty packed(9:0);
  qoh         packed(9:0) const;
  reorderPt   packed(9:0) const;
  safetyStock packed(9:0) const;
end-pr;

dcl-pr ClearReorderTable ind;
end-pr;

dcl-pr ProcessProducts;
end-pr;

// ---- Counters (module-level so helpers can update them) -----

dcl-s scanned       packed(7:0) inz(0);
dcl-s flagged       packed(7:0) inz(0);
dcl-s skippedSupInactive packed(7:0) inz(0);

// ---- Main procedure -----------------------------------------

dcl-proc Main;
  dcl-pi *n end-pi;

  exec sql
    set option commit = *none,
               datfmt = *iso,
               closqlcsr = *endactgrp;

  dsply 'Reorder Report — starting';

  if not ClearReorderTable();
    dsply 'Failed to clear REORDCND. Aborting.';
    return;
  endif;

  ProcessProducts();

  dsply '---------------------------------';
  dsply ('Products scanned:       ' + %char(scanned));
  dsply ('Flagged for reorder:    ' + %char(flagged));
  dsply ('Skipped (inactive sup): ' + %char(skippedSupInactive));
  dsply 'Reorder Report — complete';

  return;
end-proc;

// ---- Helper: clear out the REORDCND table -------------------

dcl-proc ClearReorderTable;
  dcl-pi *n ind end-pi;

  exec sql
    delete from REORDCND;

  if sqlstate <> '00000' and sqlstate <> '02000';
    return *off;
  endif;

  return *on;
end-proc;

// ---- Helper: compute a suggested reorder quantity -----------

dcl-proc CalcReorderQty;
  dcl-pi *n packed(9:0);
    qoh         packed(9:0) const;
    reorderPt   packed(9:0) const;
    safetyStock packed(9:0) const;
  end-pi;

  return reorderPt + safetyStock - qoh;
end-proc;

// ---- Helper: main product scan loop -------------------------

dcl-proc ProcessProducts;
  dcl-pi *n end-pi;

  dcl-s prod        char(15);
  dcl-s sup         char(10);
  dcl-s qoh         packed(9:0);
  dcl-s reorderPt   packed(9:0);
  dcl-s safetyStock packed(9:0);
  dcl-s suggested   packed(9:0);

  exec sql
    declare prodCursor cursor for
      select PR_PROD, PR_SUPL, PR_QOH, PR_REORDPT, PR_SAFTYSTK
        from PRODUCT
       where PR_ACTIVE = 'Y'
         and PR_QOH <= PR_REORDPT
       order by PR_SUPL, PR_PROD;

  exec sql open prodCursor;

  exec sql fetch next from prodCursor
    into :prod, :sup, :qoh, :reorderPt, :safetyStock;

  dow sqlstate = '00000';

    scanned += 1;

    // Only reorder from active suppliers
    if IsProductActive(prod) and IsSupplierActive(sup);

      suggested = CalcReorderQty(qoh : reorderPt : safetyStock);

      exec sql
        insert into REORDCND
          (RC_PROD, RC_SUPL, RC_QOH, RC_REORD, RC_SUGG, RC_CRTDT)
        values
          (:prod, :sup, :qoh, :reorderPt, :suggested, current_date);

      if sqlstate = '00000';
        flagged += 1;
      endif;

    else;
      skippedSupInactive += 1;
    endif;

    exec sql fetch next from prodCursor
      into :prod, :sup, :qoh, :reorderPt, :safetyStock;

  enddo;

  exec sql close prodCursor;

  return;
end-proc;

// ---- Helper: check if supplier is active --------------------
// Not in INVSVR service program — local to this tutorial.
// You'd promote this to INVSVR in a real codebase.

dcl-proc IsSupplierActive;
  dcl-pi *n ind;
    suplCode char(10) const;
  end-pi;

  dcl-s active char(1);

  exec sql
    select SP_ACTIVE into :active
      from SUPPLIER
     where SP_SUPL = :suplCode;

  if sqlstate <> '00000';
    return *off;
  endif;

  return (active = 'Y');
end-proc;

// ---- Prototype for local helper -----------------------------

dcl-pr IsSupplierActive ind;
  suplCode char(10) const;
end-pr;
```

## Walkthrough

### The batch structure

A batch program has a shape you'll see over and over again:

1. **Setup** — SQL options, open resources
2. **Cleanup step** — often, clear previous results (`DELETE`)
3. **Main loop** — read source records, process them
4. **Summary** — totals, counts, status
5. **Teardown** — close resources, return

REORDRPT follows this shape exactly. `Main` handles the outer orchestration; `ClearReorderTable` is the cleanup step; `ProcessProducts` is the main loop; the summary lives at the end of `Main`.

Having a shape to follow is valuable. When you're staring at a blank source file wondering "what goes first?" — use this structure.

### Two levels of filtering

Look at the cursor's WHERE clause:

```sql
where PR_ACTIVE = 'Y'
  and PR_QOH <= PR_REORDPT
```

And look at the runtime check inside the loop:

```rpgle
if IsProductActive(prod) and IsSupplierActive(sup);
```

Redundant? Not quite. `PR_ACTIVE = 'Y'` in the cursor pre-filters to candidates worth considering — it cuts the cursor's size dramatically, and it's enforced in one place by the database. `IsProductActive` is redundant with that, but I called it anyway to show how a service program fits into a cursor loop — and in a real codebase, the logic for "what counts as active" might become more sophisticated (date ranges, seasonal rules, inventory status codes) and live in the service program alone.

**`IsSupplierActive` is the non-redundant check.** The cursor pulls products, not suppliers, and joining SUPPLIER in the cursor would complicate the SQL. Doing it as a per-row lookup is cleaner at this scale — for thousands of products it's fast enough, and if performance ever mattered you'd change the cursor to a `JOIN` query (an extension below).

**The general principle:** push filtering to SQL when it's simple and big-volume; do it in RPG when it's logic-heavy or touches service programs. Knowing where to draw the line is experience you'll develop.

### Module-level state

```rpgle
dcl-s scanned       packed(7:0) inz(0);
dcl-s flagged       packed(7:0) inz(0);
dcl-s skippedSupInactive packed(7:0) inz(0);
```

These live *outside* any procedure. That makes them **module-level variables** — accessible to every procedure in this source file. `Main` initializes them (via `inz(0)`), `ProcessProducts` updates them, `Main` reads them at the end.

In small batch programs this is fine and common. In larger programs, passing state explicitly is cleaner — it's easier to follow data flow when every procedure's inputs and outputs are on its PI rather than shared through module state. The judgment is taste plus program size: under a hundred lines, module-level counters are fine; past a few hundred, pass them.

### The INSERT

```rpgle
exec sql
  insert into REORDCND
    (RC_PROD, RC_SUPL, RC_QOH, RC_REORD, RC_SUGG, RC_CRTDT)
  values
    (:prod, :sup, :qoh, :reorderPt, :suggested, current_date);
```

First SQL write in the tutorials so far. The structure is the same as the SELECT pattern — host variables prefixed with `:`, matched by position to the column list.

`current_date` is a SQL built-in that evaluates to today's date at execution time. Simpler than computing it in RPG and passing it in, and it guarantees the timestamp reflects when the row was written, not when the program started.

After every INSERT, check `sqlstate`. `'00000'` is success. Other states mean the row didn't land — bad data, constraint violation, whatever — and you shouldn't count it as flagged.

### Why the delete first

```rpgle
if not ClearReorderTable();
```

Every run of REORDRPT starts by emptying REORDCND. That's a design choice: the program produces the *current* list of reorder candidates, not an accumulated history.

If you wanted history instead, you'd skip the delete and add `RC_CRTDT` filtering to whatever reads from REORDCND — "give me today's candidates" or "give me the last 30 days." Both designs are valid.

If the delete fails, we bail. You never want to write new rows when you don't know whether old ones were cleared — you'd end up with mixed state.

## Compile and run

This tutorial requires the INVSVR service program and INVBND binding directory from Tutorial 6. If you haven't built those, do that first, or modify the source to replace `IsProductActive(...)` calls with local SQL lookups.

Compile with **CRTSQLRPGI**.

```
CALL REORDRPT
```

No parameters. You should see roughly:

```
DSPLY  Reorder Report — starting
DSPLY  ---------------------------------
DSPLY  Products scanned:       14
DSPLY  Flagged for reorder:    13
DSPLY  Skipped (inactive sup): 1
DSPLY  Reorder Report — complete
```

The numbers reflect Part 1's sample data: 14 active products are below their reorder point (we verified this in Part 2 Chapter 5); 13 of them have active suppliers; 1 is from an inactive supplier and gets skipped.

Now verify the output table in the Db2 extension:

```sql
SELECT * FROM REORDCND ORDER BY RC_SUPL, RC_PROD;
```

You should see 13 rows with today's date in RC_CRTDT. A sampling:

```
RC_PROD          RC_SUPL    RC_QOH  RC_REORD  RC_SUGG  RC_CRTDT
BOLT-M8-STEEL    BETA002    50      200       200      2026-04-24
NUT-M8-STEEL     BETA002    125     300       250      2026-04-24
GEAR-SPUR-24T    CTRL003    12      50        48       2026-04-24
...
```

Run `REORDRPT` again. You should get the same counts and the same 13 rows — but with refreshed timestamps. That's the delete-then-insert behavior working.

## Try this

Five extensions, ordered easy to hard.

**Add a report to the screen.** After the INSERT, display the product code and suggested quantity as each row is written. Turns the program from "batch report" into "batch report with live progress."

**Move `IsSupplierActive` into INVSVR.** You've now seen this procedure written locally in REORDRPT. Moving it into the service program is the natural next step. Add the prototype to `INVSVR_H`, copy the definition into `INVSVR.sqlrpgle` with `export`, rebuild the module and service program, delete the local version from REORDRPT. Recompile. It should still run identically.

**Replace the subquery check with a JOIN.** Change the cursor's WHERE clause to `INNER JOIN SUPPLIER ON PR_SUPL = SP_SUPL WHERE SP_ACTIVE = 'Y'` and add the relevant columns to the select list. Now the database does the supplier-active check before the cursor returns the row. Drop the `IsSupplierActive` call and the `skippedSupInactive` counter. Compare line count and readability.

**Parameterize the supplier.** Add a parameter to `Main` — `wk_suplCode char(10) const`. Use it in the cursor's WHERE clause so the program only processes one supplier at a time. If the caller passes `*BLANKS`, process all suppliers. That way the program is useful for both batch-all-at-once and single-supplier-on-demand workflows.

**Write a unit test.** Tutorial 12 covers RPGUnit, but here's a preview: create a new program TSTREORD that CALLs REORDRPT, then runs SQL to verify the expected counts. Basic integration test. You'll have a working test before Tutorial 12 even starts.

## What you accomplished

You built a batch program that writes to the database. You used cursors, prototypes, procedures, and a service program call — everything from Tutorials 4, 5, and 6. The structure is representative of real RPG batch work: delete previous results, walk the source set, filter and transform, insert.

If you understand every line of REORDRPT, you understand most of what production RPG looks like. The remaining tutorials add operational concerns — error handling, date math, CL orchestration, testing — but none of them will introduce concepts this big.

## What's next

Wave 2 is done. Wave 3 shifts to **operational quality** — the parts of professional code that aren't about features, but about running well in production. Tutorial 8 covers `MONITOR` for graceful error handling. Tutorial 9 is practical date math. Tutorial 10 is CL as the glue that schedules, sequences, and controls RPG programs.

Next: *Tutorial 8: Error Handling with MONITOR (coming in Wave 3)*.
