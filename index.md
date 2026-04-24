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

**[Part 4 — Going Further.]({% link part-4/index.md %})** A curated map to the broader RPG community's resources: people worth following, authoritative references, user groups, and topics this tutorial intentionally didn't cover.

**[Reference.]({% link reference.md %})** Single-page cheat sheets for free-format syntax, common BIFs, SQLSTATEs, MONITOR patterns, and daily CL commands.

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

---

## Standing on the shoulders of giants

This tutorial doesn't present any of the ideas within it as our own. Modern RPG, free-format syntax, VS Code workflows, PUB400, RPGUnit — none of it was invented here. What we've tried to do is **aggregate** decades of community teaching into a single guided journey, so that someone new to IBM i can find one coherent path from "zero" to "shipping real code."

The path exists because the following people and organizations built it.

**The content and code examples in Part 3** are adapted from a K3S internal training curriculum written by **Mike Scampini** and **Lauren Brakke**. Their originals — pedagogical, idiomatic, well-commented — are the teaching shape underneath nearly every tutorial. Thank you for letting us share this work publicly.

**The environment that makes this tutorial possible** runs on **Holger Scherer's** PUB400 public server — a free IBM i for the community, maintained continuously since 1999. Without PUB400, this site would have to assume readers already have an IBM i available. Thank you.

**The VS Code tooling** our tutorial depends on is **Code for IBM i**, originally created by **Liam Allan** and now maintained by a community of contributors. It's the single thing that makes modern RPG development on IBM i approachable to newcomers. Thank you to the entire Code for IBM i team.

**The community figures whose writing informs our approach** — whether cited directly in Part 4 or absorbed by osmosis through years of reading — include **Scott Klement**, **Nick Litten**, **Barbara Morris**, **Jon Paris**, **Susan Gantner**, and the many practitioners who publish through **Seiden Group**. Their work is where most of modern RPG's learning happens. We link to it, we don't try to replace it.

**The RPG community as a whole** — MIDRANGE-L regulars, COMMON presenters, conference speakers, bloggers, IBM Champions, and the thousands of quiet practitioners who've kept this platform alive and relevant through forty years — is what this tutorial is ultimately a tribute to. If you learned something here, it's because someone, somewhere, taught someone else who taught someone else, long before any of us got involved.

Found something missing from these credits? We'd genuinely like to get the attribution right — [open an issue or PR](https://github.com/K3S/rpg-tutorial) and we'll add it.
