---
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
