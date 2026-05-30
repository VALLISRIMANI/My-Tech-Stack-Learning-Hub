📍 **Module 1 → Concept 2 of 5: ACID Properties**

---

## 📖 THEORY — What is it and why does it exist?

### The Problem: What happens when things go wrong mid-operation?

Imagine ShopNest's checkout process. A customer pays ₹79,999 for an iPhone 15. Here's what happens behind the scenes in the database:

```
Step 1: Deduct ₹79,999 from customer's wallet
Step 2: Create a new row in the orders table
Step 3: Reduce iPhone 15 stock count by 1
Step 4: Create a payment record
```

Now imagine the **server crashes after Step 1 but before Step 2**. The customer's money is gone but no order exists. Or two customers simultaneously buy the **last iPhone in stock** — do both orders go through? These are real, catastrophic bugs that destroyed early systems.

**ACID** is the set of four guarantees that MySQL (via InnoDB engine) makes to ensure database operations are **safe, reliable, and correct** — even when hardware fails, code crashes, or multiple users hit the database simultaneously.

ACID stands for: **Atomicity, Consistency, Isolation, Durability**

---

### **A — Atomicity** ⚛️
*"All or nothing."*

A **transaction** is a group of SQL operations that must succeed or fail **together as one unit**. There is no partial success.

**Real-world analogy:** A UPI transfer. Either ₹500 leaves your account AND arrives in the recipient's account — or neither happens. The bank doesn't let money vanish in between.

For ShopNest: all 4 steps of checkout either complete fully, or if anything fails, the entire thing rolls back as if it never happened. The customer's wallet is restored. No phantom orders.

---

### **C — Consistency** 🔒
*"The database must always move from one valid state to another valid state."*

Consistency means every transaction must respect all the rules you've defined — data types, constraints, foreign keys, business rules. You cannot end up with corrupted or nonsensical data.

**Real-world analogy:** Accounting's golden rule — debits must always equal credits. Every transaction must preserve this balance. If a transaction would break it, it's rejected entirely.

For ShopNest: if you have a rule that `stock_quantity` can never go below 0, then even if 500 people simultaneously try to buy the last item, the database will never show stock = -499. Consistency enforces that rule without exception.

---

### **I — Isolation** 🧱
*"Concurrent transactions must not interfere with each other."*

When multiple users hit ShopNest simultaneously, each transaction behaves as if it's the **only transaction running**. Intermediate states of a transaction are invisible to others until it completes.

**Real-world analogy:** Two bank tellers processing transactions on the same account simultaneously. Each teller sees a consistent snapshot — they don't see each other's half-finished work. The final result is as if the transactions ran one after the other.

For ShopNest: User A is in the middle of placing an order (transaction not yet committed). User B queries order history. User B does **not** see User A's incomplete order. Isolation prevents "dirty reads" of uncommitted data.

*(We'll cover Isolation Levels in depth in Module 9 — this is the most nuanced of the four.)*

---

### **D — Durability** 💾
*"Once committed, data survives forever — even if the server crashes immediately after."*

When MySQL says "COMMIT successful," that data is written to disk permanently. A power outage, server crash, or OS failure immediately after will not lose that data.

**Real-world analogy:** Once a cheque is cleared and you get the SMS confirmation, the bank can't say "sorry, we lost it." The record is permanent.

For ShopNest: Once the customer's order is confirmed (committed), even if the server catches fire one second later, when it restarts, that order still exists. MySQL achieves this via a **write-ahead log (WAL)** — changes are written to a log file on disk before being applied to the main data files.

---

## 🔤 SYNTAX — Transactions in MySQL

ACID properties are exercised through **transactions**. Here's the syntax (full deep dive in Module 9 — for now, understand the structure):

```sql
-- Begin a transaction (group of operations)
START TRANSACTION;

    -- Step 1: Deduct from wallet
    UPDATE wallets SET balance = balance - 79999 WHERE customer_id = 1;

    -- Step 2: Create the order
    INSERT INTO orders (customer_id, product_id, amount)
    VALUES (1, 101, 79999);

    -- Step 3: Reduce stock
    UPDATE products SET stock = stock - 1 WHERE product_id = 101;

-- If ALL steps succeeded → make it permanent
COMMIT;

-- If ANYTHING went wrong → undo ALL steps
ROLLBACK;
```

| Keyword | What it does |
|---|---|
| `START TRANSACTION` | Opens a transaction block — changes are now temporary |
| `COMMIT` | Makes all changes in the block permanent (Durability kicks in) |
| `ROLLBACK` | Undoes every change made since `START TRANSACTION` (Atomicity kicks in) |

---

## 💡 EXAMPLE — ShopNest Checkout Gone Wrong (and ACID saving the day)

**Scenario:** Rahul buys an iPhone 15 (product_id = 101, price = ₹79,999). Server crashes after the wallet deduction.

```sql
-- WITHOUT a transaction (dangerous — no ACID protection):
UPDATE wallets SET balance = balance - 79999 WHERE customer_id = 1;
-- 💥 SERVER CRASHES HERE
INSERT INTO orders (customer_id, product_id, amount) VALUES (1, 101, 79999);
-- This line never runs. Rahul lost ₹79,999 with no order. 😱


-- WITH a transaction (ACID-protected):
START TRANSACTION;

    UPDATE wallets SET balance = balance - 79999 WHERE customer_id = 1;
    -- 💥 SERVER CRASHES HERE

    INSERT INTO orders (customer_id, product_id, amount) VALUES (1, 101, 79999);

COMMIT;
-- Since COMMIT was never reached, MySQL automatically rolls back
-- on restart. Rahul's ₹79,999 is restored. ✅
```

**The Consistency example — stock going negative:**
```sql
-- Suppose stock for iPhone 15 = 1
-- Two users simultaneously try to buy it

-- MySQL's Isolation ensures only ONE succeeds.
-- Consistency ensures stock never becomes -1.
-- The second transaction is either queued or rejected
-- based on isolation level. Stock stays ≥ 0. ✅
```

---

## ⚠️ GOTCHAS — What Beginners Get Wrong

**1. "MySQL is always ACID compliant"** — Only with the **InnoDB storage engine**. The older **MyISAM engine does NOT support transactions**, meaning no Atomicity, no Isolation, no Durability guarantees. We'll cover this in the next concept. Always use InnoDB for any table where data integrity matters.

**2. "COMMIT means saved to RAM"** — No. COMMIT means written to **disk**, permanently. This is the whole point of Durability.

**3. "Isolation means transactions are completely invisible to each other"** — It depends on the **isolation level** configured. MySQL has 4 isolation levels offering different trade-offs between strictness and performance. Default in MySQL is `REPEATABLE READ`. (Module 9 will cover all 4.)

**4. Forgetting ROLLBACK in error handling** — Many beginners wrap code in transactions but forget to ROLLBACK when an error occurs. This leaves the transaction open, locking rows and blocking other users — a common cause of production outages.

**5. "Consistency is enforced automatically"** — MySQL enforces constraints you *define* (like NOT NULL, FOREIGN KEY). But if you define no rules, MySQL has nothing to enforce. Consistency = your rules + MySQL's enforcement. Bad schema design = no consistency protection.

---

✅ **Quick Check:**
ShopNest runs this sequence **without** a transaction: it inserts a new order row, then tries to deduct from the customer's wallet — but the wallet update fails due to insufficient funds. Which ACID property is being violated here, and what should the developer have done to prevent this situation?
