📍 **Module 4 → Concept 2 of 6: UPDATE & DELETE**

---

## 📖 THEORY — What is it and why does it exist?

INSERT puts data in. But data changes — customers move cities, orders get cancelled, prices get updated, products go out of stock. And sometimes data needs to be removed entirely. That's `UPDATE` and `DELETE`.

These two commands are the most **dangerous** in everyday SQL — not because they're complex, but because a missing or wrong `WHERE` clause silently affects every single row in the table. This has caused real production disasters at real companies.

The discipline of writing safe UPDATE and DELETE is a professional skill as important as knowing the syntax itself.

---

## 🔤 SYNTAX — UPDATE

### Basic UPDATE

```sql
UPDATE table_name
SET    column1 = value1,
       column2 = value2,
       ...
WHERE  condition;
```

```sql
-- Update a single customer's city:
UPDATE customers
SET    city  = 'Bangalore',
       state = 'Karnataka'
WHERE  customer_id = 3;

-- Update a product's price:
UPDATE products
SET    price = 72999.00
WHERE  product_id = 1;

-- Update multiple columns at once:
UPDATE customers
SET    city         = 'Pune',
       state        = 'Maharashtra',
       loyalty_tier = 'gold'
WHERE  customer_id = 5;
```

---

### UPDATE with Expressions

```sql
-- Increase all electronics prices by 5%:
UPDATE products
SET    price = price * 1.05
WHERE  category_id = 1;

-- Decrease stock when an order is placed:
UPDATE products
SET    stock = stock - 1
WHERE  product_id = 1;

-- Add loyalty points (if we had that column):
UPDATE customers
SET    loyalty_points = loyalty_points + 100
WHERE  customer_id = 1;

-- Apply 10% discount to all pending orders over ₹50,000:
UPDATE orders
SET    total_amount = total_amount * 0.90
WHERE  status = 'pending'
  AND  total_amount > 50000;
```

---

### UPDATE with ORDER BY and LIMIT

```sql
-- Update only the 3 cheapest products — give them a price bump:
UPDATE products
SET    price = price * 1.10
ORDER BY price ASC
LIMIT  3;

-- Useful for: batch processing large tables in chunks
-- instead of one massive UPDATE that locks everything
```

---

### UPDATE across multiple conditions

```sql
-- Upgrade customers to 'silver' if they're bronze
-- AND have placed at least one delivered order
-- (subquery — we'll cover these fully in Module 6,
--  but the UPDATE syntax is what matters here):
UPDATE customers
SET    loyalty_tier = 'silver'
WHERE  loyalty_tier = 'bronze'
  AND  customer_id IN (
           SELECT DISTINCT customer_id
           FROM   orders
           WHERE  status = 'delivered'
       );
```

---

## 🔤 SYNTAX — DELETE

### Basic DELETE

```sql
DELETE FROM table_name
WHERE  condition;
```

```sql
-- Delete a specific order:
DELETE FROM orders
WHERE  order_id = 8;
-- ✅ Because order_items has ON DELETE CASCADE for orders FK,
--    all order_items for order_id = 8 are automatically deleted too

-- Delete all cancelled orders:
DELETE FROM orders
WHERE  status = 'cancelled';

-- Delete a specific customer (will fail if they have orders):
DELETE FROM customers
WHERE  customer_id = 6;
-- ✅ customer 6 (Ananya) has no orders → deletes successfully
-- ❌ customer 1 (Rahul) has orders → ERROR: FK RESTRICT blocks it
```

---

### DELETE with ORDER BY and LIMIT

```sql
-- Delete oldest 5 cancelled orders (cleanup job):
DELETE FROM orders
WHERE  status = 'cancelled'
ORDER BY order_date ASC
LIMIT  5;

-- This is the safe batch-delete pattern for large tables
-- Never run DELETE without LIMIT on multi-million row tables
-- in production without careful thought
```

---

### DELETE vs TRUNCATE — Reminder

```sql
-- DELETE all rows (slow, logged, reversible within transaction):
DELETE FROM reviews;

-- TRUNCATE all rows (fast, not logged per-row, NOT reversible):
TRUNCATE TABLE reviews;

-- DELETE fires triggers. TRUNCATE does not.
-- DELETE respects FK constraints. TRUNCATE does not (on InnoDB).
-- DELETE can use WHERE. TRUNCATE cannot.
```

---

## 🛡️ THE GOLDEN RULE — Safe UPDATE/DELETE Pattern

This is the single most important habit to build. **Always follow these three steps:**

```sql
-- ─────────────────────────────────────────────────
-- STEP 1: Write the WHERE clause as a SELECT first
--         Visually confirm EXACTLY which rows will change
-- ─────────────────────────────────────────────────
SELECT *
FROM   orders
WHERE  status = 'cancelled';
-- Output: shows order_id = 8 (Kavya's cancelled order)
-- Confirm: yes, this is what I want to affect

-- ─────────────────────────────────────────────────
-- STEP 2: Wrap in a transaction for safety
-- ─────────────────────────────────────────────────
START TRANSACTION;

DELETE FROM orders
WHERE  status = 'cancelled';

-- ─────────────────────────────────────────────────
-- STEP 3: Verify the result BEFORE committing
-- ─────────────────────────────────────────────────
SELECT COUNT(*) FROM orders WHERE status = 'cancelled';
-- Should return 0

-- If everything looks right:
COMMIT;

-- If something went wrong:
ROLLBACK;  -- instantly undoes the DELETE
```

This pattern costs 30 extra seconds and has saved careers.

---

## 💡 EXAMPLE — ShopNest Real-World UPDATE/DELETE Scenarios

```sql
-- ─────────────────────────────────────────────────
-- Scenario 1: Customer updates their profile
-- Rahul moves from Hyderabad to Bangalore
-- ─────────────────────────────────────────────────

-- Step 1: Confirm the target row
SELECT customer_id, first_name, city, state
FROM   customers
WHERE  customer_id = 1;
-- Returns: 1, Rahul, Hyderabad, Telangana ✅

-- Step 2: Safe update
START TRANSACTION;

UPDATE customers
SET    city  = 'Bangalore',
       state = 'Karnataka'
WHERE  customer_id = 1;

-- Step 3: Verify
SELECT customer_id, first_name, city, state
FROM   customers
WHERE  customer_id = 1;
-- Returns: 1, Rahul, Bangalore, Karnataka ✅

COMMIT;

-- ─────────────────────────────────────────────────
-- Scenario 2: iPhone 15 restocked — update stock
-- ─────────────────────────────────────────────────
UPDATE products
SET    stock = stock + 100
WHERE  sku = 'APPL-IPH15-001';
-- Using SKU (business identifier) instead of product_id
-- is safer — SKUs are stable business codes, IDs are
-- internal and can vary between environments

-- ─────────────────────────────────────────────────
-- Scenario 3: Order status pipeline
-- Processing → Shipped → Delivered
-- ─────────────────────────────────────────────────
UPDATE orders
SET    status = 'shipped'
WHERE  order_id = 3
  AND  status = 'processing';
-- The AND status = 'processing' guard is critical:
-- prevents accidentally shipping an already-delivered order
-- This is called an OPTIMISTIC LOCK pattern

-- ─────────────────────────────────────────────────
-- Scenario 4: Soft delete a product
-- (don't actually delete — just mark inactive)
-- ─────────────────────────────────────────────────
UPDATE products
SET    is_active = FALSE
WHERE  product_id = 6;
-- Men Formal Shirt discontinued
-- Historical order_items still reference it correctly ✅
-- Can be reactivated anytime ✅

-- ─────────────────────────────────────────────────
-- Scenario 5: Customer requests account deletion
-- GDPR-compliant approach — anonymize, don't delete
-- ─────────────────────────────────────────────────
UPDATE customers
SET    first_name = 'Deleted',
       last_name  = 'User',
       email      = CONCAT('deleted_', customer_id, '@removed.com'),
       phone      = NULL,
       is_active  = FALSE
WHERE  customer_id = 9;
-- Order history preserved for accounting
-- Personal data anonymized for compliance ✅
-- Customer cannot log in (is_active = FALSE) ✅

-- ─────────────────────────────────────────────────
-- Scenario 6: Year-end cleanup — remove old cancelled orders
-- ─────────────────────────────────────────────────
START TRANSACTION;

-- Preview first:
SELECT COUNT(*) AS orders_to_delete
FROM   orders
WHERE  status     = 'cancelled'
  AND  order_date < '2024-01-01';

-- Then delete:
DELETE FROM orders
WHERE  status     = 'cancelled'
  AND  order_date < '2024-01-01';

COMMIT;
```

---

## ⚠️ GOTCHAS — UPDATE and DELETE Disasters

**1. The missing WHERE clause — the most costly mistake in SQL**
```sql
-- Developer intends to update one customer:
UPDATE customers SET loyalty_tier = 'gold';
-- ← forgot WHERE clause
-- Every single customer is now 'gold'. Instantly. No warning.

-- Always ask yourself before running UPDATE/DELETE:
-- "If WHERE was missing, would this destroy data?"
-- If yes → double-check the WHERE clause.
```

**2. MySQL safe update mode**
```sql
-- MySQL Workbench enables safe update mode by default:
-- UPDATE/DELETE without WHERE on a primary key → blocked

-- You'll see: Error Code 1175
-- To disable for current session (use carefully):
SET SQL_SAFE_UPDATES = 0;
UPDATE products SET price = price * 1.05;  -- now works
SET SQL_SAFE_UPDATES = 1;  -- re-enable immediately after
```

**3. UPDATE doesn't tell you if WHERE matched anything**
```sql
UPDATE customers
SET    loyalty_tier = 'platinum'
WHERE  customer_id = 9999;
-- Query OK, 0 rows affected
-- No error — it just silently did nothing
-- Always check "rows affected" count or SELECT first
```

**4. Cascading deletes can surprise you**
```sql
-- order_items has ON DELETE CASCADE for orders:
DELETE FROM orders WHERE order_id = 5;
-- This ALSO deletes order_items rows for order 5
-- Sneha's MacBook + iPhone order items: gone
-- Intended? Make sure you know your cascade rules
-- before running deletes on parent tables
```

**5. Modifying data you're reading (same table)**
```sql
-- This FAILS in MySQL — can't UPDATE a table
-- while SELECTing from it in a subquery:
UPDATE products
SET    price = price * 0.9
WHERE  product_id IN (
    SELECT product_id FROM products  -- same table!
    WHERE  stock > 50
);
-- Error: You can't specify target table 'products'
-- for update in FROM clause

-- Fix — wrap in a derived table:
UPDATE products
SET    price = price * 0.9
WHERE  product_id IN (
    SELECT product_id FROM (
        SELECT product_id FROM products WHERE stock > 50
    ) AS temp
);
```

**6. DELETE with FK RESTRICT silently tells you nothing useful**
```sql
DELETE FROM customers WHERE customer_id = 1;
-- ERROR 1451: Cannot delete or update a parent row:
-- a foreign key constraint fails (`shopnest`.`orders`,
-- CONSTRAINT `fk_orders_customer` FOREIGN KEY (`customer_id`)
-- REFERENCES `customers` (`customer_id`))

-- The error is accurate but you need to know:
-- WHICH child table is blocking you?
-- Check with:
SELECT * FROM orders WHERE customer_id = 1;
-- Shows you exactly what's in the way
```

---

✅ **Quick Check:**
A ShopNest developer needs to cancel all pending orders placed before January 1, 2024 that have a total amount less than ₹1,000. They write this:

```sql
UPDATE orders SET status = 'cancelled'
WHERE total_amount < 1000;
```

Identify **two problems** with this statement — one missing condition and one missing safety practice — then write the corrected, production-safe version using the golden rule pattern.
