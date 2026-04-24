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

**[Part 1 — Get Set Up.]({% link part-1/index.md %})** Set up VS Code with the IBM i Development Pack, connect to your IBM i via the free PUB400 public server, and understand how the filesystems fit together. End with your first compiled and running Hello World.

**[Part 2 — Learn the Language.]({% link part-2/index.md %})** Free-format RPGLE fundamentals, data structures, procedures, built-in functions, embedded SQL, error handling with MONITOR, and date math. The 80% of RPG you use every day.

**[Part 3 — Build Real Things.]({% link part-3/index.md %})** Twelve progressive hands-on tutorials that build real programs — supplier lookups, batch reorder processing, service programs, price audits, CL wrappers, and unit tests — using a consistent practice dataset so examples build on each other.

**[Part 4 — Going Further]({% link part-4/index.md %})** Curated map to the broader RPG community's resources for deeper learning: Scott Klement's writing, IBM's reference guides, the Code for IBM i docs, conferences, and forums.

**Reference (coming soon).** Cheat sheets for free-format syntax, common SQLSTATEs, and daily-use CL commands.

---

## Conventions used throughout

Examples use a consistent table-prefix field naming convention you'll encounter in real IBM i shops:

- `SP_` — Supplier table fields (e.g., `SP_SUPL`, `SP_NAME`)
- `PR_` — Product table fields (e.g., `PR_PROD`, `PR_QOH`)
- `RC_` — Reorder-candidate table fields (e.g., `RC_PROD`, `RC_SUGG`)

Assumed table definitions and field lists are documented in [Part 1 Chapter 6: Create your practice tables]({% link part-1/06-practice-tables.md %}).

---

## Contributing

This site is open source. Spot a typo or want to improve a section? Use the **Edit this page on GitHub** link at the bottom of any page, or [open a pull request](https://github.com/K3S/rpg-tutorial) directly.

Contributions welcome from the RPG community — this tutorial exists because others' work pointed the way, and improving it is a community effort.

## License

Free and open source under the [MIT License](https://github.com/K3S/rpg-tutorial/blob/main/LICENSE).

---

Head to [Part 1: Get Set Up]({% link part-1/index.md %}) to begin.
