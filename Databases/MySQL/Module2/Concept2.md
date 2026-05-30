📍 **Module 2 → Concept 2 of 3: Normalization — 1NF, 2NF, 3NF, BCNF**

---

## 📖 THEORY — What is it and why does it exist?

### The Problem: What goes wrong with bad table design?

Imagine ShopNest's developer skips ER design and just throws everything into one big table:

```
┌──────────────────────────────────────────────────────────────────────┐
│                         orders (the BAD way)                         │
├────────┬──────────┬───────────────┬──────────┬────────────┬──────────┤
│order_id│cust_name │cust_email     │cust_city │product_name│category  │
├────────┼──────────┼───────────────┼──────────┼────────────┼──────────┤
│   1    │Rahul S.  │rahul@mail.com │Hyderabad │iPhone 15   │Electronics│
│   2    │Rahul S.  │rahul@mail.com │Hyderabad │AirPods     │Electronics│
│   3    │Priya N.  │priya@mail.com │Chennai   │Kurta Set   │Clothing  │
│   4    │Rahul S.  │rahul@mail.com │Hyderabad │Kurta Set   │Clothing  │
└────────┴──────────┴───────────────┴──────────┴────────────┴──────────┘
```

This design has three categories of problems called **anomalies**:

```
UPDATE anomaly:  Rahul moves to Bangalore → must update 3 rows
                 Miss one → data is now inconsistent forever

INSERT anomaly:  Can't add a new product (like "MacBook Pro")
                 until someone actually orders it

DELETE anomaly:  Priya cancels her only order (row 3) →
                 ALL information about Priya is permanently lost
```

**Normalization** is the process of restructuring tables to eliminate these anomalies. It's a series of progressive rules called **Normal Forms** — each one fixing a specific class of problem. Think of it as progressively "cleaning up" your design.

---

### The Concept of Functional Dependency

Before the normal forms, you need one key concept:

**Functional Dependency:** Column B is *functionally dependent* on Column A if knowing A's value tells you exactly one value for B.

```
customer_id → email        ✅ (one customer_id maps to exactly one email)
customer_id → city         ✅ (one customer lives in one city)
order_id    → order_date   ✅ (one order has one date)
order_id    → cust_name    ✅ (one order belongs to one customer)

product_name → category    ✅ (iPhone 15 is always Electronics)
category    → product_name ❌ (Electronics has MANY products — not functional)
```

Written as: `A → B` ("A determines B" or "B depends on A")

---

### 1NF — First Normal Form: Eliminate Repeating Groups

**Rule:** Every column must contain **atomic** (indivisible) values. No repeating groups, no arrays, no comma-separated lists in a single cell.

**Violation example:**
```
❌ NOT in 1NF:
┌──────────┬───────────────────────────────┐
│ order_id │ products                      │
├──────────┼───────────────────────────────┤
│    1     │ iPhone 15, AirPods, Phone Case │  ← multiple values in one cell
│    2     │ Kurta Set                      │
└──────────┴───────────────────────────────┘

Also violates 1NF:
┌──────────┬────────────┬────────────┬────────────┐
│ order_id │ product_1  │ product_2  │ product_3  │  ← repeating columns
├──────────┼────────────┼────────────┼────────────┤
│    1     │ iPhone 15  │ AirPods    │ Phone Case │
└──────────┴────────────┴────────────┴────────────┘
```

**Fix — make every cell atomic, one row per fact:**
```
✅ IN 1NF:
┌──────────┬─────────────┐
│ order_id │ product_name│
├──────────┼─────────────┤
│    1     │ iPhone 15   │
│    1     │ AirPods     │
│    1     │ Phone Case  │
│    2     │ Kurta Set   │
└──────────┴─────────────┘
Each cell has exactly one value. ✅
```

**1NF also requires:**
- Each column has a unique name
- All values in a column are of the same data type
- Each row is uniquely identifiable (has a primary key)

---

### 2NF — Second Normal Form: Eliminate Partial Dependencies

**Rule:** Must already be in 1NF. Additionally, every non-key column must depend on the **entire** primary key — not just part of it.

This only applies when you have a **composite primary key** (a primary key made of two or more columns).

**Violation example:**
```
order_items table with composite PK: (order_id, product_id)

┌──────────┬────────────┬──────────┬───────────────┬──────────────┐
│ order_id │ product_id │ quantity │ product_name  │ product_price│
├──────────┼────────────┼──────────┼───────────────┼──────────────┤
│    1     │    101     │    2     │ iPhone 15     │   79999      │
│    1     │    102     │    1     │ AirPods       │   12999      │
│    2     │    101     │    1     │ iPhone 15     │   79999      │
└──────────┴────────────┴──────────┴───────────────┴──────────────┘

Dependencies:
(order_id, product_id) → quantity    ✅ Full dependency — quantity depends on BOTH
product_id → product_name            ❌ Partial dependency — only depends on product_id
product_id → product_price           ❌ Partial dependency — only depends on product_id
```

`product_name` and `product_price` only need `product_id` to be determined — they don't need `order_id` at all. That's a **partial dependency** — a 2NF violation.

**Problems this causes:**
```
→ iPhone 15's name is stored in EVERY order_item row
→ Price changes require updating hundreds of rows
→ If all orders for iPhone 15 are deleted, we lose the product info
```

**Fix — move partially dependent columns to their own table:**
```
✅ IN 2NF:

order_items:                    products:
┌──────────┬────────────┬──────┐ ┌────────────┬───────────────┬───────┐
│ order_id │ product_id │ qty  │ │ product_id │ product_name  │ price │
├──────────┼────────────┼──────┤ ├────────────┼───────────────┼───────┤
│    1     │    101     │  2   │ │    101     │ iPhone 15     │ 79999 │
│    1     │    102     │  1   │ │    102     │ AirPods       │ 12999 │
│    2     │    101     │  1   │ └────────────┴───────────────┴───────┘
└──────────┴────────────┴──────┘
product_name lives in products once.
order_items only stores what's unique per order+product. ✅
```

---

### 3NF — Third Normal Form: Eliminate Transitive Dependencies

**Rule:** Must already be in 2NF. Additionally, no non-key column should depend on **another non-key column**.

In other words: non-key columns must depend on the primary key — **the whole key, and nothing but the key.**

**Violation example:**
```
customers table — PK: customer_id

┌─────────────┬───────────┬──────────┬─────────────┬──────────────┐
│ customer_id │ name      │ city     │ state       │ state_tax_pct│
├─────────────┼───────────┼──────────┼─────────────┼──────────────┤
│      1      │ Rahul S.  │Hyderabad │ Telangana   │    18%       │
│      2      │ Priya N.  │ Chennai  │ Tamil Nadu  │    18%       │
│      3      │ Amit K.   │ Mumbai   │ Maharashtra │    18%       │
└─────────────┴───────────┴──────────┴─────────────┴──────────────┘

Dependencies:
customer_id → city           ✅ Direct dependency on PK
customer_id → state          ✅ Direct dependency on PK
customer_id → state_tax_pct  ⚠️  But HOW? Via: customer→city→state→tax
state → state_tax_pct        ❌ Transitive dependency!
```

`state_tax_pct` doesn't really depend on `customer_id` — it depends on `state`, which depends on `customer_id`. This is a **transitive dependency** — a 3NF violation.

**Problems:**
```
→ Tax rate for Telangana stored in every Telangana customer row
→ Tax rate changes → update hundreds of rows → miss some → wrong taxes charged
→ A state with no customers yet can't have its tax rate stored
```

**Fix — extract the transitive dependency into its own table:**
```
✅ IN 3NF:

customers:                          states:
┌─────────────┬───────────┬──────┐  ┌─────────────┬──────────────┐
│ customer_id │ name      │ city │  │ state       │ state_tax_pct│
├─────────────┼───────────┼──────┤  ├─────────────┼──────────────┤
│      1      │ Rahul S.  │ Hyd  │  │ Telangana   │    18%       │
│      2      │ Priya N.  │ CHN  │  │ Tamil Nadu  │    18%       │
└─────────────┴───────────┴──────┘  │ Maharashtra │    18%       │
                                     └─────────────┴──────────────┘
Tax rate lives once per state.
Customers just reference their state. ✅
```

---

### BCNF — Boyce-Codd Normal Form: The Stricter 3NF

**Rule:** For every functional dependency `A → B`, A must be a **superkey** (a candidate key or the primary key). No exceptions.

BCNF is a slightly stronger version of 3NF. Most tables in 3NF are already in BCNF. Violations are rare but they do occur when a table has **multiple overlapping candidate keys**.

**Violation example (rare but real):**
```
ShopNest assigns one dedicated account_manager per customer per category:

┌─────────────┬──────────────┬─────────────────┐
│ customer_id │ category     │ account_manager  │
├─────────────┼──────────────┼─────────────────┤
│      1      │ Electronics  │ Vikram           │
│      1      │ Clothing     │ Sneha            │
│      2      │ Electronics  │ Vikram           │
│      3      │ Clothing     │ Sneha            │
└─────────────┴──────────────┴─────────────────┘

Business rule: each account_manager handles only ONE category.
So: account_manager → category  (Vikram always → Electronics)

Candidate keys: (customer_id, category) or (customer_id, account_manager)
But account_manager → category means account_manager is a determinant
yet it's NOT a superkey. → BCNF violation.
```

**Fix:**
```
✅ IN BCNF — split into two tables:

manager_categories:              customer_managers:
┌─────────────────┬──────────────┐ ┌─────────────┬─────────────────┐
│ account_manager │ category     │ │ customer_id │ account_manager │
├─────────────────┼──────────────┤ ├─────────────┼─────────────────┤
│ Vikram          │ Electronics  │ │      1      │ Vikram          │
│ Sneha           │ Clothing     │ │      1      │ Sneha           │
└─────────────────┴──────────────┘ │      2      │ Vikram          │
                                    └─────────────┴─────────────────┘
```

---

## 💡 EXAMPLE — ShopNest Normalization Journey

```sql
-- THE UNNORMALIZED MESS (0NF):
CREATE TABLE orders_mess (
    order_id        INT,
    customer_name   VARCHAR(100),   -- should be in customers table
    customer_email  VARCHAR(100),   -- should be in customers table
    customer_city   VARCHAR(50),    -- should be in customers table
    products        TEXT,           -- "iPhone15:2, AirPods:1" ← 1NF violation
    category        VARCHAR(50),    -- depends on product, not order ← 2NF violation
    state_tax       DECIMAL(5,2)    -- depends on city→state ← 3NF violation
);

-- AFTER FULL NORMALIZATION (3NF) — ShopNest's actual design:

-- customers (no transitive dependencies)
CREATE TABLE customers (
    customer_id  INT PRIMARY KEY,
    first_name   VARCHAR(50),
    last_name    VARCHAR(50),
    email        VARCHAR(100),
    phone        VARCHAR(15),
    city         VARCHAR(50),
    state        VARCHAR(50)
    -- state_tax removed → goes in states table
);

-- categories (extracted from products)
CREATE TABLE categories (
    category_id  INT PRIMARY KEY,
    name         VARCHAR(50),
    description  TEXT
);

-- products (no customer/order data here)
CREATE TABLE products (
    product_id   INT PRIMARY KEY,
    category_id  INT,             -- FK to categories
    name         VARCHAR(100),
    price        DECIMAL(10,2),
    stock        INT
);

-- orders (no product details here)
CREATE TABLE orders (
    order_id     INT PRIMARY KEY,
    customer_id  INT,             -- FK to customers
    order_date   DATETIME,
    status       VARCHAR(20),
    total_amount DECIMAL(10,2)
);

-- order_items (junction — resolves N:M between orders & products)
CREATE TABLE order_items (
    order_item_id  INT PRIMARY KEY,
    order_id       INT,           -- FK to orders
    product_id     INT,           -- FK to products
    quantity       INT,
    unit_price     DECIMAL(10,2)  -- price AT TIME OF ORDER (historical record)
);
```

---

## ⚠️ GOTCHAS — Normalization Mistakes

**1. "Always normalize to BCNF"**
In practice, most production databases target **3NF**. BCNF and beyond (4NF, 5NF exist too) can sometimes make queries more complex without meaningful benefit. Stop at 3NF unless you have a specific reason to go further.

**2. "Normalization always makes things faster"**
Wrong. Normalization reduces redundancy but increases the number of **JOIN operations** needed to query data. A heavily normalized database can be *slower* for read-heavy workloads. This is why data warehouses often **denormalize** intentionally — trading storage for query speed.

**3. "`unit_price` in `order_items` looks like redundancy — shouldn't it just reference `products.price`?"**
No — and this is a crucial design decision. Product prices change over time. If order #1001 from 2023 references `products.price` directly, and you later update the iPhone price to ₹85,000, the 2023 order now shows the wrong price. `unit_price` in `order_items` is a **historical snapshot** — intentional denormalization for data integrity.

**4. Splitting too aggressively**
Not every repeated value needs its own table. `city` in `customers` is fine as a plain column unless you need city-specific metadata (delivery zones, regional pricing). Don't over-normalize.

**5. Forgetting that 2NF only applies to composite keys**
If your table has a single-column primary key, it's automatically in 2NF (you can't have a partial dependency on a single column). Only check for 2NF violations when your PK spans multiple columns.

---

✅ **Quick Check:**
ShopNest has this table in their early design:

```
order_items: (order_id, product_id, quantity, product_name, category_name)
Primary Key: (order_id, product_id)
```

Identify: **(a)** which Normal Form is violated and why, and **(b)** exactly how you would fix it — what tables would you create and what columns would move where?
