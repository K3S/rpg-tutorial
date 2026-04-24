---
title: "5. Connect to PUB400"
parent: "Part 1: Get Set Up"
nav_order: 5
---

# Connect VS Code to PUB400

You have VS Code with the IBM i Development Pack. You have a PUB400 account. Time to connect them.

## What to do

Nick Litten, a long-time IBM i developer and writer, maintains an excellent step-by-step guide for this exact workflow: [Connecting to PUB400 with VS-Code for IBM i](https://www.nicklitten.com/connecting-to-pub400-with-vs-code-for-ibm-i/). Follow his instructions — he keeps it current and his screenshots match what you'll see.

## The values you'll need

When his guide asks for connection values, use these:

| Field          | Value                                   |
|----------------|-----------------------------------------|
| **Name**       | Your choice — e.g., `PUB400`            |
| **Host**       | `pub400.com`                            |
| **Port**       | `22`                                    |
| **Username**   | Your PUB400 user profile (from Ch. 2)   |
| **Password**   | Your PUB400 password (from Ch. 2)       |

## A note on authentication

Nick's guide covers password auth, which works. As you get comfortable, consider switching to **SSH key authentication** — it's more secure and means you don't type your password every connection. The [Code for IBM i docs](https://codefori.github.io/docs/#/pages/connect/ssh) cover the switch when you're ready.

## When you're done

In the VS Code IBM i sidebar, you can see your PUB400 connection listed. You can expand the **User Library List** and **IFS Browser** nodes and see actual content — your libraries and your IFS home directory.

If something's off, the [Code for IBM i troubleshooting docs](https://codefori.github.io/docs/#/pages/troubleshooting/index) are the canonical reference.

Next: [Chapter 6: Create your practice tables]({% link part-1/06-practice-tables.md %}).
