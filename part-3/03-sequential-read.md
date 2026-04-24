---
title: "3. Sequential Reading with SETLL and READ"
parent: "Part 3: Build Real Things"
nav_order: 3
---

# Tutorial 3: Sequential Reading with SETLL and READ

Tutorial 2 read one record at a time using `CHAIN`. This tutorial reads *all* the records, one after another, using `SETLL` and `READ`. It's how native RPG walks a table from start to finish.

You'll still see both techniques mixed in production code. SQL cursors (Tutorial 4) have largely replaced this pattern for new work, but understanding it means you can read the vast body of legacy RPG that uses it — and there are still cases where it's the cleanest approach.

## What you'll build

A program called **SUPCNT** (Supplier Count) that walks the SUPPLIER file from start to finish, counts how many suppliers are active vs. inactive, and displays the totals. No parameters — just runs, counts, reports.

## What you'll learn

- Positioning a file pointer with `SETLL *start`
- Reading records sequentially in a loop with `READ`
- Using `%eof` to detect when you've read past the last record
- Why the "read, then loop" pattern is standard
- How this compares to the SQL cursor pattern in Tutorial 4

## Before you start

- Part 1 complete — your tables exist
- Tutorial 2 completed — `dcl-f` and `likerec` will be familiar

## The program

Create a new member in `QRPGLESRC` called **SUPCNT**, type **RPGLE**. Paste in:

```rpgle
**free
ctl-opt dftactgrp(*no) actgrp(*caller) option(*nodebugio:*srcstmt);
// ************************************************************
// * Name: SUPCNT
// * Type: ILE RPG Program
// * Desc: Supplier Count Report
// *       Walks the SUPPLIER file sequentially, counting
// *       active and inactive suppliers, then displays
// *       the totals.
// * Tutorial: Part 3, Tutorial 3
// ************************************************************

dcl-f SUPPLIER usage(*input) keyed;

dcl-ds supRec likerec(SP_REC);

dcl-s activeCount   packed(5:0) inz(0);
dcl-s inactiveCount packed(5:0) inz(0);
dcl-s totalCount    packed(5:0) inz(0);

// Position the file pointer at the very beginning
setll *start SUPPLIER;

// Prime the read — get the first record before entering the loop
read SUPPLIER supRec;

dow not %eof(SUPPLIER);

  totalCount += 1;

  if supRec.SP_ACTIVE = 'Y';
    activeCount += 1;
  else;
    inactiveCount += 1;
  endif;

  // Read the next record. The loop exits when this hits EOF.
  read SUPPLIER supRec;

enddo;

// Report
dsply 'Supplier Count Report';
dsply '---------------------';
dsply ('Active:   ' + %char(activeCount));
dsply ('Inactive: ' + %char(inactiveCount));
dsply ('Total:    ' + %char(totalCount));

*inlr = *on;
return;
```

## Walkthrough

### Positioning with SETLL

```rpgle
setll *start SUPPLIER;
```

`SETLL` is "set lower limit" — it positions the file's internal read pointer at a specific location. `*start` is a special value meaning "the very beginning." After this operation, the next `READ` returns the first record in key sequence.

You can also pass a key value to `SETLL` — e.g., `setll 'M' SUPPLIER` positions at the first supplier whose code starts with `M`. Useful when you only want to process a subset of the file. For this tutorial, we want everything, so `*start` is the right choice.

### The read-then-loop pattern

```rpgle
read SUPPLIER supRec;

dow not %eof(SUPPLIER);
  ...
  read SUPPLIER supRec;
enddo;
```

This is the classic native sequential read pattern. Look at it carefully because the structure matters:

1. **Read once before the loop** — this is the "priming read." It either gets you the first record or sets `%eof` to true (if the file is empty).
2. **Loop while NOT at end of file** — process the record you have.
3. **Read again at the bottom of the loop** — this is the "advancing read." It either gets the next record or sets `%eof` to true.
4. **Loop condition is re-checked** — if EOF, the loop exits; otherwise, process the new record.

This pattern is everywhere in RPG. New developers often try to "simplify" it by reading inside the loop condition, which ends up processing garbage data because `%eof` is checked before the read completes. Don't fight the pattern — use it.

### %eof

```rpgle
dow not %eof(SUPPLIER);
```

`%eof(SUPPLIER)` returns true if the most recent read operation against SUPPLIER hit end of file. Like `%found` in Tutorial 2, it's only valid until the next I/O — check it immediately after the read.

Note the filename argument. If you have multiple files open, `%eof(SUPPLIER)` is specific to SUPPLIER; `%eof(PRODUCT)` would be a separate status. Always name the file you mean.

### The counting logic

```rpgle
if supRec.SP_ACTIVE = 'Y';
  activeCount += 1;
else;
  inactiveCount += 1;
endif;
```

Plain conditional counting. Notice we're using the qualified DS field `supRec.SP_ACTIVE` rather than the bare `SP_ACTIVE` that would also be available (since we declared the file). Qualified access is clearer and future-proof.

## Compile and run

Compile with **CRTBNDRPG** (no SQL, no SQLRPGI needed).

```
CALL SUPCNT
```

No parameters. You should see:

```
DSPLY  Supplier Count Report
DSPLY  ---------------------
DSPLY  Active:   12
DSPLY  Inactive: 3
DSPLY  Total:    15
```

Those numbers match Part 1's sample data: 15 total suppliers, 3 of which (Delta, Gamma, Mike's) are marked inactive.

## SETLL + READ vs SQL cursors

Tutorial 4 will do the same thing — walk a set of records — using SQL. Here's the honest comparison so you have it in advance.

**Native is shorter when reading an entire file in key sequence.** The program above is about 30 lines of executable code. The SQL cursor version you'll write in Tutorial 4 is similar in line count but has more ceremony (declare, open, fetch, close).

**SQL is more flexible.** If you wanted just active suppliers, sorted by name, you'd have to do that filtering and sorting inside the RPG loop with native I/O (reading everything, branching, maybe storing in an array first for sorting). With SQL, you add a `WHERE SP_ACTIVE = 'Y' ORDER BY SP_NAME` clause and you're done.

**SQL carries less file-declaration overhead.** Native I/O ties your program to a specific file structure at compile time. Change the file's record format and you recompile the program. SQL is more loosely coupled.

**Native wins in one specific case.** When your processing needs to follow a *specific* logical or physical path through the file in key order, especially with `READE` (read equal — read records that match a specific key prefix), and when you have tight performance requirements. These cases exist but they're rare in modern business code.

**The pattern is the same, though.** Whichever technique you use, the shape is always: position, get-first, loop-while-more, process, get-next. Internalize that shape and you'll recognize it whether it's written as `SETLL`+`READ` or `DECLARE`+`FETCH`.

## Try this

**Filter to active only.** Modify the program so it reads every record but only counts (and displays) the active ones. Simplest change.

**Break out by supplier-name first letter.** Add a data structure array indexed by A-Z, incrementing the matching letter's counter as you read each record. After the loop, display a mini-histogram. Tests your understanding of arrays and DS arrays from [Part 2 Chapter 2]({% link part-2/02-data-structures.md %}).

**Use SETLL with a specific key.** Change `setll *start` to `setll 'I' SUPPLIER`, meaning "position at the first supplier whose code starts with I or later." Run it. You should get a smaller count. Figure out why that letter produces the specific number you see.

**Read in reverse.** Look up the `READP` opcode (read prior). Change the program to count from the end of the file backward. You'll also need `SETGT *END` instead of `SETLL *START`. Does the total match?

## What's next

You've now done sequential reads with native I/O. Tutorial 4 does the same work using SQL cursors — the modern equivalent — and introduces a production-quality pattern that makes multi-row SQL feel clean.

Next: [Tutorial 4: Multi-Row Queries with SQL Cursors]({% link part-3/04-sql-cursors.md %}).
