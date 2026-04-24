---
title: "1. What you're learning"
parent: "Part 1: Get Set Up"
nav_order: 1
---

# What you're learning

Before the installers, a few paragraphs of orientation. If you already know what IBM i and RPG are, skim the first two sections and move on.

## What IBM i is

IBM i is an operating system that runs on IBM Power hardware. It's not Linux. It's not Windows. It's not Unix, though you can open a Unix-style shell inside it. Its lineage goes back to the AS/400, introduced in 1988, which itself inherited concepts from the System/38 from 1978.

The platform is distinguished by a few ideas that were ahead of their time and remain unusual today. The database is part of the operating system — you don't install DB2 on IBM i, you just *have* it. Security is object-level and integrated with user profiles; there's no filesystem permission model to paper over. Work is organized into *jobs*, each with its own library list, journaling, and priority. Programs compile to machine-independent bytecode that the OS translates once to native Power instructions, which is why an RPG program compiled for a 1995 AS/400 will still run on a 2026 Power10.

IBM i hosts the core business systems of a surprisingly large share of the Fortune 500: banks, insurers, manufacturers, distributors, retailers. Most of that code is RPG.

## What RPG is

RPG started in 1959 as "Report Program Generator." The old fixed-format RPG you might find in ancient IBM documentation is unrecognizable compared to the modern language. Since 2013, IBM i supports **fully free-format RPGLE** — a statically-typed, compiled, procedural language that, syntactically, looks closer to Pascal or Ada than to its own ancestors.

Modern RPG has procedures, data structures, parameter passing by reference or value, exception handling, embedded SQL, date/time types with compiler-enforced arithmetic, and clean integration with the platform's DB2 database. It's verbose but readable. It's fast — RPG compiled programs routinely process millions of database rows per second on modest hardware. It doesn't have closures, generics, or async. That's the tradeoff.

**CL (Control Language)** is the scripting and glue language of the platform. You use it to set library lists, submit batch jobs, override files, and schedule work. You can't avoid it on IBM i, but you only need enough to be productive.

## Why learn this in 2026

Three reasons.

**The installed base is enormous and not going anywhere.** IBM i shops tend to keep their systems for decades — there are businesses running RPG code written in the 1990s that still processes their daily order flow. Major industries structurally depend on this platform.

**The workforce is retiring faster than it's being replaced.** Average RPG developer age is well above 50. Shops that can't find modern RPG talent pay a premium for it.

**It's interesting work.** You touch the core logic of how a business actually operates — inventory, pricing, orders, supply chain, settlement. Not UI layers built on top of other UI layers.

## What this tutorial covers

This is a four-part journey.

**Part 1** — where you are now — gets you from zero to running your first RPG program on a real IBM i.

**Part 2** teaches the language: free-format RPGLE syntax, embedded SQL, procedures, error handling, date math. The 80% of RPG you use daily.

**Part 3** is ten progressive hands-on tutorials that build real programs — lookups, batch processing, service programs, testing — using a consistent set of practice tables so the examples build on each other.

**Part 4** points to the deeper resources maintained by the broader RPG community: Scott Klement's writing, IBM's reference guides, the Code for IBM i documentation, conferences, and forums. Once you've worked through Parts 1–3, you don't need another tutorial. You need a map of where to go next.

Ready? Move on to [Chapter 2: Sign up for PUB400]({% link part-1/02-sign-up-for-pub400.md %}).
