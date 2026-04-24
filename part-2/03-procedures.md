---
title: "3. Procedures"
parent: "Part 2: Learn the Language"
nav_order: 3
---

# Procedures

A **procedure** is RPG's term for what other languages call a function, method, or subroutine. Modern RPG is written as collections of small procedures, not monolithic programs.

If you've written a function in any mainstream language in the last twenty years, the concept is identical. The syntax is where RPG has its own flavor.

## Anatomy of a procedure

```rpgledcl-proc CalculateReorderQty;
dcl-pi *n packed(9:0);
onHand       packed(9:0) const;
reorderPoint packed(9:0) const;
safetyStock  packed(9:0) const;
end-pi;dcl-s qty packed(9:0);if onHand < reorderPoint;
qty = reorderPoint + safetyStock - onHand;
else;
qty = 0;
endif;return qty;
end-proc;

Breaking it down:

- **`dcl-proc` / `end-proc`** — opens and closes the procedure
- **`dcl-pi *n packed(9:0)` / `end-pi`** — declares the **procedure interface** (PI): the return type and the parameters. `*n` means "use the same name as the procedure" — you rarely need anything else
- **Parameters** inside the PI — each has a name, type, and optional keywords
- **Local variables** declared inside the procedure body
- **`return`** — the keyword for exiting with a value

## Parameter passing

RPG passes parameters **by reference** by default. That means the called procedure can modify the caller's variable — usually not what you want.

### `const` — read-only

```rpgledcl-pi *n packed(9:0);
onHand packed(9:0) const;
end-pi;

`const` tells the compiler "this procedure will not modify this parameter." The compiler enforces it. Use `const` on every input parameter you don't intend to change.

### `value` — pass by value

```rpgledcl-pi *n packed(9:0);
onHand packed(9:0) value;
end-pi;

`value` literally copies the value in. The procedure gets its own copy. Useful for small scalar types (integers, small decimals) where copying is cheap.

Practical rule: use `const` for most things. Use `value` occasionally for small numerics when you want to be explicit. Plain by-reference is rare in new code.

### Optional parameters

```rpgledcl-pi *n ind;
suplCode  char(10) const;
includeInactive ind const options(*nopass);
end-pi;dcl-s checkInactive ind;if %parms() >= 2;
checkInactive = includeInactive;
else;
checkInactive = *off;
endif;

`options(*nopass)` means the caller can omit this parameter. `%parms()` tells you how many were actually passed. Optional parameters go at the end of the parameter list.

## Return values

The return type goes right after the `*n` in the PI:

```rpgledcl-pi *n varchar(50);  // returns varchar(50)
dcl-pi *n ind;          // returns indicator
dcl-pi *n int(10);      // returns integer
dcl-pi *n;              // returns nothing (a procedure, not a function)

For procedures that return nothing, omit the return type and use bare `return;` to exit early, or let execution fall off the end.

## Local vs. exported

By default, procedures are **local** — callable only from within the same module. To make a procedure callable from another program or service program, mark it `export`:

```rpgledcl-proc GetSupplierName export;
// ...
end-proc;

You'll use this heavily when we get to service programs in Part 3, Tutorial 4.

## Calling a procedure

Call a procedure like a function in any other language:

```rpgledcl-s needed packed(9:0);needed = CalculateReorderQty(PR_QOH : PR_REORDPT : PR_SAFTYSTK);

Note the **colon separator** between parameters — not comma. That's an RPG-specific syntax choice you'll get used to quickly.

## Where procedures live

In a program that's a single source file, you can put all procedures in the same file. One of them is the **main procedure** — the program's entry point — declared in `ctl-opt`:

```rpglectl-opt main(MainProgram);dcl-proc MainProgram;
// program entry
end-proc;dcl-proc HelperOne;
// local helper
end-proc;dcl-proc HelperTwo;
// local helper
end-proc;

For procedures you want to share across multiple programs, you build a **service program** — covered in Part 3, Tutorial 4.

Next: [Chapter 4: Built-in Functions]({% link part-2/04-built-in-functions.md %}).
