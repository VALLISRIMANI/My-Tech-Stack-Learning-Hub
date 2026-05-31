## 🏆 Module 2 Mini Project: Design ShopNest's Reviews Feature

Using everything from this module, complete the following:

```
TASK 1 — Logical Design
  ShopNest wants to add product reviews.
  Define the logical schema:
  - What entities are involved?
  - What attributes does each have?
  - What are the relationships and their cardinalities?
  - What business rules apply?
    (e.g. one review per customer per product,
     rating must be 1-5, review text is optional)

TASK 2 — Physical Design
  Translate your logical schema into a CREATE TABLE statement.
  Choose appropriate:
  - Data types for each column
  - NULL / NOT NULL constraints
  - DEFAULT values where sensible
  - PRIMARY KEY and FOREIGN KEYS
  - UNIQUE constraints where business rules require
  - ENUM where values are fixed
  - ENGINE = InnoDB

TASK 3 — Normalization Check
  Look at your reviews table.
  Is it in 3NF? Identify any dependencies and confirm
  no partial or transitive dependencies exist.

TASK 4 — Reflection
  Why is unit_price stored in order_items rather than
  looked up from products.price at query time?
  What real-world problem does this solve?
```

---


## TASK 1 — Logical Design

**Entities involved:**
- `CUSTOMER` — the person writing the review
- `PRODUCT` — the item being reviewed
- `REVIEW` — the review itself (this is the junction/associative entity)

**Attributes:**

| Entity | Attributes |
|---|---|
| CUSTOMER | customer_id (PK), first_name, last_name, email, ... |
| PRODUCT | product_id (PK), name, price, ... |
| REVIEW | review_id (PK), customer_id (FK), product_id (FK), rating, review_text, created_at |

**Relationships & Cardinalities:**
- `CUSTOMER` → `REVIEW` : **1:N** — one customer can write many reviews
- `PRODUCT` → `REVIEW` : **1:N** — one product can receive many reviews
- `CUSTOMER` ↔ `PRODUCT` : **N:M** — resolved through the `REVIEW` junction entity

**Business Rules:**
- A customer can review **many products**, but only **one review per product** (enforced via UNIQUE on `customer_id + product_id`)
- `rating` must be between 1 and 5 (inclusive)
- `review_text` is optional (nullable)
- A review cannot exist without both a valid customer and a valid product
- `created_at` is automatically recorded at time of submission

---

## TASK 2 — Physical Design
```sql
CREATE TABLE product_reviews (
    review_id   INT           UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    customer_id INT           UNSIGNED NOT NULL,
    product_id  INT           UNSIGNED NOT NULL,
    rating      TINYINT       UNSIGNED NOT NULL,
    review_text VARCHAR(1000) NULL,
    created_at  DATETIME      NOT NULL DEFAULT CURRENT_TIMESTAMP,

    -- One review per customer per product
    UNIQUE KEY uq_customer_product (customer_id, product_id),

    -- Rating must be between 1 and 5
    CONSTRAINT chk_rating CHECK (rating >= 1 AND rating <= 5),

    -- Foreign key constraints
    CONSTRAINT fk_review_customer FOREIGN KEY (customer_id)
        REFERENCES customers(customer_id),
    CONSTRAINT fk_review_product  FOREIGN KEY (product_id)
        REFERENCES products(product_id)

) ENGINE = InnoDB;
```

---

## TASK 3 — Normalization Check

**Is `product_reviews` in 3NF? Yes. Here is the proof:**

**Primary Key:** `review_id` (single-column surrogate key)

**Functional Dependencies:**
```
review_id → customer_id     ✅ Direct dependency on PK
review_id → product_id      ✅ Direct dependency on PK
review_id → rating          ✅ Direct dependency on PK
review_id → review_text     ✅ Direct dependency on PK
review_id → created_at      ✅ Direct dependency on PK
```

**1NF check:** ✅
- Every cell is atomic (one value per cell)
- No repeating groups or arrays
- Every row uniquely identified by `review_id`

**2NF check:** ✅
- Primary key is a single column (`review_id`)
- Partial dependencies are impossible with a single-column PK
- Table is automatically in 2NF

**3NF check:** ✅
- No non-key column depends on another non-key column
- `rating` does not determine `review_text` (or vice versa)
- `customer_id` does not determine `product_id`
- No transitive dependencies exist

**Conclusion:** `product_reviews` is in 3NF (and BCNF) as designed.

---

## TASK 4 — Reflection

`unit_price` is stored in `order_items` rather than looked up from `products.price`
because **product prices change over time**, and order records must reflect
the **price at the exact moment of purchase** — not the current price.

**What goes wrong without it:**

```
Timeline:
  Jan 2024 — Customer buys iPhone 15 at ₹79,999  → order recorded
  Mar 2024 — ShopNest raises price to ₹85,999    → products.price updated
  Apr 2024 — Customer requests invoice for Jan order
           → query reads products.price = ₹85,999
           → invoice shows wrong amount ❌
           → financial records are incorrect
           → GST calculations are wrong
           → customer disputes the charge
```

**With `unit_price` in `order_items`:**

```
Jan 2024 — order_items.unit_price = 79999.00  → saved permanently ✅
Mar 2024 — products.price updated to 85999.00
Apr 2024 — Jan invoice still reads 79999.00   → correct forever ✅
```

This is **intentional, justified denormalization** — a deliberate deviation
from strict normalization to preserve a historical fact. The rule is:

> Normalize to eliminate redundancy caused by design flaws.
> Denormalize deliberately to preserve facts that change over time.

Other examples of the same pattern in ShopNest:
- `order_items.unit_price` — price at time of purchase
- Storing customer address on an order — shipping address at time of order, independent of any later address change