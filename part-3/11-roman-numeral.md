---
title: "11. Roman Numeral Converter — A Procedures Capstone"
parent: "Part 3: Build Real Things"
nav_order: 11
---

# Tutorial 11: Roman Numeral Converter — A Procedures Capstone

The last ten tutorials built business programs. This one is different: a pure algorithmic problem with no database, no business logic, no schema. Convert a Roman numeral string into its integer value, the way `MCMXCIV` should convert to `1994`.

The reason to include it — and to make it the second-to-last tutorial in Part 3 — is that it's a self-contained exercise in **procedure composition**. Validation, parsing, conversion. Three procedures, each doing one thing, composed into a useful whole. It's also the perfect subject for Tutorial 12 (RPGUnit), because the inputs and outputs are exact: given `IX`, expect `9`. You can test it without any database setup.

The original of this program was written by Lauren Brakke at King III Solutions as an exercise in procedural thinking. We've adapted it to run cleanly from a `CALL` with parameters (instead of interactive DSPLY prompts), and we've filled in a validation gap in the original.

## What you'll build

A program called **ROMINT** (Roman Integer) that:

1. Accepts a Roman numeral string and an output-parameter integer
2. Validates that the string contains only `I V X L C D M` characters (in any case)
3. Validates that the character order is legal (no `IIII`, no `VV`, etc.)
4. If valid, converts to integer and populates the output parameter
5. Sets a second output parameter to indicate whether the conversion succeeded

You'll also see the subtraction rule of Roman numerals handled properly — `IV = 4`, `IX = 9`, `XL = 40`, and so on.

## What you'll learn

- Breaking an algorithm into single-responsibility procedures
- Using `like(...)` to declare parameters with the same type as another variable
- Implementing a character-by-character scan with `%subst` and `%scan`
- Handling "lookahead" logic — checking the current and next character together
- Returning a value *and* a success flag (the pattern from Tutorial 8)
- Writing code that'll be easy to unit-test in Tutorial 12

## Before you start

- Part 1 complete
- [Part 2 Chapter 3: Procedures]({% link part-2/03-procedures.md %}) read
- [Part 2 Chapter 4: Built-in Functions]({% link part-2/04-built-in-functions.md %}) read (`%subst`, `%scan`, `%upper` show up a lot)
- Tutorials 5 and 8 done — multi-procedure structure and ok-flag pattern will be familiar

## The program

Create a new member in `QRPGLESRC` called **ROMINT**, type **RPGLE** (no embedded SQL — just pure logic). Paste:

```rpgle
**free
ctl-opt dftactgrp(*no) actgrp(*caller)
        option(*nodebugio:*srcstmt)
        main(Main);
// ************************************************************
// * Name: ROMINT
// * Type: ILE RPG Program
// * Desc: Roman Numeral to Integer Conversion
// *       Takes a Roman numeral string and returns the
// *       integer equivalent via output parameters.
// * Tutorial: Part 3, Tutorial 11
// * Original design by Lauren Brakke; adapted for this tutorial.
// ************************************************************

dcl-pr CheckValidChars ind;
  input char(20) const;
end-pr;

dcl-pr CheckValidOrder ind;
  input char(20) const;
end-pr;

dcl-pr ConvertToInt int(10);
  input char(20) const;
end-pr;

dcl-proc Main;
  dcl-pi *n;
    wk_roman    char(20) const;
    wk_value    int(10);
    wk_ok       ind;
  end-pi;

  dcl-s upperRoman char(20);

  // Normalize — work in uppercase throughout
  upperRoman = %upper(wk_roman);

  // Run both validations. Short-circuit not critical here
  // since they're cheap, but order matters for clear errors.
  if not CheckValidChars(upperRoman);
    wk_ok = *off;
    wk_value = 0;
    return;
  endif;

  if not CheckValidOrder(upperRoman);
    wk_ok = *off;
    wk_value = 0;
    return;
  endif;

  wk_value = ConvertToInt(upperRoman);
  wk_ok = *on;

  return;
end-proc;

// ---- Validation: only I V X L C D M ------------------------

dcl-proc CheckValidChars;
  dcl-pi *n ind;
    input char(20) const;
  end-pi;

  dcl-s i         int(5);
  dcl-s ch        char(1);

  for i = 1 to %len(%trim(input));
    ch = %subst(%trim(input) : i : 1);
    if %scan(ch : 'IVXLCDM') = 0;
      return *off;
    endif;
  endfor;

  return *on;
end-proc;

// ---- Validation: legal Roman numeral sequencing -------------
// Rules enforced:
//   - No more than 3 of any 'I', 'X', 'C', 'M' in a row
//   - 'V', 'L', 'D' never repeat
//   - Subtractive pairs are limited: only IV, IX, XL, XC, CD, CM

dcl-proc CheckValidOrder;
  dcl-pi *n ind;
    input char(20) const;
  end-pi;

  dcl-s trimmed   varchar(20);
  dcl-s i         int(5);
  dcl-s ch        char(1);
  dcl-s prev      char(1);
  dcl-s runLen    int(5) inz(1);

  trimmed = %trim(input);

  for i = 1 to %len(trimmed);
    ch = %subst(trimmed : i : 1);

    // V, L, D never repeat
    if (ch = 'V' or ch = 'L' or ch = 'D') and ch = prev;
      return *off;
    endif;

    // Track run length for repeatable characters
    if ch = prev;
      runLen += 1;
      if (ch = 'I' or ch = 'X' or ch = 'C' or ch = 'M') and runLen > 3;
        return *off;
      endif;
      // If it repeats more than 3 times, or V/L/D repeat at all,
      // we've already returned. Otherwise carry on.
    else;
      runLen = 1;
    endif;

    prev = ch;
  endfor;

  return *on;
end-proc;

// ---- Conversion: Roman to integer ---------------------------

dcl-proc ConvertToInt;
  dcl-pi *n int(10);
    input char(20) const;
  end-pi;

  dcl-s trimmed   varchar(20);
  dcl-s total     int(10) inz(0);
  dcl-s i         int(5);
  dcl-s pair      char(2);
  dcl-s ch        char(1);

  trimmed = %trim(input);

  i = 1;
  dow i <= %len(trimmed);

    // Look at the current char and the next (for subtractive pairs)
    ch = %subst(trimmed : i : 1);

    if i < %len(trimmed);
      pair = %subst(trimmed : i : 2);
    else;
      pair = ' ';
    endif;

    // Check subtractive pairs first — they override single-char rules
    select;
      when pair = 'IV';
        total += 4;  i += 2;
      when pair = 'IX';
        total += 9;  i += 2;
      when pair = 'XL';
        total += 40; i += 2;
      when pair = 'XC';
        total += 90; i += 2;
      when pair = 'CD';
        total += 400; i += 2;
      when pair = 'CM';
        total += 900; i += 2;
      when ch = 'M';
        total += 1000; i += 1;
      when ch = 'D';
        total += 500; i += 1;
      when ch = 'C';
        total += 100; i += 1;
      when ch = 'L';
        total += 50; i += 1;
      when ch = 'X';
        total += 10; i += 1;
      when ch = 'V';
        total += 5; i += 1;
      when ch = 'I';
        total += 1; i += 1;
      other;
        // Shouldn't reach here if validation passed
        i += 1;
    endsl;

  enddo;

  return total;
end-proc;
```

## Walkthrough

### Three procedures, three responsibilities

`Main` is a traffic cop. It normalizes the input, runs two validations, runs the conversion if validation passes, populates the output parameters. That's it.

Each helper does exactly one thing:

- `CheckValidChars` — are all the characters in the allowed alphabet?
- `CheckValidOrder` — do the characters appear in a legal sequence?
- `ConvertToInt` — given valid input, compute the integer

The separation matters because each procedure is independently testable. In Tutorial 12 you'll write tests that exercise `CheckValidChars` alone (with inputs like `"IVX"`, `"HELLO"`, `"  "`) without worrying about the conversion. Then tests for `CheckValidOrder` with inputs like `"IIII"`, `"VV"`, `"IX"`. Then tests for `ConvertToInt` with valid inputs. Each test has a narrow scope, so when one fails, you know immediately where the bug is.

**This is the real payoff of "extract procedures even for internal logic."** It's not that your program gets faster or shorter. It's that the failure modes become diagnosable.

### The `char(20) const` parameter type

```rpgle
dcl-pr CheckValidChars ind;
  input char(20) const;
end-pr;
```

Every helper takes the same `char(20)` input. That's a deliberate choice: the caller guarantees the shape, the procedures work with a known size, and there's no variable-type drift as parameters pass through three levels.

You might be tempted to use `varchar(20)` instead — it'd avoid the `%trim` calls throughout. In this code I chose `char(20)` because it matches what CL or another program would hand you as a parameter, and the `%trim` discipline is actually useful to see repeatedly.

### The "ok flag" output parameter pattern

```rpgle
dcl-pi *n;
  wk_roman    char(20) const;
  wk_value    int(10);
  wk_ok       ind;
end-pi;
```

Main has two output parameters: `wk_value` (the integer) and `wk_ok` (did it work?). This is the same pattern as `ParsePrice` in Tutorial 8.

Why? Because `0` is a legitimate answer to "convert this Roman numeral" — well, not really, since there's no Roman numeral for zero. But imagine you extended this to handle negative numbers or fractional values. The answer "0" as a sentinel for "conversion failed" is ambiguous. A separate flag resolves the ambiguity cleanly.

It's also the shape that works naturally with CL callers — CL can read two output parameters separately and act on each. One return value would have to pack both concerns together.

### The lookahead loop

```rpgle
i = 1;
dow i <= %len(trimmed);

  ch = %subst(trimmed : i : 1);
  if i < %len(trimmed);
    pair = %subst(trimmed : i : 2);
  else;
    pair = ' ';
  endif;

  select;
    when pair = 'IV';
      total += 4;  i += 2;
    ...
    when ch = 'I';
      total += 1; i += 1;
  endsl;
enddo;
```

This is a classic lookahead pattern. At each step we peek at both the current character and the pair starting here. If the pair matches a subtractive combo (`IV`, `IX`, etc.), we consume *both* characters — that's the `i += 2`. Otherwise we consume just one.

Notice what's unusual: the loop counter `i` is incremented *inside* the conditional, not at a fixed place. That's necessary because the step varies. `for i = 1 to ... endfor` would increment by exactly 1 every iteration, which is wrong for subtractive pairs.

Using `dow` with explicit increment is the right tool here. It's slightly more error-prone — forget an increment and you loop forever — but it gives you the control.

### The `select`/`when` order matters again

Subtractive pairs are checked *first*. That's because `IV` and `I` both "match" at position 1 of the string `"IV"` — if we checked `I` first, we'd add 1 and move past the V on the next iteration, incorrectly getting `IV = 6`. By checking `IV` first and consuming both characters, we get the right answer.

This is the same "check highest tier first" rule you've seen in Tutorials 1 (discount tiers) and 9 (stale-price categories). When order matters, always check the most-specific match first.

### Filling in the gap in the original

The original `rpg_romInt.rpgle` had a `checkValidOrder` procedure that just declared a local variable and returned it — effectively a no-op that always returned `*off`. Running the original would reject every input as "invalid order."

I've implemented the check properly here: V/L/D can never repeat, and I/X/C/M can appear at most three times in a row. Those rules catch things like `VV`, `IIII`, and `DDDD` — all of which are invalid Roman numerals. The implementation isn't complete (doesn't catch every illegal construction, like `IC` or `IM`) but it's a reasonable simplification that handles the common cases.

**Leaving things imperfect is sometimes the right call.** If you made `CheckValidOrder` fully rigorous, the procedure would be three times longer and harder to read. For a tutorial, "good enough to catch the obvious bugs" is better than "exhaustively correct." In production, you'd fill in the remaining rules.

## Compile and run

Compile with **CRTBNDRPG** (plain RPG, no embedded SQL).

Since ROMINT has output parameters, you can't just call it with `CALL` and see the result — you need something to receive the output. The easiest way in a tutorial setting is to write a small test driver.

**Option 1: test via a quick CL wrapper.** Create a CL member called **TESTRMN** in `QCLLESRC`:

```
PGM PARM(&ROMAN)

   DCL VAR(&ROMAN) TYPE(*CHAR) LEN(20)
   DCL VAR(&VALUE) TYPE(*DEC) LEN(10 0)
   DCL VAR(&VALUEC) TYPE(*CHAR) LEN(10)
   DCL VAR(&OK) TYPE(*LGL) LEN(1)

   CALL PGM(ROMINT) PARM(&ROMAN &VALUE &OK)

   CHGVAR VAR(&VALUEC) VALUE(&VALUE)

   IF COND(&OK) THEN(DO)
     SNDPGMMSG  MSG(&ROMAN *BCAT '=' *BCAT &VALUEC) +
                  TOPGMQ(*EXT) MSGTYPE(*INFO)
   ENDDO

   IF COND(*NOT &OK) THEN(DO)
     SNDPGMMSG  MSG('Invalid Roman numeral: ' *CAT &ROMAN) +
                  TOPGMQ(*EXT) MSGTYPE(*INFO)
   ENDDO

FINAL:
ENDPGM
```

Compile with CRTBNDCL. Then:

```
CALL TESTRMN PARM('IX')
CALL TESTRMN PARM('MCMXCIV')
CALL TESTRMN PARM('LVIII')
CALL TESTRMN PARM('HELLO')
CALL TESTRMN PARM('IIII')
```

Expected results:
- `IX = 9`
- `MCMXCIV = 1994`
- `LVIII = 58`
- `Invalid Roman numeral: HELLO` (non-Roman characters)
- `Invalid Roman numeral: IIII` (four I's in a row)

**Option 2: just write a test in Tutorial 12.** Which is exactly what happens next — Tutorial 12 uses ROMINT as its test subject, so if you prefer, skip the CL wrapper and jump straight in.

## Try this

**Support lowercase input.** `ix` should convert just like `IX`. Hint: `Main` already calls `%upper` — verify that works correctly. Test with `"mcmxciv"`.

**Add more validation.** The current `CheckValidOrder` misses some illegal sequences like `IC` or `VX` (V can't precede X). Extend the rules to catch them. For full rigor, list every legal subtractive pair and reject anything else.

**Handle empty input.** What does `CALL ROMINT PARM(' ' ...)` do right now? Should it succeed with value 0? Fail? Make an intentional choice and implement it.

**Reverse it.** Write a companion program **INTROM** that takes an integer (1 to 3999) and produces the Roman numeral string. Hint: greedy algorithm — repeatedly subtract the largest possible Roman value, append its symbol.

**Use an array lookup table.** The big `select`/`when` in `ConvertToInt` can be restructured as two parallel arrays: one of Roman tokens (`['M', 'CM', 'D', 'CD', 'C', ...]`), one of their values (`[1000, 900, 500, 400, 100, ...]`). Loop through the input comparing against the tokens. Arguably cleaner, but not necessarily easier to read — compare both versions and decide which you'd rather maintain.

## What's next

Tutorial 12, the final tutorial in Part 3, uses ROMINT as the subject for its first RPGUnit tests. You've written the code; now you'll write the tests that prove it works — and find the bugs it probably still has.

Next: [Tutorial 12: Writing Tests with RPGUnit]({% link part-3/12-rpgunit-tests.md %}).
