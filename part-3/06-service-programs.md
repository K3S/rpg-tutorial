---
title: "6. Service Programs"
parent: "Part 3: Build Real Things"
nav_order: 6
---

# Tutorial 6: Service Programs

Tutorial 5 built `CalcLineValue` and it worked. But it's locked inside the INVVAL source. If another program wants to compute a line value, it has to redefine the procedure — or the team copies code, which is how consistency dies.

A **service program** is IBM i's answer. You package procedures into an object that other programs *bind* to at compile time. The procedures are defined once and callable from any program that binds them in. That's how teams build shared business logic on IBM i.

## What you'll build

Three pieces of code working together.

**A service program called `INVSVR`** (Inventory Services) that exports two procedures:

- `GetPriceForProduct(prodCode)` — returns a product's price
- `IsProductActive(prodCode)` — returns a boolean indicating whether it's an active product

**A header file called `INVSVR_H`** that contains the prototypes. Any program that wants to call these procedures `/copy`s this header.

**A caller program called `PRCALC2`** — a refactored version of Tutorial 1's PRCALC that uses the service program instead of doing its own SQL.

When you're done, you'll have made two SQL lookups reusable across your entire codebase.

## What you'll learn

- The difference between a `*PGM` object (a program) and a `*SRVPGM` object (a service program)
- How `export` on a procedure makes it callable from outside its module
- How `/copy` brings prototypes into a program at compile time
- The three-step build: compile as module, create service program, bind from caller
- Binding directories, which make the third step less painful

## Before you start

- Part 1 complete
- Tutorial 5 completed — prototypes and procedure definitions should feel familiar
- Understand the call flow from Tutorial 1 — PRCALC2 will replicate that flow

## Step 1 — Write the header file

Create a new member in your source physical file, but this time create it in a **different** source PF: `QCPYLESRC`. This is the conventional location for copy headers.

If you don't have `QCPYLESRC` yet, create it the same way you created `QRPGLESRC` in Part 1 Chapter 7:

```
CRTSRCPF FILE(YOUR_LIBRARY/QCPYLESRC) RCDLEN(112)
```

Then create a new member in `QCPYLESRC` called **INVSVR_H**, type **RPGLE**. Paste:

```rpgle
// ************************************************************
// * Name: INVSVR_H
// * Type: RPG copy header
// * Desc: Prototypes for the INVSVR service program.
// *       /copy this into any program that calls
// *       GetPriceForProduct or IsProductActive.
// * Tutorial: Part 3, Tutorial 6
// ************************************************************

dcl-pr GetPriceForProduct packed(9:2);
  prodCode char(15) const;
end-pr;

dcl-pr IsProductActive ind;
  prodCode char(15) const;
end-pr;
```

That's the whole file. No body, no `ctl-opt`, no `return`. Just prototypes. This is what callers will include to know the shape of procedures they can call.

## Step 2 — Write the service program source

Create a new member in `QRPGLESRC` called **INVSVR**, type **SQLRPGLE**. Paste:

```rpgle
**free
ctl-opt nomain option(*nodebugio:*srcstmt);
// ************************************************************
// * Name: INVSVR
// * Type: ILE RPG Module (SQLRPGLE, NOMAIN)
// * Desc: Inventory service procedures — price lookup and
// *       active-status check. Exported for use by any
// *       program that binds to the INVSVR service program.
// * Tutorial: Part 3, Tutorial 6
// ************************************************************

/copy QCPYLESRC,INVSVR_H

dcl-proc GetPriceForProduct export;
  dcl-pi *n packed(9:2);
    prodCode char(15) const;
  end-pi;

  dcl-s price packed(9:2);

  exec sql
    set option commit = *none, closqlcsr = *endactgrp;

  exec sql
    select PR_PRICE into :price
      from PRODUCT
     where PR_PROD = :prodCode;

  if sqlstate <> '00000';
    return 0;
  endif;

  return price;
end-proc;

dcl-proc IsProductActive export;
  dcl-pi *n ind;
    prodCode char(15) const;
  end-pi;

  dcl-s active char(1);

  exec sql
    set option commit = *none, closqlcsr = *endactgrp;

  exec sql
    select PR_ACTIVE into :active
      from PRODUCT
     where PR_PROD = :prodCode;

  if sqlstate <> '00000';
    return *off;
  endif;

  return (active = 'Y');
end-proc;
```

A few things to notice:

- **`ctl-opt nomain`** — this source file has no main procedure. It's a collection of procedures meant to be called from outside. Without `nomain`, the compiler expects an entry point.
- **`/copy QCPYLESRC,INVSVR_H`** — pulls the prototypes from your header file into this source. They have to match the definitions below. The compiler checks.
- **`export` keyword on each `dcl-proc`** — this is the critical keyword. Without `export`, the procedure is module-local — invisible to callers. `export` makes it part of the service program's public API.

## Step 3 — Build the service program (two sub-steps)

Unlike a regular program, a service program takes two compile commands:

### 3a. Compile INVSVR as a module

Right-click the INVSVR member. Choose Compile. Pick **CRTSQLRPGI**. But before running, look at the compile command options — you need to change `OBJTYPE` from `*PGM` to `*MODULE`.

The command should look like:

```
CRTSQLRPGI OBJ(YOUR_LIBRARY/INVSVR)
           SRCSTMF('...')
           OBJTYPE(*MODULE)
           COMMIT(*NONE)
           DBGVIEW(*SOURCE)
           CVTCCSID(*JOB)
           COMPILEOPT('TGTCCSID(*JOB)')
```

Run it. You should see `Module INVSVR created in library YOUR_LIBRARY`.

A module is an intermediate compiled artifact. It can't be called directly. Think of it as compiled code waiting to be packaged.

### 3b. Create the service program from the module

From the VS Code IBM i terminal or a 5250 session, run:

```
CRTSRVPGM SRVPGM(YOUR_LIBRARY/INVSVR)
          MODULE(YOUR_LIBRARY/INVSVR)
          EXPORT(*ALL)
          ACTGRP(*CALLER)
```

- `SRVPGM` — the service program object we're creating
- `MODULE` — the modules it's built from (here, just one)
- `EXPORT(*ALL)` — every exported procedure from every module is callable from outside
- `ACTGRP(*CALLER)` — runs in the caller's activation group (the right default for most services)

You should see `Service program INVSVR created in library YOUR_LIBRARY`. If you `WRKOBJ YOUR_LIBRARY/INVSVR`, you'll now see two entries: the `*MODULE` from step 3a and the `*SRVPGM` you just created.

## Step 4 — Set up a binding directory (once)

When a caller program compiles, the compiler needs to know "where do I find these procedures?" You can tell it explicitly every time, or you can create a **binding directory** — a named list of service programs that the compiler checks automatically.

Create one:

```
CRTBNDDIR BNDDIR(YOUR_LIBRARY/INVBND) TEXT('Inventory tutorial binding directory')
ADDBNDDIRE BNDDIR(YOUR_LIBRARY/INVBND) OBJ((YOUR_LIBRARY/INVSVR *SRVPGM))
```

That's it. INVBND is a list with one entry — the INVSVR service program. Adding more service programs later is `ADDBNDDIRE` again.

## Step 5 — Write the caller

Create a new member in `QRPGLESRC` called **PRCALC2**, type **SQLRPGLE**. Paste:

```rpgle
**free
ctl-opt dftactgrp(*no) actgrp(*caller)
        option(*nodebugio:*srcstmt)
        bnddir('INVBND')
        main(Main);
// ************************************************************
// * Name: PRCALC2
// * Type: ILE RPG Program (SQLRPGLE)
// * Desc: Product Price Calculator, refactored to use the
// *       INVSVR service program from Tutorial 6.
// * Tutorial: Part 3, Tutorial 6
// ************************************************************

/copy QCPYLESRC,INVSVR_H

dcl-proc Main;
  dcl-pi *n;
    wk_productCode char(15) const;
    wk_quantity    packed(7:0) const;
  end-pi;

  dcl-s price      packed(9:2);
  dcl-s totalPrice packed(11:2);

  if not IsProductActive(wk_productCode);
    dsply ('Product ' + %trim(wk_productCode) + ' is inactive or missing');
    return;
  endif;

  price = GetPriceForProduct(wk_productCode);

  totalPrice = price * wk_quantity;

  dsply ('Product: ' + %trim(wk_productCode));
  dsply ('Price:   ' + %char(price));
  dsply ('Qty:     ' + %char(wk_quantity));
  dsply ('Total:   ' + %char(totalPrice));

  return;
end-proc;
```

Compare it to Tutorial 1's PRCALC. Same functionality. Different structure:

- No embedded SQL here — it's in INVSVR.
- The "is the product usable" check is one line — `IsProductActive(wk_productCode)`.
- The price lookup is one line — `GetPriceForProduct(...)`.
- The file is dramatically shorter.

The `bnddir('INVBND')` option on the `ctl-opt` line is what tells the compiler to check the INVBND binding directory for any procedures it can't find locally. That's what resolves the two service program calls.

Compile with **CRTSQLRPGI** (normal program compile, not module). The binding is automatic because of `bnddir`.

## Run it

```
CALL PRCALC2 PARM('WIDGET-STD-001' 10)
```

Should produce output identical to Tutorial 1's PRCALC with the same parameters. The difference is invisible to the user — which is the whole point. The caller is simpler, and if you write a third program tomorrow that needs `IsProductActive`, it's a one-line `/copy` and a `bnddir`.

## What you just did

You separated a chunk of logic from its callers. The SQL lookups for product existence and price now live in one place. Five programs could call them. If the PRODUCT schema changes and the lookup needs updating, you update one module, recompile, and re-create the service program — and every caller picks it up at runtime without recompiling.

This is how small teams build codebases that don't rot. Shared business rules go in service programs; callers stay focused on their specific workflow.

## Try this

**Add a third procedure.** Extend INVSVR with `GetOnHandQty(prodCode)` that returns `PR_QOH`. Add its prototype to `INVSVR_H`, its definition to `INVSVR.sqlrpgle`, recompile the module, re-create the service program. Modify PRCALC2 to display the QOH alongside the total. No binding directory change needed.

**Break EXPORT.** Remove `export` from `IsProductActive` and try to recompile the service program and PRCALC2. What error do you get? Now you know what "not exported" looks like in practice.

**Look at the service program's signatures.** Run `DSPSRVPGM SRVPGM(YOUR_LIBRARY/INVSVR) DETAIL(*PROCEXP)`. It shows you what the service program considers its public API. Handy for debugging "why can't this caller see the procedure."

**What happens when you change a prototype but not the definition?** Change `IsProductActive` in `INVSVR_H` to return `char(1)` instead of `ind`. Leave the .sqlrpgle unchanged. Recompile PRCALC2. What's the error?  Fix the mismatch. This is the #1 bug that bites new IBM i developers working with service programs — the header says one thing, the implementation says another. The compiler catches it, but only when something is using the header.

## What's next

You've made logic shareable. Tutorial 7 is the biggest one in Part 3 — a batch reorder report that brings together cursors, procedures, service programs, and the REORDCND table you set up in Part 1. It's the capstone.

Next: [Tutorial 7: Batch Reorder Report — The Capstone]({% link part-3/07-batch-reorder.md %}).
