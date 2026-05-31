📍 **Module 3 → Concept 1 of 5: Data Types — Numeric, String, Date/Time**

---

## 📖 THEORY — What is it and why does it exist?

### The Problem: Why can't everything just be text?

A beginner's instinct is to store everything as `VARCHAR` — it works for names, it works for prices, it works for dates. Why bother with anything else?

Here's what goes wrong:

```
Storing price as VARCHAR:
  '79999.00' vs '9999.00' vs '100000.00'

  ORDER BY price DESC → sorts alphabetically:
  '9999.00'    ← first (because '9' > '7' > '1' alphabetically)
  '79999.00'
  '100000.00'  ← last

  Actual correct order should be:
  100000.00  ← most expensive
  79999.00
  9999.00    ← cheapest

  Arithmetic also breaks:
  '79999.00' + '100' = '79999.00100' (string concat, not math)
```

**The right data type for each column gives you:**
- Correct sorting and arithmetic
- Efficient storage (INT uses 4 bytes; VARCHAR(20) for "12345" uses 6+ bytes)
- Built-in validation (can't store "banana" in an INT column)
- Faster indexes (fixed-width types index better than variable-width)

MySQL's data types fall into three families. Let's go through each.

---

## 🔢 NUMERIC TYPES

### Integer Types

| Type | Bytes | Signed Range | Unsigned Range | ShopNest Use |
|---|---|---|---|---|
| `TINYINT` | 1 | -128 to 127 | 0 to 255 | Boolean flags, ratings (1-5) |
| `SMALLINT` | 2 | -32,768 to 32,767 | 0 to 65,535 | Age, small counts |
| `MEDIUMINT` | 3 | -8.3M to 8.3M | 0 to 16.7M | Medium-scale IDs |
| `INT` | 4 | -2.1B to 2.1B | 0 to 4.2B | Most IDs, quantities |
| `BIGINT` | 8 | -9.2Q to 9.2Q | 0 to 18.4Q | Large-scale IDs, financial totals |

```sql
-- UNSIGNED doubles positive range, prevents negatives
stock         INT UNSIGNED        -- 0 to 4,294,967,295 (no negative stock)
rating        TINYINT UNSIGNED    -- 0 to 255 (we'll constrain to 1-5 with CHECK)
customer_id   INT UNSIGNED        -- 0 to 4,294,967,295

-- AUTO_INCREMENT must be used with a key
product_id    INT UNSIGNED AUTO_INCREMENT PRIMARY KEY
```

**BOOL / BOOLEAN** — MySQL stores these internally as `TINYINT(1)`:
```sql
is_active     TINYINT(1) NOT NULL DEFAULT 1   -- 1 = true, 0 = false
-- Or equivalently:
is_active     BOOLEAN NOT NULL DEFAULT TRUE
```

---

### Decimal / Floating Point Types

| Type | Storage | Precision | Use |
|---|---|---|---|
| `DECIMAL(M,D)` | Variable | Exact | Money, measurements — ALWAYS for financial data |
| `FLOAT` | 4 bytes | ~7 digits | Scientific data where approximation is OK |
| `DOUBLE` | 8 bytes | ~15 digits | Scientific data, higher precision than FLOAT |

```sql
-- DECIMAL(M, D):
--   M = total number of significant digits
--   D = digits after the decimal point
--   M - D = digits before the decimal point

price         DECIMAL(10, 2)   -- up to 99,999,999.99  ✅ for money
tax_rate      DECIMAL(5, 4)    -- up to 9.9999  e.g. 0.1800 for 18%
weight_kg     DECIMAL(8, 3)    -- up to 99999.999 kg

-- FLOAT / DOUBLE — approximate, never for money:
sensor_temp   FLOAT            -- 36.6000023... acceptable for sensors
pi_value      DOUBLE           -- 3.14159265358979...
```

---

## 🔤 STRING TYPES

| Type | Max Size | Storage | Use |
|---|---|---|---|
| `CHAR(N)` | 255 chars | Fixed N bytes always | Fixed-length codes |
| `VARCHAR(N)` | 65,535 bytes | Actual length + 1-2 bytes | Most text columns |
| `TEXT` | 65,535 bytes | On-disk, not in row | Long descriptions |
| `MEDIUMTEXT` | 16 MB | On-disk | Blog posts, large content |
| `LONGTEXT` | 4 GB | On-disk | Massive documents |
| `ENUM(...)` | 65,535 options | 1-2 bytes | Fixed set of values |
| `SET(...)` | 64 members | 1-8 bytes | Multiple values from fixed set |

```sql
-- CHAR: fixed width — pads with spaces to fill N bytes
-- Good for values that are ALWAYS the same length
country_code  CHAR(2)          -- 'IN', 'US', 'GB' — always 2 chars
order_ref     CHAR(10)         -- 'ORD0001234' — always 10 chars

-- VARCHAR: variable width — stores only what's needed
-- Good for text that varies in length
first_name    VARCHAR(50)      -- 'Raj' uses 4 bytes, 'Rahul' uses 6 bytes
email         VARCHAR(100)
product_name  VARCHAR(100)

-- TEXT family: stored off-row (slower access, can't have DEFAULT)
description   TEXT             -- product descriptions, reviews
-- Cannot do: description TEXT DEFAULT 'No description' ← ERROR

-- ENUM: one value from a fixed list — stored as integer internally
status        ENUM('pending', 'processing', 'shipped',
                   'delivered', 'cancelled')
-- Stored as 1 byte. Validates automatically.
-- MySQL rejects: status = 'banana' → Error

-- SET: multiple values from a fixed list
notifications SET('email', 'sms', 'push')
-- A customer can have: 'email', 'email,sms', 'sms,push', etc.
```

**CHAR vs VARCHAR — the key trade-off:**
```
CHAR(10) storing 'Hi':      H i _ _ _ _ _ _ _ _   (10 bytes, padded)
VARCHAR(10) storing 'Hi':   2 H i                  (3 bytes: length + data)

CHAR  → slightly faster reads (fixed offset), wastes space for short values
VARCHAR → efficient storage, tiny overhead for length byte
Rule: CHAR for always-fixed-length data. VARCHAR for everything else.
```

---

## 📅 DATE AND TIME TYPES

| Type | Format | Range | Storage | Use |
|---|---|---|---|---|
| `DATE` | YYYY-MM-DD | 1000-01-01 to 9999-12-31 | 3 bytes | Birthdates, deadlines |
| `TIME` | HH:MM:SS | -838:59:59 to 838:59:59 | 3 bytes | Durations, times of day |
| `DATETIME` | YYYY-MM-DD HH:MM:SS | 1000-01-01 to 9999-12-31 | 8 bytes | Business timestamps |
| `TIMESTAMP` | YYYY-MM-DD HH:MM:SS | 1970-01-01 to 2038-01-19 | 4 bytes | Auto-tracking, audit trails |
| `YEAR` | YYYY | 1901 to 2155 | 1 byte | Year-only fields |

```sql
-- DATE: just the calendar date
date_of_birth   DATE                          -- '1995-08-15'
delivery_date   DATE                          -- '2024-02-20'

-- DATETIME: date + time, timezone-naive (stores what you give it)
order_date      DATETIME NOT NULL
                DEFAULT CURRENT_TIMESTAMP     -- auto-sets to now on insert

-- TIMESTAMP: date + time, timezone-aware (converts to/from UTC)
--            Updates automatically when row changes (if configured)
--            WARNING: only valid until 2038-01-19 (Y2K38 problem)
created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                          ON UPDATE CURRENT_TIMESTAMP
-- updated_at auto-updates every time the row is modified ✅

-- TIME: duration or time-of-day
delivery_window TIME                          -- '14:00:00' (2 PM)
call_duration   TIME                          -- '00:45:30' (45 min 30 sec)

-- YEAR
model_year      YEAR                          -- 2024
```

**DATETIME vs TIMESTAMP — the critical difference:**
```
DATETIME:
  → Stores exactly what you insert: '2024-08-15 14:30:00'
  → No timezone conversion — always same value regardless of server timezone
  → Range: year 1000 to 9999
  → Use for: order dates, appointment times, any business date

TIMESTAMP:
  → Converts to UTC on storage, converts back on retrieval
  → If server timezone changes, all TIMESTAMP values shift
  → Range: 1970 to 2038 (DANGER for systems running past 2038)
  → Use for: created_at, updated_at (automatic tracking columns)
  → Benefit: ON UPDATE CURRENT_TIMESTAMP magic
```

---

## 💡 EXAMPLE — ShopNest Data Types in Context

```sql
-- Complete ShopNest customers table with justified data types:
CREATE TABLE customers (
    customer_id   INT UNSIGNED        AUTO_INCREMENT PRIMARY KEY,
    -- INT UNSIGNED: IDs are never negative, up to 4.2 billion customers

    first_name    VARCHAR(50)         NOT NULL,
    last_name     VARCHAR(50)         NOT NULL,
    -- VARCHAR(50): names vary in length, 50 chars is generous

    email         VARCHAR(100)        NOT NULL UNIQUE,
    -- VARCHAR(100): emails vary, 100 is the RFC standard max

    phone         VARCHAR(15)         NULL,
    -- VARCHAR not INT: preserves +91 prefix, leading zeros, dashes

    city          VARCHAR(50)         NOT NULL,
    state         VARCHAR(50)         NOT NULL,
    -- VARCHAR: city/state names vary in length

    date_of_birth DATE                NULL,
    -- DATE: we only need the calendar date, not time

    loyalty_tier  ENUM('bronze','silver','gold','platinum')
                                      NOT NULL DEFAULT 'bronze',
    -- ENUM: fixed set of tiers, stored as 1 byte, validated automatically

    is_active     BOOLEAN             NOT NULL DEFAULT TRUE,
    -- BOOLEAN (TINYINT(1)): simple flag

    created_at    TIMESTAMP           NOT NULL DEFAULT CURRENT_TIMESTAMP,
    -- TIMESTAMP: auto-set on insert, we want UTC tracking

    updated_at    TIMESTAMP           NOT NULL DEFAULT CURRENT_TIMESTAMP
                                      ON UPDATE CURRENT_TIMESTAMP
    -- TIMESTAMP + ON UPDATE: auto-updates whenever row changes
) ENGINE = InnoDB;
```

---

## ⚠️ GOTCHAS — Data Type Traps

**1. The FLOAT money trap (again — it bears repeating)**
```sql
-- In MySQL:
SELECT 0.1 + 0.2;          -- Returns 0.30000000000000004 with FLOAT
SELECT 0.1 + 0.2 = 0.3;    -- Returns 0 (FALSE!) with FLOAT

-- With DECIMAL:
SELECT CAST(0.1 AS DECIMAL(10,2)) + CAST(0.2 AS DECIMAL(10,2));
-- Returns 0.30 exactly ✅
```

**2. VARCHAR(255) everywhere is lazy but costly**
```sql
-- country_code VARCHAR(255)  ← wastes index space, misleads developers
-- country_code CHAR(2)       ← correct, self-documenting, efficient
-- Right-size your columns. It matters for indexes and joins.
```

**3. ENUM seems great until you need to add a value**
```sql
-- Adding a new ENUM value requires ALTER TABLE:
ALTER TABLE orders
MODIFY status ENUM('pending','processing','shipped',
                   'delivered','cancelled','returned');
-- In MySQL 8, this is fast (instant DDL)
-- In older MySQL, this rebuilds the entire table
-- If values change frequently, use a lookup table instead of ENUM
```

**4. TEXT columns can't have DEFAULT values**
```sql
description TEXT DEFAULT 'No description'  -- ❌ ERROR in MySQL
description TEXT NULL                      -- ✅ NULL means "not provided"
-- If you need a default, use VARCHAR instead of TEXT
```

**5. DATETIME doesn't auto-populate**
```sql
-- This does NOT auto-set the time:
order_date DATETIME NOT NULL
-- Inserting without specifying order_date → ERROR

-- This DOES auto-set:
order_date DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ✅
```

**6. Storing phone as INT loses data**
```sql
-- Phone: +91-98765-43210
-- As INT → impossible (has non-numeric characters)
-- As BIGINT → 9876543210 (loses +91, loses formatting)
-- As VARCHAR(15) → '+91-98765-43210' ✅ stores exactly as-is
```

---

✅ **Quick Check:**
ShopNest's developer designs this column for product ratings:

```sql
rating FLOAT NOT NULL DEFAULT 0
```

Identify **three specific problems** with this choice and write the corrected column definition with proper data type, constraints, and justification for each decision.
