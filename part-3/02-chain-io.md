---
title: "2. Reading Records with CHAIN"
parent: "Part 3: Build Real Things"
nav_order: 2
---

# Tutorial 2: Reading Records with CHAIN

Part 2 Chapter 5 taught embedded SQL, which is how most modern RPG programs access data. But if you work on an IBM i anywhere, you'll encounter **native file I/O** — RPG's older, non-SQL way of reading and writing records. You need to know both. This tutorial teaches the first and most common native operation: `CHAIN`.

## What you'll build

A program called **SUPINFO** that takes a supplier code as a parameter, uses `CHAIN` to look the record up directly by key, and displays the supplier's name, active status, and onboarding date. If the supplier code doesn't exist, it tells the user and exits cleanly.

Functionally, this is the same thing the `exec sql ... select ... where ...` lookup in Tutorial 1 did. That's the point — you'll see the two approaches side by side and understand when each is the right tool.

## What you'll learn

- Declaring a file for native I/O with `dcl-f`
- Using `CHAIN` to perform a random-access read by key
- Checking `%found` to see whether the key was located
- Using `likerec` to declare a data structure that matches a file's record format
- The tradeoffs between native I/O and embedded SQL

## Before you start

- Part 1 complete — SUPPLIER, PRODUCT, and REORDCND exist with sample data
- [Part 2 Chapter 2]({% link part-2/02-data-structures.md %}) read — this tutorial uses `likerec`
- Tutorial 1 completed — understanding program parameters will help here

## The program

Create a new member in `QRPGLESRC` called **SUPINFO**, type **RPGLE** (not SQLRPGLE — this program uses native I/O, no embedded SQL). Paste this in:

```rpgle
**free
ctl-opt dftactgrp(*no) actgrp(*caller) option(*nodebugio:*srcstmt);
// ************************************************************
// * Name: SUPINFO
// * Type: ILE RPG Program
// * Desc: Supplier Info Lookup
// *       Given a supplier code, look up the supplier
// *       in the SUPPLIER file using native CHAIN I/O
// *       and display its details.
// * Tutorial: Part 3, Tutorial 2
// ************************************************************

// Declare the SUPPLIER file for input-only native I/O
dcl-f SUPPLIER usage(*input) keyed;

// Data structure matching the SUPPLIER record format
dcl-ds supRec likerec(SP_REC);

// Program interface — a single supplier code parameter
dcl-pi *n;
  wk_suplCode char(10) const;
end-pi;

// Look the supplier up by key
chain wk_suplCode SUPPLIER supRec;

if not %found(SUPPLIER);

  // Supplier code doesn't exist in SUPPLIER
  dsply ('Supplier ' + %trim(wk_suplCode) + ' not found');

else;

  // Found it — display the details
  dsply ('Supplier: ' + %trim(supRec.SP_SUPL));
  dsply ('Name:     ' + %trim(supRec.SP_NAME));

  if supRec.SP_ACTIVE = 'Y';
    dsply 'Status:   Active';
  else;
    dsply 'Status:   Inactive';
  endif;

  dsply ('Onboard:  ' + %char(supRec.SP_ONBOARD));

endif;

*inlr = *on;
return;
```

## Walkthrough

### Declaring the file

```rpgle
dcl-f SUPPLIER usage(*input) keyed;
```

`dcl-f` tells the compiler this program will do native I/O against a file. `usage(*input)` means read-only — we won't write, update, or delete records. `keyed` means we want keyed access (so `CHAIN` can work by key), rather than arrival-sequence access.

The file name — `SUPPLIER` — is the actual table name. RPG resolves it through your library list at runtime.

One catch worth knowing: the compiler needs to *find* SUPPLIER at compile time to read its schema. If you get a "file not found" error during compile, your compile-time library list probably doesn't include your practice library. The Code for IBM i extension lets you set the library list in the connection settings.

### The record-format data structure

```rpgle
dcl-ds supRec likerec(SP_REC);
```

When you read a record with native I/O, RPG populates the file's fields — `SP_SUPL`, `SP_NAME`, `SP_ACTIVE`, `SP_ONBOARD` — as global variables. That works, but pollutes your program's namespace and makes it hard to track where values came from.

`likerec(SP_REC)` declares a data structure whose shape matches SUPPLIER's record format (named `SP_REC` in the DDL from Part 1 Chapter 6). Reading into it gives you `supRec.SP_SUPL`, `supRec.SP_NAME`, and so on — qualified access, cleaner code.

The qualified-DS habit from [Part 2 Chapter 2]({% link part-2/02-data-structures.md %}) applies to file I/O too. Use it.

### The CHAIN operation

```rpgle
chain wk_suplCode SUPPLIER supRec;
```

Three parts:

- **The key value** — here, `wk_suplCode`. This is what we're looking up.
- **The file** — `SUPPLIER`. The file we declared above.
- **Optional target DS** — `supRec`. Where the found record lands.

`CHAIN` does one thing: "find the record whose key matches this value, and give it to me." If found, the target DS is populated and `%found(SUPPLIER)` returns `*on`. If not found, the target DS is cleared and `%found(SUPPLIER)` returns `*off`.

Note that there's no loop, no cursor, no open/close. `CHAIN` is the native random-access primitive. It's the direct equivalent of `SELECT ... INTO ... WHERE key = ...` in SQL, but without the SQL preprocessing step.

### The found check

```rpgle
if not %found(SUPPLIER);
  ...
else;
  ...
endif;
```

Always check `%found` immediately after a `CHAIN`. It's the only way to distinguish "I got a record" from "there was no such record." The value is valid only until the next I/O operation — don't read another record and then check, you'll get the result for the later read.

This matches the pattern from Tutorial 1 where we checked `sqlstate = '02000'` after a `SELECT INTO`. Same idea, different API.

### Displaying the date

```rpgle
dsply ('Onboard:  ' + %char(supRec.SP_ONBOARD));
```

`SP_ONBOARD` is a `date` field. `%char` converts it to its string representation — the default format on IBM i is `*iso` (`YYYY-MM-DD`). [Part 2 Chapter 7]({% link part-2/07-date-math.md %}) covered date-to-string conversion.

## Compile and run

### Compile

Right-click the member, choose Compile, pick **CRTBNDRPG** (not CRTSQLRPGI — this program has no embedded SQL).

### Run

```
CALL SUPINFO PARM('ACME001')
```

You should see:

```
DSPLY  Supplier: ACME001
DSPLY  Name:     Acme Distributors
DSPLY  Status:   Active
DSPLY  Onboard:  2018-03-15
```

Try an inactive supplier:

```
CALL SUPINFO PARM('DLTA004')
```

Should show `Status:   Inactive`.

Try a code that doesn't exist:

```
CALL SUPINFO PARM('NOPE999')
```

Should show `Supplier NOPE999   not found`. (Notice the trailing spaces — `wk_suplCode` is `char(10)`, padded with spaces. A `%trim` in the message would clean that up, and you can add it as an extension below.)

## CHAIN vs embedded SQL — when to use which

You've now looked up a record two ways:

- **Tutorial 1** — `exec sql select ... from PRODUCT where PR_PROD = :wk_productCode`
- **Tutorial 2** — `chain wk_suplCode SUPPLIER supRec`

Both do the same fundamental thing. Knowing when to reach for each is worth a few honest paragraphs.

**CHAIN is simpler for pure "give me one record by key."** No preprocessor step, no SQLSTATE to check, no host variables. Also: compile is faster (no CRTSQLRPGI), and the runtime is marginally faster because there's no SQL plan.

**Embedded SQL is better when you need anything else.** Joining two tables, filtering by a non-key column, computing an aggregate, selecting only some columns, using a subquery — CHAIN can't do any of that. CHAIN also can't handle "give me all the matching records" — for that you'd need `SETLL` + `READE` (covered in Tutorial 3) or a cursor (Tutorial 4).

**Modern shops lean SQL.** Over the last decade, most new RPG code uses embedded SQL for almost everything, even simple lookups, because consistency is valuable. If every data access in your codebase is SQL, there's one mental model to keep in your head. The legacy code you'll inherit has plenty of CHAIN, so you need to read it — but when you're writing new code, SQL is usually the default.

One situation where CHAIN genuinely wins: inside a tight loop where you're chaining on millions of records by key. The SQL overhead can matter at scale. That's a real but narrow case.

## Try this

**Clean up the "not found" message.** Use `%trim` to drop the trailing spaces from `wk_suplCode` so the message reads `Supplier NOPE999 not found` instead of `Supplier NOPE999    not found`.

**Rewrite this tutorial using embedded SQL instead.** Create a second program, SUPINFO2, that does the same thing using SQLRPGLE and `exec sql select into`. Compare line counts. Compare readability. Which one do you prefer?

**Show the PRODUCT count.** After finding the supplier, do a second lookup — this time with embedded SQL — for `SELECT COUNT(*) FROM PRODUCT WHERE PR_SUPL = :wk_suplCode AND PR_ACTIVE = 'Y'`. Display "Active products: N" as an additional line. You'll need to change the file type back to SQLRPGLE to do this, which is itself a useful exercise in "when does a program become SQL."

**What happens if SUPPLIER isn't in your library list?** Remove it from your library list (temporarily) and try running SUPINFO again. What error do you get? This is worth doing once — it's a common production issue, and knowing what the failure mode looks like means you'll recognize it fast.

## What's next

You've now done random-access reads two ways. Next up: sequential reading — walking through every record in a file, one at a time. That's Tutorial 3.

Next: [Tutorial 3: Sequential Reading with SETLL and READ]({% link part-3/03-sequential-read.md %}).
