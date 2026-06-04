📍 **Module 4 → Concept 1 of 6: INSERT INTO — Adding Data to ShopNest**

---

## 📖 THEORY — What is it and why does it exist?

DDL built the empty rooms. **DML (Data Manipulation Language)** moves the furniture in. `INSERT INTO` is how rows get added to tables — every customer registration, every order placed, every product listed on ShopNest starts with an INSERT.

There are three things to understand about INSERT deeply:

1. How to insert **single vs multiple rows**
2. How to handle **column defaults and NULLs** correctly
3. How INSERT interacts with **constraints** — what happens when rules are violated

---

## 🔤 SYNTAX — Every Form of INSERT

---

### ① Basic Single Row INSERT

```sql
-- Full syntax — specify every column explicitly:
INSERT INTO table_name (col1, col2, col3, ...)
VALUES (val1, val2, val3, ...);

-- Example — inserting a category:
INSERT INTO categories (name, description, is_active)
VALUES ('Electronics', 'Phones, laptops, audio and accessories', TRUE);

-- Columns with DEFAULT or AUTO_INCREMENT can be omitted:
INSERT INTO categories (name)
VALUES ('Clothing');
-- category_id  → AUTO_INCREMENT (MySQL generates it)
-- description  → NULL           (column is nullable, defaults to NULL)
-- is_active    → TRUE           (DEFAULT TRUE)
-- created_at   → current time   (DEFAULT CURRENT_TIMESTAMP)
```

---

### ② Column List — Always Specify It

```sql
-- ❌ INSERT without column list (fragile):
INSERT INTO categories
VALUES (NULL, 'Home & Kitchen', 'Cookware and appliances', TRUE, NOW());
-- Depends on exact column ORDER in table definition
-- If someone adds or reorders a column → silent wrong data or error

-- ✅ INSERT with explicit column list (robust):
INSERT INTO categories (name, description, is_active)
VALUES ('Home & Kitchen', 'Cookware and appliances', TRUE);
-- Order-independent. Self-documenting. Always use this form.
```

---

### ③ Multi-Row INSERT — Insert Many Rows at Once

```sql
-- One INSERT statement, multiple value sets:
INSERT INTO categories (name, description)
VALUES
    ('Electronics',   'Phones, laptops, audio and accessories'),
    ('Clothing',      'Men and women apparel and fashion'),
    ('Home & Kitchen','Cookware, furniture and appliances'),
    ('Sports',        'Fitness equipment and sportswear'),
    ('Books',         'Fiction, non-fiction and educational');

-- Why multi-row is better than 5 separate INSERTs:
-- → One round trip to the server (vs 5)
-- → One transaction (atomic — all succeed or all fail)
-- → Significantly faster for large datasets
-- → MySQL optimizes bulk inserts internally
```

---

### ④ INSERT with Expressions and Functions

```sql
-- You can use functions and expressions in VALUES:
INSERT INTO customers (first_name, last_name, email, phone, city, state)
VALUES (
    'Rahul',
    'Sharma',
    LOWER('Rahul.Sharma@Gmail.Com'),  -- function: normalize email case
    '+91-98765-43210',
    'Hyderabad',
    'Telangana'
);

-- UPPER, LOWER, NOW(), CURDATE() all work inside VALUES:
INSERT INTO orders (customer_id, order_date, status, total_amount)
VALUES (1, NOW(), 'pending', 0.00);
```

---

### ⑤ INSERT ... SELECT — Insert from another table

```sql
-- Insert rows from a query result:
INSERT INTO categories (name, description)
SELECT name, description
FROM categories_staging
WHERE is_approved = TRUE;

-- Useful for: migrations, copying filtered data, seeding from staging
```

---

## 💡 EXAMPLE — Populating ShopNest's Full Dataset

Let's seed all the data we'll use for the rest of the course:

```sql
USE shopnest;

-- ─────────────────────────────────
-- Seed categories
-- ─────────────────────────────────
INSERT INTO categories (name, description) VALUES
    ('Electronics',    'Phones, laptops, audio and accessories'),
    ('Clothing',       'Men and women apparel and fashion'),
    ('Home & Kitchen', 'Cookware, furniture and appliances'),
    ('Sports',         'Fitness equipment and sportswear'),
    ('Books',          'Fiction, non-fiction and educational');

-- Verify:
SELECT * FROM categories;
-- category_id: 1=Electronics, 2=Clothing, 3=Home & Kitchen,
--              4=Sports, 5=Books

-- ─────────────────────────────────
-- Seed customers
-- ─────────────────────────────────
INSERT INTO customers (first_name, last_name, email, phone, city, state, loyalty_tier) VALUES
    ('Rahul',   'Sharma',  'rahul.sharma@gmail.com',  '+91-98765-43210', 'Hyderabad', 'Telangana',   'gold'),
    ('Priya',   'Nair',    'priya.nair@gmail.com',    '+91-91234-56789', 'Chennai',   'Tamil Nadu',  'silver'),
    ('Amit',    'Kumar',   'amit.kumar@yahoo.com',    NULL,              'Mumbai',    'Maharashtra', 'bronze'),
    ('Sneha',   'Reddy',   'sneha.reddy@gmail.com',   '+91-99887-76655', 'Hyderabad', 'Telangana',   'platinum'),
    ('Vikram',  'Singh',   'vikram.singh@outlook.com','+91-88776-65544', 'Delhi',     'Delhi',       'silver'),
    ('Ananya',  'Iyer',    'ananya.iyer@gmail.com',   NULL,              'Bangalore', 'Karnataka',   'bronze'),
    ('Rohan',   'Mehta',   'rohan.mehta@gmail.com',   '+91-77665-54433', 'Pune',      'Maharashtra', 'gold'),
    ('Kavya',   'Pillai',  'kavya.pillai@gmail.com',  '+91-66554-43322', 'Kochi',     'Kerala',      'silver'),
    ('Arjun',   'Patel',   'arjun.patel@yahoo.com',   '+91-55443-32211', 'Ahmedabad', 'Gujarat',     'bronze'),
    ('Divya',   'Menon',   'divya.menon@gmail.com',   '+91-44332-21100', 'Chennai',   'Tamil Nadu',  'gold');

-- ─────────────────────────────────
-- Seed products
-- ─────────────────────────────────
INSERT INTO products (category_id, name, price, stock, sku) VALUES
    (1, 'iPhone 15',           79999.00,  50, 'APPL-IPH15-001'),
    (1, 'Samsung Galaxy S24',  69999.00,  35, 'SAMS-S24-001'),
    (1, 'Sony WH-1000XM5',     24999.00, 100, 'SONY-WH1000-001'),
    (1, 'MacBook Air M2',     114999.00,  20, 'APPL-MBA-M2-001'),
    (1, 'OnePlus 12',          64999.00,  45, 'OP-12-001'),
    (2, 'Men Formal Shirt',      1299.00, 200, 'CLT-MFS-001'),
    (2, 'Women Kurta Set',       1899.00, 150, 'CLT-WKS-001'),
    (2, 'Denim Jeans',           2499.00, 120, 'CLT-DNM-001'),
    (3, 'Instant Pot Duo',       8999.00,  60, 'HK-IPD-001'),
    (3, 'Philips Air Fryer',     6499.00,  80, 'HK-PAF-001');

-- ─────────────────────────────────
-- Seed orders
-- ─────────────────────────────────
INSERT INTO orders (customer_id, status, total_amount) VALUES
    (1, 'delivered', 79999.00),
    (1, 'delivered', 24999.00),
    (2, 'shipped',   69999.00),
    (3, 'pending',    8999.00),
    (4, 'delivered', 194998.00),
    (5, 'processing', 2499.00),
    (7, 'delivered',  64999.00),
    (8, 'cancelled',   1299.00),
    (10,'delivered',  24999.00),
    (2, 'pending',    16498.00);

-- ─────────────────────────────────
-- Seed order_items
-- ─────────────────────────────────
INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES
    (1,  1, 1, 79999.00),   -- Rahul: iPhone 15
    (2,  3, 1, 24999.00),   -- Rahul: Sony headphones
    (3,  2, 1, 69999.00),   -- Priya: Samsung S24
    (4,  9, 1,  8999.00),   -- Amit: Instant Pot
    (5,  4, 1,114999.00),   -- Sneha: MacBook Air
    (5,  1, 1, 79999.00),   -- Sneha: iPhone 15 (same order)
    (6,  8, 1,  2499.00),   -- Vikram: Denim Jeans
    (7,  5, 1, 64999.00),   -- Rohan: OnePlus 12
    (8,  6, 1,  1299.00),   -- Kavya: Formal Shirt (cancelled)
    (9,  3, 1, 24999.00),   -- Divya: Sony headphones
    (10, 9, 1,  8999.00),   -- Priya: Instant Pot
    (10,10, 1,  6499.00);   -- Priya: Air Fryer (same order)

-- ─────────────────────────────────
-- Seed reviews
-- ─────────────────────────────────
INSERT INTO reviews (customer_id, product_id, rating, title, body) VALUES
    (1, 1, 5, 'Excellent phone!',      'Best iPhone yet. Camera is outstanding.'),
    (1, 3, 4, 'Great headphones',      'Noise cancellation is top notch.'),
    (2, 2, 4, 'Solid Android phone',   'Great display, smooth performance.'),
    (4, 4, 5, 'Best laptop ever',      'M2 chip is blazing fast. Worth every rupee.'),
    (4, 1, 5, 'Amazing!',              'Second iPhone and still impressed.'),
    (7, 5, 4, 'Good value phone',      'OnePlus never disappoints.'),
    (10,3, 3, 'Decent headphones',     'Good but overpriced for Indian market.');
```

---

## ⚠️ GOTCHAS — INSERT Mistakes That Bite

**1. Inserting in wrong FK order**
```sql
-- This FAILS — products references categories, which doesn't exist yet:
INSERT INTO products (category_id, name, price, stock)
VALUES (1, 'iPhone 15', 79999.00, 10);
-- ERROR: foreign key constraint fails
-- Always insert parent tables before child tables:
-- categories → customers → products → orders → order_items
```

**2. String values need quotes, numbers don't**
```sql
-- ❌ Wrong:
INSERT INTO categories (name) VALUES (Electronics);   -- treated as column name
INSERT INTO products (price) VALUES ('79999.00');     -- works but bad practice

-- ✅ Correct:
INSERT INTO categories (name) VALUES ('Electronics'); -- strings in quotes
INSERT INTO products (price) VALUES (79999.00);       -- numbers without quotes
```

**3. NULL vs empty string — they are NOT the same**
```sql
INSERT INTO customers (first_name, phone) VALUES ('Amit', NULL);
-- phone is NULL — means "not provided, unknown"

INSERT INTO customers (first_name, phone) VALUES ('Amit', '');
-- phone is '' — means "provided, but empty"
-- These behave differently in WHERE clauses:
WHERE phone IS NULL      -- finds NULL values
WHERE phone = ''         -- finds empty string values
WHERE phone IS NOT NULL  -- finds both '' and actual numbers!
```

**4. AUTO_INCREMENT value after failed insert**
```sql
-- Attempt 1: INSERT violates constraint → fails → rolls back
-- But AUTO_INCREMENT counter already advanced!
INSERT INTO products (category_id, name, price) VALUES (999, 'Test', 99);
-- FAILS (bad category_id) but internal counter went from 10 to 11

-- Next successful insert:
INSERT INTO products (category_id, name, price) VALUES (1, 'Test', 99);
-- Gets product_id = 11, not 10 (gap of 1 exists)
-- This is normal and expected — don't panic about gaps
```

**5. Implicit vs explicit NULL for omitted columns**
```sql
-- If you omit 'phone' from INSERT and it's defined as NULL:
INSERT INTO customers (first_name, last_name, email, city, state)
VALUES ('Test', 'User', 'test@test.com', 'Mumbai', 'Maharashtra');
-- phone → automatically NULL (column allows NULL)

-- If you omit 'first_name' and it's NOT NULL with no DEFAULT:
INSERT INTO customers (last_name, email, city, state)
VALUES ('User', 'test@test.com', 'Mumbai', 'Maharashtra');
-- ERROR: Field 'first_name' doesn't have a default value
```

---

✅ **Quick Check:**
ShopNest's developer writes this INSERT to add a new order item:

```sql
INSERT INTO order_items
VALUES (1, 3, 2, 1299.00);
```

Name **two specific problems** with this statement — one related to style/safety and one related to the data itself given our existing seed data — and write the corrected version.
