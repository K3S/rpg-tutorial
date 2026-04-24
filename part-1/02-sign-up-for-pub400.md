---
title: "2. Sign up for PUB400"
parent: "Part 1: Get Set Up"
nav_order: 2
---

# Sign up for PUB400

To write and run RPG code, you need an IBM i to connect to. You don't have one. IBM doesn't sell a desktop version. Cloud providers offer IBM i, but it's enterprise-priced and not where you want to start.

**PUB400.com** is a free, public IBM i server that has been running since 1999, maintained by Holger Scherer in Germany. It exists for exactly this purpose: people learning IBM i. Over 70,000 users have passed through it over the years.

What you get:

- A personal user profile with PGMR-level access
- **Two private libraries** — your code and data live here
- **500 MB of disk storage**
- Full access to RPG, CL, SQL, COBOL, and open-source tooling (including Node.js)
- The current OS — IBM i 7.5, running on real Power hardware

What you can't do:

- Commercial workloads — this is for learning only
- Create more than one account per person
- Install licensed program products that aren't part of the base

## What to do

Go to [pub400.com](https://pub400.com) and follow their signup process. Their site walks you through it; we won't duplicate their instructions here because they maintain them and we don't.

A few things to capture as you go, because you'll need them in Chapter 5:

- Your **username** (user profile name)
- Your **password**
- The **hostname** for SSH connections (it's `pub400.com`)
- The **port** (standard SSH — 22)

Read their terms carefully. They're brief and reasonable, and they keep the service sustainable for everyone.

## When you're done

You have a PUB400 account, you've noted your credentials, and you can keep moving. Next up: [Chapter 3: Install VS Code]({% link part-1/03-install-vs-code.md %}).
