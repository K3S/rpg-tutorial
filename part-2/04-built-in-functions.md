---
title: "4. Built-in Functions"
parent: "Part 2: Learn the Language"
nav_order: 4
---

# Built-in Functions

**Built-in functions** — BIFs — are RPG's standard library. They're always prefixed with `%`, so when you see `%trim`, `%date`, `%eof`, you know it's a BIF rather than a procedure you wrote.

There are dozens. This chapter covers the ones you'll use every day, grouped by what they do.

## String manipulation

### `%trim`, `%trimr`, `%triml`

```rpgle
name = %trim(SP_NAME);   // trim leading and trailing spaces
name = %trimr(SP_NAME);  // trim trailing only (right)
name = %triml(SP_NAME);  // trim leading only (left)
```

The most-used BIF in RPG. Database character fields are frequently space-padded; `%trim` removes the padding.

### `%len`

```rpgle
nameLength = %len(%trim(SP_NAME));  // actual length of trimmed name
maxLength  = %len(SP_NAME);         // declared length of the field
```

Returns the current length of a varchar, or the declared length of a fixed-length char.

### `%subst`

```rpgle
firstThree = %subst(productCode : 1 : 3);   // chars 1-3
tail       = %subst(productCode : 5);        // from position 5 to end
```

Substring extraction. Takes position and optional length. Positions are **1-based**.

### `%scan`, `%scanrpl`

```rpgle
pos = %scan('-' : productCode);              // position of first '-'
clean = %scanrpl('bad' : 'good' : message);  // replace all 'bad' with 'good'
```

`%scan` finds a substring's position. `%scanrpl` replaces all occurrences.

### `%upper`, `%lower`

```rpgle
normalized = %upper(userInput);
friendly   = %lower(headerText);
```

Case conversion.

### `%char`, `%dec`

```rpgle
label = 'Qty: ' + %char(PR_QOH);          // number to char
price = %dec(priceString : 9 : 2);         // char to decimal, with precision
```

`%char` converts any type to its character representation. `%dec` converts character or numeric to packed decimal — with required precision arguments (digits:decimals).

### Concatenation

```rpgle
message = 'Supplier ' + %trim(SP_NAME)
        + ' has ' + %char(productCount)
        + ' active products.';
```

Plain `+` concatenates strings. This is why you often see `%char` sprinkled in — to turn numbers into strings first.

## File I/O status

### `%eof`, `%found`

```rpgle
read SUPPLIER;
if %eof(SUPPLIER); leave; endif;

chain 'ACME001' SUPPLIER;
if not %found(SUPPLIER);
  dsply 'Supplier not found';
endif;
```

`%eof` — true if the last read hit end of file. `%found` — true if the last CHAIN (or similar positioning operation) found a match. Check these immediately after the operation — they change on the next I/O.

### `%error`, `%status`

```rpgle
write SP_REC;
if %error;
  dsply ('Write failed with status: ' + %char(%status));
endif;
```

`%error` flags that the last operation raised an error. `%status` gives the numeric status code. More on these in [Chapter 7]({% link part-2/07-error-handling.md %}).

## Date and time

Briefly — full coverage in [Chapter 8]({% link part-2/08-date-math.md %}).

```rpgle
today  = %date();                  // today's date
now    = %time();                  // current time
stamp  = %timestamp();             // full timestamp
future = today + %days(30);        // 30 days from now
diff   = %diff(endDate : startDate : *days);  // days between
```

## Numeric

### `%abs`

```rpgle
variance = %abs(actual - expected);
```

Absolute value.

### `%int`, `%dec`

```rpgle
whole = %int(decimalValue);         // truncate to integer
exact = %dec(amount : 11 : 4);      // convert to packed, specified precision
```

## Control and lookup

### `%parms`

```rpgle
if %parms() >= 2;
  // second parameter was passed
endif;
```

Number of parameters actually passed to the current procedure. Used with optional parameters.

### `%lookup`

```rpgle
dcl-s regionCodes char(3) dim(50);
// ... regionCodes populated ...

idx = %lookup('NEU' : regionCodes);
if idx > 0;
  // found at position idx
endif;
```

Finds a value in an array. Returns the position or 0 if not found.

## One habit worth building

When you're writing expressions, reach for BIFs first. If you find yourself writing a loop to trim spaces, convert a type, or search a string, there's almost certainly a BIF that does it in one line. Read the IBM RPG reference occasionally just to remind yourself what's available — you'll discover BIFs you didn't know existed and simplify whole sections of code.

Next: [Chapter 5: Embedded SQL]({% link part-2/05-embedded-sql.md %}).
