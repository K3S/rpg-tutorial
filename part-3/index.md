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

## What you'll build

Tutorial count will grow as the site does. Each one produces a working program you can run against the practice data.

## Tutorial format

Every tutorial in Part 3 follows the same structure so you know what to expect:

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
