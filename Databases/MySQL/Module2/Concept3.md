📍 **Module 2 → Concept 3 of 3: Physical vs Logical Schema Design**

---

## 📖 THEORY — What is it and why does it exist?

### The Problem: There's a gap between "what data means" and "how it's stored"

When you design a database, you're actually making decisions at **two completely different levels** simultaneously — and confusing them is one of the most common sources of bad database design.

Think about ShopNest's `price` column. At the business level, price is "the amount a customer pays for a product in Indian Rupees." That's a business concept. But at the storage level, MySQL needs to know: is it `DECIMAL(10,2)`? `FLOAT`? `INT` storing paise? How many bytes? Is it indexed? These are two entirely separate conversations.

This separation is formalized as **Logical Schema** vs **Physical Schema**.

---

### Logical Schema — The "What" 🗺️

The **logical schema** describes your data in **business terms**, completely independent of any database technology. It answers:

- What entities exist?
- What attributes does each entity have?
- What are the relationships between entities?
- What business rules apply?

The logical schema is **technology-agnostic** — it would look the same whether you implement it in MySQL, PostgreSQL, Oracle, or even MongoDB. It's the output of your ER diagram work from Concept 1.

```
Logical Schema for ShopNest (technology-agnostic):

CUSTOMER
  - has a unique identifier
  - has a name (first and last)
  - has a contact email (must be unique)
  - has an optional phone number
  - belongs to one city
  - can place zero or many orders

PRODUCT
  - has a unique identifier
  - has a name
  - has a selling price (monetary value, 2 decimal places)
  - has a stock count (whole number, cannot be negative)
  - belongs to exactly one category

ORDER
  - has a unique identifier
  - belongs to exactly one customer
  - has a date and time it was placed
  - has a status (pending/shipped/delivered/cancelled)
  - contains one or many order items

ORDER_ITEM
  - belongs to exactly one order
  - references exactly one product
  - has a quantity (positive whole number)
  - records the price at time of purchase
```

Notice: no mention of `INT`, `VARCHAR`, `INDEX`, or `ENGINE`. Pure business logic.

---

### Physical Schema — The "How" ⚙️

The **physical schema** is the actual implementation in a specific database system. It translates every logical decision into concrete technical choices:

- What **data type** stores each attribute?
- What are the **size constraints**?
- Which columns are **indexed** for performance?
- What **storage engine** is used?
- What are the exact **constraint** definitions?
- How are **foreign keys** enforced?

The physical schema is what you actually write as `CREATE TABLE` statements in MySQL.

---

### The Translation Process — Logical → Physical

Here's how every logical decision maps to a physical one:

```
LOGICAL DECISION              PHYSICAL IMPLEMENTATION
─────────────────────────────────────────────────────
"unique identifier"        →  INT AUTO_INCREMENT PRIMARY KEY
"name (first and last)"    →  first_name VARCHAR(50) NOT NULL,
                              last_name  VARCHAR(50) NOT NULL
"contact email (unique)"   →  email VARCHAR(100) NOT NULL UNIQUE
"optional phone"           →  phone VARCHAR(15) NULL
"monetary value"           →  DECIMAL(10,2) NOT NULL
"stock (non-negative)"     →  INT UNSIGNED NOT NULL DEFAULT 0
"date and time placed"     →  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
"status (fixed options)"   →  ENUM('pending','shipped',
                                   'delivered','cancelled')
"belongs to one customer"  →  customer_id INT NOT NULL,
                              FOREIGN KEY (customer_id)
                              REFERENCES customers(customer_id)
```

---

### Key Physical Design Decisions to Make

**① Choosing the right data type**

This is the most impactful physical decision. Wrong types waste space, slow queries, and cause subtle bugs:

```
Logical: "product price"

❌ FLOAT      → stores 79999.99 as ~79999.9921875 (floating point error)
               Never use FLOAT for money.

❌ INT        → can't store decimals. ₹79999 becomes ₹79999 (paise lost)

✅ DECIMAL(10,2) → exact decimal storage. 79999.99 = 79999.99 always.
                  (10 = total digits, 2 = digits after decimal point)
                  Supports up to ₹99,999,999.99
```

```
Logical: "customer email (unique identifier for login)"

❌ TEXT       → can't be indexed efficiently in MySQL
❌ CHAR(100)  → wastes space (always 100 bytes even for short emails)
✅ VARCHAR(100) → variable length, indexable, right-sized
```

**② NULL vs NOT NULL**

Every column must be explicitly decided:
```
first_name  NOT NULL  → every customer must have a first name (business rule)
phone       NULL      → phone is optional (business rule)
deleted_at  NULL      → NULL means "not deleted" (soft delete pattern)
```

Leaving columns nullable by default (MySQL's default behavior) when they shouldn't be is one of the most common physical schema mistakes.

**③ AUTO_INCREMENT for surrogate keys**

```sql
-- Natural key: using a real-world value as PK
-- (email as PK — seems logical but problematic)
customer_id VARCHAR(100) PRIMARY KEY  -- email as PK
-- Problem: emails change. Updating a PK cascades everywhere. Slow joins.

-- Surrogate key: a system-generated meaningless integer
customer_id INT AUTO_INCREMENT PRIMARY KEY
-- MySQL generates 1, 2, 3... automatically
-- Fast joins (INT comparison), never changes, always unique
```

Always prefer **surrogate keys** (AUTO_INCREMENT INT or BIGINT) over natural keys for primary keys in MySQL.

**④ UNSIGNED for non-negative numbers**

```sql
stock_quantity INT UNSIGNED NOT NULL DEFAULT 0
-- UNSIGNED means no negative values allowed (0 to 4,294,967,295)
-- vs signed INT: (-2,147,483,648 to 2,147,483,647)
-- Also doubles the positive range for free
```

**⑤ Choosing INT size wisely**

| Type | Storage | Range (signed) | Use for |
|---|---|---|---|
| `TINYINT` | 1 byte | -128 to 127 | Status flags, small counts |
| `SMALLINT` | 2 bytes | -32,768 to 32,767 | Age, small quantities |
| `INT` | 4 bytes | -2.1B to 2.1B | Most IDs, counts |
| `BIGINT` | 8 bytes | -9.2 quintillion to 9.2Q | Large IDs, financial totals |

ShopNest today has 10,000 customers → `INT` is fine. ShopNest at Amazon scale → `BIGINT`.

---

## 🔤 SYNTAX — ShopNest Physical Schema (Complete)

This is the full `CREATE TABLE` translation of our logical design — the foundation we'll use for the rest of the course:

```sql
-- Make sure we're using the right database
USE shopnest;

-- ① Categories
CREATE TABLE categories (
    category_id   INT           UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name          VARCHAR(50)   NOT NULL UNIQUE,
    description   TEXT          NULL
) ENGINE = InnoDB;

-- ② Customers
CREATE TABLE customers (
    customer_id   INT           UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    first_name    VARCHAR(50)   NOT NULL,
    last_name     VARCHAR(50)   NOT NULL,
    email         VARCHAR(100)  NOT NULL UNIQUE,
    phone         VARCHAR(15)   NULL,
    city          VARCHAR(50)   NOT NULL,
    state         VARCHAR(50)   NOT NULL,
    created_at    DATETIME      NOT NULL DEFAULT CURRENT_TIMESTAMP
) ENGINE = InnoDB;

-- ③ Products
CREATE TABLE products (
    product_id    INT           UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    category_id   INT           UNSIGNED NOT NULL,
    name          VARCHAR(100)  NOT NULL,
    description   TEXT          NULL,
    price         DECIMAL(10,2) NOT NULL,
    stock         INT           UNSIGNED NOT NULL DEFAULT 0,
    created_at    DATETIME      NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (category_id) REFERENCES categories(category_id)
) ENGINE = InnoDB;

-- ④ Orders
CREATE TABLE orders (
    order_id      INT           UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    customer_id   INT           UNSIGNED NOT NULL,
    order_date    DATETIME      NOT NULL DEFAULT CURRENT_TIMESTAMP,
    status        ENUM('pending','processing','shipped',
                       'delivered','cancelled')
                                NOT NULL DEFAULT 'pending',
    total_amount  DECIMAL(10,2) NOT NULL DEFAULT 0.00,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
) ENGINE = InnoDB;

-- ⑤ Order Items (junction table)
CREATE TABLE order_items (
    order_item_id INT           UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    order_id      INT           UNSIGNED NOT NULL,
    product_id    INT           UNSIGNED NOT NULL,
    quantity      INT           UNSIGNED NOT NULL,
    unit_price    DECIMAL(10,2) NOT NULL,
    FOREIGN KEY (order_id)   REFERENCES orders(order_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
) ENGINE = InnoDB;
```

---

## 💡 EXAMPLE — Logical Decision → Physical Consequence

```
SCENARIO: ShopNest's analyst says
"We need to track when orders are cancelled and why."

LOGICAL DECISION:
  Orders have an optional cancellation timestamp
  Orders have an optional cancellation reason (free text)

PHYSICAL TRANSLATION:
  ALTER TABLE orders
  ADD COLUMN cancelled_at  DATETIME     NULL,
  ADD COLUMN cancel_reason VARCHAR(255) NULL;

  Why DATETIME not TIMESTAMP?
    TIMESTAMP range: 1970–2038 (the Y2K38 problem)
    DATETIME range:  1000–9999 (safe forever)
    Use DATETIME for business dates. TIMESTAMP for auto-tracking.

  Why VARCHAR(255) not TEXT?
    TEXT can't have a DEFAULT value in MySQL
    VARCHAR(255) is indexable if needed later
    255 chars is enough for a cancellation reason
```

---

## ⚠️ GOTCHAS — Physical Schema Mistakes

**1. Using FLOAT for money — the silent killer**
```sql
-- This looks fine but produces wrong results:
price FLOAT NOT NULL
-- SELECT 0.1 + 0.2 → 0.30000000000000004
-- Over thousands of transactions, rounding errors
-- compound into real financial discrepancies.
-- ALWAYS use DECIMAL for monetary values.
```

**2. Making everything VARCHAR(255) lazily**
```sql
-- status VARCHAR(255)  ← accepts "banana", "YOLO", anything
-- status ENUM('pending','shipped','delivered')
--   ← only valid values allowed, stored as 1-2 bytes (more efficient)
-- Use ENUM when you have a known fixed set of options.
```

**3. Forgetting DEFAULT values**
```sql
-- Without DEFAULT, inserting without specifying stock gives NULL or error
stock INT UNSIGNED NOT NULL          -- ❌ must always specify stock on insert

stock INT UNSIGNED NOT NULL DEFAULT 0 -- ✅ omitting stock defaults to 0
```

**4. Using VARCHAR for phone numbers**
```sql
-- phone INT    ← loses leading zeros (+91 numbers, "0" prefix)
--              ← can't store "+91-98765-43210" format
-- phone VARCHAR(15) ← stores any format, preserves leading zeros ✅
```

**5. TIMESTAMP vs DATETIME confusion**
```sql
created_at TIMESTAMP  -- auto-converts to UTC, has 2038 limit
                      -- good for "when was this row last touched"
order_date DATETIME   -- stores exactly what you give it, no timezone
                      -- good for business dates that must not shift
```

**6. Skipping the physical schema review before going live**
The logical schema is designed for humans. The physical schema is designed for MySQL. Always review: are all NOT NULLs correct? Are ENUMs used where appropriate? Are foreign keys defined? Is the storage engine InnoDB everywhere transactional data lives?

✅ **Module 2 Complete!** 