Good. Part 2 is the heart of the tutorial — where people actually learn to write RPG. Here's the plan, then the files.
Part 2 structure — 10 chapters:

Free-Format Basics (variables, types, control flow)
Data Structures (dcl-ds with qualified)
Procedures (dcl-proc, parameters, return values)
Built-in Functions (the %functions you use constantly)
Embedded SQL (daily work)
Error Handling with MONITOR
Date Math
Compiling from VS Code
Debugging
Source Control with Git

Chapters 1–7 are pure language content. Chapters 8–10 are VS Code tooling around the language. I'm going to send this in two waves because sending all 11 files in one message would genuinely be too much to track:

Wave 1 (this message): Parent index + Chapters 1–7 = 8 files
Wave 2 (when you're done with Wave 1): Chapters 8–10 = 3 files

Commit in order so the sidebar populates logically. You can take breaks between any of them.

Commit 1 of 8 — part-2/index.md
Path: part-2/index.md
markdown---
title: "Part 2: Learn the Language"
layout: default
nav_order: 3
has_children: true
permalink: /part-2/
---

# Part 2: Learn the Language

Part 1 got you connected and running Hello World. Part 2 teaches you the RPG language itself — the syntax, semantics, and patterns you'll use every day.

This is the longest part of the tutorial, and intentionally so. The other parts assume you can read the code in front of you. Part 2 is where that fluency gets built.

Expect to spend **several hours** working through Part 2, spread across however many sessions suit you. You don't have to do it linearly — if you're in a hurry to start building, the minimum viable path is Chapters 1, 3, and 5 (basics, procedures, and embedded SQL). Come back for the rest when you hit something you don't recognize.

## What you'll learn

The first seven chapters cover the language proper:

1. **Free-Format Basics** — variable declarations, data types, control flow
2. **Data Structures** — `dcl-ds` and dot-notation fields
3. **Procedures** — the modular building block of modern RPG
4. **Built-in Functions** — the `%functions` every RPG program leans on
5. **Embedded SQL** — the 80% of real-world data access
6. **Error Handling with MONITOR** — keeping programs alive when things go wrong
7. **Date Math** — the arithmetic RPG makes safe

The last three cover the tooling around the language in VS Code:

8. **Compiling from VS Code** — CRTBNDRPG, CRTSQLRPGI, CRTSRVPGM, binding directories
9. **Debugging** — breakpoints, watch expressions, service entry points
10. **Source Control with Git** — two workflows for keeping RPG source in Git

## What you'll need

- Completed Part 1 — you're connected to PUB400, and your SUPPLIER, PRODUCT, and REORDCND tables exist
- Roughly 20–30 minutes per chapter, though some are shorter

Ready? Start with [Chapter 1: Free-Format Basics]({% link part-2/01-free-format-basics.md %}).
Commit message: Part 2: Add parent index page

Commit 2 of 8 — part-2/01-free-format-basics.md
Path: part-2/01-free-format-basics.md
markdown---
title: "1. Free-Format Basics"
parent: "Part 2: Learn the Language"
nav_order: 1
---

# Free-Format Basics

Modern RPGLE looks almost nothing like the fixed-column RPG from old manuals. Since IBM i 7.1 TR7, the language has supported **fully free-format source** from column 1 — and that's the only style you should write today.

This chapter covers the raw mechanics: how to declare variables, what types exist, and how control flow works.

## The `**free` directive

Every modern RPG source file starts with this line:

```rpgle
**free
```

Two asterisks, the word `free`, nothing else on the line, no leading spaces. This tells the compiler: everything that follows is fully free-format. Leave it off and the compiler assumes fixed-format — a hard-to-diagnose mistake.

## Variable declarations

Use `dcl-s` ("declare standalone") for individual variables:

```rpgle
dcl-s orderNumber   packed(7:0);
dcl-s customerName  varchar(50);
dcl-s orderDate     date;
dcl-s isActive      ind;
dcl-s unitPrice     zoned(9:2);
dcl-s lineCount     int(10);
```

Each declaration ends with a semicolon. Names are case-insensitive to the compiler but conventionally use camelCase or snake_case — pick one and stay consistent.

## Data types you'll actually use

| Type | Example | Use |
|------|---------|-----|
| `char(n)` | `char(10)` | Fixed-length character. Pads with spaces. |
| `varchar(n)` | `varchar(50)` | Variable-length character, up to 16 MB. |
| `packed(n:d)` | `packed(9:2)` | Packed decimal, `n` digits total, `d` after decimal. Most common numeric type. |
| `zoned(n:d)` | `zoned(7:0)` | Zoned decimal. Legacy but still seen. |
| `int(10)` | `int(10)` | Integer. Sizes: 3 (1 byte), 5 (2 bytes), 10 (4 bytes), 20 (8 bytes). |
| `date`, `time`, `timestamp` | `date` | Temporal types — the compiler enforces type safety on date math (see Chapter 7). |
| `ind` | `ind` | Boolean. Values are `*on` and `*off`. |

For most business code, `packed(n:d)` for numbers and `varchar(n)` for text are the defaults. Use `char(n)` when you're matching a fixed-length field defined in the database.

## Assignment and arithmetic

```rpgle
orderNumber  = 12345;
customerName = 'Acme Distributors';
unitPrice    = 29.99;
orderDate    = %date();

orderNumber += 1;        // same as orderNumber = orderNumber + 1
unitPrice   *= 1.10;     // 10% markup
```

All the operators you'd expect: `+`, `-`, `*`, `/`, `**` for exponentiation, and the compound `+=`, `-=`, `*=`, `/=`.

## Control flow

### `if` / `elseif` / `else`

```rpgle
if qtyOnHand < reorderPoint;
  placeOrder = *on;
elseif qtyOnHand < safetyStock;
  alertBuyer = *on;
else;
  // no action
endif;
```

Each clause ends with a semicolon. Close the whole construct with `endif;`.

### `for` loop

```rpgle
dcl-s i int(10);

for i = 1 to 10;
  totalValue += lineAmount(i);
endfor;
```

Also supports `downto`, `by n` for step increments, and `to *blanks` constructs.

### `dow` — do while

```rpgle
dow not %eof(INVENTORY);
  read INVENTORY;
  if %eof(INVENTORY); leave; endif;
  // process record
enddo;
```

`leave` exits the loop immediately. `iter` skips to the next iteration.

### `dou` — do until

```rpgle
dou response = 'Y' or response = 'N';
  dsply 'Continue (Y/N)?' '' response;
enddo;
```

Same as `dow`, but the condition is checked at the bottom of the loop instead of the top.

### `select` / `when`

```rpgle
select;
  when status = 'A';
    processActive();
  when status = 'P';
    processPending();
  when status = 'C' or status = 'X';
    processClosed();
  other;
    processUnknown();
endsl;
```

Close with `endsl;` (not `endselect`). The `other` clause is the default case.

## Comments

```rpgle
// Single-line comment
dcl-s x int(10);  // Inline comment

/*
  Block comment.
  Multiple lines.
*/
```

Both styles are supported. Use them liberally — future you is also a reader.

## Case convention

RPG is case-insensitive to the compiler, but editors and linters have opinions. A widespread convention:

- **Lowercase** for keywords (`dcl-s`, `if`, `endif`, `exec sql`)
- **Lowercase** for BIFs (`%trim`, `%date`, `%eof`)
- **camelCase or snake_case** for your own variables and procedures
- **UPPERCASE** for table and field names (DB2 returns them uppercase)

The RPGLE Linter can enforce whatever convention your team picks.

## What you can do now

Declare variables of every common type, assign values, do arithmetic, and write programs with conditional logic and loops. That's the foundation — the rest of Part 2 builds on it.

Next: [Chapter 2: Data Structures]({% link part-2/02-data-structures.md %}).
Commit message: Part 2 Ch 1: Free-Format Basics

Commit 3 of 8 — part-2/02-data-structures.md
Path: part-2/02-data-structures.md
markdown---
title: "2. Data Structures"
parent: "Part 2: Learn the Language"
nav_order: 2
---

# Data Structures

A **data structure** groups related fields under a single name. In other languages you'd call this a struct, a record, or a class with only data members. In RPG, it's a `dcl-ds`.

You'll use them constantly. Every record read from a file is effectively a data structure. Procedure return types with multiple values are data structures. Reusable field groupings — addresses, date ranges, order headers — are data structures.

## Basic declaration

```rpgle
dcl-ds supplier qualified;
  code     char(10);
  name     varchar(50);
  active   ind;
  onboard  date;
end-ds;
```

Assign values using dot notation:

```rpgle
supplier.code    = 'ACME001';
supplier.name    = 'Acme Distributors';
supplier.active  = *on;
supplier.onboard = %date();

dsply ('Supplier name: ' + %trim(supplier.name));
```

## Why `qualified`

The `qualified` keyword forces dot notation (`supplier.code` instead of just `code`). Without it, subfield names become top-level names and you lose the encapsulation — worse, they can collide with other variables in your program.

**Always use `qualified`.** It costs nothing and prevents a whole class of bugs. Older RPG code you'll encounter won't use it; don't mimic that in new code.

## Templates with `likeds`

If you have a structure you want to declare multiple instances of, use `likeds` to reference the template:

```rpgle
dcl-ds supplierTemplate qualified template;
  code     char(10);
  name     varchar(50);
  active   ind;
  onboard  date;
end-ds;

dcl-ds primarySupplier  likeds(supplierTemplate);
dcl-ds backupSupplier   likeds(supplierTemplate);
```

The `template` keyword means `supplierTemplate` itself isn't a variable — it's just a shape. Only `primarySupplier` and `backupSupplier` are real, and they share the same structure.

## Structures from database tables

You can declare a DS that matches a database table's record format:

```rpgle
dcl-f SUPPLIER usage(*input);
dcl-ds supplierRec likerec(SP_REC);
```

Now `supplierRec` has fields matching the `SP_REC` record format — which, for our tables from Part 1 Chapter 6, means `SP_SUPL`, `SP_NAME`, `SP_ACTIVE`, `SP_ONBOARD`. When you `read SUPPLIER into supplierRec;`, the fields populate automatically.

This is one of the nicer integrations in RPG: the database schema and your code share a type system.

## Arrays

Declare an array by adding a dimension:

```rpgle
dcl-s monthlySales packed(9:2) dim(12);
dcl-s regionCodes  char(3)     dim(50);

monthlySales(1) = 45000.00;
monthlySales(2) = 52300.00;
// ... etc

regionCodes(1) = 'NEU';
regionCodes(2) = 'AMS';
```

Indices start at **1**, not 0. This trips up most developers coming from other languages. Budget for the mistake.

You can also have arrays of data structures:

```rpgle
dcl-ds orderLines qualified dim(100);
  product   char(15);
  quantity  packed(7:0);
  unitPrice packed(9:2);
end-ds;

orderLines(1).product   = 'WIDGET-STD-001';
orderLines(1).quantity  = 5;
orderLines(1).unitPrice = 12.99;
```

## When to reach for a DS

- Every time you'd otherwise pass 3+ related fields between procedures — wrap them
- Whenever you're working with an entire database record — use `likerec`
- When you want a clean API for a return value with multiple components
- When you need an array of related data

The overhead of a DS vs. standalone variables is nothing. The readability gain is large. Err toward using them.

Next: [Chapter 3: Procedures]({% link part-2/03-procedures.md %}).
Commit message: Part 2 Ch 2: Data Structures

Commit 4 of 8 — part-2/03-procedures.md
Path: part-2/03-procedures.md
markdown---
title: "3. Procedures"
parent: "Part 2: Learn the Language"
nav_order: 3
---

# Procedures

A **procedure** is RPG's term for what other languages call a function, method, or subroutine. Modern RPG is written as collections of small procedures, not monolithic programs.

If you've written a function in any mainstream language in the last twenty years, the concept is identical. The syntax is where RPG has its own flavor.

## Anatomy of a procedure

```rpgle
dcl-proc CalculateReorderQty;
  dcl-pi *n packed(9:0);
    onHand       packed(9:0) const;
    reorderPoint packed(9:0) const;
    safetyStock  packed(9:0) const;
  end-pi;
  
  dcl-s qty packed(9:0);
  
  if onHand < reorderPoint;
    qty = reorderPoint + safetyStock - onHand;
  else;
    qty = 0;
  endif;
  
  return qty;
end-proc;
```

Breaking it down:

- **`dcl-proc` / `end-proc`** — opens and closes the procedure
- **`dcl-pi *n packed(9:0)` / `end-pi`** — declares the **procedure interface** (PI): the return type and the parameters. `*n` means "use the same name as the procedure" — you rarely need anything else
- **Parameters** inside the PI — each has a name, type, and optional keywords
- **Local variables** declared inside the procedure body
- **`return`** — the keyword for exiting with a value

## Parameter passing

RPG passes parameters **by reference** by default. That means the called procedure can modify the caller's variable — usually not what you want.

### `const` — read-only

```rpgle
dcl-pi *n packed(9:0);
  onHand packed(9:0) const;
end-pi;
```

`const` tells the compiler "this procedure will not modify this parameter." The compiler enforces it. Use `const` on every input parameter you don't intend to change.

### `value` — pass by value

```rpgle
dcl-pi *n packed(9:0);
  onHand packed(9:0) value;
end-pi;
```

`value` literally copies the value in. The procedure gets its own copy. Useful for small scalar types (integers, small decimals) where copying is cheap.

Practical rule: use `const` for most things. Use `value` occasionally for small numerics when you want to be explicit. Plain by-reference is rare in new code.

### Optional parameters

```rpgle
dcl-pi *n ind;
  suplCode  char(10) const;
  includeInactive ind const options(*nopass);
end-pi;

dcl-s checkInactive ind;

if %parms() >= 2;
  checkInactive = includeInactive;
else;
  checkInactive = *off;
endif;
```

`options(*nopass)` means the caller can omit this parameter. `%parms()` tells you how many were actually passed. Optional parameters go at the end of the parameter list.

## Return values

The return type goes right after the `*n` in the PI:

```rpgle
dcl-pi *n varchar(50);  // returns varchar(50)
dcl-pi *n ind;          // returns indicator
dcl-pi *n int(10);      // returns integer
dcl-pi *n;              // returns nothing (a procedure, not a function)
```

For procedures that return nothing, omit the return type and use bare `return;` to exit early, or let execution fall off the end.

## Local vs. exported

By default, procedures are **local** — callable only from within the same module. To make a procedure callable from another program or service program, mark it `export`:

```rpgle
dcl-proc GetSupplierName export;
  // ...
end-proc;
```

You'll use this heavily when we get to service programs in Part 3, Tutorial 4.

## Calling a procedure

Call a procedure like a function in any other language:

```rpgle
dcl-s needed packed(9:0);

needed = CalculateReorderQty(PR_QOH : PR_REORDPT : PR_SAFTYSTK);
```

Note the **colon separator** between parameters — not comma. That's an RPG-specific syntax choice you'll get used to quickly.

## Where procedures live

In a program that's a single source file, you can put all procedures in the same file. One of them is the **main procedure** — the program's entry point — declared in `ctl-opt`:

```rpgle
ctl-opt main(MainProgram);

dcl-proc MainProgram;
  // program entry
end-proc;

dcl-proc HelperOne;
  // local helper
end-proc;

dcl-proc HelperTwo;
  // local helper
end-proc;
```

For procedures you want to share across multiple programs, you build a **service program** — covered in Part 3, Tutorial 4.

Next: [Chapter 4: Built-in Functions]({% link part-2/04-built-in-functions.md %}).
Commit message: Part 2 Ch 3: Procedures

Commit 5 of 8 — part-2/04-built-in-functions.md
Path: part-2/04-built-in-functions.md
markdown---
title: "4. Built-in Functions"
parent: "Part 2: Learn the Language"
nav_order: 4
---

# Built-in Functions

**Built-in functions** — BIFs — are RPG's standard library. They're always prefixed with `%`, so when you see `%trim`, `%date`, `%eof`, you know it's a BIF rather than a procedure you wrote.

There are dozens. This chapter covers the ones you'll use every day, grouped by what they do.

## String manipulation

### `%trim`, `%trimr`, `%triml`

```rpgle
name = %trim(SP_NAME);   // trim leading and trailing spaces
name = %trimr(SP_NAME);  // trim trailing only (right)
name = %triml(SP_NAME);  // trim leading only (left)
```

The most-used BIF in RPG. Database character fields are frequently space-padded; `%trim` removes the padding.

### `%len`

```rpgle
nameLength = %len(%trim(SP_NAME));  // actual length of trimmed name
maxLength  = %len(SP_NAME);         // declared length of the field
```

Returns the current length of a varchar, or the declared length of a fixed-length char.

### `%subst`

```rpgle
firstThree = %subst(productCode : 1 : 3);   // chars 1-3
tail       = %subst(productCode : 5);        // from position 5 to end
```

Substring extraction. Takes position and optional length. Positions are **1-based**.

### `%scan`, `%scanrpl`

```rpgle
pos = %scan('-' : productCode);              // position of first '-'
clean = %scanrpl('bad' : 'good' : message);  // replace all 'bad' with 'good'
```

`%scan` finds a substring's position. `%scanrpl` replaces all occurrences.

### `%upper`, `%lower`

```rpgle
normalized = %upper(userInput);
friendly   = %lower(headerText);
```

Case conversion.

### `%char`, `%dec`

```rpgle
label = 'Qty: ' + %char(PR_QOH);          // number to char
price = %dec(priceString : 9 : 2);         // char to decimal, with precision
```

`%char` converts any type to its character representation. `%dec` converts character or numeric to packed decimal — with required precision arguments (digits:decimals).

### Concatenation

```rpgle
message = 'Supplier ' + %trim(SP_NAME)
        + ' has ' + %char(productCount)
        + ' active products.';
```

Plain `+` concatenates strings. This is why you often see `%char` sprinkled in — to turn numbers into strings first.

## File I/O status

### `%eof`, `%found`

```rpgle
read SUPPLIER;
if %eof(SUPPLIER); leave; endif;

chain 'ACME001' SUPPLIER;
if not %found(SUPPLIER);
  dsply 'Supplier not found';
endif;
```

`%eof` — true if the last read hit end of file. `%found` — true if the last CHAIN (or similar positioning operation) found a match. Check these immediately after the operation — they change on the next I/O.

### `%error`, `%status`

```rpgle
write SP_REC;
if %error;
  dsply ('Write failed with status: ' + %char(%status));
endif;
```

`%error` flags that the last operation raised an error. `%status` gives the numeric status code. More on these in [Chapter 6]({% link part-2/06-error-handling.md %}).

## Date and time

Briefly — full coverage in [Chapter 7]({% link part-2/07-date-math.md %}).

```rpgle
today  = %date();                  // today's date
now    = %time();                  // current time
stamp  = %timestamp();             // full timestamp
future = today + %days(30);        // 30 days from now
diff   = %diff(endDate : startDate : *days);  // days between
```

## Numeric

### `%abs`

```rpgle
variance = %abs(actual - expected);
```

Absolute value.

### `%int`, `%dec`

```rpgle
whole = %int(decimalValue);         // truncate to integer
exact = %dec(amount : 11 : 4);      // convert to packed, specified precision
```

## Control and lookup

### `%parms`

```rpgle
if %parms() >= 2;
  // second parameter was passed
endif;
```

Number of parameters actually passed to the current procedure. Used with optional parameters.

### `%lookup`

```rpgle
dcl-s regionCodes char(3) dim(50);
// ... regionCodes populated ...

idx = %lookup('NEU' : regionCodes);
if idx > 0;
  // found at position idx
endif;
```

Finds a value in an array. Returns the position or 0 if not found.

## One habit worth building

When you're writing expressions, reach for BIFs first. If you find yourself writing a loop to trim spaces, convert a type, or search a string, there's almost certainly a BIF that does it in one line. Read the IBM RPG reference occasionally just to remind yourself what's available — you'll discover BIFs you didn't know existed and simplify whole sections of code.

Next: [Chapter 5: Embedded SQL]({% link part-2/05-embedded-sql.md %}).
Commit message: Part 2 Ch 4: Built-in Functions

Commit 6 of 8 — part-2/05-embedded-sql.md
Path: part-2/05-embedded-sql.md
markdown---
title: "5. Embedded SQL"
parent: "Part 2: Learn the Language"
nav_order: 5
---

# Embedded SQL

Most modern RPG programs that touch data use **embedded SQL** rather than RPG's native file operations (CHAIN, READ, WRITE). Native I/O still has its place — we cover it in Part 3, Tutorial 2 — but if you learn one data-access approach first, make it SQL. It's the bulk of real-world work.

## File extension

Save programs with embedded SQL as **`.sqlrpgle`**, not `.rpgle`. The compiler preprocesses the SQL when the extension is `.sqlrpgle`; with `.rpgle`, `exec sql` lines are treated as syntax errors.

## The `exec sql` statement

Every SQL statement is wrapped in `exec sql`:

```rpgle
exec sql
  select SP_NAME
    into :suplName
    from SUPPLIER
   where SP_SUPL = :suplCode;
```

The statement ends with a semicolon. You can split it across as many lines as readability wants.

## Host variables

Inside an SQL statement, reference RPG variables by prefixing them with `:` — these are called **host variables**:

```rpgle
dcl-s suplCode char(10);
dcl-s suplName varchar(50);

suplCode = 'ACME001';

exec sql
  select SP_NAME
    into :suplName
    from SUPPLIER
   where SP_SUPL = :suplCode;
```

The `:` prefix is only used inside SQL. In regular RPG code, you refer to variables by their plain name.

## SQLSTATE and SQLCODE

After every `exec sql`, two special variables are updated automatically:

- **`sqlstate`** — a 5-character standard SQL state code. `'00000'` = success. `'02000'` = no row found. Anything else = an error.
- **`sqlcode`** — numeric. 0 = success, negative = error, positive = warning.

Check them:

```rpgle
exec sql
  select SP_NAME into :suplName
    from SUPPLIER
   where SP_SUPL = :suplCode;

select;
  when sqlstate = '00000';
    // success — use suplName
  when sqlstate = '02000';
    dsply 'Supplier not found';
  other;
    dsply ('SQL error: ' + sqlstate);
endsl;
```

`SQLSTATE` is the modern, standards-based check. Prefer it to `SQLCODE` except when reading very old code.

## Single-row queries

Single-row reads use `SELECT ... INTO`:

```rpgle
exec sql
  select SP_NAME, SP_ACTIVE, SP_ONBOARD
    into :suplName, :suplActive, :onboardDate
    from SUPPLIER
   where SP_SUPL = :suplCode;
```

The field order in `SELECT` must match the host variable order in `INTO`.

If the `WHERE` clause returns more than one row, you'll get error `21000` ("more than one row returned"). Make sure your predicate is unique — or use a cursor.

## Cursors for multiple rows

When a query returns many rows, use a cursor:

```rpgle
dcl-s prodCode char(15);
dcl-s prodSupl char(10);
dcl-s prodQty  packed(9:0);

exec sql
  declare productCursor cursor for
    select PR_PROD, PR_SUPL, PR_QOH
      from PRODUCT
     where PR_SUPL = :targetSupplier
     order by PR_PROD;

exec sql open productCursor;

dow sqlstate = '00000';
  exec sql fetch next from productCursor
    into :prodCode, :prodSupl, :prodQty;

  if sqlstate = '00000';
    // process the row
  endif;
enddo;

exec sql close productCursor;
```

The pattern: declare, open, fetch in a loop, close. The loop exits when `FETCH` returns anything other than `'00000'` — typically `'02000'` when you've read all rows.

## Insert, update, delete

```rpgle
// INSERT
exec sql
  insert into SUPPLIER
    (SP_SUPL, SP_NAME, SP_ACTIVE, SP_ONBOARD)
  values
    (:newCode, :newName, 'Y', current_date);

// UPDATE
exec sql
  update PRODUCT
     set PR_PRICE = :newPrice,
         PR_CHGDT = current_date
   where PR_PROD = :prodCode;

// DELETE
exec sql
  delete from REORDCND
   where RC_CRTDT < current_date - 30 days;
```

These don't return rows, so no `INTO` clause. `SQLSTATE` still tells you success or failure, and an extra variable — `sqlerrd(3)` — tells you how many rows were affected.

## Transactions

For multi-statement work that needs to be atomic:

```rpgle
monitor;
  exec sql insert into ORDERS ...;
  if sqlstate <> '00000';
    exec sql rollback;
    return *off;
  endif;

  exec sql insert into ORDER_LINES ...;
  if sqlstate <> '00000';
    exec sql rollback;
    return *off;
  endif;

  exec sql commit;
  return *on;
on-error;
  exec sql rollback;
  return *off;
endmon;
```

By default on IBM i, SQL operations auto-commit unless the file is journaled and the program specifies `COMMIT` in the CRTSQLRPGI parameters. For tutorial purposes on PUB400, auto-commit is fine. In production, always set up journaling and explicit transactions for anything that modifies multiple tables.

## Practical tips

- **Always use host variables in the WHERE clause**, never string-concatenate user input. SQL injection is as real in RPG as in any other language.
- **Select only the columns you need.** `SELECT *` works but hides schema changes and wastes bandwidth.
- **Match host variable types to column types.** A `varchar(50)` column reading into a `char(50)` host variable will space-pad. Often fine, sometimes not.
- **Use `ORDER BY` explicitly** when you care about row order. SQL databases don't guarantee order without it.

## Try it

With your practice tables from Part 1 Chapter 6, try this query in VS Code's Db2 for IBM i extension:

```sql
SELECT COUNT(*) AS PRODUCTS_BELOW_REORDER
  FROM PRODUCT
 WHERE PR_ACTIVE = 'Y'
   AND PR_QOH < PR_REORDPT;
```

You should get 14. That's the 14 active products that the batch RPG program in Part 3 Tutorial 5 will write into REORDCND.

Next: [Chapter 6: Error Handling with MONITOR]({% link part-2/06-error-handling.md %}).
Commit message: Part 2 Ch 5: Embedded SQL

Commit 7 of 8 — part-2/06-error-handling.md
Path: part-2/06-error-handling.md
markdown---
title: "6. Error Handling with MONITOR"
parent: "Part 2: Learn the Language"
nav_order: 6
---

# Error Handling with MONITOR

RPG programs crash. Division by zero, numeric overflow, reading past end-of-file without checking, converting "abc" to a decimal — these are runtime errors, and by default they terminate the program with an unrecoverable message. Users see the awful green-screen halt. Your batch job dies mid-run. You get a 3 AM phone call.

The solution is `monitor` — RPG's equivalent of `try`/`catch` in other languages.

## Basic structure

```rpgle
monitor;
  // code that might fail
on-error;
  // handle any error
endmon;
```

- **`monitor`** starts a protected block
- **`on-error`** introduces an error handler
- **`endmon`** closes the block

If the code inside `monitor` raises an error, execution jumps immediately to the `on-error` handler. If no error occurs, the `on-error` block is skipped.

## A simple example

```rpgle
dcl-proc SafeDivide export;
  dcl-pi *n packed(9:4);
    numerator   packed(9:2) const;
    denominator packed(9:2) const;
  end-pi;

  dcl-s result packed(9:4);

  monitor;
    result = numerator / denominator;
  on-error;
    result = 0;
  endmon;

  return result;
end-proc;
```

If `denominator` is 0, division by zero raises an error; the handler catches it and returns 0 instead of crashing.

## Handling specific errors

You can branch on the specific error code:

```rpgle
monitor;
  result = numerator / denominator;
on-error *zero-divide;
  result = 0;
on-error *out-of-range;
  result = 9999999999.9999;
on-error;
  // everything else
  result = 0;
endmon;
```

A handful of useful named statuses:

- `*zero-divide` — division by zero (status code 00102)
- `*out-of-range` — value out of declared range (00103)
- `*string-error` — string operation failed (00100)
- `*decimal-data` — bad decimal data (00907)

You can also specify numeric ranges directly:

```rpgle
on-error 00100 : 00200;  // any status from 00100 to 00200
```

Order matters. On-error clauses are checked top to bottom; the first match wins.

## Inspecting the error

Inside an `on-error` block, two BIFs give you diagnostic info:

```rpgle
monitor;
  result = numerator / denominator;
on-error;
  dsply ('Error ' + %char(%status) + ': ' + %trim(%error()));
  result = 0;
endmon;
```

- `%status` — the numeric status code
- `%error` — a short text description

Log these when you want to know what actually went wrong. Silently swallowing errors is worse than crashing — you lose the signal.

## SQL errors are different

`exec sql` statements don't raise RPG errors when they fail — they update `sqlstate` and `sqlcode` but keep executing. So `monitor` won't catch SQL errors. You check `sqlstate` explicitly:

```rpgle
exec sql
  update PRODUCT
     set PR_PRICE = :newPrice
   where PR_PROD = :prodCode;

if sqlstate <> '00000';
  exec sql rollback;
  dsply ('SQL update failed: ' + sqlstate);
  return *off;
endif;
```

However, `monitor` *will* catch errors that happen in RPG expressions referenced by an SQL statement — for example, an overflow when you compute a host variable. The rule of thumb: `monitor` for RPG runtime errors, `sqlstate` for SQL errors.

## A realistic pattern: transactional update

```rpgle
dcl-proc UpdatePriceWithRollback export;
  dcl-pi *n ind;
    prodCode char(15)    const;
    newPrice packed(9:2) const;
  end-pi;

  monitor;
    exec sql
      update PRODUCT
         set PR_PRICE = :newPrice,
             PR_CHGDT = current_date
       where PR_PROD = :prodCode;

    if sqlstate <> '00000';
      exec sql rollback;
      return *off;
    endif;

    exec sql commit;
    return *on;

  on-error;
    exec sql rollback;
    return *off;
  endmon;
end-proc;
```

Note the double defense: explicit `sqlstate` check for SQL problems, `monitor` for any RPG-level failures (like a type conversion error computing `newPrice`). Either path rolls back the transaction.

## When to use `monitor`

- Any arithmetic that could divide by zero or overflow (price calculations, quantity conversions)
- Any type conversion from external data (`%dec` on user input, character-to-numeric parsing)
- Any file I/O where a failure shouldn't crash the whole run
- Any procedure that's called from a production batch job

## When not to bother

- Inside a small, pure-math helper where overflow would be a programmer error, not a data error
- When you genuinely want to crash — for programmer assertions during development

"Swallow every possible error" is just as bad as "don't handle any." Aim for `monitor` where real-world data can plausibly be malformed, and let programmer bugs crash loudly during testing.

Next: [Chapter 7: Date Math]({% link part-2/07-date-math.md %}).
Commit message: Part 2 Ch 6: Error Handling with MONITOR

Commit 8 of 8 — part-2/07-date-math.md
Path: part-2/07-date-math.md
markdown---
title: "7. Date Math"
parent: "Part 2: Learn the Language"
nav_order: 7
---

# Date Math

RPG has first-class support for dates, times, and timestamps. Unlike languages where dates are a library you have to import and wrestle with, RPG's compiler understands them natively — and, importantly, enforces type safety so you can't accidentally add a raw number to a date.

This matters because business logic is full of date questions. When is this order due? How many days until this product runs out? Did this shipment arrive late? Is this supplier contract still valid?

## The three temporal types

```rpgle
dcl-s orderDate      date;
dcl-s pickupTime     time;
dcl-s orderCreatedAt timestamp;
```

- **`date`** — just a date (year, month, day)
- **`time`** — just a time (hour, minute, second)
- **`timestamp`** — both, plus microseconds

All three are stored internally as numbers, but the compiler treats them as opaque temporal values. You can't read or assign them as integers.

## Getting the current date/time

```rpgle
today  = %date();          // today
now    = %time();          // current time
stamp  = %timestamp();     // right now, full precision
```

These are system calls — they return the current value each time you call them.

## Creating a specific date

```rpgle
dueDate       = %date('2026-06-15' : *iso);
anniversary   = %date('06/15/2026' : *usa);
```

The second argument is a format specifier. `*iso` is the standard `YYYY-MM-DD`. Other common ones: `*usa` (`MM/DD/YYYY`), `*eur` (`DD.MM.YYYY`), `*jis` (`YYYY-MM-DD` like ISO).

**Use `*iso` unless you have a strong reason.** Ambiguous date formats cause bugs.

## Converting between date and string

```rpgle
displayDate = %char(orderDate : *iso);     // '2026-06-15'
orderDate   = %date('2026-06-15' : *iso);  // back to date
```

`%char` with a format specifier produces the string. `%date` with a format specifier parses it.

## Duration BIFs

These create values you can add to or subtract from dates:

```rpgle
nextWeek    = today + %days(7);
nextMonth   = today + %months(1);
nextYear    = today + %years(1);
contractEnd = today + %days(365);
oneHourAgo  = now   - %hours(1);
```

Full list: `%days`, `%weeks`, `%months`, `%years`, `%hours`, `%minutes`, `%seconds`, `%ms`.

You can't skip them. This is illegal:

```rpgle
nextWeek = today + 7;  // compile error
```

The compiler enforces: to add 7 to a date, you must specify what 7 *is*. `%days(7)`? `%weeks(7)`? That's the type safety paying off.

## Calculating differences

`%diff` computes the difference between two dates, times, or timestamps:

```rpgle
daysOld     = %diff(%date()    : hiredDate    : *days);
hoursWorked = %diff(clockOut   : clockIn      : *hours);
ageInMonths = %diff(%date()    : birthDate    : *months);
```

The third argument is the unit. Positive result means the first date is later.

## Practical patterns

### Is this due?

```rpgle
if expectedDate < %date();
  dsply 'Overdue';
endif;
```

### Is this within tolerance?

```rpgle
if %diff(%date() : expectedDate : *days) > toleranceDays;
  dsply 'Past tolerance window';
endif;
```

### Time supply (inventory days remaining)

```rpgle
if avgDailyDemand > 0;
  daysOfStock = PR_QOH / avgDailyDemand;
else;
  daysOfStock = 9999;  // no demand — effectively infinite
endif;
```

### Next scheduled reorder

```rpgle
nextReorder = lastOrderDate + %days(reorderCycleDays);
```

### Age in years (decimal)

```rpgle
ageInDays  = %diff(%date() : birthDate : *days);
ageInYears = ageInDays / 365.25;
```

## Watch out for the 80/20 edge cases

- **Month math is weird.** `%date('2026-01-31') + %months(1)` returns `2026-02-28`, not `2026-02-31` (which doesn't exist). Compiler handles it sensibly, but be aware.
- **Daylight savings.** Time arithmetic across a DST boundary can produce unexpected results. If you're working with exact durations, consider timestamps in UTC.
- **Year 9999.** RPG dates max out at `9999-12-31`. You probably won't hit it, but effectively-infinite sentinel values should avoid dates.
- **Date zero.** The default value for an uninitialized date is `0001-01-01`. Watch for this when reading records where the date column might be NULL or unset — you'll sometimes see `0001-01-01` as an actual stored value.

## Try it

Using your practice tables, run this query to find products that haven't had a price change in over a year:

```sql
SELECT PR_PROD, PR_PRICE, PR_CHGDT
  FROM PRODUCT
 WHERE PR_CHGDT < CURRENT_DATE - 365 DAYS
   AND PR_ACTIVE = 'Y'
 ORDER BY PR_CHGDT;
```

In the sample data, most products have recent change dates (the past couple months). Only the inactive products and a handful with 2023–2024 dates will come back — a realistic pattern for a stale-pricing audit.

---

That's the language. Chapters 1–7 cover what you need to read and write modern RPG fluently. The next three chapters (coming next) cover the tooling around it in VS Code: compiling in its various forms, the debugger, and Git.

Next: Chapter 8 — Compiling from VS Code (coming in Wave 2).
