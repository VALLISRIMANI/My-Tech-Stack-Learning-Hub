📍 **Module 2 → Concept 1 of 3: ER Diagrams — Entities, Attributes & Relationships**

---

## 📖 THEORY — What is it and why does it exist?

### The Problem: Why plan before writing SQL?

Imagine ShopNest's lead developer jumps straight into writing `CREATE TABLE` statements without thinking through the design. Three months later:

- The `orders` table stores customer names directly → when a customer updates their name, 10,000 order rows are wrong
- There's no clean way to store multiple items per order → everything is crammed into one column as comma-separated text
- Adding a "product categories" feature requires rewriting half the database

**This is why we design first.** An **Entity-Relationship (ER) Diagram** is a visual blueprint of your database — drawn on paper or a whiteboard *before* a single line of SQL is written. It's the architect's floor plan before the construction crew arrives.

---

### The Three Building Blocks of ER Diagrams

---

### ① Entities 📦

An **entity** is a real-world *thing* that you need to store data about. In ER diagrams, entities become **tables** in your database.

**Rule of thumb:** If you'd want a list of them, it's probably an entity.

For ShopNest:
```
Things we need lists of:
  → Customers    (we want a list of all customers)
  → Products     (we want a list of all products)
  → Orders       (we want a list of all orders)
  → Categories   (Electronics, Clothing, Home Goods...)
  → Payments     (we want a record of every payment)

These become our entities → our tables.
```

**Weak Entity:** An entity that *cannot exist* without another entity. For ShopNest, an `order_item` (a single line in an order — "2x iPhone 15") cannot exist without a parent `order`. It has no independent identity. Weak entities depend on a **strong entity** for their existence.

---

### ② Attributes 🏷️

An **attribute** is a piece of information *about* an entity. Attributes become **columns** in your table.

```
Entity: Customer
Attributes:
  → customer_id   (unique identifier — this is the KEY attribute)
  → first_name
  → last_name
  → email
  → phone
  → city
  → created_at

Entity: Product
Attributes:
  → product_id    (key attribute)
  → name
  → description
  → price
  → stock_quantity
  → category_id
```

**Types of attributes you should know:**

| Type | Description | ShopNest Example |
|---|---|---|
| **Simple** | Single, indivisible value | `price`, `email` |
| **Composite** | Can be split into sub-parts | `full_name` → `first_name` + `last_name` |
| **Derived** | Calculated from other attributes | `age` derived from `date_of_birth` |
| **Multi-valued** | Can have multiple values | `phone_numbers` (a customer has 2 phones) |
| **Key** | Uniquely identifies the entity | `customer_id`, `product_id` |

**Design rule:** Always split composite attributes. Store `first_name` and `last_name` separately — not `full_name`. This lets you query `WHERE last_name = 'Sharma'` efficiently and sort by last name.

---

### ③ Relationships 🔗

A **relationship** defines how two entities are connected to each other. This is the heart of *relational* databases.

There are three types, defined by **cardinality** (how many of one entity relates to how many of another):

---

**One-to-One (1:1)**
One record in Table A relates to exactly one record in Table B.

```
Customer ──────── CustomerProfile
  (1)                   (1)

One customer has exactly one profile page.
One profile belongs to exactly one customer.

Real ShopNest use: A customer has one wallet.
One wallet belongs to one customer.
```

1:1 relationships are relatively rare. Often the two entities can just be merged into one table — unless one side is optional or the data is sensitive (e.g., keep payment details in a separate table for security).

---

**One-to-Many (1:N)**
One record in Table A relates to MANY records in Table B. But each record in B relates to only ONE record in A.

```
Customer ──────── Orders
  (1)               (N)

One customer can place MANY orders.
But each order belongs to exactly ONE customer.

Product ──────── OrderItems
  (1)               (N)

One product can appear in MANY order items.
But each order item refers to exactly ONE product.
```

1:N is the **most common relationship** in relational databases. It's implemented by placing the **primary key of the "one" side** as a **foreign key on the "many" side**.

```
customers table:          orders table:
┌─────────────┐          ┌──────────────────────┐
│ customer_id │◄─────────│ customer_id (FK)      │
│ first_name  │    1:N   │ order_id              │
│ email       │          │ total_amount          │
└─────────────┘          └──────────────────────┘
```

---

**Many-to-Many (N:M)**
Many records in Table A relate to many records in Table B.

```
Products ──────── Orders
   (N)               (M)

One order can contain MANY products.
One product can appear in MANY orders.
```

N:M relationships **cannot be directly implemented** in a relational database. You must break them apart using a **junction table** (also called a bridge table or associative table):

```
products          order_items (junction)      orders
┌──────────┐     ┌────────────────────────┐  ┌──────────┐
│product_id│◄────│product_id (FK)         │  │order_id  │
│name      │     │order_id   (FK)─────────├─►│customer_id│
│price     │     │quantity               │  │total     │
└──────────┘     │unit_price             │  └──────────┘
                 └────────────────────────┘

The junction table has TWO foreign keys — one to each side.
Together they form the primary key of the junction table.
```

---

## 🔤 SYNTAX — ER Diagram Notation (Crow's Foot)

The most common ER notation used in tools like MySQL Workbench is **Crow's Foot**:

```
Symbols at the end of relationship lines:

  │    →  exactly one (mandatory)
  O    →  zero (optional)
  <    →  many (the "crow's foot")

Examples:
  Customer ──│────<── Orders
             1        N
  "One customer has many orders"
  "Each order belongs to exactly one customer"

  Order ──│────<── OrderItems ──>────│── Product
          1            N           N          1
  "One order has many items, each item has one product"
  "One product appears in many items"
```

---

## 💡 EXAMPLE — ShopNest Complete ER Design

Here's the full ER design we'll implement throughout the course:

```
┌─────────────────┐         ┌──────────────────┐
│   CUSTOMERS     │         │    CATEGORIES    │
├─────────────────┤         ├──────────────────┤
│ customer_id  PK │         │ category_id   PK │
│ first_name      │         │ name             │
│ last_name       │         │ description      │
│ email           │         └────────┬─────────┘
│ phone           │                  │ 1
│ city            │                  │
│ created_at      │                  │ N
└────────┬────────┘         ┌────────▼─────────┐
         │ 1                │    PRODUCTS      │
         │                  ├──────────────────┤
         │ N                │ product_id    PK │
┌────────▼────────┐         │ category_id   FK │
│    ORDERS       │         │ name             │
├─────────────────┤         │ price            │
│ order_id     PK │         │ stock_quantity   │
│ customer_id  FK │         └────────┬─────────┘
│ order_date      │                  │ N
│ status          │                  │
│ total_amount    │                  │
└────────┬────────┘         ┌────────▼─────────┐
         │ 1                │   ORDER_ITEMS    │
         │                  ├──────────────────┤
         └──────────────────│ order_item_id PK │
                   1:N      │ order_id      FK │
                            │ product_id    FK │
                            │ quantity         │
                            │ unit_price       │
                            └──────────────────┘

Relationships:
  customers → orders      : 1:N  (one customer, many orders)
  orders → order_items    : 1:N  (one order, many line items)
  products → order_items  : 1:N  (one product, many line items)
  categories → products   : 1:N  (one category, many products)
  orders ↔ products       : N:M  (resolved via order_items)
```

---

## ⚠️ GOTCHAS — Design Mistakes Beginners Make

**1. Storing multiple values in one column**
```
❌ BAD — order_items stored as text in orders table:
orders: { order_id: 1, items: "iPhone15:2, AirPods:1, Case:3" }
→ You can never query "how many iPhones were sold total"

✅ GOOD — separate order_items table with one row per item
```

**2. Skipping the junction table for N:M**
```
❌ BAD:
orders: { order_id: 1, product_ids: "101,102,103" }
→ Comma-separated IDs in a column = a maintenance nightmare

✅ GOOD: order_items junction table with one row per
   product per order
```

**3. Naming confusion — don't name your junction table randomly**
Name junction tables after what they *represent*:
`order_items` (not `orders_products`), `course_enrollments` (not `students_courses`)

**4. Confusing attributes with entities**
`city` is an *attribute* of a customer — not its own entity (unless ShopNest needs to store city-specific data like delivery zones, in which case `cities` becomes an entity).

**5. Over-engineering 1:1 relationships**
If a customer always has a profile and a profile always has a customer, just merge them into one `customers` table. Only split 1:1 when one side is optional, sensitive, or accessed much less frequently.

---

✅ **Quick Check:**
ShopNest wants to add a **reviews** feature — customers can write reviews for products. One customer can review many products, and one product can have many reviews from different customers. What type of relationship is this? How would you resolve it in a relational database — what would the junction/bridge table look like, and what columns would it need?
