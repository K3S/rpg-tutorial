---
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
