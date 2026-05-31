📍 **Module 3 → Concept 2 of 5: CREATE / DROP / ALTER DATABASE**

---

## 📖 THEORY — What is it and why does it exist?

### What is a Database in MySQL's world?

Before you create a single table, you need a **database** — MySQL's term for a named container that holds a collection of related tables, views, stored procedures, and other objects.

Think of it this way:
```
MySQL Server
├── shopnest          ← database (our e-commerce store)
│   ├── customers     ← table
│   ├── orders        ← table
│   └── products      ← table
├── shopnest_staging  ← database (for testing)
│   ├── customers
│   └── ...
└── analytics_db      ← database (separate reporting system)
    └── ...
```

One MySQL server can host **hundreds of databases** simultaneously — each completely isolated from the others. A query in `shopnest` cannot accidentally touch `analytics_db` unless you explicitly cross-reference it.

MySQL uses the words **DATABASE** and **SCHEMA** interchangeably — `CREATE DATABASE` and `CREATE SCHEMA` do exactly the same thing. You'll see both in the wild.

---

### Why Manage Databases Programmatically?

Three real ShopNest scenarios where database-level DDL matters:

```
Scenario 1 — Environments
  shopnest_prod     → live data, real customers
  shopnest_staging  → copy for QA testing
  shopnest_dev      → developer sandboxes

Scenario 2 — Multi-tenancy
  shopnest_india    → Indian operations
  shopnest_uae      → UAE operations
  (separate databases per region for compliance)

Scenario 3 — Disaster Recovery
  DROP DATABASE shopnest_dev;    → wipe dev environment cleanly
  CREATE DATABASE shopnest_dev;  → start fresh for new sprint
```

---

## 🔤 SYNTAX — All Database-Level Commands

### ① CREATE DATABASE

```sql
-- Basic creation
CREATE DATABASE shopnest;

-- Safe creation — no error if it already exists
CREATE DATABASE IF NOT EXISTS shopnest;

-- Creation with explicit character set and collation
CREATE DATABASE shopnest
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

-- Full production-grade version (recommended):
CREATE DATABASE IF NOT EXISTS shopnest
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;
```

**Breaking down the options:**

| Clause | What it does |
|---|---|
| `IF NOT EXISTS` | Prevents error if DB already exists — safe to run repeatedly |
| `CHARACTER SET` | Defines which characters can be stored |
| `COLLATE` | Defines how characters are compared and sorted |

---

### ② CHARACTER SET and COLLATION — The Critical Detail

This is where most beginners make a database-wide mistake they regret later.

**Character Set** = which characters your database can store:
```
latin1      → Western European only (no Hindi, no emoji, no Arabic)
utf8        → Unicode BUT only up to 3 bytes (misses emoji! 😱)
utf8mb4     → Full Unicode, 4 bytes (emoji ✅, all languages ✅)
```

**Collation** = how characters are compared (affects ORDER BY, WHERE, indexes):
```
utf8mb4_general_ci   → fast but slightly inaccurate comparisons
utf8mb4_unicode_ci   → accurate Unicode comparisons (recommended)
utf8mb4_bin          → binary/case-sensitive comparison

_ci suffix = Case Insensitive  ('Rahul' = 'rahul' = 'RAHUL')
_cs suffix = Case Sensitive    ('Rahul' ≠ 'rahul')
_bin suffix = Binary           (byte-by-byte comparison)
```

**Real ShopNest consequence:**
```sql
-- With utf8 (wrong):
INSERT INTO products (name) VALUES ('iPhone 15 📱');
-- Silently truncates or errors — emoji requires 4 bytes, utf8 only supports 3

-- With utf8mb4 (correct):
INSERT INTO products (name) VALUES ('iPhone 15 📱');
-- Stores perfectly ✅

-- With utf8mb4_unicode_ci collation:
SELECT * FROM customers WHERE first_name = 'rahul';
-- Finds 'Rahul', 'RAHUL', 'rahul' — case insensitive ✅
```

**Always use `utf8mb4` + `utf8mb4_unicode_ci` for any modern application.**

---

### ③ USE DATABASE

```sql
-- Switch to a database (all subsequent queries run against it)
USE shopnest;

-- Check which database you're currently in
SELECT DATABASE();
-- Returns: shopnest
```

---

### ④ ALTER DATABASE

```sql
-- Change character set and collation of existing database
ALTER DATABASE shopnest
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

-- Note: this changes the DEFAULT for NEW tables created in this database
-- Existing tables keep their original character set until altered separately
```

---

### ⑤ DROP DATABASE

```sql
-- Delete a database and ALL its contents permanently
DROP DATABASE shopnest;

-- Safe version — no error if database doesn't exist
DROP DATABASE IF EXISTS shopnest;

-- ⚠️ WARNING: This is instant and IRREVERSIBLE.
-- No confirmation prompt. No recycle bin.
-- Every table, every row, every procedure — gone.
```

---

### ⑥ Inspection Commands

```sql
-- List all databases on this server
SHOW DATABASES;

-- Show the CREATE DATABASE statement for an existing database
-- (useful to check its character set and collation)
SHOW CREATE DATABASE shopnest;
-- Output:
-- CREATE DATABASE `shopnest`
-- /*!40100 DEFAULT CHARACTER SET utf8mb4
--  COLLATE utf8mb4_unicode_ci */

-- Show all available character sets
SHOW CHARACTER SET;

-- Show all available collations (filter by utf8mb4)
SHOW COLLATION WHERE Charset = 'utf8mb4';

-- Show server's default character set
SHOW VARIABLES LIKE 'character_set_server';
SHOW VARIABLES LIKE 'collation_server';
```

---

## 💡 EXAMPLE — ShopNest Full Environment Setup

```sql
-- ================================================
-- ShopNest Database Environment Setup
-- Run as root or shopnest_dba user
-- ================================================

-- Step 1: Clean slate (safe — won't error if doesn't exist)
DROP DATABASE IF EXISTS shopnest_dev;
DROP DATABASE IF EXISTS shopnest_staging;
DROP DATABASE IF EXISTS shopnest_prod;

-- Step 2: Create all three environments
CREATE DATABASE IF NOT EXISTS shopnest_prod
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

CREATE DATABASE IF NOT EXISTS shopnest_staging
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

CREATE DATABASE IF NOT EXISTS shopnest_dev
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

-- Step 3: Verify all three exist
SHOW DATABASES;
-- Output includes:
-- shopnest_dev
-- shopnest_prod
-- shopnest_staging

-- Step 4: Check prod was created correctly
SHOW CREATE DATABASE shopnest_prod;
-- Should show utf8mb4 and utf8mb4_unicode_ci

-- Step 5: Switch into prod for all further work
USE shopnest_prod;

-- Step 6: Confirm we're in the right database
SELECT DATABASE();
-- Output: shopnest_prod

-- Step 7: Sprint ended, dev environment needs a fresh start
DROP DATABASE IF EXISTS shopnest_dev;
CREATE DATABASE IF NOT EXISTS shopnest_dev
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;
-- Clean slate for the new sprint ✅
```

---

## ⚠️ GOTCHAS — Database-Level Mistakes

**1. Using `utf8` instead of `utf8mb4`**
```sql
-- MySQL's 'utf8' is NOT real UTF-8. It's a broken 3-byte subset.
-- It silently truncates or errors on emoji and many Asian characters.
-- This is a historical MySQL mistake that can't be fixed for backward compat.
-- ALWAYS use utf8mb4. No exceptions for modern applications.

CREATE DATABASE shopnest CHARACTER SET utf8;    -- ❌ broken
CREATE DATABASE shopnest CHARACTER SET utf8mb4; -- ✅ correct
```

**2. Dropping the wrong database**
```sql
-- There is NO "are you sure?" prompt in MySQL.
DROP DATABASE shopnest_prod;  -- instant, permanent, catastrophic if wrong

-- Safe practice: always verify you're targeting the right DB first
SELECT DATABASE();            -- check current context
SHOW DATABASES;               -- visually confirm the name
-- Then drop.
```

**3. Forgetting `USE` and querying the wrong database**
```sql
-- You connect to MySQL, forget to USE shopnest_prod
-- and run queries against whatever was selected last session.
-- Always start sessions with:
USE shopnest_prod;
SELECT DATABASE(); -- verify before any destructive operations
```

**4. `ALTER DATABASE` doesn't fix existing tables**
```sql
-- You realize your database was created with latin1
ALTER DATABASE shopnest CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
-- This only affects NEW tables created after this point.
-- Existing tables still have latin1. You must ALTER each table too:
ALTER TABLE customers CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
-- This is painful — get the character set right at creation time.
```

**5. Cross-database queries are possible but dangerous**
```sql
-- MySQL allows querying across databases in the same server:
SELECT * FROM shopnest_prod.customers;  -- from inside shopnest_dev
-- Useful for comparisons, dangerous if you accidentally write to prod
-- Always double-check SELECT DATABASE() before write operations
```

**6. Database names are case-sensitive on Linux, not on Windows**
```sql
-- On Windows: shopnest = SHOPNEST = ShopNest (same database)
-- On Linux:   shopnest ≠ ShopNest (different databases!)
-- Always use lowercase database names to avoid cross-platform bugs.
-- Convention: snake_case for all database and table names.
```

---

✅ **Quick Check:**
A ShopNest developer creates the production database with this command:

```sql
CREATE DATABASE shopnest CHARACTER SET utf8;
```

Six months later, customers start reporting that their names in regional languages (Telugu, Tamil, Hindi) and some product names with emoji are showing as `????` or getting truncated. 

What is the root cause, what should the original command have been, and what must the developer do now to fix **both** the database default **and** the already-existing tables?
