---
title: "2. Reading Records with CHAIN"
parent: "Part 3: Build Real Things"
nav_order: 2
---

# Tutorial 2: Reading Records with CHAIN

Tutorial 1 used embedded SQL to look up a single record. This tutorial does the same kind of work using **Record Level Access** (RLA, also called native I/O) — specifically, the `CHAIN` operation.

[Part 2 Chapter 6]({% link part-2/06-sql-vs-rla.md %}) laid out when each tool is the right choice. RLA wins when you're doing record-at-a-time work — positioning at a specific record by key, working with it, possibly updating in place. CHAIN is the random-access read that anchors that pattern. This tutorial gets you fluent in it.

## What you'll build

A program called **SUPINFO** that takes a supplier code as a parameter, uses `CHAIN` to look the record up directly by key, and displays the supplier's name, active status, and onboarding date. If the supplier code doesn't exist, it tells the user and exits cleanly.

Functionally, this is the same thing the `exec sql ... select ... where ...` lookup in Tutorial 1 did. That's the point — you'll see the two approaches side by side and understand when each is the right tool.

## What you'll learn

- Declaring a file for native I/O with `dcl-f`
- Using `CHAIN` to perform a random-access read by key
- Checking `%found` to see whether the key was located
- Using `likerec` to declare a data structure that matches a file's record format
- Why CHAIN is the right tool for "give me this specific record"

## Before you start

- Part 1 complete — SUPPLIER, PRODUCT, and REORDCND exist with sample data
- [Part 2 Chapter 2]({% link part-2/02-data-structures.md %}) read — this tutorial uses `likerec`
- [Part 2 Chapter 6]({% link part-2/06-sql-vs-rla.md %}) read — this tutorial demonstrates the RLA half of that distinction
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

Note that there's no loop, no cursor, no open/close. `CHAIN` is the native random-access primitive — it's a single operation that says "go get the record at this key." That's the access pattern RLA was designed for, and it's why the operation is so direct.

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

`SP_ONBOARD` is a `date` field. `%char` converts it to its string representation — the default format on IBM i is `*iso` (`YYYY-MM-DD`). [Part 2 Chapter 8]({% link part-2/08-date-math.md %}) covers date-to-string conversion in detail.

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

## CHAIN in context

You've now done a single-record lookup two ways — Tutorial 1 with embedded SQL, this tutorial with CHAIN. Both work. Both are fine for this specific case.

Where CHAIN really earns its keep is the pattern Part 2 Chapter 6 described: **record-at-a-time work**. When you're going to read a record, examine it, possibly update it, and move on to the next one in a tight loop, CHAIN (and its loop-friendly cousins READ and READE in Tutorial 3) avoid the per-statement overhead that SQL carries. For one CHAIN, the difference is invisible. For tens of thousands in a batch loop, it's measurable — and that's where shops still reach for native I/O on purpose.

For pure single-row lookup with no loop, like this tutorial, choose based on what's around it. If your program is already doing embedded SQL, use SQL for consistency. If it's a small, native-I/O-flavored program, use CHAIN. There's no wrong answer for the single-lookup case.

The "wrong answer" emerges in batch loops — using SQL for record-at-a-time updates against thousands of records when CHAIN + UPDATE would be cleaner and faster. We'll see that pattern in Tutorial 3 and again in Tutorial 7.

## Try this

**Clean up the "not found" message.** Use `%trim` to drop the trailing spaces from `wk_suplCode` so the message reads `Supplier NOPE999 not found` instead of `Supplier NOPE999    not found`.

**Rewrite this tutorial using embedded SQL instead.** Create a second program, SUPINFO2, that does the same thing using SQLRPGLE and `exec sql select into`. Compare line counts. Compare readability. For a single-row lookup like this, neither is dramatically better — pick what feels right.

**Show the PRODUCT count.** After finding the supplier, do a second lookup — this time with embedded SQL — for `SELECT COUNT(*) FROM PRODUCT WHERE PR_SUPL = :wk_suplCode AND PR_ACTIVE = 'Y'`. Display "Active products: N" as an additional line. You'll need to change the file type to SQLRPGLE to do this. This is a perfect example of mixing the tools — the lookup stays as RLA, the aggregate uses SQL because *aggregating across a set* is what SQL is for. Same program, both tools, each doing what it does best.

**What happens if SUPPLIER isn't in your library list?** Remove it from your library list (temporarily) and try running SUPINFO again. What error do you get? This is worth doing once — it's a common production issue, and knowing what the failure mode looks like means you'll recognize it fast.

## What's next

You've now done random-access reads two ways. Next up: sequential reading — walking through every record in a file, one at a time. That's Tutorial 3, and it's where the speed advantage of native I/O really starts to show.

Next: [Tutorial 3: Sequential Reading with SETLL and READ]({% link part-3/03-sequential-read.md %}).
