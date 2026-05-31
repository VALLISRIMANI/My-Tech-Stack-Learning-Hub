📍 **Module 3 → Concept 5 of 5: Complete DDL Wrap-up — ENUM Deep Dive, Schema Review & Module Mini Project**

---

## 📖 THEORY — Tying DDL Together

We've covered data types, database commands, table commands, and constraints. This final concept in Module 3 does three things:

1. Deep dives into **ENUM** — the most misunderstood data type in MySQL
2. Reviews the **complete ShopNest schema** as a unified whole
3. Introduces two important DDL patterns you'll use constantly in production

---

## 🔬 ENUM — The Complete Picture

### What ENUM really is under the hood

ENUM looks like a string type but behaves very differently internally. MySQL stores ENUM values as **integers** — 1, 2, 3... mapped to the list positions you define. The string labels are just a display layer.

```sql
status ENUM('pending', 'processing', 'shipped', 'delivered', 'cancelled')

-- Internal storage:
-- 'pending'    → stored as 1  (1 byte)
-- 'processing' → stored as 2  (1 byte)
-- 'shipped'    → stored as 3  (1 byte)
-- 'delivered'  → stored as 4  (1 byte)
-- 'cancelled'  → stored as 5  (1 byte)
-- NULL         → stored as 0  (if column is nullable)

-- vs VARCHAR('pending') → stored as 7 bytes of actual text
-- ENUM wins on storage AND comparison speed
```

---

### ENUM Sorting Behavior — The Surprise

Because ENUM is stored as integers, **ORDER BY sorts by definition order, not alphabetically**:

```sql
-- ENUM defined as: ('pending','processing','shipped','delivered','cancelled')

SELECT status FROM orders ORDER BY status;

-- Result (sorted by integer position, NOT alphabetically):
-- 'pending'     ← position 1
-- 'processing'  ← position 2
-- 'shipped'     ← position 3
-- 'delivered'   ← position 4
-- 'cancelled'   ← position 5

-- This is actually USEFUL — you defined the logical order,
-- and sorting follows that order naturally.

-- To sort alphabetically instead:
SELECT status FROM orders ORDER BY CAST(status AS CHAR);
```

---

### Adding ENUM Values — The Safe Way

```sql
-- Original ENUM:
status ENUM('pending','processing','shipped','delivered','cancelled')

-- Adding 'returned' — TWO ways, very different performance:

-- ❌ SLOW WAY (rebuilds entire table on older MySQL):
ALTER TABLE orders
    MODIFY COLUMN status
    ENUM('pending','processing','shipped','delivered','cancelled','returned')
    NOT NULL DEFAULT 'pending';

-- ✅ FAST WAY in MySQL 8 (instant DDL — adds to end):
-- Adding to the END of the ENUM list is instant (no table rebuild)
-- Adding in the MIDDLE requires full table rebuild (reorders integers)
-- Always append to end of ENUM list for zero-downtime changes
```

---

### When to use ENUM vs a Lookup Table

```sql
-- Use ENUM when:
--   ✅ Values are few (< 20), well-known, rarely change
--   ✅ Values are intrinsic to the column's meaning
--   ✅ You want automatic validation without a JOIN

status ENUM('pending','processing','shipped','delivered','cancelled')

-- Use a lookup/reference TABLE when:
--   ✅ Values change frequently (products need new categories often)
--   ✅ Values have their own attributes (a category has name + description + image)
--   ✅ Values are shared across multiple tables
--   ✅ Business users need to manage values without a developer

-- WRONG — using ENUM for categories:
category ENUM('Electronics','Clothing','Home','Sports','Books'...)
-- Adding a new category needs a developer + ALTER TABLE + deployment
-- A categories table lets the business team add via admin UI ✅
```

---

## 🏗️ The Complete ShopNest DDL Schema

Here is the final, production-grade DDL for everything we've designed — ready to run as a complete script:

```sql
-- ================================================
-- ShopNest E-Commerce Database
-- Complete DDL Script — Module 3 Final Version
-- ================================================

DROP DATABASE IF EXISTS shopnest;
CREATE DATABASE IF NOT EXISTS shopnest
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

USE shopnest;

-- ─────────────────────────────────────────────
-- TABLE 1: categories
-- Parent table — no foreign keys
-- ─────────────────────────────────────────────
CREATE TABLE categories (
    category_id   INT UNSIGNED      NOT NULL AUTO_INCREMENT,
    name          VARCHAR(50)       NOT NULL,
    description   TEXT              NULL,
    is_active     BOOLEAN           NOT NULL DEFAULT TRUE,
    created_at    TIMESTAMP         NOT NULL DEFAULT CURRENT_TIMESTAMP,

    PRIMARY KEY (category_id),
    CONSTRAINT uq_category_name UNIQUE (name)
) ENGINE = InnoDB DEFAULT CHARSET = utf8mb4 COLLATE = utf8mb4_unicode_ci;

-- ─────────────────────────────────────────────
-- TABLE 2: customers
-- Parent table — no foreign keys
-- ─────────────────────────────────────────────
CREATE TABLE customers (
    customer_id   INT UNSIGNED      NOT NULL AUTO_INCREMENT,
    first_name    VARCHAR(50)       NOT NULL,
    last_name     VARCHAR(50)       NOT NULL,
    email         VARCHAR(100)      NOT NULL,
    phone         VARCHAR(15)       NULL,
    city          VARCHAR(50)       NOT NULL,
    state         VARCHAR(50)       NOT NULL,
    loyalty_tier  ENUM('bronze','silver','gold','platinum')
                                    NOT NULL DEFAULT 'bronze',
    is_active     BOOLEAN           NOT NULL DEFAULT TRUE,
    created_at    TIMESTAMP         NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at    TIMESTAMP         NOT NULL DEFAULT CURRENT_TIMESTAMP
                                             ON UPDATE CURRENT_TIMESTAMP,

    PRIMARY KEY (customer_id),
    CONSTRAINT uq_customer_email UNIQUE (email),
    CONSTRAINT chk_customer_email CHECK (email LIKE '%@%.%')
) ENGINE = InnoDB DEFAULT CHARSET = utf8mb4 COLLATE = utf8mb4_unicode_ci;

-- ─────────────────────────────────────────────
-- TABLE 3: products
-- Child of categories
-- ─────────────────────────────────────────────
CREATE TABLE products (
    product_id    INT UNSIGNED      NOT NULL AUTO_INCREMENT,
    category_id   INT UNSIGNED      NOT NULL,
    name          VARCHAR(100)      NOT NULL,
    description   TEXT              NULL,
    price         DECIMAL(10,2)     NOT NULL,
    stock         INT UNSIGNED      NOT NULL DEFAULT 0,
    sku           VARCHAR(50)       NULL,
    is_active     BOOLEAN           NOT NULL DEFAULT TRUE,
    created_at    TIMESTAMP         NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at    TIMESTAMP         NOT NULL DEFAULT CURRENT_TIMESTAMP
                                             ON UPDATE CURRENT_TIMESTAMP,

    PRIMARY KEY (product_id),
    CONSTRAINT uq_product_sku    UNIQUE (sku),
    CONSTRAINT fk_products_category
        FOREIGN KEY (category_id) REFERENCES categories(category_id)
        ON DELETE RESTRICT
        ON UPDATE CASCADE,
    CONSTRAINT chk_product_price CHECK (price > 0)
) ENGINE = InnoDB DEFAULT CHARSET = utf8mb4 COLLATE = utf8mb4_unicode_ci;

-- ─────────────────────────────────────────────
-- TABLE 4: orders
-- Child of customers
-- ─────────────────────────────────────────────
CREATE TABLE orders (
    order_id      INT UNSIGNED      NOT NULL AUTO_INCREMENT,
    customer_id   INT UNSIGNED      NOT NULL,
    order_date    DATETIME          NOT NULL DEFAULT CURRENT_TIMESTAMP,
    status        ENUM('pending','processing','shipped',
                       'delivered','cancelled')
                                    NOT NULL DEFAULT 'pending',
    total_amount  DECIMAL(10,2)     NOT NULL DEFAULT 0.00,
    notes         VARCHAR(500)      NULL,

    PRIMARY KEY (order_id),
    CONSTRAINT fk_orders_customer
        FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
        ON DELETE RESTRICT
        ON UPDATE CASCADE,
    CONSTRAINT chk_order_total CHECK (total_amount >= 0)
) ENGINE = InnoDB DEFAULT CHARSET = utf8mb4 COLLATE = utf8mb4_unicode_ci;

-- ─────────────────────────────────────────────
-- TABLE 5: order_items
-- Child of orders AND products (junction table)
-- ─────────────────────────────────────────────
CREATE TABLE order_items (
    order_item_id INT UNSIGNED      NOT NULL AUTO_INCREMENT,
    order_id      INT UNSIGNED      NOT NULL,
    product_id    INT UNSIGNED      NOT NULL,
    quantity      INT UNSIGNED      NOT NULL,
    unit_price    DECIMAL(10,2)     NOT NULL,

    PRIMARY KEY (order_item_id),
    CONSTRAINT uq_order_product
        UNIQUE (order_id, product_id),
    CONSTRAINT fk_items_order
        FOREIGN KEY (order_id) REFERENCES orders(order_id)
        ON DELETE CASCADE
        ON UPDATE CASCADE,
    CONSTRAINT fk_items_product
        FOREIGN KEY (product_id) REFERENCES products(product_id)
        ON DELETE RESTRICT
        ON UPDATE CASCADE,
    CONSTRAINT chk_item_quantity  CHECK (quantity > 0),
    CONSTRAINT chk_item_price     CHECK (unit_price > 0)
) ENGINE = InnoDB DEFAULT CHARSET = utf8mb4 COLLATE = utf8mb4_unicode_ci;

-- ─────────────────────────────────────────────
-- TABLE 6: reviews
-- Child of customers AND products
-- ─────────────────────────────────────────────
CREATE TABLE reviews (
    review_id     INT UNSIGNED      NOT NULL AUTO_INCREMENT,
    customer_id   INT UNSIGNED      NOT NULL,
    product_id    INT UNSIGNED      NOT NULL,
    rating        TINYINT UNSIGNED  NOT NULL,
    title         VARCHAR(100)      NULL,
    body          TEXT              NULL,
    created_at    TIMESTAMP         NOT NULL DEFAULT CURRENT_TIMESTAMP,

    PRIMARY KEY (review_id),
    CONSTRAINT uq_one_review
        UNIQUE (customer_id, product_id),
    CONSTRAINT fk_reviews_customer
        FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
        ON DELETE CASCADE
        ON UPDATE CASCADE,
    CONSTRAINT fk_reviews_product
        FOREIGN KEY (product_id) REFERENCES products(product_id)
        ON DELETE CASCADE
        ON UPDATE CASCADE,
    CONSTRAINT chk_rating_range
        CHECK (rating BETWEEN 1 AND 5)
) ENGINE = InnoDB DEFAULT CHARSET = utf8mb4 COLLATE = utf8mb4_unicode_ci;
```

---

## 🔧 Two Important Production DDL Patterns

### Pattern 1: Safe Schema Migrations

In production you never just run raw ALTER TABLE blindly. The professional pattern:

```sql
-- Step 1: Verify current state before changing anything
DESC products;
SHOW CREATE TABLE products\G

-- Step 2: Test on staging first
USE shopnest_staging;
ALTER TABLE products ADD COLUMN weight_kg DECIMAL(8,3) NULL;
-- verify it works, check data integrity

-- Step 3: Apply to production with IF NOT EXISTS guard
USE shopnest_prod;
-- Check if column already exists before adding
-- (prevents error if migration runs twice)
SELECT COUNT(*) FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = 'shopnest_prod'
  AND TABLE_NAME   = 'products'
  AND COLUMN_NAME  = 'weight_kg';
-- If returns 0, safe to add. If returns 1, already exists.

-- Step 4: Apply
ALTER TABLE products ADD COLUMN weight_kg DECIMAL(8,3) NULL;

-- Step 5: Verify
DESC products;
```

---

### Pattern 2: Soft Deletes vs Hard Deletes

In e-commerce, you rarely truly delete data. Instead you **soft delete** — mark as inactive, preserve history:

```sql
-- Hard delete (permanent — avoid for important records):
DELETE FROM products WHERE product_id = 101;
-- Gone forever. Order history that referenced this product breaks.

-- Soft delete pattern (professional approach):
-- Products table already has: is_active BOOLEAN NOT NULL DEFAULT TRUE

UPDATE products SET is_active = FALSE WHERE product_id = 101;
-- Product is "deleted" from the user's perspective
-- But order_items history still references it correctly ✅
-- You can restore it: UPDATE products SET is_active = TRUE ...
-- You can audit: SELECT * FROM products WHERE is_active = FALSE

-- Queries then filter active records:
SELECT * FROM products WHERE is_active = TRUE;
```

---

## ⚠️ Final DDL Gotchas — The Ones That Bite in Production

**1. ENUM vs application-side validation**
```sql
-- Don't rely ONLY on ENUM for validation in your app code.
-- ENUM enforces at DB level — good.
-- But your API should also validate before hitting the DB
-- so you can return meaningful error messages to users,
-- not raw MySQL errors.
```

**2. AUTO_INCREMENT gaps are normal and expected**
```sql
-- After failed inserts or rollbacks, AUTO_INCREMENT still advances:
INSERT INTO products ... ;  -- product_id = 1 (success)
INSERT INTO products ... ;  -- product_id = 2 (fails, rolls back)
INSERT INTO products ... ;  -- product_id = 3 (success)
-- Gap at 2 is normal. Never rely on AUTO_INCREMENT being gapless.
-- Never use it to count records. Use COUNT(*) for that.
```

**3. Schema changes need to respect FK order**
```sql
-- You cannot ADD a foreign key if the data already violates it:
-- If orders has customer_id = 999 but customers has no id 999,
-- adding the FK constraint will fail:
ALTER TABLE orders ADD CONSTRAINT fk_orders_customer
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id);
-- ERROR: Cannot add foreign key constraint
-- Fix: clean the orphaned data first, then add constraint
```

**4. SHOW CREATE TABLE is your best friend**
```sql
-- Always use this to see the REAL current state of a table:
SHOW CREATE TABLE orders\G
-- Shows exact data types, constraints, indexes, engine, charset
-- Far more complete than DESC which omits constraints and indexes
```

✅ **Module 3 Complete!** 