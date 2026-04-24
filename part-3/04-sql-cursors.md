---
title: "4. Multi-Row Queries with SQL Cursors"
parent: "Part 3: Build Real Things"
nav_order: 4
---

# Tutorial 4: Multi-Row Queries with SQL Cursors

Tutorial 3 walked a file sequentially with native I/O. This tutorial does the same kind of work — read many records, process each one — but using SQL. The pattern is called a **cursor**, and once you know it, you can do anything with a set of rows that SQL allows: filter, sort, join, aggregate.

This tutorial also introduces a production-quality technique that isn't in most beginner RPG material: using `extname` to declare a data structure whose shape automatically matches the query result. It saves you from having to declare fifteen host variables by hand.

## What you'll build

A program called **PRODLIST** that takes a supplier code as a parameter, runs a cursor over all that supplier's active products (ordered by product code), displays each product with its on-hand quantity and reorder status, and finishes with a summary line showing how many products were below their reorder point.

## What you'll learn

- The cursor lifecycle: `DECLARE` → `OPEN` → `FETCH` → `CLOSE`
- Fetching an entire row into a single data structure with `FETCH INTO :someDs`
- The `extname` keyword for declaring a DS that matches a table's schema automatically
- The cursor loop pattern — how it mirrors the `SETLL`+`READ` pattern from Tutorial 3
- When to use a cursor vs. a single-row `SELECT INTO`

## Before you start

- Part 1 complete
- [Part 2 Chapter 5: Embedded SQL]({% link part-2/05-embedded-sql.md %}) read — this tutorial extends the cursor section
- Tutorial 1 completed — host variables should feel natural

## The program

Create a new member in `QRPGLESRC` called **PRODLIST**, type **SQLRPGLE** (embedded SQL — CRTSQLRPGI at compile time). Paste in:

```rpgle
**free
ctl-opt dftactgrp(*no) actgrp(*caller) option(*nodebugio:*srcstmt);
// ************************************************************
// * Name: PRODLIST
// * Type: ILE RPG Program (SQLRPGLE)
// * Desc: Active Product Listing for a Supplier
// *       Takes a supplier code, lists all active products
// *       from that supplier with stock status, and reports
// *       how many are below reorder point.
// * Tutorial: Part 3, Tutorial 4
// ************************************************************

// Program parameters
dcl-pi *n;
  wk_suplCode char(10) const;
end-pi;

// Data structure matching the PRODUCT table's shape
dcl-ds prodRec extname('PRODUCT') qualified end-ds;

// Counters
dcl-s totalCount  packed(5:0) inz(0);
dcl-s reorderCount packed(5:0) inz(0);

// Status display variable
dcl-s stockStatus char(3);

// SQL options
exec sql
  set option commit = *none,
             datfmt = *iso,
             closqlcsr = *endactgrp;

// Declare the cursor
exec sql
  declare prodCursor cursor for
    select *
      from PRODUCT
     where PR_SUPL = :wk_suplCode
       and PR_ACTIVE = 'Y'
     order by PR_PROD;

// Open it
exec sql open prodCursor;

if sqlstate <> '00000';
  dsply ('Error opening cursor: ' + sqlstate);
  *inlr = *on;
  return;
endif;

// Header
dsply ('Active products for supplier ' + %trim(wk_suplCode));
dsply '---------------------------------------';

// Fetch loop
exec sql fetch next from prodCursor into :prodRec;

dow sqlstate = '00000';

  totalCount += 1;

  // Flag if stock is at or below reorder point
  if prodRec.PR_QOH <= prodRec.PR_REORDPT;
    stockStatus = '!! ';
    reorderCount += 1;
  else;
    stockStatus = '   ';
  endif;

  dsply (stockStatus
       + %trim(prodRec.PR_PROD)
       + '  QOH: ' + %char(prodRec.PR_QOH)
       + '  ROP: ' + %char(prodRec.PR_REORDPT));

  // Fetch the next row
  exec sql fetch next from prodCursor into :prodRec;

enddo;

// Close the cursor
exec sql close prodCursor;

// Summary
dsply '---------------------------------------';
dsply ('Total:        ' + %char(totalCount));
dsply ('Below reorder: ' + %char(reorderCount));

*inlr = *on;
return;
```

## Walkthrough

### The extname data structure

```rpgle
dcl-ds prodRec extname('PRODUCT') qualified end-ds;
```

This single line is the most valuable pattern in the whole tutorial. `extname('PRODUCT')` tells the compiler: "look at the PRODUCT table's schema and build this data structure to match it." You get `prodRec.PR_PROD`, `prodRec.PR_SUPL`, `prodRec.PR_ACTIVE`, `prodRec.PR_QOH`, and every other column — with the right types — without typing a thing.

If the table schema changes (new column added, type widened), recompiling your program picks it up automatically. Compare that to writing fifteen `dcl-s` lines by hand and manually keeping them in sync. Big win.

`qualified` is here because it always should be — [Part 2 Chapter 2]({% link part-2/02-data-structures.md %}) was firm about this. No exceptions.

### Declaring the cursor

```rpgle
exec sql
  declare prodCursor cursor for
    select *
      from PRODUCT
     where PR_SUPL = :wk_suplCode
       and PR_ACTIVE = 'Y'
     order by PR_PROD;
```

`DECLARE CURSOR` doesn't execute the query. It just tells SQL "here's the query I'm going to run in a moment; give it a name I can reference later." The cursor's name — `prodCursor` — is the handle we'll use for OPEN, FETCH, and CLOSE.

Two things to notice about the SQL itself:

**`SELECT *`.** Normally you want to list columns explicitly. Here, because we're fetching into a DS that matches the full table schema, `SELECT *` is actually the right choice — it guarantees column count and order match the DS. If you fetched fewer columns than the DS has, the extra DS fields would be left undefined. Match them or declare a narrower DS.

**The host variable in WHERE.** `:wk_suplCode` uses the caller's input — same pattern as Tutorial 1. Never concatenate user input into a SQL statement.

### Opening and checking

```rpgle
exec sql open prodCursor;

if sqlstate <> '00000';
  dsply ('Error opening cursor: ' + sqlstate);
  *inlr = *on;
  return;
endif;
```

`OPEN` actually runs the query — or more precisely, prepares to return rows. If something is wrong with the SQL (table doesn't exist, column misspelled), you'll learn about it here.

Checking `sqlstate` after OPEN is good hygiene. If it fails, there's no point fetching.

### The fetch loop

```rpgle
exec sql fetch next from prodCursor into :prodRec;

dow sqlstate = '00000';
  ...
  exec sql fetch next from prodCursor into :prodRec;
enddo;
```

This is the SQL equivalent of the `SETLL`+`READ` pattern from Tutorial 3. Study the structure:

1. **Fetch once before the loop** (the priming fetch)
2. **Loop while the last fetch succeeded** (`sqlstate = '00000'`)
3. **Process the row you have**
4. **Fetch again at the bottom**
5. **Condition re-checked — exits when there are no more rows**

`'00000'` means "fetch succeeded, row is in the DS." `'02000'` means "no more rows." Any other value is an error.

The shape is identical to Tutorial 3. Internalize it once, recognize it everywhere.

### Closing

```rpgle
exec sql close prodCursor;
```

Once you're done, close the cursor. This releases the resources the database was holding for your query. The `closqlcsr = *endactgrp` option we set at the top would eventually close it anyway, but explicit is better — and if you forget the close, multiple concurrent cursors can leak.

## Compile and run

Compile with **CRTSQLRPGI** (embedded SQL → SQL preprocessor → bound RPG).

```
CALL PRODLIST PARM('ACME001')
```

You should see:

```
DSPLY  Active products for supplier ACME001
DSPLY  ---------------------------------------
DSPLY     WIDGET-STD-001  QOH: 450  ROP: 500
DSPLY     WIDGET-PRO-001  QOH: 2500  ROP: 1000
DSPLY  !! WIDGET-DLX-001  QOH: 75  ROP: 150
DSPLY  ---------------------------------------
DSPLY  Total:        3
DSPLY  Below reorder: 2
```

The `!!` flags products where on-hand quantity is at or below the reorder point — two of the three Acme widgets qualify from the sample data.

Try a supplier with many products:

```
CALL PRODLIST PARM('BETA002')
```

Four products, each with its own stock status.

Try an inactive supplier:

```
CALL PRODLIST PARM('DLTA004')
```

Delta is inactive, and its only product is also inactive, so the cursor returns zero rows and the total shows 0.

## Single-row SELECT vs cursor — when to reach for which

Tutorial 1 used `SELECT INTO` for a single-row lookup. This tutorial used a cursor. The choice between them:

**SELECT INTO.** Use when you know exactly one row will match (or when zero matching rows is an expected "not found" case). The query returns, you check `sqlstate`, you're done. Cleaner, less ceremony.

**Cursor.** Use when you need to iterate over multiple rows, when you want to filter/sort/join the results, or when you want to process each row before deciding about the next.

**A common middle ground.** If you want "the first matching row" — the most recent order for a customer, the highest-priced product in a category — you can add `LIMIT 1` (or `FETCH FIRST 1 ROWS ONLY` in stricter SQL) to a query and use `SELECT INTO`. Often cleaner than opening a cursor just to fetch once and close.

The cursor lifecycle carries real overhead (declare, open, fetch, close). Don't use a cursor when a single-row SELECT will do.

## Try this

**Swap WHERE filters.** Remove the `AND PR_ACTIVE = 'Y'` clause. Now the listing includes inactive products too. Add a column in the output to show which are active vs. inactive.

**Add a running total.** Declare `runningValue packed(11:2) inz(0)` and inside the loop compute `runningValue += prodRec.PR_PRICE * prodRec.PR_QOH`. After the loop, display "Total inventory value: " with the running total. You've just built a simple inventory valuation report.

**Parameterize the filter column.** Change the cursor's WHERE clause so it filters on on-hand quantity — only return products with `PR_QOH < 100`. Which supplier gives you the most "low stock" results?

**Two cursors in one program.** This is more advanced. Declare a second cursor for SUPPLIER with `WHERE SP_ACTIVE = 'Y'`. Outer loop: fetch each active supplier. Inner loop: fetch that supplier's active products. Write a supplier-by-supplier report. You've just written a nested-loop join in procedural code.

## What's next

You've now done both native sequential I/O and SQL cursors. The "read many records, process each one" pattern is internalized.

Tutorial 5 shifts from reading to *building* — making reusable logic by writing procedures, then packaging procedures into modules you can call from anywhere.

Next: [Tutorial 5: Building Reusable Logic with Procedures]({% link part-3/05-procedures.md %}).

