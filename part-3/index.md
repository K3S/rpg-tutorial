---
title: "Part 3: Build Real Things"
layout: default
nav_order: 4
has_children: true
permalink: /part-3/
---

# Part 3: Build Real Things

Part 2 taught the language. Part 3 puts it to work.

Each tutorial here builds a small, realistic RPG program on the PRODUCT, SUPPLIER, and REORDCND tables you created in Part 1. The programs aren't toys — they're the shape of real work you'd do in an IBM i shop: look things up, iterate through records, calculate, write results back, handle errors, test your work.

The tutorials are progressive. Later ones assume you've internalized the earlier ones. Work through them in order the first time. Come back to individual ones as reference after that.

## The twelve tutorials

**Reading data**

1. **[Product Lookup and Price Calculation]({% link part-3/01-product-lookup.md %})** — your first program that takes parameters, reads the database, applies business logic
2. **[Reading Records with CHAIN]({% link part-3/02-chain-io.md %})** — native random-access I/O, and when to use it instead of SQL
3. **[Sequential Reading with SETLL and READ]({% link part-3/03-sequential-read.md %})** — walking a file one record at a time
4. **[Multi-Row Queries with SQL Cursors]({% link part-3/04-sql-cursors.md %})** — the modern equivalent, plus the `extname` DS trick

**Building real programs**

5. **[Building Reusable Logic with Procedures]({% link part-3/05-procedures.md %})** — structuring code as main + helpers
6. **[Service Programs]({% link part-3/06-service-programs.md %})** — packaging procedures for reuse across programs
7. **[Batch Reorder Report — The Capstone]({% link part-3/07-batch-reorder.md %})** — everything so far, applied to a real batch job

**Operational quality**

8. **[Error Handling with MONITOR]({% link part-3/08-monitor-errors.md %})** — graceful handling of bad data and SQL failures
9. **[Date Math in Practice]({% link part-3/09-date-math-practice.md %})** — a stale-price audit program
10. **[CL — The Glue Language]({% link part-3/10-cl-glue.md %})** — wrappers, parameter passing, job scheduling

**Capstone and confidence**

11. **[Roman Numeral Converter — A Procedures Capstone]({% link part-3/11-roman-numeral.md %})** — a pure-algorithm exercise in procedural composition
12. **[Writing Tests with RPGUnit]({% link part-3/12-rpgunit-tests.md %})** — automated testing for the programs you've built

## Tutorial format

Every tutorial in Part 3 follows the same structure:

1. **What you'll build** — one paragraph describing the program and its inputs/outputs
2. **What you'll learn** — the concepts this tutorial anchors
3. **The program** — the full source code, ready to compile
4. **Walkthrough** — section-by-section explanation of what the code does and why
5. **Compile and run** — how to get it running on your PUB400 account
6. **Try this** — extensions to deepen your understanding
7. **What's next** — link to the next tutorial

## Before you start

- Part 1 complete — you're connected to PUB400 and your practice tables exist
- Part 2 read through at least Chapter 5 — the tutorials here assume you can read embedded SQL
- Your library name from Part 1 (you'll substitute it in a few places)

Ready? Start with [Tutorial 1: Product Lookup and Price Calculation]({% link part-3/01-product-lookup.md %}).

---

*Many programs in Part 3 are adapted from a K3S internal training curriculum written by Mike Scampini and Lauren Brakke. Their originals have been re-worked for the PUB400 environment and the public tutorial schema, but the teaching shape of the code is theirs.*
