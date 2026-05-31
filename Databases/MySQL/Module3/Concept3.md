📍 **Module 3 → Concept 3 of 5: CREATE TABLE, ALTER TABLE, RENAME, DROP, TRUNCATE**

---

## 📖 THEORY — What is it and why does it exist?

### DDL — The Language That Builds Your Database

So far we've designed ShopNest on paper. Now we build it. **DDL (Data Definition Language)** is the category of SQL commands that create, modify, and delete the *structure* of your database — not the data inside it, but the containers that hold data.

Think of DDL as **construction work**:
- `CREATE TABLE` = building a new room
- `ALTER TABLE` = renovating an existing room (adding walls, moving doors)
- `RENAME TABLE` = changing the room's nameplate
- `TRUNCATE TABLE` = emptying the room completely but keeping its structure
- `DROP TABLE` = demolishing the room entirely

The critical difference between DDL and DML (Data Manipulation Language — INSERT, UPDATE, DELETE) is that **DDL operations are auto-committed** in MySQL. They cannot be rolled back with `ROLLBACK`. Once you `DROP TABLE`, it's gone — no transaction can save you.

---

## 🔤 SYNTAX — Every Table-Level DDL Command

---

### ① CREATE TABLE

```sql
CREATE TABLE [IF NOT EXISTS] table_name (
    column1  datatype  [constraints],
    column2  datatype  [constraints],
    ...
    [table_constraints]
) ENGINE = InnoDB;
```

**Full ShopNest example with every clause explained:**

```sql
CREATE TABLE IF NOT EXISTS products (

    -- Column definition: name  datatype  column-constraints
    product_id    INT UNSIGNED        NOT NULL AUTO_INCREMENT,
    category_id   INT UNSIGNED        NOT NULL,
    name          VARCHAR(100)        NOT NULL,
    description   TEXT                NULL,
    price         DECIMAL(10,2)       NOT NULL,
    stock         INT UNSIGNED        NOT NULL    DEFAULT 0,
    is_active     BOOLEAN             NOT NULL    DEFAULT TRUE,
    created_at    TIMESTAMP           NOT NULL    DEFAULT CURRENT_TIMESTAMP,
    updated_at    TIMESTAMP           NOT NULL    DEFAULT CURRENT_TIMESTAMP
                                                  ON UPDATE CURRENT_TIMESTAMP,

    -- Table-level constraints (defined after all columns)
    PRIMARY KEY (product_id),
    FOREIGN KEY (category_id) REFERENCES categories(category_id)
        ON DELETE RESTRICT
        ON UPDATE CASCADE

) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4
  COLLATE = utf8mb4_unicode_ci;
```

**Column constraint options:**

| Constraint | Meaning |
|---|---|
| `NOT NULL` | Column must have a value — NULL not allowed |
| `NULL` | Column is optional (MySQL default if unspecified) |
| `DEFAULT value` | Value used when column is omitted on INSERT |
| `AUTO_INCREMENT` | MySQL auto-generates next integer value |
| `UNIQUE` | No two rows can have the same value in this column |
| `PRIMARY KEY` | Unique + NOT NULL + clustered index (one per table) |

---

### ② CREATE TABLE AS SELECT — Copy a table's data

```sql
-- Create a new table from the results of a SELECT query
CREATE TABLE products_backup AS
    SELECT * FROM products;

-- Create a filtered copy
CREATE TABLE high_value_orders AS
    SELECT * FROM orders WHERE total_amount > 10000;

-- ⚠️ WARNING: copies DATA only, not constraints/indexes/foreign keys
```

---

### ③ ALTER TABLE — Modify existing table structure

This is the most complex DDL command because it has many sub-operations:

```sql
-- ─────────────────────────────────────────
-- ADD a new column
-- ─────────────────────────────────────────
ALTER TABLE customers
    ADD COLUMN loyalty_points INT UNSIGNED NOT NULL DEFAULT 0;

-- Add column at a specific position (FIRST or AFTER column_name)
ALTER TABLE customers
    ADD COLUMN middle_name VARCHAR(50) NULL AFTER first_name;

ALTER TABLE customers
    ADD COLUMN customer_ref CHAR(10) NOT NULL FIRST;

-- ─────────────────────────────────────────
-- DROP a column
-- ─────────────────────────────────────────
ALTER TABLE customers
    DROP COLUMN middle_name;
-- ⚠️ Permanent. All data in that column is lost.

-- ─────────────────────────────────────────
-- MODIFY a column — change datatype or constraints
-- (keeps the column name the same)
-- ─────────────────────────────────────────
ALTER TABLE products
    MODIFY COLUMN name VARCHAR(150) NOT NULL;
-- Changed from VARCHAR(100) to VARCHAR(150)

ALTER TABLE customers
    MODIFY COLUMN phone VARCHAR(20) NULL;
-- Changed from VARCHAR(15) to VARCHAR(20), made nullable

-- ─────────────────────────────────────────
-- CHANGE a column — rename AND/OR change definition
-- ─────────────────────────────────────────
ALTER TABLE products
    CHANGE COLUMN is_active is_published BOOLEAN NOT NULL DEFAULT FALSE;
-- Renamed 'is_active' to 'is_published' AND changed default

-- ─────────────────────────────────────────
-- ADD a constraint
-- ─────────────────────────────────────────
ALTER TABLE orders
    ADD CONSTRAINT chk_total
    CHECK (total_amount >= 0);

ALTER TABLE products
    ADD CONSTRAINT uq_product_name UNIQUE (name);

-- ─────────────────────────────────────────
-- DROP a constraint
-- ─────────────────────────────────────────
ALTER TABLE products
    DROP INDEX uq_product_name;       -- drop UNIQUE constraint

ALTER TABLE orders
    DROP CHECK chk_total;             -- drop CHECK constraint

ALTER TABLE products
    DROP FOREIGN KEY fk_name;         -- drop FOREIGN KEY (need constraint name)

-- ─────────────────────────────────────────
-- Multiple changes in ONE ALTER TABLE
-- (more efficient — one table rebuild instead of many)
-- ─────────────────────────────────────────
ALTER TABLE customers
    ADD COLUMN loyalty_points  INT UNSIGNED NOT NULL DEFAULT 0,
    ADD COLUMN loyalty_tier    ENUM('bronze','silver','gold','platinum')
                               NOT NULL DEFAULT 'bronze',
    MODIFY COLUMN phone        VARCHAR(20) NULL,
    DROP COLUMN customer_ref;
```

---

### ④ RENAME TABLE

```sql
-- Rename a single table
RENAME TABLE products TO shop_products;

-- Rename multiple tables in one atomic operation
RENAME TABLE
    products        TO shop_products,
    categories      TO shop_categories,
    customers       TO shop_customers;
-- All renames happen together — no partial state

-- Alternative syntax (renames one table):
ALTER TABLE shop_products RENAME TO products;
```

---

### ⑤ TRUNCATE TABLE — Empty a table, keep its structure

```sql
-- Remove ALL rows from the table instantly
TRUNCATE TABLE order_items;

-- What TRUNCATE does differently from DELETE:
-- 1. Drops and recreates the table internally (much faster)
-- 2. Resets AUTO_INCREMENT counter back to 1
-- 3. Cannot be rolled back (DDL, not DML)
-- 4. Does not fire BEFORE/AFTER DELETE triggers
-- 5. Cannot TRUNCATE a table referenced by a foreign key
--    (unless the referencing table is truncated first)
```

---

### ⑥ DROP TABLE — Delete table structure AND all data

```sql
-- Drop a single table
DROP TABLE order_items;

-- Safe version
DROP TABLE IF EXISTS order_items;

-- Drop multiple tables (order matters for foreign keys)
-- Drop child tables before parent tables:
DROP TABLE IF EXISTS order_items;   -- child (has FKs to orders, products)
DROP TABLE IF EXISTS orders;        -- child (has FK to customers)
DROP TABLE IF EXISTS products;      -- child (has FK to categories)
DROP TABLE IF EXISTS customers;     -- parent
DROP TABLE IF EXISTS categories;    -- parent

-- ⚠️ Cannot drop a parent table if child tables still reference it:
DROP TABLE IF EXISTS customers;
-- ERROR: Cannot drop table 'customers' referenced by a foreign key
-- constraint in table 'orders'
```

---

### ⑦ Inspection Commands

```sql
-- List all tables in current database
SHOW TABLES;

-- Show full CREATE TABLE statement for a table
-- (essential for understanding existing tables)
SHOW CREATE TABLE products;

-- Show column definitions
DESCRIBE products;
-- or shorthand:
DESC products;

-- Show more detailed column info
SHOW COLUMNS FROM products;

-- Show all indexes on a table
SHOW INDEX FROM products;
```

---

## 💡 EXAMPLE — ShopNest Schema Evolution Over Time

```sql
-- ================================================
-- DAY 1: Initial launch schema
-- ================================================
USE shopnest_prod;

CREATE TABLE IF NOT EXISTS categories (
    category_id  INT UNSIGNED     AUTO_INCREMENT PRIMARY KEY,
    name         VARCHAR(50)      NOT NULL UNIQUE,
    description  TEXT             NULL
) ENGINE = InnoDB DEFAULT CHARSET = utf8mb4 COLLATE = utf8mb4_unicode_ci;

CREATE TABLE IF NOT EXISTS products (
    product_id   INT UNSIGNED     AUTO_INCREMENT PRIMARY KEY,
    category_id  INT UNSIGNED     NOT NULL,
    name         VARCHAR(100)     NOT NULL,
    price        DECIMAL(10,2)    NOT NULL,
    stock        INT UNSIGNED     NOT NULL DEFAULT 0,
    created_at   TIMESTAMP        NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (category_id) REFERENCES categories(category_id)
) ENGINE = InnoDB DEFAULT CHARSET = utf8mb4 COLLATE = utf8mb4_unicode_ci;

-- ================================================
-- WEEK 2: Business decides to add product images
-- ================================================
ALTER TABLE products
    ADD COLUMN image_url VARCHAR(500) NULL AFTER name;

-- ================================================
-- MONTH 1: Add product weight for shipping calc
-- and a SKU code for warehouse management
-- ================================================
ALTER TABLE products
    ADD COLUMN weight_kg  DECIMAL(8,3)  NULL,
    ADD COLUMN sku        VARCHAR(50)   NULL UNIQUE;

-- ================================================
-- MONTH 3: 'description' column needs to be longer,
-- and we realize VARCHAR(100) for name is too short
-- ================================================
ALTER TABLE products
    MODIFY COLUMN name        VARCHAR(200) NOT NULL,
    MODIFY COLUMN description MEDIUMTEXT   NULL;

-- ================================================
-- MONTH 6: Rename for consistency with new naming convention
-- ================================================
RENAME TABLE categories TO product_categories;

-- ================================================
-- YEAR 1: Dev environment needs a fresh start
-- ================================================
-- In dev only — clear all orders data, reset counters
TRUNCATE TABLE order_items;
TRUNCATE TABLE orders;
-- AUTO_INCREMENT resets to 1, structure preserved ✅

-- ================================================
-- Verify current state
-- ================================================
SHOW TABLES;
DESC products;
SHOW CREATE TABLE products\G  -- \G = vertical output, easier to read
```

---

## ⚠️ GOTCHAS — DDL Traps That Cost Hours

**1. DDL cannot be rolled back**
```sql
START TRANSACTION;
DROP TABLE orders;    -- auto-committed immediately
ROLLBACK;             -- does NOTHING — orders is already gone
-- Always back up before destructive DDL operations.
```

**2. DROP TABLE vs TRUNCATE vs DELETE — know the difference**
```sql
DELETE FROM orders;
-- ✅ Can be rolled back (DML)
-- ✅ Fires DELETE triggers
-- ✅ Respects WHERE clause
-- ❌ Slow on large tables (row by row)
-- ❌ Does NOT reset AUTO_INCREMENT

TRUNCATE TABLE orders;
-- ❌ Cannot be rolled back (DDL)
-- ❌ Does NOT fire triggers
-- ❌ Cannot use WHERE (all or nothing)
-- ✅ Instant even on huge tables
-- ✅ Resets AUTO_INCREMENT to 1
-- ❌ Fails if foreign keys reference this table

DROP TABLE orders;
-- Removes structure AND data
-- Use only when you want the table gone entirely
```

**3. MODIFY vs CHANGE — easy to mix up**
```sql
-- MODIFY: change datatype/constraints, keep same name
ALTER TABLE products MODIFY COLUMN name VARCHAR(200) NOT NULL;

-- CHANGE: rename AND/OR change datatype (must repeat full definition)
ALTER TABLE products CHANGE COLUMN name product_name VARCHAR(200) NOT NULL;
--                                   ^^^^  ^^^^^^^^^^^
--                                old name  new name
-- If you forget the new definition, MySQL errors
```

**4. Foreign key drop order**
```sql
-- WRONG — trying to drop parent before child:
DROP TABLE customers;  -- ❌ ERROR: referenced by orders

-- CORRECT — children first, then parents:
DROP TABLE order_items;  -- remove junction table first
DROP TABLE orders;       -- remove child of customers
DROP TABLE customers;    -- now safe to drop parent ✅
```

**5. ALTER TABLE on large tables locks the table**
```sql
-- On a table with 50 million rows:
ALTER TABLE orders ADD COLUMN notes TEXT NULL;
-- In older MySQL: locks the table for minutes/hours
-- In MySQL 8 with InnoDB: most ALTERs are instant (online DDL)
-- But some changes (changing column type) still require table rebuild
-- Always test ALTER TABLE on a copy first in production scenarios
```

**6. `CREATE TABLE AS SELECT` doesn't copy constraints**
```sql
CREATE TABLE orders_backup AS SELECT * FROM orders;
-- ✅ Copies all data
-- ❌ No PRIMARY KEY
-- ❌ No FOREIGN KEYs
-- ❌ No indexes
-- ❌ No NOT NULL constraints
-- Always add constraints manually after, or use mysqldump for real backups
```

---

✅ **Quick Check:**
ShopNest's team runs the following on their production database:

```sql
START TRANSACTION;

ALTER TABLE products
    DROP COLUMN description;

ROLLBACK;
```

They expect the `description` column to be restored after `ROLLBACK`. Will it be? Explain exactly what happens and why — and what they should have done instead to safely test this change.
