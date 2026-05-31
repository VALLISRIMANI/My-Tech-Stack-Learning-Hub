📍 **Module 3 → Concept 4 of 5: Constraints — PRIMARY KEY, FOREIGN KEY, UNIQUE, NOT NULL, CHECK, DEFAULT**

---

## 📖 THEORY — What is it and why does it exist?

### The Problem: Data Without Rules is Just Noise

Imagine ShopNest's database with no constraints:

```
customers table — no rules enforced:
┌─────────────┬────────────┬───────────────────┬──────┐
│ customer_id │ first_name │ email             │ city │
├─────────────┼────────────┼───────────────────┼──────┤
│      1      │ Rahul      │ rahul@mail.com    │ Hyd  │
│      1      │ NULL       │ rahul@mail.com    │ NULL │  ← duplicate ID!
│      3      │ Priya      │ NULL              │ NULL │  ← no email!
│      3      │ Priya      │ rahul@mail.com    │ NULL │  ← duplicate email!
└─────────────┴────────────┴───────────────────┴──────┘

orders table — no rules enforced:
┌──────────┬─────────────┬───────────────┐
│ order_id │ customer_id │ total_amount  │
├──────────┼─────────────┼───────────────┤
│    1     │    9999     │   -500.00     │  ← customer 9999 doesn't exist!
│    1     │    1        │   -500.00     │  ← negative total!
└──────────┴─────────────┴───────────────┘
```

Every one of these corruptions would silently succeed without constraints. Your application logic might catch some of them — but application code has bugs, gets bypassed, gets rewritten. **Constraints are the database's own immune system** — they enforce data integrity at the lowest possible level, unconditionally, regardless of which application or user inserts the data.

MySQL has six constraint types. Let's go through each precisely.

---

## 🔤 SYNTAX + THEORY — Each Constraint in Depth

---

### ① PRIMARY KEY

**What it does:** Uniquely identifies each row in a table. Enforces both `UNIQUE` and `NOT NULL` simultaneously. Every table should have exactly one primary key. MySQL automatically creates a **clustered index** on the primary key — rows are physically sorted by PK on disk, making PK lookups extremely fast.

```sql
-- Column-level (single column PK):
CREATE TABLE customers (
    customer_id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    -- ↑ inline definition
    ...
);

-- Table-level (preferred for clarity, required for composite PK):
CREATE TABLE customers (
    customer_id INT UNSIGNED NOT NULL AUTO_INCREMENT,
    ...
    PRIMARY KEY (customer_id)
);

-- Composite primary key (two columns together form the unique identifier):
CREATE TABLE order_items (
    order_id    INT UNSIGNED NOT NULL,
    product_id  INT UNSIGNED NOT NULL,
    quantity    INT UNSIGNED NOT NULL,
    unit_price  DECIMAL(10,2) NOT NULL,

    PRIMARY KEY (order_id, product_id)
    -- The COMBINATION must be unique
    -- order_id=1 + product_id=101 is unique
    -- order_id=1 + product_id=102 is also fine
    -- order_id=1 + product_id=101 AGAIN → ERROR (duplicate PK)
);
```

---

### ② FOREIGN KEY

**What it does:** Links a column in one table (child) to the primary key of another table (parent). Enforces **referential integrity** — you cannot have an order for a customer that doesn't exist, or an order item for a product that doesn't exist.

```sql
CREATE TABLE orders (
    order_id    INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    customer_id INT UNSIGNED NOT NULL,
    ...

    -- Inline foreign key definition:
    FOREIGN KEY (customer_id)
        REFERENCES customers(customer_id)
        ON DELETE RESTRICT        -- what happens when parent row is deleted
        ON UPDATE CASCADE         -- what happens when parent PK is updated
);

-- Named foreign key (recommended — easier to drop later):
CREATE TABLE orders (
    order_id    INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    customer_id INT UNSIGNED NOT NULL,
    ...

    CONSTRAINT fk_orders_customer
        FOREIGN KEY (customer_id)
        REFERENCES customers(customer_id)
        ON DELETE RESTRICT
        ON UPDATE CASCADE
);
```

**ON DELETE and ON UPDATE actions — this is critical:**

| Action | What happens to child rows when parent is deleted/updated |
|---|---|
| `RESTRICT` | **Blocks** the delete/update if child rows exist. Parent protected. |
| `CASCADE` | Child rows are **automatically deleted/updated** to match |
| `SET NULL` | Child FK column is set to **NULL** (column must be nullable) |
| `SET DEFAULT` | Child FK set to its **DEFAULT value** (rarely used) |
| `NO ACTION` | Same as RESTRICT in MySQL (checked immediately) |

```sql
-- Real ShopNest scenarios:

-- Scenario 1: Cannot delete a customer who has orders
CONSTRAINT fk_orders_customer
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
    ON DELETE RESTRICT     -- blocks deletion of customer with orders
    ON UPDATE CASCADE      -- if customer_id changes, propagate to orders

-- Scenario 2: Deleting an order cascades to its items
CONSTRAINT fk_items_order
    FOREIGN KEY (order_id) REFERENCES orders(order_id)
    ON DELETE CASCADE      -- delete order → automatically deletes its items
    ON UPDATE CASCADE

-- Scenario 3: Product deleted → order items keep history (set NULL)
CONSTRAINT fk_items_product
    FOREIGN KEY (product_id) REFERENCES products(product_id)
    ON DELETE SET NULL     -- product removed but order_item history preserved
    ON UPDATE CASCADE
    -- (requires product_id in order_items to be nullable)
```

---

### ③ UNIQUE

**What it does:** Ensures no two rows have the same value in the specified column(s). Unlike PRIMARY KEY, a table can have multiple UNIQUE constraints, and UNIQUE columns **can contain NULL** (in MySQL, multiple NULLs in a UNIQUE column are allowed — NULL ≠ NULL).

```sql
-- Single column unique:
CREATE TABLE customers (
    customer_id  INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    email        VARCHAR(100) NOT NULL UNIQUE,
    -- ↑ inline definition
    ...
);

-- Named unique constraint (table-level — preferred):
CREATE TABLE customers (
    customer_id  INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    email        VARCHAR(100) NOT NULL,
    phone        VARCHAR(15)  NULL,
    ...

    CONSTRAINT uq_customer_email UNIQUE (email),
    CONSTRAINT uq_customer_phone UNIQUE (phone)
    -- phone can be NULL in multiple rows (NULL ≠ NULL in UNIQUE)
);

-- Composite unique — combination must be unique:
CREATE TABLE reviews (
    review_id   INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    customer_id INT UNSIGNED NOT NULL,
    product_id  INT UNSIGNED NOT NULL,
    rating      TINYINT UNSIGNED NOT NULL,
    ...

    CONSTRAINT uq_one_review_per_customer_product
        UNIQUE (customer_id, product_id)
    -- One customer can review one product only once
    -- customer_id=1 + product_id=101 → allowed once only
);
```

---

### ④ NOT NULL

**What it does:** Prevents NULL values in a column. Seems simple but is one of the most important data quality decisions you make per column.

```sql
CREATE TABLE products (
    product_id   INT UNSIGNED    AUTO_INCREMENT PRIMARY KEY,
    name         VARCHAR(100)    NOT NULL,       -- must always have a name
    description  TEXT            NULL,           -- optional
    price        DECIMAL(10,2)   NOT NULL,       -- must have a price
    image_url    VARCHAR(500)    NULL,           -- optional
    stock        INT UNSIGNED    NOT NULL DEFAULT 0,  -- required, defaults to 0
    ...
);

-- Adding NOT NULL to existing column (requires no existing NULLs):
ALTER TABLE products
    MODIFY COLUMN image_url VARCHAR(500) NOT NULL DEFAULT '';
-- ⚠️ Fails if any existing rows have NULL in image_url
-- Must UPDATE those rows first, then ALTER

-- Removing NOT NULL constraint:
ALTER TABLE products
    MODIFY COLUMN image_url VARCHAR(500) NULL;
```

---

### ⑤ CHECK

**What it does:** Validates that column values satisfy a custom boolean expression. Fully enforced in **MySQL 8.0.16+** (earlier versions parsed but silently ignored CHECK constraints — a famous MySQL gotcha).

```sql
CREATE TABLE products (
    product_id   INT UNSIGNED    AUTO_INCREMENT PRIMARY KEY,
    name         VARCHAR(100)    NOT NULL,
    price        DECIMAL(10,2)   NOT NULL,
    stock        INT UNSIGNED    NOT NULL DEFAULT 0,
    discount_pct DECIMAL(5,2)    NOT NULL DEFAULT 0.00,

    -- Named CHECK constraints:
    CONSTRAINT chk_price_positive
        CHECK (price > 0),

    CONSTRAINT chk_discount_range
        CHECK (discount_pct >= 0 AND discount_pct <= 100),

    CONSTRAINT chk_name_length
        CHECK (CHAR_LENGTH(name) >= 2)
);

-- Multi-column CHECK (cross-column validation):
CREATE TABLE orders (
    order_id       INT UNSIGNED    AUTO_INCREMENT PRIMARY KEY,
    order_date     DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
    delivery_date  DATETIME        NULL,
    total_amount   DECIMAL(10,2)   NOT NULL,

    CONSTRAINT chk_delivery_after_order
        CHECK (delivery_date IS NULL OR delivery_date > order_date),

    CONSTRAINT chk_total_non_negative
        CHECK (total_amount >= 0)
);

-- Testing CHECK enforcement:
INSERT INTO products (name, price, stock, discount_pct)
VALUES ('iPhone 15', -500, 10, 0);
-- ERROR 3819: Check constraint 'chk_price_positive' is violated ✅
```

---

### ⑥ DEFAULT

**What it does:** Specifies the value MySQL inserts automatically when a column is omitted from an INSERT statement. Reduces repetitive code and ensures sensible fallback values.

```sql
CREATE TABLE orders (
    order_id      INT UNSIGNED    AUTO_INCREMENT PRIMARY KEY,
    customer_id   INT UNSIGNED    NOT NULL,

    -- Literal defaults:
    status        ENUM('pending','processing','shipped',
                       'delivered','cancelled')
                                  NOT NULL DEFAULT 'pending',
    total_amount  DECIMAL(10,2)   NOT NULL DEFAULT 0.00,
    is_gift       BOOLEAN         NOT NULL DEFAULT FALSE,

    -- Function defaults:
    order_date    DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
    -- ↑ auto-sets to current date/time on INSERT

    updated_at    TIMESTAMP       NOT NULL DEFAULT CURRENT_TIMESTAMP
                                           ON UPDATE CURRENT_TIMESTAMP,
    -- ↑ auto-updates on every row modification

    notes         VARCHAR(500)    NULL DEFAULT NULL
    -- explicit NULL default (same as just NULL, but self-documenting)
);

-- DEFAULT in action:
INSERT INTO orders (customer_id, total_amount)
VALUES (1, 79999.00);
-- status    → 'pending'          (DEFAULT applied)
-- is_gift   → FALSE              (DEFAULT applied)
-- order_date→ current timestamp  (DEFAULT applied)
-- updated_at→ current timestamp  (DEFAULT applied)
```

---

## 💡 EXAMPLE — ShopNest Full Constrained Schema

```sql
USE shopnest_prod;

-- Categories
CREATE TABLE IF NOT EXISTS categories (
    category_id  INT UNSIGNED     NOT NULL AUTO_INCREMENT,
    name         VARCHAR(50)      NOT NULL,
    description  TEXT             NULL,
    is_active    BOOLEAN          NOT NULL DEFAULT TRUE,

    PRIMARY KEY (category_id),
    CONSTRAINT uq_category_name UNIQUE (name)
) ENGINE = InnoDB DEFAULT CHARSET = utf8mb4 COLLATE = utf8mb4_unicode_ci;

-- Customers
CREATE TABLE IF NOT EXISTS customers (
    customer_id  INT UNSIGNED     NOT NULL AUTO_INCREMENT,
    first_name   VARCHAR(50)      NOT NULL,
    last_name    VARCHAR(50)      NOT NULL,
    email        VARCHAR(100)     NOT NULL,
    phone        VARCHAR(15)      NULL,
    city         VARCHAR(50)      NOT NULL,
    state        VARCHAR(50)      NOT NULL,
    created_at   TIMESTAMP        NOT NULL DEFAULT CURRENT_TIMESTAMP,

    PRIMARY KEY (customer_id),
    CONSTRAINT uq_customer_email UNIQUE (email),
    CONSTRAINT chk_email_format  CHECK (email LIKE '%@%.%')
) ENGINE = InnoDB DEFAULT CHARSET = utf8mb4 COLLATE = utf8mb4_unicode_ci;

-- Products
CREATE TABLE IF NOT EXISTS products (
    product_id   INT UNSIGNED     NOT NULL AUTO_INCREMENT,
    category_id  INT UNSIGNED     NOT NULL,
    name         VARCHAR(100)     NOT NULL,
    description  TEXT             NULL,
    price        DECIMAL(10,2)    NOT NULL,
    stock        INT UNSIGNED     NOT NULL DEFAULT 0,
    created_at   TIMESTAMP        NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at   TIMESTAMP        NOT NULL DEFAULT CURRENT_TIMESTAMP
                                           ON UPDATE CURRENT_TIMESTAMP,

    PRIMARY KEY (product_id),
    CONSTRAINT fk_products_category
        FOREIGN KEY (category_id) REFERENCES categories(category_id)
        ON DELETE RESTRICT
        ON UPDATE CASCADE,
    CONSTRAINT chk_product_price
        CHECK (price > 0)
) ENGINE = InnoDB DEFAULT CHARSET = utf8mb4 COLLATE = utf8mb4_unicode_ci;

-- Orders
CREATE TABLE IF NOT EXISTS orders (
    order_id     INT UNSIGNED     NOT NULL AUTO_INCREMENT,
    customer_id  INT UNSIGNED     NOT NULL,
    order_date   DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP,
    status       ENUM('pending','processing','shipped',
                      'delivered','cancelled')
                                  NOT NULL DEFAULT 'pending',
    total_amount DECIMAL(10,2)    NOT NULL DEFAULT 0.00,

    PRIMARY KEY (order_id),
    CONSTRAINT fk_orders_customer
        FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
        ON DELETE RESTRICT
        ON UPDATE CASCADE,
    CONSTRAINT chk_order_total
        CHECK (total_amount >= 0)
) ENGINE = InnoDB DEFAULT CHARSET = utf8mb4 COLLATE = utf8mb4_unicode_ci;

-- Order Items
CREATE TABLE IF NOT EXISTS order_items (
    order_item_id INT UNSIGNED    NOT NULL AUTO_INCREMENT,
    order_id      INT UNSIGNED    NOT NULL,
    product_id    INT UNSIGNED    NOT NULL,
    quantity      INT UNSIGNED    NOT NULL,
    unit_price    DECIMAL(10,2)   NOT NULL,

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
    CONSTRAINT chk_quantity_positive
        CHECK (quantity > 0),
    CONSTRAINT chk_unit_price_positive
        CHECK (unit_price > 0)
) ENGINE = InnoDB DEFAULT CHARSET = utf8mb4 COLLATE = utf8mb4_unicode_ci;
```

---

## ⚠️ GOTCHAS — Constraint Mistakes

**1. CHECK constraints silently ignored before MySQL 8.0.16**
```sql
-- On MySQL 5.7 or early 8.0:
CONSTRAINT chk_price CHECK (price > 0)
-- Parses without error but is NEVER enforced
-- INSERT with price = -999 succeeds silently 😱
-- Always verify your MySQL version: SELECT VERSION();
-- Always test that your CHECKs actually reject bad data
```

**2. FOREIGN KEY requires matching data types exactly**
```sql
-- parent:
customer_id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY

-- child (wrong — signed vs unsigned mismatch):
customer_id INT NOT NULL  -- ❌ ERROR: types don't match

-- child (correct):
customer_id INT UNSIGNED NOT NULL  -- ✅ exact match required
```

**3. NULL in UNIQUE columns**
```sql
-- UNIQUE constraint allows multiple NULLs in MySQL:
phone VARCHAR(15) NULL UNIQUE
-- Three customers with phone = NULL → all allowed ✅
-- (NULL is not equal to NULL in SQL)
-- Two customers with phone = '9876543210' → ERROR ✅
-- This is correct SQL standard behavior, just surprising
```

**4. ON DELETE CASCADE can be dangerous**
```sql
-- With ON DELETE CASCADE on order_items → orders:
DELETE FROM orders WHERE order_id = 1;
-- Automatically deletes ALL order_items for order 1 too
-- Intended? Maybe. Accidental mass deletion? Also possible.
-- For financial data, prefer RESTRICT and archive instead of delete
```

**5. Forgetting to name constraints**
```sql
-- Unnamed constraint (MySQL generates a name like 'products_chk_1'):
CHECK (price > 0)

-- Named constraint:
CONSTRAINT chk_product_price CHECK (price > 0)

-- Why naming matters:
ALTER TABLE products DROP CHECK products_chk_1;   -- hard to know which
ALTER TABLE products DROP CHECK chk_product_price; -- self-documenting ✅
```

**6. DEFAULT doesn't work with TEXT columns**
```sql
description TEXT DEFAULT 'No description'  -- ❌ ERROR
description TEXT NULL                      -- ✅ use NULL as the "no value" signal
-- Or switch to VARCHAR if you need a non-NULL default:
description VARCHAR(1000) NOT NULL DEFAULT ''
```

---

✅ **Quick Check:**
ShopNest's `order_items` table has this foreign key:

```sql
CONSTRAINT fk_items_order
    FOREIGN KEY (order_id) REFERENCES orders(order_id)
    ON DELETE CASCADE
```

And this foreign key:

```sql
CONSTRAINT fk_items_product
    FOREIGN KEY (product_id) REFERENCES products(product_id)
    ON DELETE RESTRICT
```

**Question:** A customer calls to cancel their order. The support team runs `DELETE FROM orders WHERE order_id = 5`. Meanwhile, the warehouse team tries to discontinue a product by running `DELETE FROM products WHERE product_id = 101` — but product 101 appears in order_item rows.

What happens in each case, and why did ShopNest intentionally choose different `ON DELETE` actions for these two foreign keys?
