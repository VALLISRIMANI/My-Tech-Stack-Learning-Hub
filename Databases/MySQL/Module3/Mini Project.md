## 🏆 Module 3 Mini Project: Build ShopNest From Scratch

Run the full DDL script above in your MySQL environment, then complete these tasks:

```
TASK 1 — Run & Verify
  Execute the complete DDL script.
  Run SHOW TABLES to confirm all 6 tables exist.
  Run DESC on each table and verify columns look correct.

TASK 2 — Constraint Testing
  Try each of these — predict what will happen, then run it:

  a) INSERT INTO categories (name) VALUES ('Electronics');
     INSERT INTO categories (name) VALUES ('Electronics');
     → What constraint fires? Why?

  b) INSERT INTO products
         (category_id, name, price, stock)
     VALUES (999, 'iPhone 15', 79999.00, 10);
     → What constraint fires? Why?

  c) INSERT INTO products
         (category_id, name, price, stock)
     VALUES (1, 'iPhone 15', -500.00, 10);
     → What constraint fires? Why?

  d) INSERT INTO reviews
         (customer_id, product_id, rating)
     VALUES (1, 1, 7);
     → What constraint fires? Why?

TASK 3 — ALTER TABLE Practice
  ShopNest adds two new business requirements:
  a) Products now need a 'brand' column (VARCHAR 100, optional)
  b) Orders now support a 'coupon_code' column (CHAR 10, optional)
  Write the ALTER TABLE statements for both.

TASK 4 — Schema Design Question
  ShopNest wants to add a 'payments' table.
  Design it yourself — logical schema first, then physical.
  It should track: which order was paid, amount paid,
  payment method (card/upi/netbanking/cod),
  payment status (pending/completed/failed/refunded),
  transaction reference number (unique), and timestamp.
  Write the full CREATE TABLE statement with all
  appropriate constraints.

TASK 5 — Reflection
  Why does order_items use ON DELETE CASCADE for its
  orders foreign key, but ON DELETE RESTRICT for its
  products foreign key? What real-world business logic
  does each choice protect?
```

---


## TASK 1 — Run & Verify

```sql
SHOW TABLES;
-- Expected output:
-- categories
-- customers
-- order_items
-- orders
-- products
-- reviews

DESCRIBE categories;
DESCRIBE customers;
DESCRIBE products;
DESCRIBE orders;
DESCRIBE order_items;
DESCRIBE reviews;
```

---

## TASK 2 — Constraint Testing

### a) Duplicate category name

```sql
INSERT INTO categories (name) VALUES ('Electronics'); -- succeeds
INSERT INTO categories (name) VALUES ('Electronics'); -- fails
-- ERROR 1062: Duplicate entry 'Electronics' for key 'uq_category_name'
-- Constraint fired: UNIQUE (uq_category_name)
-- Reason: name column has a UNIQUE constraint; 'Electronics' already exists.
```

### b) Invalid category_id (foreign key violation)

```sql
INSERT INTO products (category_id, name, price, stock)
VALUES (999, 'iPhone 15', 79999.00, 10);
-- ERROR 1452: Cannot add or update a child row: a foreign key constraint fails
-- Constraint fired: FOREIGN KEY fk_products_category ON DELETE RESTRICT
-- Reason: category_id 999 does not exist in the categories table.
```

### c) Negative price (CHECK violation)

```sql
INSERT INTO products (category_id, name, price, stock)
VALUES (1, 'iPhone 15', -500.00, 10);
-- ERROR 3819: Check constraint 'chk_product_price' is violated
-- Constraint fired: CHECK (chk_product_price)
-- Reason: price must be > 0; -500.00 violates this rule.
```

### d) Rating out of range (CHECK violation)

```sql
INSERT INTO reviews (customer_id, product_id, rating)
VALUES (1, 1, 7);
-- ERROR 3819: Check constraint 'chk_rating_range' is violated
-- Constraint fired: CHECK (chk_rating_range)
-- Reason: rating must be BETWEEN 1 AND 5; 7 exceeds the allowed range.
```

---

## TASK 3 — ALTER TABLE Practice

### a) Add brand column to products

```sql
ALTER TABLE products
    ADD COLUMN brand VARCHAR(100) NULL;
```

### b) Add coupon_code column to orders

```sql
ALTER TABLE orders
    ADD COLUMN coupon_code CHAR(10) NULL;
```

---

## TASK 4 — Payments Table

**Logical Schema:**

- `payment_id` — unique identifier for each payment
- `order_id` — which order this payment belongs to (mandatory, one order per payment)
- `amount` — monetary value paid (must be positive)
- `payment_method` — fixed set: card, upi, netbanking, cod
- `payment_status` — fixed set: pending, completed, failed, refunded
- `transaction_ref` — unique reference number from payment gateway (optional until payment is processed)
- `created_at` — timestamp of when payment record was created

**Physical Schema:**

```sql
CREATE TABLE payments (
    payment_id        INT UNSIGNED        NOT NULL AUTO_INCREMENT,
    order_id          INT UNSIGNED        NOT NULL,
    amount            DECIMAL(10,2)       NOT NULL,
    payment_method    ENUM('card','upi','netbanking','cod')
                                          NOT NULL,
    payment_status    ENUM('pending','completed','failed','refunded')
                                          NOT NULL DEFAULT 'pending',
    transaction_ref   VARCHAR(100)        NULL,
    created_at        TIMESTAMP           NOT NULL DEFAULT CURRENT_TIMESTAMP,

    PRIMARY KEY (payment_id),
    CONSTRAINT uq_transaction_ref
        UNIQUE (transaction_ref),
    CONSTRAINT fk_payments_order
        FOREIGN KEY (order_id) REFERENCES orders(order_id)
        ON DELETE RESTRICT
        ON UPDATE CASCADE,
    CONSTRAINT chk_payment_amount
        CHECK (amount > 0)

) ENGINE = InnoDB DEFAULT CHARSET = utf8mb4 COLLATE = utf8mb4_unicode_ci;
```

---

## TASK 5 — Reflection

`order_items` uses `ON DELETE CASCADE` for its `orders` foreign key because an order item has no independent meaning without its parent order — if an order is deleted, all its line items should be deleted automatically as a single logical unit.

`order_items` uses `ON DELETE RESTRICT` for its `products` foreign key because a product being discontinued should not silently erase purchase history. The restriction forces the business to explicitly archive or soft-delete the product rather than accidentally destroying historical order records that reference it.