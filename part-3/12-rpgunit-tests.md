---
title: "12. Writing Tests with RPGUnit"
parent: "Part 3: Build Real Things"
nav_order: 12
---

# Tutorial 12: Writing Tests with RPGUnit

Every tutorial so far has ended with "run it and eyeball the output." That's fine for learning. It's not fine for maintaining a codebase. You can't eyeball your way to catching a regression in ROMINT six months after you wrote it.

**Automated tests are the answer.** In RPG land, the tool is **RPGUnit** — an open-source unit-testing framework from the IBM i community. You write tests that assert specific outputs for specific inputs, you run the test suite, and either everything passes or RPGUnit tells you exactly which assertions failed.

This tutorial teaches the minimum viable RPGUnit workflow. You'll write a test file for ROMINT from Tutorial 11 and watch it both pass and fail — because we'll run the tests against the known bugs in ROMINT's validation logic on purpose.

## What you'll build

A test program called **TSTROMINT** that tests the ROMINT converter from Tutorial 11. It contains multiple test procedures, each asserting specific conversions or validation results. You'll run the suite through the Code for IBM i extension and see the pass/fail output inline in VS Code.

## What you'll learn

- What RPGUnit is and how it fits into the IBM i ecosystem
- Installing RPGUnit on PUB400 (there's a subtlety we'll address honestly)
- The structure of a test source file: `ctl-opt nomain`, setup, test procedures
- Assertions: `aEqual`, `aIsTrue`, `aIsFalse`, `aNotEqual`
- Running a test suite from the Code for IBM i panel
- Interpreting test output
- The red-green-refactor cycle in an IBM i context

## Before you start

- Part 1 complete
- Tutorial 11 completed — ROMINT compiled and working
- Some willingness to deal with external tooling setup (RPGUnit isn't part of base IBM i)

## What is RPGUnit?

**RPGUnit** is an open-source unit testing framework for ILE RPG. It's modeled on JUnit — same structure, same mental model. You write a test program that declares test procedures; the framework discovers them by naming convention, runs each one, collects pass/fail results, and reports.

The project's official home is [https://github.com/tools-400/irpgunit](https://github.com/tools-400/irpgunit). It's maintained by the IBM i community (Rapid Fire team). It's free. It's been in use for over a decade.

The Code for IBM i VS Code extension has first-class RPGUnit support — when you right-click a `TSTxxx` file, you get a "Run tests" option that executes your tests and displays results inline, JUnit-style. That's what we'll use.

### Honest note on PUB400

RPGUnit is pre-installed on PUB400. You can use it without any setup — the RPGUnit runtime (a library called `RPGUNIT`) is already available to all users on the public server.

If you're working on a private IBM i instead, you'd need to install RPGUnit once — download the release from the GitHub link above, follow their install instructions. It's a one-time task, maybe 30 minutes.

For PUB400, it just works.

## Write the test source

Create a new member in `QRPGLESRC` called **TSTROMINT**, type **RPGLE** (no embedded SQL in this test — just procedural logic and assertions). The `TST` prefix is convention: Code for IBM i looks for files starting with `TST` (or `T_`) to identify test sources.

Paste:

```rpgle
**free
ctl-opt nomain
        bnddir('RPGUNIT')
        option(*nodebugio:*srcstmt);
// ************************************************************
// * Name: TSTROMINT
// * Type: ILE RPG Module (RPGUnit test)
// * Desc: Unit tests for ROMINT (Tutorial 11).
// * Tutorial: Part 3, Tutorial 12
// ************************************************************

/include QINCLUDE,TESTCASE

// ---- Prototype for the program under test -------------------

dcl-pr ROMINT extpgm('ROMINT');
  roman char(20) const;
  value int(10);
  ok    ind;
end-pr;

// ---- Helper that wraps the CALL and returns its result ------

dcl-proc CallROMINT;
  dcl-pi *n;
    roman    char(20) const;
    value    int(10);
    ok       ind;
  end-pi;

  ROMINT(roman : value : ok);
end-proc;

// ============================================================
// Test procedures — any procedure starting with "test" is
// automatically discovered and run by RPGUnit.
// ============================================================

dcl-proc test_single_I;
  dcl-s value int(10);
  dcl-s ok    ind;

  CallROMINT('I' : value : ok);

  aIsTrue(ok : 'Single I should be valid');
  aEqual(1 : value : 'I should convert to 1');
end-proc;

dcl-proc test_single_V;
  dcl-s value int(10);
  dcl-s ok    ind;

  CallROMINT('V' : value : ok);

  aIsTrue(ok : 'Single V should be valid');
  aEqual(5 : value : 'V should convert to 5');
end-proc;

dcl-proc test_subtractive_IX;
  dcl-s value int(10);
  dcl-s ok    ind;

  CallROMINT('IX' : value : ok);

  aIsTrue(ok : 'IX should be valid');
  aEqual(9 : value : 'IX should convert to 9');
end-proc;

dcl-proc test_subtractive_XL;
  dcl-s value int(10);
  dcl-s ok    ind;

  CallROMINT('XL' : value : ok);

  aIsTrue(ok : 'XL should be valid');
  aEqual(40 : value : 'XL should convert to 40');
end-proc;

dcl-proc test_full_LVIII;
  dcl-s value int(10);
  dcl-s ok    ind;

  CallROMINT('LVIII' : value : ok);

  aIsTrue(ok : 'LVIII should be valid');
  aEqual(58 : value : 'LVIII should convert to 58');
end-proc;

dcl-proc test_full_MCMXCIV;
  dcl-s value int(10);
  dcl-s ok    ind;

  CallROMINT('MCMXCIV' : value : ok);

  aIsTrue(ok : 'MCMXCIV should be valid');
  aEqual(1994 : value : 'MCMXCIV should convert to 1994');
end-proc;

dcl-proc test_lowercase_input;
  dcl-s value int(10);
  dcl-s ok    ind;

  CallROMINT('ix' : value : ok);

  aIsTrue(ok : 'Lowercase ix should be valid (normalization)');
  aEqual(9 : value : 'ix should convert to 9');
end-proc;

dcl-proc test_invalid_chars;
  dcl-s value int(10);
  dcl-s ok    ind;

  CallROMINT('HELLO' : value : ok);

  aIsFalse(ok : 'HELLO should be rejected as invalid chars');
end-proc;

dcl-proc test_invalid_run;
  dcl-s value int(10);
  dcl-s ok    ind;

  CallROMINT('IIII' : value : ok);

  aIsFalse(ok : 'IIII should be rejected as invalid (four Is)');
end-proc;

dcl-proc test_v_repeats;
  dcl-s value int(10);
  dcl-s ok    ind;

  CallROMINT('VV' : value : ok);

  aIsFalse(ok : 'VV should be rejected (V cannot repeat)');
end-proc;

// ============================================================
// Intentional failing test — demonstrates what a failure looks
// like in the output. The ROMINT validator misses this case.
// ============================================================

dcl-proc test_illegal_pair_IC;
  dcl-s value int(10);
  dcl-s ok    ind;

  CallROMINT('IC' : value : ok);

  aIsFalse(ok : 'IC is not a legal subtractive pair, should be rejected');
end-proc;
```

## What each assertion does

RPGUnit's assertion BIFs (really, procedures — they start with `a` as their convention):

- `aEqual(expected : actual : message)` — fails if expected ≠ actual
- `aNotEqual(notExpected : actual : message)` — fails if they are equal
- `aIsTrue(condition : message)` — fails if condition is `*off`
- `aIsFalse(condition : message)` — fails if condition is `*on`
- `aAssert(condition : message)` — general-purpose, equivalent to `aIsTrue`
- `aFail(message)` — always fails (useful as a placeholder for "not implemented yet")

If an assertion fails, RPGUnit records it and continues through the rest of the test procedure — other assertions may still pass. After the procedure finishes, RPGUnit moves to the next test.

The string argument on each assertion is the **message** shown when the assertion fails. Make it specific — "MCMXCIV should convert to 1994" is much more useful than "conversion test" when you see it in a failure report.

## Compile and run

### Compile the test

Compile TSTROMINT with **CRTBNDRPG**, but note: it needs access to the RPGUnit service program and the `QINCLUDE` copy file. If compile fails with unresolved references, add `BNDDIR(RPGUNIT)` or similar to the compile command — Code for IBM i usually handles this.

On PUB400 where RPGUnit is already installed, the include path and binding directory should resolve automatically with the `bnddir('RPGUNIT')` in the `ctl-opt`.

### Run the test suite

In the VS Code IBM i sidebar, find TSTROMINT in your library's object listing. Right-click and choose **Run tests**. (This option appears only on test programs — it's recognized by the filename pattern.)

Code for IBM i executes the RPGUnit runner against your compiled test program. Results appear in the Test Explorer panel (usually on the left side of VS Code), showing each test procedure as either green (passed) or red (failed).

Expected results if ROMINT from Tutorial 11 is intact:

- `test_single_I` — PASS
- `test_single_V` — PASS
- `test_subtractive_IX` — PASS
- `test_subtractive_XL` — PASS
- `test_full_LVIII` — PASS
- `test_full_MCMXCIV` — PASS
- `test_lowercase_input` — PASS (ROMINT normalizes to uppercase in Main)
- `test_invalid_chars` — PASS
- `test_invalid_run` — PASS
- `test_v_repeats` — PASS
- `test_illegal_pair_IC` — **FAIL** (the validator misses this case)

The last one failing is intentional. ROMINT's `CheckValidOrder` only rejects obvious cases (V/L/D repeats, I/X/C/M runs > 3). It doesn't catch `IC` — an illegal subtractive pair. The failing test *documents* this gap in the validator.

**This is the red in red-green-refactor.** You've identified a bug through a test. The next step is to fix `CheckValidOrder` in ROMINT — add the rule that only `IV, IX, XL, XC, CD, CM` are legal subtractive pairs, and every other "small before large" pattern is invalid. Recompile ROMINT, re-run the test. It should go green.

That cycle — write a test that fails, write code that makes it pass, clean up if needed — is the whole of TDD in a nutshell. You've just done it.

## What testing gets you on IBM i

Two things worth saying honestly, because "write tests" is easier said than done in a real RPG shop.

**The realistic case.** You probably won't test-drive every function you write. Nobody in a real RPG codebase does. What you *will* do is write a test when you're about to modify code you don't trust — either because the logic is tricky (like Roman numeral validation) or because it's critical (like tax calculation). The test captures the current behavior. You change the code. The test confirms you didn't break what was working. That's the highest-value testing pattern in practice.

**The regression shield.** Months after you write code, someone else — or future you — will come back to modify it. Without tests, every change feels risky, and either they don't change anything (preserving bugs forever) or they change things and break other things. Tests turn "is this safe to change?" into a five-second question.

Tests don't replace code review or QA. They supplement them. A test suite is documentation that runs.

## Try this

**Fix the IC bug.** Modify `CheckValidOrder` in ROMINT to reject illegal subtractive pairs. Any "smaller before larger" where the pair isn't in `{IV, IX, XL, XC, CD, CM}` should fail. Recompile ROMINT, run the test suite again. All green.

**Add more tests.** What other edge cases exist? What about `MMMCMXCIX` (3999, the largest standard Roman numeral)? What about just whitespace? What about numbers that Roman numerals can't represent (zero, negatives)? Add a test for each, and decide whether to make ROMINT handle them gracefully or reject them outright.

**Write tests before writing code.** Pick a small new feature — say, an `InRange` helper that checks if a number is between min and max inclusive. Write the test file first, with tests like `aIsTrue(InRange(5 : 1 : 10))`, `aIsFalse(InRange(11 : 1 : 10))`, etc. Run the tests — they fail because the procedure doesn't exist. Now write the procedure. Watch tests go green. This is textbook TDD.

**Write tests for one of the Part 3 business programs.** INVVAL from Tutorial 5 is a good candidate because it takes clean inputs (a supplier code) and produces computable output (a total). Create test data first (a supplier whose totals you've pre-computed), then tests that call INVVAL and assert the expected totals. This is integration testing — harder than pure unit testing because you're involving the database, but the pattern is identical.

**Look at the RPGUnit assertion catalog.** `aEqual` / `aIsTrue` / `aIsFalse` are the basics, but RPGUnit has more — `aContains`, `aStartsWith`, dates, numerics with precision. See the [RPGUnit documentation](https://tools-400.github.io/irpgunit/) for the full list.

## You made it

You've written twelve tutorials worth of RPG, from Hello World in Part 1 through a fully tested Roman numeral converter here. You can:

- Connect VS Code to an IBM i and operate it
- Read and write the filesystems, libraries, and objects
- Write free-format RPG fluently
- Do embedded SQL and native I/O, and know when to choose each
- Structure programs with procedures and service programs
- Handle errors gracefully with MONITOR
- Do real date arithmetic
- Write CL wrappers that orchestrate RPG programs
- Write unit tests with RPGUnit

That's a complete picture of what mid-level RPG work looks like in 2026. Not everything — no screens (subfiles, display files), no job queue internals, no Integrated Web Services, no Node.js integration, no RDi-specific workflows — but a solid center.

Part 4 will point you to the deeper resources maintained by the broader RPG community for everything else: Scott Klement's writing, IBM's reference guides, the Code for IBM i docs, conferences and forums. Once you've worked through Parts 1–3, you don't need another tutorial. You need a map of where to go next.

Next: *Part 4: Going Further (coming soon)*.
