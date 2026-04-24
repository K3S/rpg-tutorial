---
title: "4. Install the IBM i Development Pack"
parent: "Part 1: Get Set Up"
nav_order: 4
---

# Install the IBM i Development Pack

VS Code by itself doesn't know anything about IBM i. You need the **IBM i Development Pack**, an extension pack maintained jointly by the open-source community (under the "Code for IBM i" project) and IBM. Installing one pack installs several related extensions at once.

## What to do

In VS Code, open the Extensions panel (**Ctrl+Shift+X** on Windows, **Cmd+Shift+X** on macOS) and search for:

IBM i Development Pack

Click Install. VS Code will pull in the bundled extensions. Restart VS Code when prompted.

## What you just got

The pack bundles these extensions:

- **Code for IBM i** — connect to IBM i, browse libraries and the IFS, compile, run commands
- **RPGLE Language Tools** — content assist, outline view, hover info, the linter
- **IBMi Languages** — syntax highlighting for RPG, CL, DDS, MI
- **Db2 for IBM i** — SQL editor with inline query results
- **IBM i Renderer** — rendering for display files (DSPF) and printer files (PRTF)
- **IBM i Project Explorer** — local-project workflows with Source Orbit or Better Object Builder

For step-by-step installation guidance (including troubleshooting), see the official [Code for IBM i documentation](https://codefori.github.io/docs/).

## When you're done

Look at the Activity Bar on the left side of VS Code. You should see a new **IBM i** icon (it looks like a small mainframe). Click it. You'll see "Connect to an IBM i" — but don't click it yet. That's the next chapter.

Next: [Chapter 5: Connect to PUB400]({% link part-1/05-connect-to-pub400.md %}).
