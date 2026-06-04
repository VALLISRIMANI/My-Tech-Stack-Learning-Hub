📍 **Module 4 → Concept 3 of 6: INSERT IGNORE & ON DUPLICATE KEY UPDATE**

---

## 📖 THEORY — What is it and why does it exist?

### The Problem: What happens when you INSERT a row that already exists?

Normal INSERT is all-or-nothing — if a row violates a UNIQUE or PRIMARY KEY constraint, the entire statement fails with an error:

```sql
INSERT INTO categories (name) VALUES ('Electronics');
-- First time → success ✅

INSERT INTO categories (name) VALUES ('Electronics');
-- Second time → ERROR 1062: Duplicate entry 'Electronics'
--               for key 'uq_category_name' ❌
```

This is correct behavior for most cases. But real applications hit three common scenarios where you need smarter behavior:

```
Scenario 1 — Idempotent seeding:
  Your app runs a "seed initial data" script on every deploy.
  You don't want it to fail if categories already exist.
  → INSERT IGNORE

Scenario 2 — Upsert (Update or Insert):
  ShopNest syncs product inventory from a supplier feed daily.
  If the product exists → update its stock and price.
  If it doesn't exist → insert it fresh.
  → ON DUPLICATE KEY UPDATE

Scenario 3 — Silent skip on conflict:
  Bulk importing 10,000 customer records from a CSV.
  A few emails already exist in the database.
  Skip duplicates silently, process the rest.
  → INSERT IGNORE
```

MySQL gives you two tools to handle these exactly.

---

## 🔤 SYNTAX — INSERT IGNORE

```sql
INSERT IGNORE INTO table_name (col1, col2, ...)
VALUES (val1, val2, ...);
```

`INSERT IGNORE` tells MySQL: if this INSERT would violate a UNIQUE or PRIMARY KEY constraint, **silently skip this row** and continue. No error. No rollback.

```sql
-- Without IGNORE — fails on duplicate:
INSERT INTO categories (name, description)
VALUES ('Electronics', 'Phones and gadgets');
-- ERROR 1062 if 'Electronics' already exists ❌

-- With IGNORE — silently skips duplicate:
INSERT IGNORE INTO categories (name, description)
VALUES ('Electronics', 'Phones and gadgets');
-- Query OK, 0 rows affected (if duplicate)
-- Query OK, 1 row affected  (if new)
-- No error either way ✅

-- Powerful with multi-row INSERT:
INSERT IGNORE INTO categories (name, description)
VALUES
    ('Electronics',    'Phones and gadgets'),   -- exists → skipped
    ('Clothing',       'Apparel and fashion'),   -- exists → skipped
    ('Toys',           'Kids toys and games'),   -- new    → inserted ✅
    ('Automotive',     'Car accessories');       -- new    → inserted ✅
-- Query OK, 2 rows affected (only the 2 new ones inserted)
```

---

### What INSERT IGNORE actually ignores

```sql
-- IGNORE suppresses these errors:
--  ✅ Duplicate PRIMARY KEY
--  ✅ Duplicate UNIQUE key
--  ✅ Row too long warnings

-- IGNORE does NOT suppress:
--  ❌ Foreign key violations (still errors)
--  ❌ Data type mismatches (still errors)
--  ❌ CHECK constraint violations (still errors in MySQL 8)

-- Example — FK violation still fails even with IGNORE:
INSERT IGNORE INTO products (category_id, name, price, stock)
VALUES (9999, 'Test Product', 999.00, 10);
-- ERROR: Cannot add or update a child row:
-- foreign key constraint fails ❌ (IGNORE didn't help here)
```

---

## 🔤 SYNTAX — ON DUPLICATE KEY UPDATE

```sql
INSERT INTO table_name (col1, col2, col3, ...)
VALUES (val1, val2, val3, ...)
ON DUPLICATE KEY UPDATE
    col2 = new_value2,
    col3 = new_value3;
```

This is MySQL's **UPSERT** — if the row is new, INSERT it. If it already exists (PK or UNIQUE violation), UPDATE the specified columns instead.

```sql
-- Basic upsert — insert product or update price/stock if SKU exists:
INSERT INTO products (category_id, name, price, stock, sku)
VALUES (1, 'iPhone 15', 82999.00, 75, 'APPL-IPH15-001')
ON DUPLICATE KEY UPDATE
    price = 82999.00,
    stock = 75;

-- If 'APPL-IPH15-001' doesn't exist → full INSERT
-- If 'APPL-IPH15-001' already exists → only price and stock are updated
-- category_id, name stay unchanged ✅
```

---

### The VALUES() function — reference the attempted INSERT value

```sql
-- VALUES(column_name) refers to the value you tried to INSERT:
INSERT INTO products (category_id, name, price, stock, sku)
VALUES (1, 'iPhone 15', 82999.00, 75, 'APPL-IPH15-001')
ON DUPLICATE KEY UPDATE
    price = VALUES(price),    -- use the new price from INSERT attempt
    stock = VALUES(stock);    -- use the new stock from INSERT attempt

-- In MySQL 8.0.20+, VALUES() is deprecated.
-- Use aliases instead (cleaner, recommended):
INSERT INTO products (category_id, name, price, stock, sku)
VALUES (1, 'iPhone 15', 82999.00, 75, 'APPL-IPH15-001') AS new_vals
ON DUPLICATE KEY UPDATE
    price = new_vals.price,
    stock = new_vals.stock;
```

---

### Incrementing on duplicate — the counter pattern

```sql
-- Track how many times each product is viewed:
-- (hypothetical product_views table)
CREATE TABLE product_views (
    product_id   INT UNSIGNED NOT NULL,
    view_count   INT UNSIGNED NOT NULL DEFAULT 1,
    last_viewed  TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP
                                       ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (product_id)
);

-- First view of product 1 → inserts with count=1
-- Subsequent views → increments count
INSERT INTO product_views (product_id, view_count)
VALUES (1, 1)
ON DUPLICATE KEY UPDATE
    view_count  = view_count + 1;
    -- ↑ References the EXISTING row's value, not the INSERT value
    -- This is the increment pattern — extremely common in analytics
```

---

### Rows affected return values — understanding the signals

```sql
-- ON DUPLICATE KEY UPDATE returns different "rows affected" counts:
-- 0 = row existed, UPDATE was run but no values actually changed
-- 1 = new row was INSERTed
-- 2 = row existed and was successfully UPDATEd
-- (MySQL counts the duplicate match as 1 + the update as 1 = 2)

-- This lets your application code know exactly what happened:
-- affected_rows == 1 → fresh insert
-- affected_rows == 2 → existing row updated
-- affected_rows == 0 → existed but nothing changed
```

---

## 💡 EXAMPLE — ShopNest Real-World Upsert Scenarios

```sql
-- ─────────────────────────────────────────────────
-- Scenario 1: Daily supplier inventory sync
-- Supplier sends updated stock + price for all products
-- If product exists → update. If new → insert.
-- ─────────────────────────────────────────────────
INSERT INTO products (category_id, name, price, stock, sku)
VALUES
    (1, 'iPhone 15',          82999.00,  60, 'APPL-IPH15-001'),
    (1, 'Samsung Galaxy S24', 67999.00,  40, 'SAMS-S24-001'),
    (1, 'iPhone 15 Pro',      129999.00, 25, 'APPL-IPH15P-001')
AS supplier_feed
ON DUPLICATE KEY UPDATE
    price = supplier_feed.price,
    stock = supplier_feed.stock;

-- iPhone 15 (SKU exists)        → price and stock updated ✅
-- Samsung S24 (SKU exists)      → price and stock updated ✅
-- iPhone 15 Pro (SKU is new)    → full row inserted ✅
-- category_id, name untouched for existing products ✅

-- ─────────────────────────────────────────────────
-- Scenario 2: Idempotent database seeding script
-- Runs on every deployment — safe to run multiple times
-- ─────────────────────────────────────────────────
INSERT IGNORE INTO categories (name, description)
VALUES
    ('Electronics',    'Phones, laptops, audio and accessories'),
    ('Clothing',       'Men and women apparel and fashion'),
    ('Home & Kitchen', 'Cookware, furniture and appliances'),
    ('Sports',         'Fitness equipment and sportswear'),
    ('Books',          'Fiction, non-fiction and educational');
-- First deployment  → all 5 inserted ✅
-- Second deployment → all 5 skipped silently ✅
-- No errors, no duplicates, same result every time ✅

-- ─────────────────────────────────────────────────
-- Scenario 3: Customer loyalty points accumulation
-- Award 50 points per delivered order
-- If customer already in loyalty table → add points
-- If not → create their record
-- ─────────────────────────────────────────────────
CREATE TABLE loyalty_points (
    customer_id  INT UNSIGNED  NOT NULL,
    points       INT UNSIGNED  NOT NULL DEFAULT 0,
    updated_at   TIMESTAMP     NOT NULL DEFAULT CURRENT_TIMESTAMP
                                        ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (customer_id),
    CONSTRAINT fk_loyalty_customer
        FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
        ON DELETE CASCADE ON UPDATE CASCADE
);

-- Award points to Rahul (customer_id = 1):
INSERT INTO loyalty_points (customer_id, points)
VALUES (1, 50)
ON DUPLICATE KEY UPDATE
    points = points + 50;
-- First delivered order  → inserts row with points = 50
-- Second delivered order → updates to points = 100
-- Third delivered order  → updates to points = 150 ✅

-- ─────────────────────────────────────────────────
-- Scenario 4: Bulk customer import with IGNORE
-- Importing 5 customers from a CSV
-- 2 emails already exist — skip them, import the rest
-- ─────────────────────────────────────────────────
INSERT IGNORE INTO customers
    (first_name, last_name, email, city, state)
VALUES
    ('Rahul',  'Sharma', 'rahul.sharma@gmail.com', 'Hyderabad', 'Telangana'),
    ('Priya',  'Nair',   'priya.nair@gmail.com',   'Chennai',   'Tamil Nadu'),
    ('Kiran',  'Rao',    'kiran.rao@gmail.com',     'Mysore',    'Karnataka'),
    ('Meera',  'Das',    'meera.das@gmail.com',     'Kolkata',   'West Bengal'),
    ('Suresh', 'Babu',   'suresh.babu@gmail.com',   'Vizag',     'Andhra Pradesh');
-- Rahul and Priya already exist (duplicate email) → silently skipped
-- Kiran, Meera, Suresh are new                    → inserted ✅
-- Query OK, 3 rows affected ✅
```

---

## ⚠️ GOTCHAS — Common Mistakes

**1. INSERT IGNORE silently hides real problems**
```sql
INSERT IGNORE INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (9999, 1, 2, 79999.00);
-- order_id 9999 doesn't exist → FK violation
-- But IGNORE doesn't suppress FK errors → still errors
-- However, if you had a duplicate PK issue AND a FK issue,
-- IGNORE would hide the duplicate but expose the FK error
-- Making debugging confusing. Use IGNORE surgically.
```

**2. ON DUPLICATE KEY UPDATE accidentally updates the wrong columns**
```sql
-- DANGEROUS — updating category_id on duplicate:
INSERT INTO products (category_id, name, price, stock, sku)
VALUES (3, 'iPhone 15', 82999.00, 75, 'APPL-IPH15-001')
ON DUPLICATE KEY UPDATE
    category_id = VALUES(category_id),  -- ❌ moves iPhone to Home & Kitchen!
    price       = VALUES(price),
    stock       = VALUES(stock);
-- Only update columns that should legitimately change (price, stock)
-- Never update FK columns or business identifiers on duplicate
```

**3. AUTO_INCREMENT jumps on ON DUPLICATE KEY UPDATE**
```sql
-- Even when the UPDATE branch runs (no INSERT),
-- AUTO_INCREMENT counter still advances by 1 in some MySQL versions.
-- After 1000 upserts where 990 were updates, your next
-- real INSERT might get product_id = 1001 instead of 11.
-- Again — gaps in AUTO_INCREMENT are normal. Don't rely on gapless IDs.
```

**4. Confusing INSERT IGNORE with ON DUPLICATE KEY UPDATE**
```sql
-- INSERT IGNORE: skip the row entirely on duplicate
--   → existing row is completely UNCHANGED
--   → useful when you just want new rows added

-- ON DUPLICATE KEY UPDATE: modify existing row on duplicate
--   → existing row IS changed
--   → useful for syncing/updating existing data

-- Wrong tool for the job:
INSERT IGNORE INTO products ... -- wants to update price on duplicate
-- ❌ Price won't update — the row is just skipped

-- Right tool:
INSERT INTO products ... ON DUPLICATE KEY UPDATE price = new_price
-- ✅ Price updates on duplicate ✅
```

**5. Multiple UNIQUE keys and which one triggers the duplicate**
```sql
-- products has UNIQUE on both product_id (PK) and sku:
INSERT INTO products (product_id, category_id, name, price, stock, sku)
VALUES (1, 1, 'Different Phone', 50000.00, 20, 'APPL-IPH15-001')
ON DUPLICATE KEY UPDATE price = 50000.00;

-- Which duplicate triggered the update?
-- product_id = 1 exists? OR sku 'APPL-IPH15-001' exists?
-- Could be either — MySQL fires on WHICHEVER unique key
-- matches first. With multiple unique keys this gets ambiguous.
-- Prefer tables with a single natural unique key for upserts.
```

---

✅ **Quick Check:**
ShopNest runs a nightly job that awards 100 loyalty points to every customer who had an order delivered that day. The job inserts into the `loyalty_points` table.

A customer can receive points multiple times (once per delivered order). Write the correct SQL statement that:
- Creates their record with 100 points if they don't exist yet
- Adds 100 to their existing points if they do exist

Use customer_id = 2 as your example, and explain why `INSERT IGNORE` would be the **wrong** tool for this job.
