---
title: "Reference"
layout: default
nav_order: 6
permalink: /reference/
---

# Reference

A single-page cheat sheet covering the patterns you'll reach for most often. Not exhaustive — the [IBM i RPG Reference](https://www.ibm.com/docs/en/ssw_ibm_i_76/pdf/sc092508.pdf) is the full spec. This page is what to keep open in a second tab while you work.

Jump to:

- [Free-format RPGLE](#free-format-rpgle)
- [Common BIFs](#common-bifs)
- [Embedded SQL essentials](#embedded-sql-essentials)
- [SQLSTATEs worth knowing](#sqlstates-worth-knowing)
- [MONITOR error handling](#monitor-error-handling)
- [Daily-use CL commands](#daily-use-cl-commands)
- [Compile commands](#compile-commands)

---

## Free-format RPGLE

### Source file header

```rpgle
**free
ctl-opt dftactgrp(*no) actgrp(*caller)
        option(*nodebugio:*srcstmt)
        main(Main);
```

- `**free` — must be on line 1, column 1
- `dftactgrp(*no)` — required for modern procedure-based programs
- `actgrp(*caller)` — run in the caller's activation group
- `option(*nodebugio:*srcstmt)` — better debugger line-mapping
- `main(Main)` — declare an explicit main procedure

### Data types

| Type | Syntax | Notes |
|------|--------|-------|
| Fixed-length char | `char(n)` | Pads with spaces |
| Variable-length char | `varchar(n)` | No padding, up to 16 MB |
| Packed decimal | `packed(n:d)` | `n` digits total, `d` decimals. Most common numeric |
| Zoned decimal | `zoned(n:d)` | Legacy |
| Integer | `int(3)`, `int(5)`, `int(10)`, `int(20)` | 1, 2, 4, 8 bytes |
| Date | `date` | Stored internally, compiler-enforced arithmetic |
| Time | `time` | HH:MM:SS |
| Timestamp | `timestamp` | Date + time + microseconds |
| Boolean | `ind` | Values `*on` / `*off` |
| Pointer | `pointer` | Raw pointer, rarely needed |

### Variable declarations

```rpgle
dcl-s simpleVar     packed(9:2);
dcl-s initedVar     int(10) inz(0);
dcl-s sizedArray    char(10) dim(50);
dcl-s constVar      packed(5:2) const inz(3.14);
```

### Data structures

```rpgle
// Always use qualified
dcl-ds item qualified;
  code   char(15);
  name   varchar(50);
  price  packed(9:2);
end-ds;

// Template — no memory allocated, just a shape
dcl-ds itemTemplate qualified template;
  code   char(15);
  name   varchar(50);
end-ds;

// Declare another DS matching the template
dcl-ds anotherItem likeds(itemTemplate);

// DS matching a database table's schema
dcl-ds prodRec extname('PRODUCT') qualified end-ds;

// DS matching a record format
dcl-ds supRec likerec(SP_REC);

// Array of DSes
dcl-ds orderLines qualified dim(100);
  product  char(15);
  quantity packed(7:0);
end-ds;
```

### Control flow

```rpgle
// If / elseif / else
if condition;
  ...
elseif otherCondition;
  ...
else;
  ...
endif;

// Select / when
select;
  when x = 1;
    ...
  when x = 2;
    ...
  other;
    ...
endsl;

// For loop
for i = 1 to 10;        // ascending
  ...
endfor;

for i = 10 downto 1;    // descending
  ...
endfor;

for i = 1 to 100 by 5;  // step
  ...
endfor;

// Do-while (condition checked at top)
dow condition;
  ...
  if earlyExit; leave; endif;   // break
  if skipRest;  iter;  endif;   // continue
enddo;

// Do-until (condition checked at bottom)
dou condition;
  ...
enddo;

// FOR-EACH (iterate array or DS array)
for-each element in myArray;
  ...
endfor;
```

### Procedures

```rpgle
// Prototype — declare at top for forward reference
dcl-pr CalcPrice packed(11:2);
  qty   packed(7:0) const;
  price packed(9:2) const;
end-pr;

// Definition — the body
dcl-proc CalcPrice;
  dcl-pi *n packed(11:2);
    qty   packed(7:0) const;
    price packed(9:2) const;
  end-pi;

  return qty * price;
end-proc;

// Exported from service program
dcl-proc GetPrice export;
  ...
end-proc;

// Parameter keywords
const              // read-only, can't be modified
value              // pass by value (copy)
options(*nopass)   // optional parameter, check with %parms()
options(*omit)     // allow *OMIT placeholder
options(*varsize)  // accept variable-size strings
```

### Indicators

```rpgle
dcl-s isActive ind;

isActive = *on;
if isActive;
  ...
endif;

// Toggle
isActive = not isActive;

// Assign from comparison
isActive = (status = 'A');
```

---

## Common BIFs

Grouped by purpose.

### Strings

```rpgle
%trim(s)              // trim leading and trailing spaces
%trimr(s)             // trim trailing (right) only
%triml(s)             // trim leading (left) only
%len(s)               // length of value (for varchar) or declared (for char)
%subst(s : pos : len) // substring; len optional (to end if omitted)
%scan(needle : haystack)         // position of first match, 0 if not found
%scanrpl(old : new : s)          // replace all occurrences
%upper(s)             // convert to uppercase
%lower(s)             // convert to lowercase
%replace(new : s : pos : len)    // replace a section
%xlate(from : to : s)            // character-by-character translate
```

### Type conversion

```rpgle
%char(value)                 // anything to character
%char(date : *iso)           // date formatted as ISO
%int(s)                      // char or numeric to integer
%float(s)                    // to floating-point
%dec(s : digits : decimals)  // to packed decimal, with precision
%date(s : *iso)              // string to date
%time(s : *iso)              // string to time
%timestamp(s)                // string to timestamp
```

### Numeric

```rpgle
%abs(n)                // absolute value
%rem(a : b)            // remainder (modulo)
%div(a : b)            // integer division (like / but returns int)
%int(n)                // truncate to integer
%round(n : places)     // round half-up to decimals
```

### Dates

```rpgle
%date()                           // today
%time()                           // current time
%timestamp()                      // now
%days(n)                          // duration: n days
%weeks(n)                         // n weeks
%months(n)                        // n months
%years(n)                         // n years
%hours(n), %minutes(n), %seconds(n)
%diff(d1 : d2 : *days)            // difference in days (d1 - d2)
%subdt(dateValue : *d)            // extract day of month
%subdt(dateValue : *m)            // extract month
%subdt(dateValue : *y)            // extract year
```

### File and SQL status

```rpgle
%found(FILENAME)    // last CHAIN/SETLL found a match
%eof(FILENAME)      // last READ hit end of file
%error              // last operation raised an error
%status             // numeric error status

// Inside a monitor block
%status             // current error status code
%error              // short error description text
```

### Arrays

```rpgle
%elem(array)           // number of elements
%lookup(needle : array)              // returns position or 0
%lookup(needle : array : startIdx)   // starting at specific position
%xfoot(numericArray)                 // sum all elements
```

### Parameters and misc

```rpgle
%parms()          // number of parameters passed to current procedure
%addr(var)        // address (pointer) to a variable
```

---

## Embedded SQL essentials

### File extension

Use `.sqlrpgle` for any source with `exec sql`. Not `.rpgle`.

### Standard SQL options (paste at top of every SQL program)

```rpgle
exec sql
  set option commit = *none,
             datfmt = *iso,
             closqlcsr = *endactgrp;
```

### Single-row query

```rpgle
exec sql
  select PR_NAME, PR_PRICE
    into :localName, :localPrice
    from PRODUCT
   where PR_PROD = :inputCode;

if sqlstate <> '00000';
  // handle error
endif;
```

### Cursor for multi-row query

```rpgle
// Declare (doesn't execute)
exec sql
  declare prodCursor cursor for
    select PR_PROD, PR_QOH
      from PRODUCT
     where PR_ACTIVE = 'Y'
     order by PR_PROD;

// Open (executes the query)
exec sql open prodCursor;

// Fetch loop
exec sql fetch next from prodCursor into :prodVar, :qohVar;

dow sqlstate = '00000';
  // process row
  exec sql fetch next from prodCursor into :prodVar, :qohVar;
enddo;

// Close
exec sql close prodCursor;
```

### Fetching into a data structure

```rpgle
dcl-ds prodRec extname('PRODUCT') qualified end-ds;

exec sql
  declare prodCursor cursor for
    select * from PRODUCT where PR_SUPL = :suplCode;

exec sql open prodCursor;
exec sql fetch next from prodCursor into :prodRec;

dow sqlstate = '00000';
  // prodRec.PR_PROD, prodRec.PR_PRICE, etc.
  exec sql fetch next from prodCursor into :prodRec;
enddo;

exec sql close prodCursor;
```

### Insert / Update / Delete

```rpgle
exec sql
  insert into PRODUCT
    (PR_PROD, PR_SUPL, PR_PRICE, PR_ACTIVE, PR_CHGDT)
  values
    (:newCode, :newSupl, :newPrice, 'Y', current_date);

exec sql
  update PRODUCT
     set PR_PRICE = :newPrice,
         PR_CHGDT = current_date
   where PR_PROD = :prodCode;

exec sql
  delete from REORDCND
   where RC_CRTDT < current_date - 30 days;

// Check rows affected
if sqlerrd(3) = 0;
  // no rows were affected — might be worth flagging
endif;
```

### Transactions

```rpgle
exec sql commit;
exec sql rollback;
```

### Dynamic SQL (when you need a table name as variable)

```rpgle
dcl-s sql varchar(500);

sql = 'select count(*) from ' + %trim(tableName);

exec sql prepare stmt from :sql;
exec sql execute stmt into :count;
```

---

## SQLSTATEs worth knowing

The five-character code returned after every SQL statement. First two characters are the class.

### Success and "expected" results

| State | Meaning |
|-------|---------|
| `00000` | Success |
| `01xxx` | Warning (operation completed, but with caveats) |
| `02000` | No row found (SELECT INTO) or no more rows (FETCH) |

### Common errors

| State | Meaning |
|-------|---------|
| `21000` | Result set has more than one row (for SELECT INTO) |
| `22001` | String data right truncation (your target is too small) |
| `22003` | Numeric value out of range |
| `22007` | Invalid date/time format |
| `22012` | Division by zero |
| `23502` | NULL in a NOT NULL column |
| `23505` | Unique constraint violation (duplicate key) |
| `23503` | Foreign key violation |
| `24501` | Cursor is not open |
| `24504` | Cursor is in an invalid state (e.g., positioned after end) |
| `42601` | Syntax error |
| `42703` | Column doesn't exist |
| `42704` | Object doesn't exist (table, etc.) |
| `42802` | Wrong number of values for an INSERT |
| `42818` | Operands of different types |

### Pattern-matching class codes

When you want to branch on error category rather than exact code:

- `22xxx` — data exception (type conversion, range, format)
- `23xxx` — integrity constraint violation
- `42xxx` — syntax or access violation
- `57xxx` — resource not available or operator intervention
- `58xxx` — system error

Branch with a range check:

```rpgle
if %subst(sqlstate : 1 : 2) = '23';
  // any constraint violation
endif;
```

---

## MONITOR error handling

### Basic structure

```rpgle
monitor;
  // risky code
on-error;
  // handle any error
endmon;
```

### Specific error codes

```rpgle
monitor;
  result = numerator / denominator;
on-error *zero-divide;
  result = 0;
on-error *out-of-range;
  result = 9999999999.99;
on-error 00100 : 00199;     // range
  // catches status 00100 through 00199 inclusive
on-error;
  // catch-all
endmon;
```

### Useful named statuses

| Status | Meaning |
|--------|---------|
| `*zero-divide` | Division by zero (00102) |
| `*out-of-range` | Value out of range (00103) |
| `*string-error` | String operation failed (00100) |
| `*decimal-data` | Invalid decimal data (00907) |

### Inspect the error

```rpgle
on-error;
  dsply ('Error ' + %char(%status) + ': ' + %trim(%error));
endmon;
```

### Key rule

`monitor` catches **RPG runtime errors** only. SQL errors set `sqlstate` instead. Check both:

- `monitor` / `on-error` — for `%dec`, divisions, overflows, array bounds, pointers
- `sqlstate` check — for every `exec sql` operation

---

## Daily-use CL commands

### Library and library list

| Command | What it does |
|---------|--------------|
| `WRKLIB` | Work with libraries |
| `CRTLIB` | Create library |
| `DLTLIB` | Delete library |
| `DSPLIBL` | Display library list |
| `ADDLIBLE` | Add library to library list |
| `RMVLIBLE` | Remove library from library list |
| `CHGLIBL` | Change library list (replace entire list) |
| `CHGCURLIB` | Change current library |

### Objects

| Command | What it does |
|---------|--------------|
| `WRKOBJ` | Work with objects |
| `DSPOBJD` | Display object description |
| `CRTDUPOBJ` | Duplicate an object |
| `MOVOBJ` | Move object to a different library |
| `DLTOBJ` | Delete object |
| `RNMOBJ` | Rename object |
| `CHGOBJOWN` | Change object owner |

### Source physical files and members

| Command | What it does |
|---------|--------------|
| `CRTSRCPF` | Create source physical file |
| `WRKMBRPDM` | Work with source members (PDM) |
| `ADDPFM` | Add physical file member |
| `RMVM` | Remove member |
| `CPYSRCF` | Copy source file / members |

### Jobs

| Command | What it does |
|---------|--------------|
| `WRKACTJOB` | Work with active jobs |
| `WRKSBMJOB` | Work with submitted jobs |
| `WRKJOB` | Work with current or specific job |
| `DSPJOB` | Display job details |
| `SBMJOB` | Submit job to batch |
| `HLDJOB` / `RLSJOB` / `ENDJOB` | Hold, release, end job |
| `WRKJOBSCDE` | Work with job schedule entries |
| `ADDJOBSCDE` | Add scheduled job |

### Job log and messages

| Command | What it does |
|---------|--------------|
| `DSPJOBLOG` | Display job log |
| `DSPMSG` | Display messages |
| `SNDPGMMSG` | Send program message |
| `SNDMSG` | Send message to user or queue |
| `WRKMSG` | Work with messages |

### Spool files

| Command | What it does |
|---------|--------------|
| `WRKSPLF` | Work with spool files |
| `DSPSPLF` | Display spool file |
| `CPYSPLF` | Copy spool file to a PF |
| `DLTSPLF` | Delete spool file |

### Files and data

| Command | What it does |
|---------|--------------|
| `DSPFD` | Display file description |
| `DSPFFD` | Display file field descriptions |
| `DSPPFM` | Display physical file member (view data) |
| `CLRPFM` | Clear physical file member (empty data) |
| `RGZPFM` | Reorganize PFM (reclaim deleted-record space) |
| `STRSQL` | Interactive SQL session |
| `RUNSQLSTM` | Run SQL statements from a source member |

### Overrides

| Command | What it does |
|---------|--------------|
| `OVRDBF` | Override database file |
| `OVRPRTF` | Override printer file |
| `OVRDSPF` | Override display file |
| `DSPOVR` | Display active overrides |
| `DLTOVR` | Delete override |

### Authority and users

| Command | What it does |
|---------|--------------|
| `DSPUSRPRF` | Display user profile |
| `WRKUSRPRF` | Work with user profiles |
| `DSPOBJAUT` | Display object authority |
| `GRTOBJAUT` | Grant object authority |
| `RVKOBJAUT` | Revoke object authority |

### Service and diagnostics

| Command | What it does |
|---------|--------------|
| `DSPSRVPGM` | Display service program info (including exports) |
| `DSPMOD` | Display module info |
| `DSPBNDDIR` | Display binding directory |
| `WRKBNDDIR` | Work with binding directories |

---

## Compile commands

### RPG

```
CRTBNDRPG PGM(YOURLIB/YOURPGM)
          SRCFILE(YOURLIB/QRPGLESRC)
          SRCMBR(YOURPGM)
          DBGVIEW(*SOURCE)
          OPTION(*EVENTF)
```

Compiles a bound RPG program. Use for plain `.rpgle` files with no embedded SQL.

### SQLRPGLE

```
CRTSQLRPGI OBJ(YOURLIB/YOURPGM)
           SRCFILE(YOURLIB/QRPGLESRC)
           SRCMBR(YOURPGM)
           OBJTYPE(*PGM)
           COMMIT(*NONE)
           DBGVIEW(*SOURCE)
           OPTION(*EVENTF *XREF)
```

For `.sqlrpgle` — the SQL preprocessor runs first, then CRTBNDRPG internally.

For a module (service program input), change `OBJTYPE(*PGM)` to `OBJTYPE(*MODULE)`.

### CL

```
CRTBNDCL PGM(YOURLIB/YOURPGM)
         SRCFILE(YOURLIB/QCLLESRC)
         SRCMBR(YOURPGM)
         DBGVIEW(*SOURCE)
```

### Service program (two-step)

```
// Step 1: Compile as module
CRTSQLRPGI OBJ(YOURLIB/SVCMOD)
           SRCFILE(YOURLIB/QRPGLESRC)
           SRCMBR(SVCMOD)
           OBJTYPE(*MODULE)
           ...

// Step 2: Create service program from module(s)
CRTSRVPGM SRVPGM(YOURLIB/SVCPGM)
          MODULE(YOURLIB/SVCMOD)
          EXPORT(*ALL)
          ACTGRP(*CALLER)
```

### Binding directories

```
// Create
CRTBNDDIR BNDDIR(YOURLIB/MYBND)

// Add a service program to it
ADDBNDDIRE BNDDIR(YOURLIB/MYBND)
           OBJ((YOURLIB/SVCPGM *SRVPGM))

// Reference in ctl-opt of a consuming program:
// ctl-opt ... bnddir('MYBND');
```

### Logical files from IFS source

CL doesn't have a built-in CRTLF from IFS, but most shops use a utility (like `CRTLFIFS` from the Code for IBM i community). Check the Code for IBM i actions config for the command pattern.

---

## A few final patterns

### The "ok flag" output parameter

Return a value *and* a success indicator — so the caller can distinguish "succeeded with value 0" from "failed, ignore value":

```rpgle
dcl-proc ParseAmount;
  dcl-pi *n packed(9:2);
    input char(20) const;
    ok    ind;
  end-pi;

  dcl-s result packed(9:2);

  monitor;
    result = %dec(%trim(input) : 9 : 2);
    ok = *on;
  on-error;
    ok = *off;
    result = 0;
  endmon;

  return result;
end-proc;
```

### The read-then-loop pattern (native I/O)

```rpgle
setll *start FILE;
read FILE;

dow not %eof(FILE);
  // process current record
  read FILE;
enddo;
```

Priming read before the loop; advancing read at the bottom. Never inside the loop condition.

### The cursor-fetch pattern (SQL)

```rpgle
exec sql declare c1 cursor for ...;
exec sql open c1;
exec sql fetch next from c1 into :vars;

dow sqlstate = '00000';
  // process
  exec sql fetch next from c1 into :vars;
enddo;

exec sql close c1;
```

Same shape as the native pattern. Internalize once, recognize everywhere.

### Tiered conditionals — always check highest tier first

```rpgle
if qty >= 100;
  discount = 0.10;
elseif qty >= 50;
  discount = 0.05;
else;
  discount = 0;
endif;
```

Checking `qty >= 50` first would eat the `>= 100` branch — a common bug. Always check highest threshold first in any tiered logic.

---

*This page is a living reference. If something's missing or unclear, [open a PR](https://github.com/K3S/rpg-tutorial) or suggest edits via the link at the bottom.*
