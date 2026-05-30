📍 **Module 1 → Concept 5 of 5: User Management — CREATE USER, GRANT, REVOKE, ALTER root password**

---

## 📖 THEORY — What is it and why does it exist?

### The Problem: Why not just use root for everything?

The `root` user is MySQL's **superuser** — it can do anything: drop databases, delete all data, create users, shut down the server. Using root for your application is like giving every employee at ShopNest the master key to the entire building, the safe, and the server room.

**What actually goes wrong:**
- Your Node.js/Python app connects as root → a SQL injection attack now has full server control
- A junior developer accidentally runs `DROP DATABASE shopnest;` — with root, it just works. Gone.
- You want your analytics team to only *read* data, not modify it — root can't be restricted

MySQL's solution is **granular user management**:
- Create separate users for separate purposes
- Grant only the minimum permissions each user needs
- Revoke permissions when they're no longer needed

This principle is called **Principle of Least Privilege** — one of the most important security concepts in software engineering.

---

### How MySQL Identifies Users

MySQL identifies users by **two things together**: username AND the host they connect from.

```
'rahul'@'localhost'   → user "rahul" connecting from the same machine
'rahul'@'192.168.1.5' → user "rahul" connecting from IP 192.168.1.5
'rahul'@'%'           → user "rahul" connecting from ANY host (wildcard)
```

`'rahul'@'localhost'` and `'rahul'@'%'` are treated as **completely different users** by MySQL — even though the username is the same. This is a critical concept beginners always miss.

---

## 🔤 SYNTAX — All User Management Commands

### ① CREATE USER
```sql
CREATE USER 'username'@'host' IDENTIFIED BY 'password';

-- Examples:
-- App user connecting from same machine
CREATE USER 'shopnest_app'@'localhost' IDENTIFIED BY 'App@Secure123';

-- Analytics user connecting from any host
CREATE USER 'shopnest_analyst'@'%' IDENTIFIED BY 'Read@Only456';

-- Admin connecting only from specific IP
CREATE USER 'shopnest_admin'@'192.168.1.10' IDENTIFIED BY 'Admin@789';
```

---

### ② GRANT — Give permissions
```sql
GRANT privilege(s) ON database.table TO 'username'@'host';

-- Grant ALL privileges on shopnest database (admin-level)
GRANT ALL PRIVILEGES ON shopnest.* TO 'shopnest_admin'@'192.168.1.10';

-- Grant only SELECT on all tables (read-only analyst)
GRANT SELECT ON shopnest.* TO 'shopnest_analyst'@'%';

-- Grant SELECT, INSERT, UPDATE on specific table only (app user)
GRANT SELECT, INSERT, UPDATE ON shopnest.orders TO 'shopnest_app'@'localhost';

-- Grant everything on everything (avoid — dangerous)
GRANT ALL PRIVILEGES ON *.* TO 'superuser'@'localhost';

-- After granting, reload privilege tables immediately
FLUSH PRIVILEGES;
```

**Privilege levels — from broadest to narrowest:**
```
*.*           → all databases, all tables (global)
shopnest.*    → all tables within shopnest database
shopnest.orders → only the orders table in shopnest
```

**Common privileges:**

| Privilege | What it allows |
|---|---|
| `SELECT` | Read rows from tables |
| `INSERT` | Add new rows |
| `UPDATE` | Modify existing rows |
| `DELETE` | Remove rows |
| `CREATE` | Create new databases/tables |
| `DROP` | Delete databases/tables |
| `ALTER` | Modify table structure |
| `INDEX` | Create/drop indexes |
| `ALL PRIVILEGES` | Everything above |

---

### ③ REVOKE — Remove permissions
```sql
REVOKE privilege(s) ON database.table FROM 'username'@'host';

-- Remove DELETE permission from app user
REVOKE DELETE ON shopnest.* FROM 'shopnest_app'@'localhost';

-- Remove ALL privileges
REVOKE ALL PRIVILEGES ON shopnest.* FROM 'shopnest_analyst'@'%';

-- Always flush after revoking
FLUSH PRIVILEGES;
```

---

### ④ ALTER USER — Change password
```sql
-- Change password for any user (run as root)
ALTER USER 'shopnest_app'@'localhost' IDENTIFIED BY 'NewPassword@123';

-- Change YOUR OWN password (while logged in as that user)
ALTER USER USER() IDENTIFIED BY 'NewPassword@123';

-- Change root password
ALTER USER 'root'@'localhost' IDENTIFIED BY 'NewRoot@Secure999';

-- Apply changes
FLUSH PRIVILEGES;
```

---

### ⑤ Inspection Commands — See what exists
```sql
-- List all users on the server
SELECT user, host FROM mysql.user;

-- See all privileges granted to a specific user
SHOW GRANTS FOR 'shopnest_app'@'localhost';

-- See privileges for currently logged-in user
SHOW GRANTS FOR CURRENT_USER();

-- See currently logged-in user
SELECT CURRENT_USER();
```

---

### ⑥ DROP USER — Delete a user entirely
```sql
DROP USER 'shopnest_analyst'@'%';
-- Removes the user AND all their privileges simultaneously
```

---

## 💡 EXAMPLE — Setting Up ShopNest's Complete User Structure

Here's a realistic, production-style user setup for ShopNest:

```sql
-- ============================================
-- ShopNest User Management Setup
-- Run as root
-- ============================================

-- 1. The application backend (Node.js/Python app)
--    Needs to read/write orders, customers, products
--    Connects from localhost only
--    Should NOT be able to drop tables or create users
CREATE USER 'shopnest_app'@'localhost' IDENTIFIED BY 'App@Secure123';
GRANT SELECT, INSERT, UPDATE, DELETE ON shopnest.* TO 'shopnest_app'@'localhost';

-- 2. The analytics/reporting team
--    Only needs to read data for dashboards
--    Connects from their office IP range
CREATE USER 'shopnest_analyst'@'%' IDENTIFIED BY 'Read@Only456';
GRANT SELECT ON shopnest.* TO 'shopnest_analyst'@'%';

-- 3. The DBA (database administrator)
--    Full control over shopnest database
--    Restricted to internal network IP only
CREATE USER 'shopnest_dba'@'192.168.1.10' IDENTIFIED BY 'Dba@Ultra789';
GRANT ALL PRIVILEGES ON shopnest.* TO 'shopnest_dba'@'192.168.1.10';

-- Apply all privilege changes
FLUSH PRIVILEGES;

-- ============================================
-- Verify the setup
-- ============================================
SELECT user, host FROM mysql.user;

SHOW GRANTS FOR 'shopnest_app'@'localhost';
-- Output:
-- GRANT SELECT, INSERT, UPDATE, DELETE ON `shopnest`.* TO `shopnest_app`@`localhost`

SHOW GRANTS FOR 'shopnest_analyst'@'%';
-- Output:
-- GRANT SELECT ON `shopnest`.* TO `shopnest_analyst`@`%`

-- ============================================
-- Later: analyst gets promoted to app developer
-- ============================================
GRANT INSERT, UPDATE, DELETE ON shopnest.* TO 'shopnest_analyst'@'%';
FLUSH PRIVILEGES;

-- ============================================
-- Analyst leaves the company
-- ============================================
DROP USER 'shopnest_analyst'@'%';
FLUSH PRIVILEGES;
```

---

## ⚠️ GOTCHAS — Critical Mistakes Beginners Make

**1. Forgetting the host part**
```sql
-- This creates 'app'@'%' — connects from ANYWHERE
CREATE USER 'app' IDENTIFIED BY 'pass';

-- Always be explicit about the host:
CREATE USER 'app'@'localhost' IDENTIFIED BY 'pass';
```

**2. GRANT doesn't automatically create the user**
```sql
-- This FAILS if 'newuser'@'localhost' doesn't exist yet
GRANT SELECT ON shopnest.* TO 'newuser'@'localhost';
-- Error: There is no such grant defined for user 'newuser' on host 'localhost'

-- Always CREATE USER first, then GRANT
```

**3. Forgetting FLUSH PRIVILEGES**
In older MySQL versions, privilege changes don't take effect until you run `FLUSH PRIVILEGES`. In MySQL 8+, `GRANT` and `REVOKE` flush automatically — but it's still good practice to always include it. Skipping it has caused hours of debugging ("why isn't my new user working?!").

**4. Using `'%'` for root**
```sql
-- NEVER do this — root accessible from anywhere on the internet
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%';
-- This is how databases get hacked and ransomed
```

**5. Granting ALL PRIVILEGES to the app user**
Your application almost never needs `CREATE`, `DROP`, or `ALTER` at runtime. Those are schema changes done by migrations. App users should only have `SELECT, INSERT, UPDATE, DELETE` at most.

**6. Passwords in plain sight**
```sql
-- Anyone watching your screen or logs sees this
CREATE USER 'app'@'localhost' IDENTIFIED BY 'MyPassword123';
-- In production, use environment variables and never hardcode passwords
```

✅ **Module 1 Complete!**