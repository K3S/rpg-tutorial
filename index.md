---
title: Home
layout: default
nav_order: 1
description: "A free, open-source tutorial for modern RPGLE and CL development on IBM i using VS Code."
permalink: /
---

# RPG Tutorial

A free, open-source tutorial for modern RPGLE and CL development on IBM i, using VS Code and contemporary tooling.

Sponsored by [King III Solutions](https://k3s.com).

---

## Who this is for

- RPG developers migrating from fixed-format to free-format
- Teams modernizing their IBM i workflow with VS Code and Git
- Developers coming to IBM i from other platforms
- Students learning RPGLE and CL for the first time
- Anyone who wants concrete, working examples of real-world RPG patterns

## What you'll find here

**Getting Started.** Set up VS Code with the IBM i Development Pack, connect to your IBM i, and understand how the filesystems fit together.

**Writing Your First Programs.** Free-format RPGLE fundamentals, embedded SQL in SQLRPGLE, and compiling from VS Code.

**Debugging and Quality.** The interactive debugger, the RPGLE linter, and productivity tips from working in modern tooling.

**Source Control.** Git workflows that work with both IFS stream files and QSYS library members.

**Hands-On Tutorials.** Ten progressive tutorials that build real programs — supplier lookups, batch reorder processing, service programs, error handling, date math, PHP/XMLSERVICE bridges, CL glue, and RPGUnit testing.

**Reference.** Cheat sheets for free-format syntax, common SQLSTATEs, and daily-use CL commands.

---

## Conventions used throughout

Examples use a consistent table-prefix field naming convention you'll encounter in real IBM i shops:

- `SP_` — Supplier table fields (e.g., `SP_SUPL`, `SP_NAME`)
- `PR_` — Product table fields (e.g., `PR_PROD`, `PR_QOH`)
- `RC_` — Reorder-candidate table fields (e.g., `RC_PROD`, `RC_SUGG`)

Assumed table definitions and field lists are documented on the relevant tutorial pages.

---

## Contributing

This site is open source. Spot a typo or want to improve a section? Use the **Edit this page on GitHub** link at the bottom of any page, or [open a pull request](https://github.com/K3S/rpg-tutorial) directly.

## License

Free and open source under the [MIT License](https://github.com/K3S/rpg-tutorial/blob/main/LICENSE).
