---
title: "Part 4: Going Further"
layout: default
nav_order: 5
has_children: false
permalink: /part-4/
---

# Part 4: Going Further

Parts 1–3 covered the core. You can connect to an IBM i, write modern RPG, build real programs, and test them. That's the foundation.

Part 4 is different: **it's a map, not a tutorial.**

The RPG community has already built excellent material on every topic this tutorial didn't cover. Rather than try to replace that work, this page points you to the best of it — organized by what you're trying to do.

One honest note about links: resources on the web move. If something 404s, search for the resource name and you'll almost certainly find its new home. The people and projects below aren't going anywhere; only their URLs might.

---

## Tooling and environment

**[Code for IBM i Docs](https://codefori.github.io/docs/)** — the official documentation for the VS Code extension you're using. Covers connection setup, filesystem navigation, the linter, local development with Git, actions configuration, and much more than this tutorial could touch. When you want to go deeper on the tooling itself, start here.

**[Code for IBM i — GitHub organization](https://github.com/codefori)** — where the extensions live, where issues get filed, and where you'll find related projects like `vscode-rpgle` (the language tools) and `vscode-db2i` (the SQL editor).

**[Nick Litten's Getting Started with IBM i RPG](https://www.nicklitten.com/course/getting-started-with-ibm-i-rpg-a-guide-for-new-programmers/)** — a parallel free guide. Strong on the "getting your first program running" onboarding, which is a useful complement to everything Part 1 covered.

**[PUB400.com](https://pub400.com/)** — your free account's home. The site includes a welcome PDF, rules of use, and a link to Holger Scherer's company (RZKH) for anyone interested in commercial IBM i hosting.

**[Scott Klement's open-source tools](https://www.scottklement.com/)** — Scott maintains several widely-used libraries you'll encounter in production shops: **HTTPAPI** (for calling HTTP services from RPG), **YAJL** (JSON parser/generator for RPG), and several others. If you need to do web services, HTTP calls, or JSON work from RPG, start here.

---

## Official language references

**[IBM i 7.5 ILE RPG Programmer's Guide (PDF)](https://www.ibm.com/docs/en/ssw_ibm_i_75/pdf/sc092507.pdf)** — IBM's authoritative guide to the RPG compiler, the Integrated Language Environment, program creation, debugging, and advanced topics. This is the source of truth for language behavior.

**[IBM i 7.5 ILE RPG Reference (PDF)](https://www.ibm.com/docs/en/ssw_ibm_i_76/pdf/sc092508.pdf)** — the language specification itself. Every opcode, every BIF, every keyword, with full syntax rules and examples. Heavier reading than the Programmer's Guide; use as a reference, not a tutorial.

**[IBM Documentation — IBM i](https://www.ibm.com/docs/en/i)** — the root of IBM's online docs. Contains every manual for every version. When you need something authoritative and current, this is where to search.

**[What's New in RPG (Barbara Morris presentations)](https://www.tug.ca/)** — Barbara Morris is IBM's RPG compiler architect. Her "What's New" talks at TUG (Toronto User's Group) and other conferences are the best way to track new language features as they ship. Search "Barbara Morris What's New RPG" on YouTube or TUG's site for the latest.

---

## People worth following

These are practitioners whose writing and speaking has shaped how modern RPG is taught. Following their work is how you keep current.

**[Scott Klement](https://www.scottklement.com/)** — decades of deep technical articles on RPG, web services, HTTP, JSON, APIs, and interoperability. His homepage is a library; his conference presentations (many linked from the site) are the best treatment of web services from RPG that exists.

**[Nick Litten](https://www.nicklitten.com/)** — working IBM i developer who blogs prolifically about practical RPG, modernization, VS Code workflows, and career topics. High signal-to-noise, current, and written with real-shop perspective.

**Barbara Morris** — IBM's RPG compiler architect. Presentations and posts, rather than a single website. Search for her by name for authoritative content on where the language is going.

**[Jon Paris & Susan Gantner](https://systemideveloper.com/)** — long-time RPG educators and authors. Published the book *Free-Format RPG IV* which is the canonical reference for modern RPG syntax. Their site hosts training materials and articles.

---

## Communities and discussion

**[MIDRANGE-L mailing list](https://www.midrange.com/)** — the classic RPG/IBM i mailing list. Decades of archived discussion, still active. Post a question, get answers from practitioners with thirty years of experience.

**[Code for IBM i GitHub Discussions](https://github.com/codefori/vscode-ibmi/discussions)** — for questions specifically about the VS Code extension. The maintainers are responsive.

**[COMMON](https://www.common.org/)** — the IBM i user association. Runs annual conferences (COMMON POWERUp in North America) where most of the writers above speak. Worth attending once.

**[COMMON Europe](https://www.comeur.org/)** — European counterpart, hosts Common Europe Congress annually.

**[IT Jungle](https://www.itjungle.com/)** — the main independent publication covering IBM i. Weekly articles on technology, community, IBM's strategy, and industry news. Subscribe to the newsletter.

**[IBM i on Reddit (r/ibmi)](https://www.reddit.com/r/ibmi/)** — smaller but growing. Good for asking "is this a dumb question?" questions that feel too basic for MIDRANGE-L.

---

## Topics this tutorial intentionally didn't cover

There's a lot more to RPG than twelve tutorials can hold. Here are the major areas Parts 1–3 skipped, and where to learn them.

### Display files and subfiles

The green-screen 5250 UIs you see on IBM i are driven by **display files** written in a language called **DDS**. **Subfiles** are the patterns for showing and interacting with lists of records — think of them as 5250-era React components. Neither is going away — thousands of production applications still use them — but they're a distinct skill.

- IBM's [Display File topic in the iSeries Information Center](https://www.ibm.com/docs/en/i) (search "display files") is the reference
- Jon Paris and Susan Gantner cover subfiles extensively in their articles and books
- Nick Litten has several blog posts on display-file modernization

### Integrated Web Services (IWS) and REST APIs from RPG

Exposing RPG programs as REST endpoints — or calling REST APIs from RPG — is a major modernization topic. Your RPG business logic becomes callable from web apps, mobile apps, and other systems.

- Scott Klement's [Providing RPG Web Services presentation](https://www.scottklement.com/) is the canonical treatment
- His **HTTPAPI** library is what most shops use for outbound HTTP
- IBM's Integrated Web Services (IWS) tool is the no-code path — search IBM docs for current setup

### Open-source ecosystem on IBM i

IBM ships the **Open Source Package Manager** (`yum` on IBM i) with Node.js, Python, Git, and a full Unix-style toolchain. You can run modern web apps directly on IBM i and have them call RPG.

- [IBM's Open Source on IBM i page](https://www.ibm.com/support/pages/node/1118781)
- Search for Nick Litten's posts on Node.js for IBM i
- [Litmis](https://github.com/litmis) — open-source tooling and examples on GitHub

### XMLSERVICE — calling RPG from PHP, Python, Node.js

The bridge layer that lets non-RPG languages call RPG programs. Every modern IBM i shop that's doing web UI work in Node.js or PHP is using XMLSERVICE under the hood.

- [XMLSERVICE on GitHub](https://github.com/IBM/xmlservice)
- **itoolkit** libraries exist for Python, Node.js, PHP on the GitHub IBM organization

### RDi (Rational Developer for i)

IBM's paid Eclipse-based IDE for RPG. Many shops still use it (often alongside or instead of VS Code). Worth knowing exists even if you don't use it.

- [IBM's RDi product page](https://www.ibm.com/products/rational-developer-for-i)
- Some tutorials and workflows only exist in RDi form

### Performance tuning and database internals

Once you've written RPG, you hit questions about query plans, journaling, index design, and job tuning.

- **Mike Cain** and the **DB2 for i Center of Excellence** publish the definitive content on Db2 for i performance
- IBM's [Database on IBM i](https://www.ibm.com/docs/en/i/7.5?topic=database) documentation
- The **SQL Performance Center** on IBM i itself (accessible through ACS) is an under-used diagnostic tool

### Service program versioning and binding language

Once you have more than one service program and more than one caller, you need to understand **signatures**, **binding language**, and how to evolve APIs without breaking existing callers. This is a real-world topic that's easy to get wrong.

- IBM's ILE Concepts manual is the foundation (search "ILE Concepts" in IBM docs)
- Scott Klement has written extensively on signature management

---

## Where to turn when you're stuck

Three practical moves when you hit something you can't solve:

**Search MIDRANGE-L archives.** Almost every reasonable RPG question has been asked before. Add `site:midrange.com` to your search.

**Ask on MIDRANGE-L or GitHub Discussions.** Be specific, show the code that's failing and the error message. The community is generous to well-formed questions.

**Read the Code for IBM i extension logs.** Weird connection or compile behavior often shows up in the extension's output panel (View → Output → select "Code for IBM i"). Copy the log, paste it when asking for help.

---

## Contributing to Part 4

This page should grow. If you know a resource that belongs here — a blog, a tool, a tutorial, a person — [open a pull request](https://github.com/K3S/rpg-tutorial) or an issue on the repo. The goal is a living, community-maintained map.

Particular gaps we'd love contributions on:

- **Regional user groups** beyond COMMON (TUG, OMNI, and others)
- **Non-English resources** — German, Japanese, Spanish IBM i communities
- **Specific excellent blog posts** on topics above, where the author's homepage URL isn't enough
- **Newer voices** — people who started writing about RPG in the last few years whose work deserves more attention

---

## Thank you

You've reached the end of the tutorial. If anything in Parts 1–3 was unclear, broken, or could be better, please [let us know](https://github.com/K3S/rpg-tutorial/issues). This is an open resource and community feedback is how it stays useful.

Go build something.
