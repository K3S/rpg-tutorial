---
title: "7. Hello World"
parent: "Part 1: Get Set Up"
nav_order: 7
---

# Hello World

Your first RPG program. We'll write it, compile it, and run it. That's the end of Part 1.

## Where RPG source lives

A small detour. On Windows or macOS, source code is just files in folders. On IBM i, there's a traditional layout that's still the most common place to put source:

- **Libraries** contain objects
- A **source physical file** is an object that contains **members**
- Each **member** is a single source file — `HELLO.RPGLE`, `SUPLKUP.SQLRPGLE`, etc.

Source physical files are conventionally named by language: `QRPGLESRC` for RPG source, `QCLSRC` for CL, `QDDSSRC` for DDS, and so on. You'll create one in your library to hold your source going forward.

(Modern workflows using Git and the IFS let you skip this structure, but learning it first is easier — it's the world you'll encounter in most shops.)

## Create a source physical file

In VS Code, open the Command Palette (**F1**) and search for "IBM i: Run CL Command." Or find a 5250 session however you prefer (there's a green-screen option in VS Code too).

Run:

CRTSRCPF FILE(YOUR_LIBRARY/QRPGLESRC) RCDLEN(112)

Replace `YOUR_LIBRARY` with the same library name you used in Chapter 6. `RCDLEN(112)` is the traditional record length that fits fixed-format RPG. Free-format source will use much less but you want compatibility.

You should see a message that the file was created.

## Create the HELLO member

In the VS Code IBM i sidebar, expand your library in the Object Browser. You should see your new `QRPGLESRC` file.

Right-click `QRPGLESRC` and choose **Create Member**. Fill in:

- **Name:** `HELLO`
- **Type:** `RPGLE`
- **Text description:** `Hello World`

VS Code opens the empty member. Paste this in:

```rpgle
**free
ctl-opt dftactgrp(*no) actgrp(*caller)
        option(*nodebugio:*srcstmt)
        main(HelloWorld);

dcl-proc HelloWorld;
  dsply 'Hello, World!';
end-proc;
```

Save with **Ctrl+S** / **Cmd+S**.

Quick line-by-line:

- `**free` — first line tells the compiler this is fully free-format RPGLE. Required.
- `ctl-opt` — the control options. `dftactgrp(*no)` opts out of the default activation group (required for modern procedure-based programs). `main(HelloWorld)` tells the compiler that `HelloWorld` is the program's entry procedure.
- `dcl-proc` / `end-proc` — declare a procedure.
- `dsply` — a legacy "display message" operation that writes a line to the job log (or interactive screen if you're on a 5250).

## Compile

Right-click the member in the Object Browser and choose **Compile**. Pick **CRTBNDRPG** (Create Bound RPG Program) when asked which compile action to use.

Watch the bottom status bar. You'll see "Compiling..." briefly. When it finishes, one of two things happens:

- **Success** — a green message, and a new `*PGM` object named `HELLO` appears in your library
- **Error** — a red message; the Problems tab in VS Code shows what went wrong with line numbers you can click through to

If you got errors, the most common cause is a typo in the source. Fix and recompile.

## Run it

In a 5250 session (SSH into PUB400 with a terminal emulator or use the 5250 option in VS Code), type:

CALL HELLO

You'll see `DSPLY  Hello, World!` on the screen. Press Enter to dismiss.

You can also run it from the VS Code IBM i terminal by issuing the same command via the "Run Command" action.

## You finished Part 1

You have a free IBM i account, a configured VS Code environment, a set of practice tables, and a compiled RPG program that ran successfully. That's the foundation.

Part 2 teaches the language itself. Part 3 is ten tutorials that build real programs on the tables you just created. Part 4 points to the broader RPG community's resources for deeper learning.

Take a break. Part 2 is waiting.
