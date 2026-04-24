# Learning RPG via VS Code

A practical tutorial for developers coming to IBM i RPG programming through modern tooling. This guide covers both the Visual Studio Code environment setup and free-format RPGLE fundamentals, using examples aligned with K3S database conventions.

---

## Table of Contents

1. [Why VS Code for RPG](#1-why-vs-code-for-rpg)
2. [Prerequisites](#2-prerequisites)
3. [Installing VS Code and the IBM i Development Pack](#3-installing-vs-code-and-the-ibm-i-development-pack)
4. [Connecting to Your IBM i](#4-connecting-to-your-ibm-i)
5. [Understanding the IBM i File Systems](#5-understanding-the-ibm-i-file-systems)
6. [Your First RPG Program: Hello World](#6-your-first-rpg-program-hello-world)
7. [Free-Format RPGLE Fundamentals](#7-free-format-rpgle-fundamentals)
8. [Working with DB2 in SQLRPGLE](#8-working-with-db2-in-sqlrpgle)
9. [Compiling from VS Code](#9-compiling-from-vs-code)
10. [Debugging](#10-debugging)
11. [The RPGLE Linter](#11-the-rpgle-linter)
12. [Productivity Tips](#12-productivity-tips)
13. [Source Control with Git](#13-source-control-with-git)

**Part 2: Hands-On Tutorials** — ten progressive coding examples:
- Tutorial 1: Supplier Lookup — Your First Complete Program
- Tutorial 2: Native File I/O — CHAIN
- Tutorial 3: Sequential Reading — Counting Active Suppliers
- Tutorial 4: Service Programs — The Modular RPG Pattern
- Tutorial 5: Batch Processing — The Reorder Candidates Report
- Tutorial 6: Bulletproof Error Handling with MONITOR
- Tutorial 7: Date Math — Time Supply Calculations
- Tutorial 8: Bridging RPG and PHP with XMLSERVICE
- Tutorial 9: CL — The Glue Language
- Tutorial 10: Writing Tests with RPGUnit

14. [Where to Go Next](#14-where-to-go-next)

---

## 1. Why VS Code for RPG

For decades, IBM i developers worked in SEU (Source Entry Utility) — a green-screen editor — or in Rational Developer for i (RDi), IBM's Eclipse-based commercial IDE. Both still exist, but VS Code has become the dominant development environment on the platform for good reason:

- **It's free and cross-platform.** Runs on Windows, macOS, and Linux.
- **It's modern.** Familiar to every developer who has touched JavaScript, Python, or any other mainstream language in the last decade.
- **It's fast.** Significantly lighter weight than RDi.
- **It has first-class IBM support.** As of September 2025, IBM offers an enterprise support tier for the IBM i Extensions for VS Code, backed by Level 1, 2, and 3 support teams.
- **It unifies your stack.** If you're also writing PHP, React, Node.js, Python, or SQL scripts alongside your RPG — and most shops are — you stay in one editor.

The ecosystem is built around the **IBM i Development Pack**, an extension pack maintained jointly by the open-source community and IBM. This is what you install.

---

## 2. Prerequisites

Before starting, make sure you have:

- **IBM i user profile** with authority to the libraries you need, SSH access enabled, and a home directory in the IFS.
- **SSH Daemon started** on the IBM i. Verify with `STRTCPSVR SERVER(*SSHD)` if needed.
- **Network access** from your workstation to the IBM i over port 22 (SSH). If you're behind a corporate firewall, confirm the route is open.
- **Your user profile password** or, better, an **SSH key pair** (more on this below).
- A **library list** that gives you access to the source physical files and compile targets you'll work with.

If you're unsure whether SSH is configured on your partition, ask your system administrator. The `QSHELL` and OpenSSH components must be installed and started; these are standard on modern IBM i releases but occasionally disabled.

---

## 3. Installing VS Code and the IBM i Development Pack

### Step 1 — Install VS Code

Download from [code.visualstudio.com](https://code.visualstudio.com/) and run the installer. On Windows, accept the defaults; the "Add to PATH" option is useful. On macOS, drag to Applications.

### Step 2 — Install the IBM i Development Pack

Open VS Code, press **Ctrl+Shift+X** (or **Cmd+Shift+X** on macOS) to open the Extensions panel, and search for:

```
IBM i Development Pack
```

Click **Install**. This single pack installs the core extensions you need:

| Extension | Purpose |
|---|---|
| **Code for IBM i** | Connect, browse libraries and IFS, compile, run commands |
| **RPGLE Language Tools** | Content assist, outline view, hover info, lint for free-format RPG |
| **IBMi Languages** | Syntax highlighting for RPG, CL, DDS, MI |
| **Db2 for IBM i** | SQL editor with inline results |
| **IBM i Renderer** | Rendering for DSPF/PRTF source |
| **IBM i Project Explorer** | Local project workflow with Source Orbit / BOB |

After installation, restart VS Code. You'll see a new **IBM i** icon appear in the Activity Bar (left sidebar).

### Optional but useful companions

- **IBM i Testing** — run RPGUnit tests inline with inline results.
- **IBM i Code Coverage** — coverage reports for RPG and COBOL.
- **Git Client for IBM i** — Git-over-SSH workflow against IFS directories.

---

## 4. Connecting to Your IBM i

Click the **IBM i** icon in the Activity Bar, then click **Connect to an IBM i**.

You'll be asked for:

- **Name** — a label you choose (e.g., `PROD-LPAR1`, `DEV-AS400`).
- **Host or IP Address** — DNS name or IP of the partition.
- **Port** — typically `22`.
- **Username** — your IBM i user profile.
- **Authentication** — password or SSH private key.

### Password vs. SSH key

Password authentication works, but SSH keys are strongly preferred in production shops:

1. Generate a key pair on your workstation: `ssh-keygen -t ed25519 -C "your.email@k3s.com"`
2. Copy the public key (`~/.ssh/id_ed25519.pub`) to your IBM i home directory at `/home/<YOURUSER>/.ssh/authorized_keys`.
3. Ensure the `.ssh` directory is `700` and `authorized_keys` is `600`.
4. In the VS Code connection dialog, point to the private key file.

Once connected, you'll see three tree views in the IBM i sidebar:

- **User Library List** — your library list as set by your job description.
- **Object Browser** — libraries and objects you've added.
- **IFS Browser** — the Integrated File System, Unix-style.

---

## 5. Understanding the IBM i File Systems

One concept that trips up developers new to IBM i: there are effectively **three places your source code can live**, and the toolchain treats them differently.

### 5.1 Source Physical File Members (QSYS.LIB)

Traditional home of RPG source. A source physical file (e.g., `QRPGLESRC` in library `MYLIB`) holds members, each a separate source file. You'll see these in VS Code as:

```
MYLIB/QRPGLESRC/INVUPD.RPGLE
```

Members have a maximum record length (usually 112) and record-level source dates — both concepts that don't exist in normal file systems.

### 5.2 IFS Stream Files

The IBM i's Unix-style file system. RPG sources here are normal files, typically with a `.rpgle` or `.sqlrpgle` extension. Example path:

```
/home/YOURUSER/projects/inventory/src/invupd.rpgle
```

Modern IBM i development trends toward IFS — it plays well with Git, has no record length limits, and uses standard line endings.

### 5.3 Local Workspace

Source files on your workstation, potentially synced to Git, compiled remotely via tools like **BOB** (Better Object Builder) or **Source Orbit**. This is the fully modern workflow and the direction the IBM i community is heading.

**For learning purposes, start with source members** (option 5.1). It matches the mental model of most IBM i shops today. Graduate to IFS and local projects as you get comfortable.

---

## 6. Your First RPG Program: Hello World

In the IBM i Object Browser, add the library containing your source file (or create one). Right-click a source physical file like `QRPGLESRC` and select **Create Member**.

Name it `HELLO` and give it a type of `RPGLE`. VS Code opens the new member.

Type the following:

```rpgle
**free
ctl-opt dftactgrp(*no) actgrp(*caller)
        option(*nodebugio:*srcstmt)
        main(HelloWorld);

dcl-proc HelloWorld;
  dsply 'Hello, World!';
end-proc;
```

Save with **Ctrl+S**. Now let's break down what you just wrote.

| Line | What it does |
|---|---|
| `**free` | Declares this as a fully free-format source. Mandatory on line 1 for modern RPG. |
| `ctl-opt` | Control options — formerly the H-spec. Controls activation group, compile options, and the program entry procedure. |
| `dftactgrp(*no)` | Don't use the default activation group; required for procedure-based programs. |
| `main(HelloWorld)` | Tells the compiler that `HelloWorld` is the program entry procedure. |
| `dcl-proc` / `end-proc` | Defines a procedure. |
| `dsply` | Legacy "display" operation — writes a message to the job log or interactive screen. |

We'll compile and run this in section 9.

---

## 7. Free-Format RPGLE Fundamentals

Modern RPG is nothing like the fixed-column RPG/400 code you may have seen in old manuals. As of IBM i 7.1 TR7, RPGLE supports **fully free-format source** from column 1, and that's the only style you should be writing today.

### 7.1 Variable Declarations

```rpgle
dcl-s orderNumber   packed(7:0);
dcl-s customerName  varchar(50);
dcl-s orderDate     date;
dcl-s isActive      ind;
dcl-s unitPrice     zoned(9:2);
```

`dcl-s` declares a standalone variable. The types you'll use most:

- **`char(n)`** — fixed-length character
- **`varchar(n)`** — variable-length character (up to 16MB)
- **`packed(n:d)`** — packed decimal, `n` digits total, `d` after decimal
- **`zoned(n:d)`** — zoned decimal (legacy, but still common)
- **`int(10)`** — 4-byte signed integer (`int(3)`, `int(5)`, `int(10)`, `int(20)` for different sizes)
- **`date`**, **`time`**, **`timestamp`** — temporal types
- **`ind`** — indicator, equivalent to boolean

### 7.2 Data Structures

```rpgle
dcl-ds supplier qualified;
  code     char(10);
  name     varchar(50);
  active   ind;
  onboard  date;
end-ds;

supplier.code = 'ACME001';
supplier.name = 'Acme Distributors';
```

The `qualified` keyword forces you to use dot-notation — which is how you should always write structures in new code.

### 7.3 Control Flow

```rpgle
// IF / ELSEIF / ELSE
if qtyOnHand < reorderPoint;
  placeOrder = *on;
elseif qtyOnHand < safetyStock;
  alertBuyer = *on;
else;
  // no action
endif;

// FOR loop
for i = 1 to 10;
  totalValue += lineAmount(i);
endfor;

// DOW (do while)
dow not %eof(INVENTORY);
  read INVENTORY;
  // process
enddo;

// SELECT / WHEN (like switch/case)
select;
  when status = 'A';
    processActive();
  when status = 'P';
    processPending();
  other;
    processUnknown();
endsl;
```

### 7.4 Procedures

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

- `dcl-pi` / `end-pi` — the procedure interface, describing return type and parameters.
- `const` — the parameter is read-only inside the procedure. Always use it unless you specifically need to modify the caller's variable.
- `*n` means "use the same name as the procedure" for the PI.

### 7.5 Built-in Functions

RPG has dozens of built-in functions (BIFs), all prefixed with `%`. Some you'll use constantly:

```rpgle
len = %len(customerName);       // string length
trimmed = %trim(fieldValue);    // trim leading/trailing spaces
upper = %upper(inputText);      // uppercase
num = %dec(textField : 9 : 2);  // convert text to packed decimal
str = %char(numericValue);      // convert numeric to character
today = %date();                // current date
reached = %eof(FILENAME);       // end of file reached?
```

---

## 8. Working with DB2 in SQLRPGLE

In practice, almost all modern RPG programs that touch data use **embedded SQL** rather than native file I/O (READ/CHAIN). Save these programs with the `.sqlrpgle` extension; the compiler preprocesses the SQL.

### 8.1 A Simple Query

This example uses the K3S naming convention where table-prefix fields identify the source (e.g., `SP_` for supplier table):

```rpgle
**free
ctl-opt dftactgrp(*no) option(*nodebugio:*srcstmt) main(LookupSupplier);

dcl-proc LookupSupplier;
  dcl-pi *n;
    suplCode char(10) const;
  end-pi;
  
  dcl-s suplName   varchar(50);
  dcl-s suplActive char(1);
  
  exec sql
    select SP_NAME, SP_ACTIVE
      into :suplName, :suplActive
      from SUPPLIER
      where SP_SUPL = :suplCode;
  
  if sqlstate = '00000';
    dsply ('Supplier: ' + %trim(suplName));
  elseif sqlstate = '02000';
    dsply 'Supplier not found';
  else;
    dsply ('SQL error: ' + sqlstate);
  endif;
end-proc;
```

Key points:

- `:variable` — host variable reference; precede RPG variables with a colon inside SQL statements.
- `sqlstate` — standard 5-character error code. `'00000'` means success; `'02000'` means no row found.
- `sqlcode` — numeric equivalent (negative = error, zero = success, positive = warning).

### 8.2 Cursors for Multiple Rows

```rpgle
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
    // process row
  endif;
enddo;

exec sql close productCursor;
```

### 8.3 Running Ad-Hoc SQL in VS Code

The Db2 for IBM i extension lets you write SQL in a `.sql` file and execute it against your connection. Press **Ctrl+R** (or **Cmd+R**) with your cursor in a statement to run it; results appear inline below the editor. This is enormously faster than STRSQL on a green screen.

Useful prefix hints the extension supports:

- `-- cl:` prefix on a line treats it as a CL command instead of SQL
- `-- json:` returns results as JSON
- `-- csv:` returns results as CSV

---

## 9. Compiling from VS Code

With your source member saved, compile via one of these paths:

### Right-click approach
Right-click the member in the Object Browser and pick **Compile**. Then choose an action such as **CRTBNDRPG** (create bound RPG program) or **CRTSQLRPGI** (create SQL-RPG object).

### Keyboard approach
With the source open in the editor, press **Ctrl+E** (Windows) or **Cmd+E** (macOS) to open the compile menu.

### Errors in the editor

Compile errors and warnings appear inline in the editor with red/yellow squiggles, and the full compile listing is saved to the IFS for inspection. Click an error in the **Problems** panel to jump directly to the offending line — the feature that makes VS Code feel radically better than SEU.

### Customizing compile actions

Compile commands aren't hardcoded; they're stored as JSON-defined "Actions" in your Code for IBM i settings. You can add custom actions for your shop's conventions — for example, a compile action that builds into a specific library with specific parameters for your R7-to-legacy bridge objects.

---

## 10. Debugging

Install the **IBM i Debug** extension (usually already present in the Development Pack). Prerequisites on the partition:

- The IBM i Debug Service must be installed and running (`STRDBGSVR` if using the standard service; newer PTFs use the `IBM i Debug Service`).
- Your compile must be done with debug info. For RPG: `DBGVIEW(*SOURCE)` on the compile command, or set it in your action.

### Setting a breakpoint

Click in the gutter to the left of the line number. A red dot appears. Start the program (from a 5250 session, or by submitting a batch job) — execution halts at your breakpoint and VS Code shows the call stack, variables, and watch expressions in the Debug panel.

### Service Entry Point (SEP) debugging

For debugging procedures called from other jobs (e.g., a module called by a web request), set a Service Entry Point instead of a regular breakpoint. SEPs trigger when any job calls the procedure, attaching the debugger automatically. This is especially useful for debugging RPG procedures called from PHP via XMLSERVICE — a common pattern in K3S-style hybrid architectures.

---

## 11. The RPGLE Linter

The RPGLE Language Tools extension includes a configurable linter. To enable it, run the command **Open RPGLE lint configuration** from the Command Palette (**F1**). If no configuration exists, it creates one at `<LIB>/VSCODE/RPGLINT.JSON`.

Sample configuration:

```json
{
  "indent": 2,
  "RequireBlankSpecial": true,
  "RequiresParameter": true,
  "RequiresProcedureDescription": false,
  "NoOCCURS": true,
  "NoGOTO": true,
  "NoSELECTAll": true,
  "UseCLP_opcode": true,
  "ForceOptionalParens": true,
  "SpecificCasing": [
    { "operation": "*BIFS", "expected": "lower" },
    { "operation": "*DIRECTIVES", "expected": "lower" }
  ]
}
```

Lint errors show as squiggles in the editor, with **Quick Fix** (Ctrl+.) available for most of them. Adopt a shared `rpglint.json` across your team — commit it to Git alongside the source — and you get free style enforcement.

---

## 12. Productivity Tips

A few time-savers worth knowing from day one:

- **Ctrl+P** — Quick file open. Start typing a member name to jump to it.
- **F12** — Go to Definition. Works on procedure names, fields, and copy-included definitions.
- **Shift+F12** — Find all references.
- **Ctrl+Shift+O** — Jump to a procedure or field within the current file.
- **Ctrl+/** — Toggle line comment.
- **Alt+Up / Alt+Down** — Move the current line up/down.
- **Shift+Alt+F** — Format document.
- **Ctrl+Shift+P** → "Focus on Outline View" — shows all procedures, subprocedures, and data structures.
- **Compare with another member** — Right-click a member and choose Compare. Much more capable than SEU's compare.
- **5250 terminal in the editor** — Launch a green-screen session right inside VS Code via the Code for IBM i extension's terminal integration.
- **Notebooks (`.inb` files)** — Mix SQL, CL, Shell, and Markdown into one runnable notebook, with chart output. Useful for reports and tutorials.

---

## 13. Source Control with Git

Two workflows are common:

### IFS + Git Client for IBM i

The **Git Client for IBM i** extension adds Git operations against IFS directories. You develop in `/home/USER/projects/myapp/`, commit from the IBM i side, and push to GitHub/GitLab over HTTPS.

### Local Project + Source Orbit / BOB

The more modern approach. You edit source on your workstation, under Git version control. A build tool (Source Orbit or Better Object Builder) pushes source to IBM i and compiles it. This is what the **IBM i Project Explorer** extension enables.

For a team migrating from pure-QSYS to modern workflows, start with IFS + Git and graduate to local projects as your team gets comfortable.

---

# Part 2: Hands-On Tutorials

These ten tutorials build on each other. Each introduces new concepts while reinforcing earlier ones, and each produces a working program you can actually compile and run. Examples use K3S-style table-prefix field naming throughout (`SP_` for supplier-table fields, `PR_` for product-table fields, `RC_` for reorder-candidate fields) so the convention becomes second nature.

Assumed objects for these tutorials (create them or substitute your own equivalents):

- **`SUPPLIER`** — physical file with fields `SP_SUPL` (char 10, key), `SP_NAME` (varchar 50), `SP_ACTIVE` (char 1), `SP_ONBOARD` (date).
- **`PRODUCT`** — physical file with fields `PR_PROD` (char 15, key), `PR_SUPL` (char 10), `PR_ACTIVE` (char 1), `PR_QOH` (packed 9:0), `PR_REORDPT` (packed 9:0), `PR_SAFTYSTK` (packed 9:0), `PR_PRICE` (packed 9:2), `PR_CHGDT` (date).
- **`REORDCND`** — physical file with record format `RC_REC`, fields `RC_PROD`, `RC_SUPL`, `RC_QOH`, `RC_REORD`, `RC_SUGG`, `RC_CRTDT`.

---

## Tutorial 1: Supplier Lookup — Your First Complete Program

**Goal:** Build a command-line callable RPG program that takes a supplier code, queries DB2, and displays the result. This ties together parameters, embedded SQL, error handling, and free-format syntax.

Create a source member `SUPLKUP` (type `SQLRPGLE`):

```rpgle
**free
//
//  SUPLKUP - Supplier Lookup
//  --------------------------
//  Takes a supplier code as a parameter, writes details to
//  the job log. Demonstrates embedded SQL with SQLSTATE handling.
//
//  Compile:  right-click, choose CRTSQLRPGI
//  Invoke:   CALL SUPLKUP PARM('ACME001   ')
//

ctl-opt dftactgrp(*no) option(*nodebugio:*srcstmt);

dcl-pi *n;
  inSuplCode char(10) const;
end-pi;

dcl-s suplName    varchar(50);
dcl-s suplActive  char(1);
dcl-s onboardDate date;
dcl-s msg         varchar(100);

exec sql
  select SP_NAME, SP_ACTIVE, SP_ONBOARD
    into :suplName, :suplActive, :onboardDate
    from SUPPLIER
   where SP_SUPL = :inSuplCode;

select;
  when sqlstate = '00000';
    msg = 'Found: ' + %trim(suplName)
        + ' (Active=' + suplActive
        + ', Onboard=' + %char(onboardDate) + ')';
  when sqlstate = '02000';
    msg = 'Supplier ' + %trim(inSuplCode) + ' not found';
  other;
    msg = 'SQL error ' + sqlstate;
endsl;

dsply msg;

*inlr = *on;
return;
```

**What's new here:**

- `dcl-pi *n` declares the program's entry parameter interface — the modern equivalent of the old `*ENTRY PLIST`. Because this is a top-level program, `*n` means "use the same name as the program."
- `*inlr = *on` closes files and releases resources before returning. Always set this on program exit unless you specifically need the program to stay active between calls.
- `SP_SUPL`, `SP_NAME`, `SP_ACTIVE`, `SP_ONBOARD` follow the K3S convention where the prefix tells you which table owns the field at a glance.

**To test from a 5250 session:**

```
CALL SUPLKUP PARM('ACME001   ')
```

Trailing spaces matter — command-line parameters must match the declared length exactly. The `DSPLY` message appears in the interactive session; press Enter to dismiss.

---

## Tutorial 2: Native File I/O — CHAIN

**Goal:** Access DB2 data without SQL. You'll encounter plenty of this pattern in legacy code, and it's occasionally faster for simple single-record lookups by key.

Assume a logical file `SUPPLR01` keyed on `SP_SUPL`. Create member `SUPLKUP2` (type `RPGLE`):

```rpgle
**free
//
//  SUPLKUP2 - Supplier Lookup using CHAIN
//  ---------------------------------------
//  Same result as SUPLKUP but with native file I/O.
//

ctl-opt dftactgrp(*no) option(*nodebugio:*srcstmt);

dcl-f SUPPLR01 keyed usage(*input);

dcl-pi *n;
  inSuplCode char(10) const;
end-pi;

dcl-s msg varchar(100);

chain inSuplCode SUPPLR01;

if %found(SUPPLR01);
  msg = 'Found: ' + %trim(SP_NAME)
      + ' (Active=' + SP_ACTIVE + ')';
else;
  msg = 'Supplier ' + %trim(inSuplCode) + ' not found';
endif;

dsply msg;

*inlr = *on;
return;
```

**What's new here:**

- `dcl-f` declares a file and opens it at program start. `keyed` means access by key, `usage(*input)` means read-only.
- `chain` positions the file cursor at the matching record and reads it. The record's fields become available as program variables automatically — that's why `SP_NAME` and `SP_ACTIVE` just work without being declared.
- `%found(filename)` tells you whether the chain succeeded. Always check this; never assume.

**Why SQL is usually preferred:** SQL lets the database engine optimize access paths; it handles joins, aggregations, and sorts far more concisely; and it's portable. Use native I/O for simple by-key reads and for maintaining legacy code that already uses it.

---

## Tutorial 3: Sequential Reading — Counting Active Suppliers

**Goal:** Loop through a file, applying logic to each record.

```rpgle
**free
//
//  SUPLCNT - Count active vs inactive suppliers
//

ctl-opt dftactgrp(*no) option(*nodebugio:*srcstmt);

dcl-f SUPPLIER usage(*input);

dcl-s activeCount int(10);
dcl-s totalCount  int(10);
dcl-s msg         varchar(100);

dow not %eof(SUPPLIER);
  read SUPPLIER;
  if %eof(SUPPLIER); leave; endif;

  totalCount += 1;
  if SP_ACTIVE = 'Y';
    activeCount += 1;
  endif;
enddo;

msg = 'Active: ' + %char(activeCount)
    + ' of ' + %char(totalCount) + ' total';
dsply msg;

*inlr = *on;
return;
```

**What's new here:**

- `read` advances the file cursor and reads the next record. After the last record, `%eof(filename)` becomes `*on`.
- `leave` exits the current loop immediately; `iter` skips to the next iteration (you'll see this in Tutorial 5).
- `+=` works on numeric fields exactly as in any modern language.

**SQL equivalent:** You could replace the entire loop with a single query:

```sql
SELECT COUNT(*) AS TOTAL,
       SUM(CASE WHEN SP_ACTIVE = 'Y' THEN 1 ELSE 0 END) AS ACTIVE
  FROM SUPPLIER;
```

One statement, delegated to the database engine. Almost always the right call for counts, sums, or joins.

---

## Tutorial 4: Service Programs — The Modular RPG Pattern

**Goal:** Build reusable procedures that live in a service program, callable from any other program. This is how serious RPG is written today — small, testable exported procedures rather than monolithic programs.

**Step 1: The prototype file (header)**

Create member `SUPLSVR_H` (type `RPGLEINC`) in `QCPYLESRC`:

```rpgle
**free
//
//  SUPLSVR_H - Prototypes for SUPLSVR service program
//

dcl-pr GetSupplierName varchar(50);
  suplCode char(10) const;
end-pr;

dcl-pr IsSupplierActive ind;
  suplCode char(10) const;
end-pr;

dcl-pr CountActiveSuppliers int(10) end-pr;
```

**Step 2: The service program source**

Create member `SUPLSVR` (type `SQLRPGLE`):

```rpgle
**free
//
//  SUPLSVR - Supplier service procedures
//  --------------------------------------
//  Compile:  CRTSQLRPGI OBJ(K3SLIB/SUPLSVR) OBJTYPE(*MODULE)
//  Bind:     CRTSRVPGM SRVPGM(K3SLIB/SUPLSVR)
//                      MODULE(SUPLSVR) EXPORT(*ALL)
//

ctl-opt nomain option(*nodebugio:*srcstmt);

/copy QCPYLESRC,SUPLSVR_H

dcl-proc GetSupplierName export;
  dcl-pi *n varchar(50);
    suplCode char(10) const;
  end-pi;

  dcl-s name varchar(50);

  exec sql
    select SP_NAME into :name
      from SUPPLIER
     where SP_SUPL = :suplCode;

  if sqlstate <> '00000';
    name = '';
  endif;

  return name;
end-proc;

dcl-proc IsSupplierActive export;
  dcl-pi *n ind;
    suplCode char(10) const;
  end-pi;

  dcl-s active char(1);

  exec sql
    select SP_ACTIVE into :active
      from SUPPLIER
     where SP_SUPL = :suplCode;

  return (sqlstate = '00000' and active = 'Y');
end-proc;

dcl-proc CountActiveSuppliers export;
  dcl-pi *n int(10) end-pi;
  dcl-s cnt int(10);

  exec sql
    select count(*) into :cnt
      from SUPPLIER
     where SP_ACTIVE = 'Y';

  return cnt;
end-proc;
```

**What's new here:**

- `ctl-opt nomain` declares this is a service-program source with no main entry — only callable procedures.
- `dcl-proc ... export` marks a procedure as callable from outside this module.
- `/copy` includes the prototype file so callers agree on the interface.

**Step 3: Using it from another program**

```rpgle
**free
ctl-opt dftactgrp(*no) bnddir('SUPLBND');

/copy QCPYLESRC,SUPLSVR_H

dcl-s name  varchar(50);
dcl-s count int(10);
dcl-s msg   varchar(100);

name  = GetSupplierName('ACME001');
count = CountActiveSuppliers();

msg = 'Name: ' + %trim(name)
    + ', Active total: ' + %char(count);
dsply msg;

*inlr = *on;
return;
```

`bnddir('SUPLBND')` tells the compiler to look in the `SUPLBND` binding directory for service programs containing the referenced procedures. Create the binding directory with `CRTBNDDIR` and add entries with `ADDBNDDIRE`.

**Why this matters for K3S:** Service programs are the clean way to expose RPG business logic to PHP, Node.js, or the R7 Mezzio backend. Each exported procedure becomes a candidate for an XMLSERVICE call or a stored-procedure wrapper.

---

## Tutorial 5: Batch Processing — The Reorder Candidates Report

**Goal:** Real-world business logic. Read through the product master, identify items below reorder point, and write candidates to an output file.

```rpgle
**free
//
//  REORDCAND - Build reorder candidates
//  ------------------------------------
//  Reads PRODUCT, writes candidates to REORDCND.
//  Simplified example; production K3S logic uses
//  Dynamic Exponential Smoothing and seasonality profiles.
//

ctl-opt dftactgrp(*no) option(*nodebugio:*srcstmt);

dcl-f PRODUCT  usage(*input);
dcl-f REORDCND usage(*output);

dcl-s candidatesCount int(10);
dcl-s suggestedQty    packed(9:0);
dcl-s msg             varchar(100);

// Clear prior candidates in one DB operation
exec sql delete from REORDCND;

dow not %eof(PRODUCT);
  read PRODUCT;
  if %eof(PRODUCT); leave; endif;

  // Skip inactive products
  if PR_ACTIVE <> 'Y'; iter; endif;

  // Is quantity on hand below reorder point?
  if PR_QOH < PR_REORDPT;
    suggestedQty = PR_REORDPT + PR_SAFTYSTK - PR_QOH;

    // Populate and write the output record
    clear RC_REC;
    RC_PROD  = PR_PROD;
    RC_SUPL  = PR_SUPL;
    RC_QOH   = PR_QOH;
    RC_REORD = PR_REORDPT;
    RC_SUGG  = suggestedQty;
    RC_CRTDT = %date();

    write RC_REC;
    candidatesCount += 1;
  endif;
enddo;

msg = %char(candidatesCount) + ' reorder candidates written';
dsply msg;

*inlr = *on;
return;
```

**What's new here:**

- `iter` skips to the next loop iteration without leaving the loop.
- `clear` resets all fields of a record format or structure to zeros/blanks — essential before populating output records so stale data from prior writes doesn't leak through.
- Output file writes use the **record format name** (`RC_REC`), not the file name.
- Mixing SQL and native I/O in the same program is perfectly legal. Here we use SQL to clear the file once (fast, handles journaling properly) and native I/O for record-by-record writes.

**Supply chain note:** Real K3S reorder logic is substantially more sophisticated — exponential smoothing on demand, seasonality profiles, lead time variability, supplier order cycles, PE checks, multi-echelon considerations. The loop structure stays the same; the math inside the `if` gets richer.

---

## Tutorial 6: Bulletproof Error Handling with MONITOR

**Goal:** Wrap risky operations so unexpected errors don't crash your program.

```rpgle
**free
ctl-opt nomain;

dcl-proc SafeDivide export;
  dcl-pi *n packed(9:4);
    numerator   packed(9:2) const;
    denominator packed(9:2) const;
  end-pi;

  dcl-s result packed(9:4);

  monitor;
    result = numerator / denominator;
  on-error *zero-divide;
    result = 0;
  on-error;
    // Any other arithmetic error
    result = 0;
  endmon;

  return result;
end-proc;

dcl-proc ParseDecimal export;
  dcl-pi *n packed(9:2);
    input varchar(20) const;
  end-pi;

  dcl-s result packed(9:2);

  monitor;
    result = %dec(input : 9 : 2);
  on-error;
    // Non-numeric input — return zero rather than crash
    result = 0;
  endmon;

  return result;
end-proc;
```

**What's new here:**

- `monitor` / `on-error` / `endmon` blocks work like `try` / `catch` / `end` in other languages.
- You can specify status codes like `*zero-divide` (`00102`), ranges, or use a bare `on-error` to catch anything.
- `%status` and `%error` give diagnostic info inside an on-error block.

**A real-world pattern — SQL update with rollback:**

```rpgle
dcl-proc UpdatePriceWithRollback export;
  dcl-pi *n ind;
    prodCode char(15) const;
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

---

## Tutorial 7: Date Math — Time Supply Calculations

**Goal:** Practical supply chain quantities using date arithmetic. Time Supply, reorder cycles, and overdue detection all come down to date math.

```rpgle
**free
//
//  DATEUTIL - Date calculation service procedures
//

ctl-opt nomain;

// Time Supply = QOH / avg_daily_demand
// Expressed as days of inventory remaining
dcl-proc CalcTimeSupply export;
  dcl-pi *n packed(7:2);
    qoh            packed(9:0) const;
    avgDailyDemand packed(9:4) const;
  end-pi;

  dcl-s days packed(7:2);

  monitor;
    if avgDailyDemand > 0;
      days = qoh / avgDailyDemand;
    else;
      days = 9999; // effectively infinite
    endif;
  on-error;
    days = 0;
  endmon;

  return days;
end-proc;

// Next scheduled order based on supplier order cycle
dcl-proc NextReorderDate export;
  dcl-pi *n date;
    lastOrderDate   date  const;
    cycleLengthDays int(10) const;
  end-pi;

  return lastOrderDate + %days(cycleLengthDays);
end-proc;

// Days between two dates (positive if toDate is later)
dcl-proc DaysBetween export;
  dcl-pi *n int(10);
    fromDate date const;
    toDate   date const;
  end-pi;

  return %diff(toDate : fromDate : *days);
end-proc;

// Is a scheduled delivery past its tolerance window?
dcl-proc IsOrderOverdue export;
  dcl-pi *n ind;
    expectedDate  date   const;
    toleranceDays int(5) const;
  end-pi;

  return %date() > expectedDate + %days(toleranceDays);
end-proc;
```

**What's new here:**

- `%date()` returns today's date.
- `%days(n)`, `%months(n)`, `%years(n)` produce duration values you can add to or subtract from dates.
- `%diff(d1 : d2 : unit)` returns the difference between two dates in the specified unit (`*days`, `*months`, `*years`, etc.).
- Date arithmetic is type-safe — the compiler catches attempts to add a raw number to a date directly. You must go through `%days()`.

---

## Tutorial 8: Bridging RPG and PHP with XMLSERVICE

**Goal:** Expose an RPG procedure so the Mezzio/PHP R7 backend can call it over XMLSERVICE. This is the integration pattern that keeps legacy DB2/RPG logic available to the modern frontend.

**Step 1: RPG side — a program designed for toolkit calls**

Create `GETSUPL` (type `SQLRPGLE`):

```rpgle
**free
//
//  GETSUPL - Supplier lookup for toolkit callers
//  ---------------------------------------------
//  Fixed-length character parameters only.
//  No null indicators. All I/O explicitly marked.
//

ctl-opt dftactgrp(*no) option(*nodebugio:*srcstmt);

dcl-pi *n;
  inSuplCode char(10) const;  // input
  outName    char(50);        // output
  outActive  char(1);         // output (Y/N)
  outOnboard char(10);        // output (YYYY-MM-DD)
  outStatus  char(10);        // output (OK/NOTFOUND/ERROR)
end-pi;

dcl-s onboardDate date;

outName    = *blanks;
outActive  = 'N';
outOnboard = *blanks;
outStatus  = 'ERROR';

exec sql
  select SP_NAME, SP_ACTIVE, SP_ONBOARD
    into :outName, :outActive, :onboardDate
    from SUPPLIER
   where SP_SUPL = :inSuplCode;

select;
  when sqlstate = '00000';
    outOnboard = %char(onboardDate : *iso);
    outStatus  = 'OK';
  when sqlstate = '02000';
    outStatus  = 'NOTFOUND';
  other;
    outStatus  = 'ERROR';
endsl;

*inlr = *on;
return;
```

**Step 2: PHP side — calling via the Zend/ibm_db2 Toolkit**

```php
<?php
use ToolkitApi\Toolkit;

$tk = new Toolkit($dbConn, $schema, $user, $password, 'iPDO');
$tk->setOptions(['stateless' => true]);

$params = [
    $tk->AddParameterChar('both', 10, 'inSuplCode', 'inSuplCode', 'ACME001'),
    $tk->AddParameterChar('both', 50, 'outName',    'outName',    ''),
    $tk->AddParameterChar('both',  1, 'outActive',  'outActive',  ''),
    $tk->AddParameterChar('both', 10, 'outOnboard', 'outOnboard', ''),
    $tk->AddParameterChar('both', 10, 'outStatus',  'outStatus',  ''),
];

$result = $tk->PgmCall('GETSUPL', 'K3SLIB', $params);

if ($result && trim($result['io_param']['outStatus']) === 'OK') {
    echo "Name:    " . trim($result['io_param']['outName'])    . "\n";
    echo "Active:  " . trim($result['io_param']['outActive'])  . "\n";
    echo "Onboard: " . trim($result['io_param']['outOnboard']) . "\n";
} else {
    echo "Lookup failed: " . trim($result['io_param']['outStatus'] ?? 'ERROR');
}
```

**Gotchas worth internalizing:**

- **Parameter lengths must match exactly.** If the RPG declares `char(10)` and PHP sends more or fewer bytes, XMLSERVICE will either truncate silently or fail mysteriously.
- **CCSID 65535 members.** Older source files sometimes default to CCSID 65535 (binary, no translation). PHP can't read these correctly. Set CCSID 37 (or whatever matches your job) on both source members and any relevant tables.
- **Stateless vs. stateful.** Stateless is simpler but pays the cost of a job start per call. Stateful caches a persistent job — much faster, but you must manage cleanup.
- **Return status explicitly.** Never rely on empty return values to signal "not found." Always include a dedicated status field like `outStatus` above so the PHP side can distinguish outcomes unambiguously.

---

## Tutorial 9: CL — The Glue Language

**Goal:** You can't escape CL on IBM i. It's the glue that sets library lists, overrides files, clears working tables, and schedules batch jobs. Worth knowing just enough to be productive.

**A complete nightly job:**

```cl
/*  NIGHTLYRUN - Nightly reorder processing  */
             PGM

             ADDLIBLE   LIB(K3SLIB)
             ADDLIBLE   LIB(K3SDTA)

             /*  Route the report to the overnight print queue  */
             OVRPRTF    FILE(REORDRPT) OUTQ(PRINT01) HOLD(*YES)

             /*  Clear any prior candidates  */
             CLRPFM     FILE(K3SDTA/REORDCND)

             /*  Run the batch RPG program  */
             CALL       PGM(K3SLIB/REORDCAND)

             /*  Log completion  */
             SNDPGMMSG  MSG('Nightly reorder candidates refreshed.')

             DLTOVR     FILE(*ALL)

             ENDPGM
```

Save as `NIGHTLYRUN` (type `CLLE` — the modern Integrated Language Environment CL).

**Calling an RPG program with parameters:**

```cl
             PGM        PARM(&SUPL)

             DCL        VAR(&SUPL) TYPE(*CHAR) LEN(10)

             CALL       PGM(SUPLKUP) PARM(&SUPL)

             ENDPGM
```

**Calling a CL command from RPG via QCMDEXC:**

```rpgle
dcl-pr RunCommand extproc('QCMDEXC');
  command char(500) const options(*varsize);
  length  packed(15:5) const;
end-pr;

RunCommand('CLRPFM K3SDTA/REORDCND' : 24);
RunCommand('OVRDBF FILE(CUSTMAST) TOFILE(K3SDTA/CUSTARCH)' : 47);
```

**CL commands you'll see constantly:**

| Command | Purpose |
|---|---|
| `ADDLIBLE` / `RMVLIBLE` | Add/remove a library from the list |
| `CHGCURLIB` | Change the current library |
| `OVRDBF` / `DLTOVR` | Override/remove a file override |
| `CLRPFM` | Clear all records from a physical file |
| `SBMJOB` | Submit a job to batch |
| `DSPJOBLOG` | Display the job log |
| `WRKSPLF` | Work with spooled files |
| `WRKACTJOB` | Work with active jobs |
| `SNDPGMMSG` | Send a message to the job log or user |
| `CPYF` | Copy file (often used for data migration) |

---

## Tutorial 10: Writing Tests with RPGUnit

**Goal:** Automated tests for your service program procedures. These patterns work with the **IBM i Testing** VS Code extension.

Install the **IBM i Testing** extension from the marketplace if it isn't already present. Create test member `TSUPLSVR` (type `SQLRPGLE`):

```rpgle
**free
//
//  TSUPLSVR - Unit tests for SUPLSVR service program
//

ctl-opt nomain;

/copy QCPYLESRC,SUPLSVR_H
/include QINCLUDE,TESTCASE

dcl-proc setUpSuite export;
  // Runs once before all tests in this suite
  exec sql
    insert into SUPPLIER
      (SP_SUPL, SP_NAME, SP_ACTIVE, SP_ONBOARD)
    values
      ('TEST001',    'Test Supplier One', 'Y', '2024-01-15'),
      ('TEST002',    'Test Supplier Two', 'N', '2023-06-01');
end-proc;

dcl-proc tearDownSuite export;
  // Runs once after all tests
  exec sql
    delete from SUPPLIER where SP_SUPL like 'TEST%';
end-proc;

dcl-proc testGetSupplierName_found export;
  dcl-s name varchar(50);
  name = GetSupplierName('TEST001');
  aEqual('Test Supplier One' : %trim(name));
end-proc;

dcl-proc testGetSupplierName_notFound export;
  dcl-s name varchar(50);
  name = GetSupplierName('NOTEXIST');
  aEqual('' : %trim(name));
end-proc;

dcl-proc testIsSupplierActive_yes export;
  aIsTrue(IsSupplierActive('TEST001'));
end-proc;

dcl-proc testIsSupplierActive_no export;
  aIsFalse(IsSupplierActive('TEST002'));
end-proc;

dcl-proc testCountActiveSuppliers_includesTest export;
  dcl-s cnt int(10);
  cnt = CountActiveSuppliers();
  aIsTrue(cnt >= 1);
end-proc;
```

**What's new here:**

- Test procedures follow a `testXxx` naming convention; the framework discovers them automatically by prefix.
- `setUpSuite` / `tearDownSuite` wrap the entire test file. `setUp` / `tearDown` (without `Suite`) wrap each individual test.
- Assertions use RPGUnit helpers: `aEqual`, `aIsTrue`, `aIsFalse`, `aNotEqual`.

**Running tests from VS Code:** With the IBM i Testing extension installed, a beaker icon appears next to each `testXxx` procedure. Click to run that specific test, or run the whole suite from the testing sidebar. Failures show inline with the assertion message, and pass/fail counts appear in the status bar.

**Tie this to CI/CD:** The companion testing CLI runs in PASE, so you can wire it into Jenkins, GitHub Actions, or any pipeline that can SSH to IBM i. Run tests on every commit; block merges when tests fail. This is how you bring RPG development under the same quality bar as your PHP and React code.

---

## Bringing It Together

Work through these ten tutorials in order and a new team member should emerge with:

- Confidence in the VS Code + Code for IBM i workflow
- Fluency in modern free-format RPG syntax
- Comfort with both embedded SQL and native file I/O
- An understanding of modular service-program architecture
- Working knowledge of the PHP/RPG bridge pattern
- Enough CL to navigate any shop
- A testing discipline that scales to CI/CD

The rest comes with reps, reading other people's code, and shipping real work.

---

## 14. Where to Go Next

- **Official docs** — `https://codefori.github.io/docs/` — the canonical reference for Code for IBM i.
- **IBM's RPG Programmer's Guide** — for the language itself. IBM publishes it free online.
- **Scott Klement's articles** — the best writing on modern RPG you'll find anywhere. Search his name on IT Jungle or iProDeveloper archives.
- **IBM i Sandbox** — free one-day sandbox access to test Code for IBM i against a real partition without touching production.
- **Common Europe Congress / COMMON NAUG** — conferences where the Code for IBM i maintainers present new features.

### Suggested exercises for new team members

1. Write a procedure that reads a supplier code, queries the `SUPPLIER` table, and displays the name and onboarding date.
2. Convert a small fixed-format RPG program your team maintains to fully free-format. (The RPGLE Language Tools extension can help with the mechanical parts.)
3. Write an RPG procedure that's callable from PHP via XMLSERVICE, set a Service Entry Point breakpoint, and trigger it from a browser request.
4. Build a `.inb` notebook that runs a DB2 query, charts the result, and exports to HTML — a useful tool for quick reports.

---

*This tutorial is intended as a starting point. The IBM i development ecosystem moves quickly, and the Code for IBM i extension receives updates roughly every two to three weeks. Keep the extension updated and watch the release notes for new capabilities.*
