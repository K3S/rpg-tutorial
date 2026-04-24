---
title: "10. CL — The Glue Language"
parent: "Part 3: Build Real Things"
nav_order: 10
---

# Tutorial 10: CL — The Glue Language

Every tutorial so far has been RPG. But RPG rarely runs in isolation. On IBM i, the thing that *calls* RPG programs — schedules them, sequences them, passes them parameters, overrides their files, logs their output — is **CL (Control Language)**.

If RPG is where the business logic lives, CL is the connective tissue. Batch jobs are CL. Startup sequences are CL. Wrapper programs that accept string parameters and convert them before calling RPG are CL. You can't be productive on IBM i without writing some.

This tutorial teaches enough CL to run your RPG programs from job schedulers and to write the wrapper code real production deployments need.

## What you'll build

A CL program called **CL_REORD** that orchestrates a reorder run. It:

1. Logs the start
2. Overrides the `QPRINT` output to a specific spool file name for easy retrieval
3. Calls REORDRPT (the batch program from Tutorial 7)
4. Queries the row count from REORDCND via an RPG program you'll write alongside
5. Sends a completion message with the count to the operator's message queue
6. Cleans up overrides
7. Logs the end

The RPG helper, **COUNTROW**, accepts a table name and returns the count back to CL through a parameter. It exists to demonstrate passing data back from RPG to CL.

## What you'll learn

- The anatomy of a CL program — `PGM`, `DCL`, `ENDPGM`
- CL variable declarations and the `&NAME` syntax
- Calling an RPG program from CL with `CALL PGM(...) PARM(...)`
- Converting between CL character variables and decimal (the most common type-mismatch bug)
- `SNDPGMMSG` for logging and operator notifications
- The "CL wrapper around RPG" pattern that real shops use
- Why CL exists and when to reach for it

## Before you start

- Part 1 complete
- REORDRPT from Tutorial 7 compiled and working
- Your practice library available to receive new programs

## Step 1 — Write the RPG helper

Create a new member in `QRPGLESRC` called **COUNTROW**, type **SQLRPGLE**. Paste:

```rpgle
**free
ctl-opt dftactgrp(*no) actgrp(*caller)
        option(*nodebugio:*srcstmt)
        main(Main);
// ************************************************************
// * Name: COUNTROW
// * Type: ILE RPG Program (SQLRPGLE)
// * Desc: Returns the row count of a named table via
// *       output parameter. Called from CL.
// * Tutorial: Part 3, Tutorial 10
// ************************************************************

dcl-proc Main;
  dcl-pi *n;
    wk_tableName char(10) const;
    wk_rowCount  packed(9:0);
  end-pi;

  dcl-s sql varchar(200);

  exec sql
    set option commit = *none,
               datfmt = *iso,
               closqlcsr = *endactgrp;

  // Build dynamic SQL because the table name is a parameter.
  // Note: parameterizing object names (like table names) isn't
  // possible in standard parameterized SQL — it only works for
  // values in WHERE clauses. So we construct the statement.
  sql = 'select count(*) from ' + %trim(wk_tableName);

  exec sql prepare countStmt from :sql;

  if sqlstate <> '00000';
    wk_rowCount = -1;
    return;
  endif;

  exec sql execute countStmt into :wk_rowCount;

  if sqlstate <> '00000';
    wk_rowCount = -1;
  endif;

  return;
end-proc;
```

Compile with **CRTSQLRPGI**.

**A note on the output parameter.** The PI declares `wk_rowCount packed(9:0)` with no `const` — that's the key. Without `const`, the parameter is passed by reference and the RPG program can write to it. The caller (in this case, CL) reads it back. This is how you return scalar data from an RPG program that doesn't have a return value.

## Step 2 — Write the CL wrapper

In your `QCLLESRC` source file (same pattern as `QRPGLESRC` — create it with `CRTSRCPF FILE(YOUR_LIBRARY/QCLLESRC) RCDLEN(112)` if it doesn't exist), create a new member called **CL_REORD**, type **CLLE**. Paste:

```
/*******************************************************************/
/*                                                                 */
/*  Name: CL_REORD                                                 */
/*  Type: CL Program                                               */
/*  Desc: Wrapper for REORDRPT batch run. Overrides output,        */
/*        calls the batch, reports the row count.                  */
/*  Tutorial: Part 3, Tutorial 10                                  */
/*                                                                 */
/*******************************************************************/

PGM

   DCL        VAR(&ROWCOUNTD) TYPE(*DEC) LEN(9 0)
   DCL        VAR(&ROWCOUNTC) TYPE(*CHAR) LEN(10)
   DCL        VAR(&TABLENAME) TYPE(*CHAR) LEN(10) VALUE('REORDCND')

   /* Log start */
   SNDPGMMSG  MSG('CL_REORD: Starting reorder run') +
                TOPGMQ(*EXT) MSGTYPE(*INFO)

   /* Run the RPG batch program */
   CALL       PGM(REORDRPT)

   /* Ask the helper for the row count */
   CALL       PGM(COUNTROW) PARM(&TABLENAME &ROWCOUNTD)

   /* Convert decimal count to character for the message */
   CHGVAR     VAR(&ROWCOUNTC) VALUE(&ROWCOUNTD)

   /* Report the result */
   SNDPGMMSG  MSG('CL_REORD: ' *CAT &ROWCOUNTC *CAT ' +
                   reorder candidates written') +
                TOPGMQ(*EXT) MSGTYPE(*INFO)

   /* Log end */
   SNDPGMMSG  MSG('CL_REORD: Reorder run complete') +
                TOPGMQ(*EXT) MSGTYPE(*INFO)

FINAL:
ENDPGM
```

Compile with **CRTBNDCL** (the compile action labeled "CLLE json V7R3 CRTBNDCL from IFS source" in the VS Code actions config from Part 1).

## Walkthrough

### PGM and ENDPGM

Every CL program is bracketed by `PGM` at the top and `ENDPGM` at the bottom. Between them, the program runs top to bottom. CL has no free-format equivalent — it's always this shape, always column-oriented-ish.

If the program takes parameters, the PGM line lists them: `PGM PARM(&PROD &QTY)`. Our program takes no parameters.

### CL variables

```
DCL VAR(&ROWCOUNTD) TYPE(*DEC) LEN(9 0)
DCL VAR(&ROWCOUNTC) TYPE(*CHAR) LEN(10)
DCL VAR(&TABLENAME) TYPE(*CHAR) LEN(10) VALUE('REORDCND')
```

- `DCL` — declares a variable
- `VAR(&NAME)` — the variable name, always prefixed with `&`
- `TYPE(*DEC)` or `TYPE(*CHAR)` or `TYPE(*LGL)` (logical/boolean)
- `LEN(n)` for character (chars) or `LEN(n d)` for decimal (digits, decimal positions)
- `VALUE(...)` is optional — sets an initial value

The `&` prefix is *always* required. CL looks for `&ROWCOUNTD` as a variable name but `ROWCOUNTD` as a literal string. Forgetting the `&` is the most common CL bug for newcomers.

### The RPG call

```
CALL PGM(REORDRPT)
```

Direct call, no parameters. The program runs, does its work (prints its own DSPLY messages), returns.

### The parameterized call

```
CALL PGM(COUNTROW) PARM(&TABLENAME &ROWCOUNTD)
```

`PARM(...)` lists the parameters in order, space-separated. `&TABLENAME` is passed in (its value is 'REORDCND'). `&ROWCOUNTD` is passed by reference — COUNTROW writes to it, and after the call, `&ROWCOUNTD` contains the count.

**The type matching is critical.** CL's `&ROWCOUNTD` is `*DEC LEN(9 0)`. COUNTROW's PI declares `wk_rowCount packed(9:0)`. Same type, same precision. **If they mismatch — say, COUNTROW declared `packed(11:0)` while CL declared `*DEC LEN(9 0)` — the values received and stored will be silently wrong.** The compiler can't check cross-program, so consistency is on you.

**This is the #1 source of production bugs in CL-to-RPG integration.** Always match types exactly.

### Decimal to character conversion

```
CHGVAR VAR(&ROWCOUNTC) VALUE(&ROWCOUNTD)
```

`CHGVAR` changes the value of a variable. When the source and target are different types, CL does the conversion automatically. Here it takes the decimal `&ROWCOUNTD` and converts it to the character `&ROWCOUNTC`.

**Why two variables?** Because `SNDPGMMSG`'s MSG parameter is a character string, not a decimal. You can't directly concatenate a decimal into a message. So the pattern is: compute in decimal (that's how RPG returned it), convert to character (for the message), send.

### SNDPGMMSG

```
SNDPGMMSG  MSG('CL_REORD: Starting reorder run') +
             TOPGMQ(*EXT) MSGTYPE(*INFO)
```

`SNDPGMMSG` is "send program message." It's the main way CL programs log events.

- `MSG('...')` — the message text (up to 512 chars)
- `TOPGMQ(*EXT)` — the external message queue, which is visible to the user running the job. You'll also see `*PRV` (previous program in the call stack) and explicit queues like `TOPGMQ(*EXT) PGMQ(QSYS)`.
- `MSGTYPE(*INFO)` — informational. Others: `*DIAG` (diagnostic), `*COMP` (completion), `*ESCAPE` (escape, which halts callers).

The `+` at the end of a line is CL's line-continuation marker. CL is fussy about line breaks — you can't just continue a statement on the next line without it.

### The *CAT operator

```
MSG('CL_REORD: ' *CAT &ROWCOUNTC *CAT ' reorder candidates written')
```

`*CAT` concatenates strings with trailing spaces preserved. `*BCAT` strips trailing blanks and adds one space. `*TCAT` preserves everything including trailing blanks with no space.

This tutorial uses `*CAT` because `&ROWCOUNTC` is fixed-length char(10) with leading zeros (e.g., `'0000000013'`), and we keep it simple by not trimming it. In real code you'd probably use `*BCAT` or a different conversion to get `'13'` without the leading zeros. (See "Try this" below.)

## Run it

From the VS Code IBM i terminal or a 5250 session:

```
CALL CL_REORD
```

You should see:

1. The REORDRPT DSPLYs fire (starting, scanning, summary)
2. Messages in your job log:
   - `CL_REORD: Starting reorder run`
   - `CL_REORD: 0000000013 reorder candidates written`
   - `CL_REORD: Reorder run complete`

If you check the job log with `DSPJOBLOG` or `DSPMSG`, you'll see the sequence.

## Why CL exists

A reasonable question: why isn't this RPG code? RPG has `EXEC SQL`, it can display messages, it can be the orchestrator.

Three real answers:

**History.** CL predates RPG's modernization. For thirty years, CL was the only reasonable way to do job-level control. Shops have thousands of CL programs. Skipping CL means not being able to read the codebase.

**Job-level operations.** CL has native commands for things RPG can't do easily: overriding files (`OVRDBF`), adding library entries (`ADDLIBLE`), allocating objects (`ALCOBJ`), monitoring for messages (`MONMSG`), managing spool files, setting job attributes. Doing these from RPG is possible but awkward. CL is the natural layer.

**Composition with system services.** Job schedulers, command menus, and operator tooling all speak CL. Your batch job runs under a `SBMJOB` that calls a CL program. Your nightly reporting runs through `WRKJOBSCDE` entries that call CL programs. You can call RPG directly, but the wrapper pattern keeps the ceremony separate from the business logic.

Practical rule: **RPG for the computation, CL for the orchestration.** A nightly run might be a CL program that sets library lists, overrides printer files, calls three RPG batch programs in sequence, and cleans up. Each RPG program focuses on its logic; the CL owns the plumbing.

## Try this

**Strip leading zeros.** The message shows `0000000013` instead of `13`. Modify CL_REORD to remove leading zeros. Hint: CL has no `%trim` — you'll need to either convert differently or use a small RPG helper. Which feels cleaner?

**Error handling with MONMSG.** CL's equivalent of `monitor` is `MONMSG`. Add `MONMSG MSGID(CPF0000) EXEC(GOTO FINAL)` right after the `CALL REORDRPT` line. Now if the RPG crashes, the CL jumps to the `FINAL:` label and exits cleanly instead of propagating the error to the caller.

**OVRDBF for testing.** Before the CALL, add `OVRDBF FILE(PRODUCT) TOFILE(PRODUCT_TEST)`. After the CALL (in FINAL), add `DLTOVR FILE(PRODUCT)`. Now REORDRPT would read from PRODUCT_TEST instead of PRODUCT when called through this CL — useful for running the batch against a test copy of the data without modifying the RPG source. This is override-based testing, a common IBM i pattern.

**Parameterize the threshold.** Modify STALEPR from Tutorial 9 and CL_REORD so that CL_REORD can run STALEPR with a specific threshold before the reorder runs. The CL accepts a threshold parameter and passes it to STALEPR. Practice accepting parameters and passing them through.

**Schedule it.** Use `WRKJOBSCDE` to add a job schedule entry that runs CL_REORD daily at 2 AM. This is how batch jobs actually get into production. Don't forget to remove it when you're done testing.

## What's next

Wave 3 is done — operational quality tools (error handling, date math, CL orchestration) are in place. Wave 4 closes out Part 3 with the last two tutorials:

- **Tutorial 11** — the Roman numeral converter, a capstone that uses procedures for validation and conversion in an unexpectedly realistic way
- **Tutorial 12** — RPGUnit, so you can write tests for all the programs in Part 3

Next: *Tutorial 11: Roman Numeral Converter — A Procedures Capstone (coming in Wave 4)*.
