---
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

Next: [Chapter 6: Choosing Between SQL and Native I/O]({% link part-2/06-sql_vs_rla.md %}).
