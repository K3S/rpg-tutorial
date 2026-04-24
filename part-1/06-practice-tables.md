---
title: "6. Create your practice tables"
parent: "Part 1: Get Set Up"
nav_order: 6
---

# Create your practice tables

The tutorials in Parts 2 and 3 use three tables: **SUPPLIER**, **PRODUCT**, and **REORDCND**. Every code example references the same tables and the same sample data, so what you learn in Chapter 1 of Part 2 still fits when you get to Tutorial 10 in Part 3.

This chapter gives you the SQL that creates those tables and populates them with realistic sample data — 15 suppliers, 31 products, and an empty REORDCND you'll populate later in Tutorial 5.

## Find your library name

PUB400 provisions two libraries per user, typically named after your user profile. To find yours, in VS Code's IBM i sidebar, expand **User Library List**. The first library you have write access to is the one you'll use. Note the name — you'll substitute it into the script below.

If you're unsure, you can also run this SQL in the Db2 for IBM i extension:

```sql
SELECT CURRENT_USER AS MY_USER
  FROM SYSIBM.SYSDUMMY1;
```

Your private library name is usually equal to or derived from your user profile.

## The script

Create a new file in VS Code called `setup-tables.sql`. Paste the script below. **Change `YOUR_LIBRARY` on the `SET SCHEMA` line** to your actual library name before running.

```sql
-- ================================================================
--  RPG Tutorial — Practice Tables Setup
-- ================================================================
--  Creates SUPPLIER, PRODUCT, and REORDCND tables in your
--  personal library, with realistic sample data.
--
--  BEFORE RUNNING: Replace YOUR_LIBRARY below with your actual
--  PUB400 library name (visible in the VS Code IBM i sidebar
--  under User Library List).
-- ================================================================

SET SCHEMA = 'YOUR_LIBRARY';

-- ----------------------------------------------------------------
-- SUPPLIER — master list of suppliers
-- ----------------------------------------------------------------
CREATE TABLE SUPPLIER (
  SP_SUPL     CHAR(10)     NOT NULL,
  SP_NAME     VARCHAR(50)  NOT NULL,
  SP_ACTIVE   CHAR(1)      NOT NULL DEFAULT 'Y',
  SP_ONBOARD  DATE         NOT NULL,
  CONSTRAINT SUPPLIER_PK PRIMARY KEY (SP_SUPL)
)
RCDFMT SP_REC;

INSERT INTO SUPPLIER
  (SP_SUPL, SP_NAME, SP_ACTIVE, SP_ONBOARD) VALUES
  ('ACME001',  'Acme Distributors',       'Y', '2018-03-15'),
  ('BETA002',  'Beta Supply Co',          'Y', '2019-07-22'),
  ('CTRL003',  'Central Manufacturing',   'Y', '2015-01-08'),
  ('DLTA004',  'Delta Wholesale',         'N', '2014-05-12'),
  ('EPSL005',  'Epsilon Products',        'Y', '2020-11-03'),
  ('FOXT006',  'Foxtrot Inc',             'Y', '2017-09-30'),
  ('GAMA007',  'Gamma Industries',        'N', '2013-02-14'),
  ('HOTL008',  'Hotel Partners',          'Y', '2021-06-18'),
  ('INDG009',  'Indigo Holdings',         'Y', '2016-12-01'),
  ('JULT010',  'Juliet Trading',          'Y', '2022-04-27'),
  ('KILO011',  'Kilo Corp',               'Y', '2019-08-15'),
  ('LIMA012',  'Lima Supplies',           'Y', '2020-03-09'),
  ('MIKE013',  'Mike''s Warehouse',       'N', '2012-10-21'),
  ('NOVE014',  'November Solutions',      'Y', '2023-01-05'),
  ('OSCR015',  'Oscar Distribution',      'Y', '2018-11-28');

-- ----------------------------------------------------------------
-- PRODUCT — product master with inventory data
-- ----------------------------------------------------------------
CREATE TABLE PRODUCT (
  PR_PROD      CHAR(15)     NOT NULL,
  PR_SUPL      CHAR(10)     NOT NULL,
  PR_ACTIVE    CHAR(1)      NOT NULL DEFAULT 'Y',
  PR_QOH       DECIMAL(9,0) NOT NULL DEFAULT 0,
  PR_REORDPT   DECIMAL(9,0) NOT NULL DEFAULT 0,
  PR_SAFTYSTK  DECIMAL(9,0) NOT NULL DEFAULT 0,
  PR_PRICE     DECIMAL(9,2) NOT NULL DEFAULT 0,
  PR_CHGDT     DATE         NOT NULL,
  CONSTRAINT PRODUCT_PK PRIMARY KEY (PR_PROD),
  CONSTRAINT PRODUCT_SUPPLIER_FK
    FOREIGN KEY (PR_SUPL) REFERENCES SUPPLIER (SP_SUPL)
)
RCDFMT PR_REC;

INSERT INTO PRODUCT
  (PR_PROD, PR_SUPL, PR_ACTIVE, PR_QOH, PR_REORDPT, PR_SAFTYSTK, PR_PRICE, PR_CHGDT) VALUES
  -- Acme
  ('WIDGET-STD-001', 'ACME001', 'Y',  450,  500,  100, 12.99, '2026-02-15'),
  ('WIDGET-PRO-001', 'ACME001', 'Y', 2500, 1000,  200, 29.99, '2026-03-01'),
  ('WIDGET-DLX-001', 'ACME001', 'Y',   75,  150,   30, 49.99, '2026-02-10'),
  -- Beta
  ('BOLT-M8-STEEL',  'BETA002', 'Y',   50,  200,   50,  0.15, '2026-01-20'),
  ('BOLT-M10-STEEL', 'BETA002', 'Y',  890,  500,  100,  0.22, '2026-02-28'),
  ('BOLT-M12-STEEL', 'BETA002', 'Y', 1500,  800,  200,  0.35, '2026-03-08'),
  ('NUT-M8-STEEL',   'BETA002', 'Y',  125,  300,   75,  0.08, '2026-02-14'),
  -- Central
  ('GEAR-SPUR-24T',  'CTRL003', 'Y',   12,   50,   10,  8.75, '2026-03-10'),
  ('GEAR-SPUR-48T',  'CTRL003', 'Y',  180,  100,   25, 14.50, '2026-03-12'),
  ('BEARING-608ZZ',  'CTRL003', 'Y', 1200,  800,  200,  1.85, '2026-03-05'),
  ('BEARING-6205',   'CTRL003', 'Y',  340,  200,   50,  4.25, '2026-02-28'),
  -- Delta (inactive supplier)
  ('SPROCKET-15T',   'DLTA004', 'N',    0,    0,    0,  4.50, '2023-12-15'),
  -- Epsilon
  ('SENSOR-TEMP-A1', 'EPSL005', 'Y',   85,  100,   25, 45.00, '2026-02-22'),
  ('SENSOR-TEMP-A2', 'EPSL005', 'Y',  320,  100,   25, 52.50, '2026-03-12'),
  ('SENSOR-PRES-P1', 'EPSL005', 'Y',   48,   75,   15, 68.00, '2026-02-25'),
  -- Foxtrot
  ('BRACKET-L-12IN', 'FOXT006', 'Y',    5,   75,   20,  3.25, '2026-02-08'),
  ('BRACKET-L-18IN', 'FOXT006', 'Y',  220,  100,   25,  4.75, '2026-03-01'),
  ('BRACKET-Z-12IN', 'FOXT006', 'Y',  650,  300,   75,  5.50, '2026-02-19'),
  -- Gamma (inactive supplier)
  ('CLAMP-LEGACY',   'GAMA007', 'N',   12,    0,    0,  2.00, '2022-08-14'),
  -- Hotel
  ('HOSE-HYD-1IN',   'HOTL008', 'Y',   38,   50,   10, 28.50, '2026-02-11'),
  ('HOSE-HYD-2IN',   'HOTL008', 'Y',  115,   75,   20, 42.00, '2026-03-03'),
  ('FITTING-HYD-1',  'HOTL008', 'Y',  480,  250,   50,  8.25, '2026-02-27'),
  -- Indigo
  ('PAINT-WHT-1GAL', 'INDG009', 'Y',   28,  100,   25, 35.00, '2026-02-05'),
  ('PAINT-BLK-1GAL', 'INDG009', 'Y',  155,  100,   25, 35.00, '2026-03-06'),
  ('PAINT-RED-1GAL', 'INDG009', 'Y',   72,  100,   25, 38.00, '2026-02-18'),
  -- Juliet
  ('LABEL-STD-2X3',  'JULT010', 'Y', 8500, 5000, 1000,  0.02, '2026-03-11'),
  ('LABEL-LRG-4X6',  'JULT010', 'Y',  980, 2000,  400,  0.05, '2026-02-21'),
  -- Kilo
  ('FILTER-AIR-STD', 'KILO011', 'Y',   65,  150,   30, 18.50, '2026-02-16'),
  ('FILTER-OIL-STD', 'KILO011', 'Y',  240,  200,   40, 22.00, '2026-03-04'),
  -- Lima
  ('ADHESIVE-EPX-A', 'LIMA012', 'Y',   92,  120,   30, 16.75, '2026-02-24'),
  ('ADHESIVE-CNA-B', 'LIMA012', 'Y',  310,  150,   35, 14.50, '2026-03-07');

-- ----------------------------------------------------------------
-- REORDCND — reorder candidates (output table)
-- Starts empty. Populated by the batch RPG program in Tutorial 5.
-- ----------------------------------------------------------------
CREATE TABLE REORDCND (
  RC_PROD    CHAR(15)     NOT NULL,
  RC_SUPL    CHAR(10)     NOT NULL,
  RC_QOH     DECIMAL(9,0) NOT NULL,
  RC_REORD   DECIMAL(9,0) NOT NULL,
  RC_SUGG    DECIMAL(9,0) NOT NULL,
  RC_CRTDT   DATE         NOT NULL
)
RCDFMT RC_REC;

-- ----------------------------------------------------------------
-- Verification queries — run these after the script completes
-- ----------------------------------------------------------------
SELECT COUNT(*) AS SUPPLIER_COUNT FROM SUPPLIER;
-- Expected: 15

SELECT COUNT(*) AS PRODUCT_COUNT FROM PRODUCT;
-- Expected: 31

SELECT COUNT(*) AS REORDCND_COUNT FROM REORDCND;
-- Expected: 0

-- How many active products are below their reorder point?
SELECT COUNT(*) AS BELOW_REORDER
  FROM PRODUCT
 WHERE PR_ACTIVE = 'Y' AND PR_QOH < PR_REORDPT;
-- Expected: 14
```

## How to run it

With the Db2 for IBM i extension installed (part of the Development Pack from Chapter 4), you have a few options:

**Run the whole script at once** — Right-click anywhere in the editor and choose "Run all statements."

**Run one statement at a time** — Place your cursor inside a statement and press **Ctrl+R** (Windows) or **Cmd+R** (macOS). Useful for the verification queries at the bottom, where you want to see each count separately.

Results appear in a panel below the editor.

## The field-prefix convention

Notice every field name starts with a two-letter prefix: `SP_` for supplier fields, `PR_` for product fields, `RC_` for reorder-candidate fields. This is a convention used throughout the tutorial examples. The reason: when you're looking at RPG code with twenty variables in scope, the prefix tells you at a glance which table each came from. Real shops enforce conventions like this, and reading code is much easier when every field carries this kind of self-identifying context.

## What's in the sample data

A few things worth noticing, because they come up in the tutorials:

- **3 inactive suppliers** (Delta, Gamma, Mike) — Tutorial 5's reorder report must skip their products even if stock is low
- **2 inactive products** (Sprocket-15T, Clamp-Legacy) — should be skipped by any active-inventory query
- **14 active products below their reorder point** — these are exactly the records Tutorial 5's batch program will write into REORDCND
- **Extreme cases** — a bracket with QOH=5 against a reorder point of 75 (severely below), a label with QOH=8500 (well-stocked), products with prices ranging from $0.02 to $68.00

## When you're done

Your three tables exist, the verification counts match, and you're ready to write some RPG. Next: [Chapter 7: Hello World]({% link part-1/07-hello-world.md %}).
