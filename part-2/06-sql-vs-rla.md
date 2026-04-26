---
title: "6. Choosing Between SQL and Native I/O"
parent: "Part 2: Learn the Language"
nav_order: 6
---

# Choosing Between SQL and Native I/O

Chapter 5 introduced embedded SQL. RPG also has an older, equally valid way of accessing the database: **Record Level Access** (RLA), also called **native I/O**. CHAIN, READ, READE, UPDATE, WRITE, DELETE — operations that work directly against the file at the record level.

Modern teaching often skips RLA, or treats it as legacy code you tolerate but don't write. That framing is wrong, and this chapter is here to correct it. **SQL and RLA are two different tools for two different access patterns**, and a working RPG developer reaches for both.

Get this distinction right early and the rest of Part 3 makes more sense.

## The two access patterns

Most database work falls into one of two shapes.

**Set-based access.** "Give me all active products from this supplier, sorted by code." "Total the value of this warehouse's inventory." "Find every order placed last quarter where the customer is in California." You're describing a *group* of records, defined by criteria, and asking the database to return or aggregate them. The database's optimizer plans the query, uses indexes, and hands you the result.

**Record-at-a-time access.** "Get the supplier whose code is `ACME001`. Update its onboarding date. Move on to the next supplier in alphabetical order. Get its details. Decide whether to update it. Repeat." You're navigating through specific records by key, often updating them in place, often making decisions about each one before moving to the next.

Both are legitimate. Both come up in real business code. The mistake is using one tool for both patterns.

## SQL excels at set-based work

SQL is *declarative*. You describe what you want; the optimizer figures out how. For set-based access this is enormously powerful:

- **Filters and joins** — "active products from suppliers in this region" is one statement
- **Aggregates** — `SUM`, `COUNT`, `AVG`, `GROUP BY` happen at the database
- **Sorting** — `ORDER BY` is essentially free; doing the equivalent in RPG would mean reading everything into memory first
- **Optimizer-driven indexes** — the database picks the best access path
- **Portability** — the same SQL runs against Db2 for i, Db2 LUW, Postgres, anywhere

When a query needs to look at lots of rows, filter or aggregate them, and produce a set of results, SQL is faster, shorter, and clearer. This is the bulk of reporting, analytics, and bulk data movement.

## Native I/O excels at record-at-a-time work

RLA works *imperatively*. You position the file pointer with `SETLL`, you `READ` the next record, you `UPDATE` it, you `READ` the next one. Each operation is a direct call to the file system — no SQL parser, no optimizer, no cursor management.

For record-at-a-time access, this matters in three ways:

**Speed.** SQL has per-statement overhead — statement parsing (even when cached), optimizer engagement, host-variable marshaling. For *one* `select`, the overhead is invisible. For thousands of single-row lookups inside a loop, the overhead adds up. RLA reads from the file's buffer directly, with none of that machinery in the path.

**Updating in place.** When you've just read a record and want to update it, RLA's `UPDATE` operation works on the record you're currently positioned at — no need to identify it again by key. SQL would require an `UPDATE ... WHERE PR_PROD = :prodCode` that re-locates the row.

**Cleaner pointer-style navigation.** "Position at this key, walk forward through matching records, stop when the key changes" maps cleanly to `SETLL` + `READE` (read equal). The same logic in SQL works, but feels less natural.

The speed difference isn't theoretical. For tight loops doing record-at-a-time updates against keyed files, RLA can be an order of magnitude faster than equivalent SQL. Real shops doing nightly batch processing against millions of records *measure* this difference and pick RLA on that basis.

## What this looks like in code

You'll write the working code in Part 3 (Tutorials 2, 3, 4 cover the mechanics). For now, here's the same task in both forms so you can see the shape difference.

**Task: read every supplier in code order, mark inactive ones with a flag.**

In SQL:

```rpgle
exec sql
  declare supCursor cursor for
    select SP_SUPL, SP_ACTIVE from SUPPLIER order by SP_SUPL
    for update of SP_ACTIVE;

exec sql open supCursor;
exec sql fetch next from supCursor into :supCode, :supActive;

dow sqlstate = '00000';
  if SomeCondition(supCode);
    exec sql update SUPPLIER set SP_ACTIVE = 'N'
              where current of supCursor;
  endif;
  exec sql fetch next from supCursor into :supCode, :supActive;
enddo;

exec sql close supCursor;
```

In RLA:

```rpgle
dcl-f SUPPLIER usage(*update) keyed;
dcl-ds supRec likerec(SP_REC : *all);

setll *start SUPPLIER;
read SUPPLIER supRec;

dow not %eof(SUPPLIER);
  if SomeCondition(supRec.SP_SUPL);
    supRec.SP_ACTIVE = 'N';
    update SP_REC supRec;
  endif;
  read SUPPLIER supRec;
enddo;
```

Both work. The RLA version is shorter, has no cursor lifecycle to manage, and runs faster on a tight loop. The SQL version is more flexible — change "every supplier" to "suppliers from a specific region" and it's a one-line WHERE clause; in RLA you'd add an `if` inside the loop.

That tradeoff — RLA's simplicity and speed against SQL's flexibility — is the whole story.

## How to choose

A practical rule that holds up well:

**Start by describing what you're doing in plain language.**

- "I want a *list* of records that meet some criteria" → **SQL**
- "I want to *aggregate* across many records" → **SQL**
- "I want to *join* records from multiple tables" → **SQL**
- "I want to *position* at a specific record and work with it" → **RLA**
- "I want to *walk through* records in key order, possibly updating in place" → **RLA**
- "I want a single record by key, no other logic" → **either**, choose based on what's easier in context

That last case is the genuine gray area. Tutorial 1 used SQL for a single-row lookup; Tutorial 2 will do the same lookup with CHAIN. Both are fine. Choose based on whether the surrounding code is already doing SQL or RLA — consistency within a program matters more than the exact choice.

## A few honest caveats

**Mixing SQL and RLA in the same program is fine.** A common pattern: use SQL to find a small set of candidate records (the set-based filter), then loop through them with RLA to update each one. You're using each tool for what it's best at.

**Don't optimize prematurely.** If your program runs once a day and processes 100 records, the choice between SQL and RLA is invisible — both finish instantly. Reach for RLA when you've measured the loop and it matters, not because you suspect it might.

**Code conventions vary by shop.** Some K3S customers, and many others, have entire codebases built on RLA. Other shops have moved to SQL for almost everything. Both are reasonable. Read whichever style your codebase uses and write new code in the same style unless you have a specific reason to deviate.

**SQL has gotten faster.** Db2 for i's optimizer has improved enormously over the last decade. For many workloads where RLA used to be the obvious choice, SQL is now within a few percent of native speeds — sometimes even faster, when an index helps SQL find a record that RLA would have to scan for. The "SQL is slow" intuition from older shops is increasingly outdated. **Always measure before assuming.**

## When you'll see this distinction in Part 3

- **[Tutorial 2: Reading Records with CHAIN]({% link part-3/02-chain-io.md %})** — RLA random access by key. The single-record-lookup case where RLA shines.
- **[Tutorial 3: Sequential Reading with SETLL and READ]({% link part-3/03-sequential-read.md %})** — RLA's classic walk-the-file pattern.
- **[Tutorial 4: Multi-Row Queries with SQL Cursors]({% link part-3/04-sql-cursors.md %})** — SQL's set-based equivalent. Same task, different tool.
- **[Tutorial 7: Batch Reorder Report]({% link part-3/07-batch-reorder.md %})** — uses SQL to identify candidates (set-based), then writes back. Pure set-based work.

After working through those four tutorials, the choice between tools should feel natural. Until then, the rule "set-based → SQL, record-at-a-time → RLA" will get you 90% of the way there.

Next: [Chapter 7: Error Handling with MONITOR]({% link part-2/07-error-handling.md %}).
