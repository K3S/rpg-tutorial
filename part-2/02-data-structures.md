---
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
