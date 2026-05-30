## 🏆 Module 1 Mini Project: ShopNest Server Setup

Complete all of the following tasks in order. Each one tests a concept from this module.

```
TASK 1 — Environment Check
Run SHOW VARIABLES LIKE 'default_storage_engine';
Confirm it returns InnoDB. If not, note what it returns.

TASK 2 — Database Creation
Create the ShopNest database:
  CREATE DATABASE shopnest;
  USE shopnest;

TASK 3 — User Setup
Create three users as follows:
  a) 'sn_app'@'localhost'    → password: 'App@123'
     Permissions: SELECT, INSERT, UPDATE, DELETE on shopnest.*

  b) 'sn_readonly'@'%'       → password: 'Read@123'
     Permissions: SELECT only on shopnest.*

  c) 'sn_dba'@'localhost'    → password: 'Dba@123'
     Permissions: ALL PRIVILEGES on shopnest.*

TASK 4 — Verification
Run SHOW GRANTS for each of the three users
and confirm each shows exactly the right permissions.

TASK 5 — Revoke & Cleanup
The readonly analyst no longer needs access.
Revoke all their privileges, then drop the user entirely.
Verify they no longer appear in SELECT user, host FROM mysql.user;

TASK 6 — Reflection Question
Why did we NOT grant ALL PRIVILEGES to 'sn_app'@'localhost'
even though it's the main application user?
```

---


## TASK 1 — Environment Check

```sql
SHOW VARIABLES LIKE 'default_storage_engine';
```

**Output:**
```
+------------------------+--------+
| Variable_name          | Value  |
+------------------------+--------+
| default_storage_engine | InnoDB |
+------------------------+--------+
```
✅ Confirmed: InnoDB is the default storage engine.
InnoDB provides full ACID compliance, row-level locking,
foreign key enforcement, and automatic crash recovery.

---

## TASK 2 — Database Creation

```sql
CREATE DATABASE shopnest;
USE shopnest;
```

✅ ShopNest database created and selected as the active database.
All subsequent tables and users will be scoped to this database.

---

## TASK 3 — User Setup

### a) Application User
```sql
CREATE USER 'sn_app'@'localhost' IDENTIFIED BY 'App@123';
GRANT SELECT, INSERT, UPDATE, DELETE ON shopnest.* TO 'sn_app'@'localhost';
```
- **Purpose:** Main backend application (Node.js / Python)
- **Host restriction:** localhost only — cannot connect remotely
- **Permissions:** Read and write rows; cannot alter structure

### b) Read-Only Analyst User
```sql
CREATE USER 'sn_readonly'@'%' IDENTIFIED BY 'Read@123';
GRANT SELECT ON shopnest.* TO 'sn_readonly'@'%';
```
- **Purpose:** Analytics and reporting team
- **Host restriction:** Any host (office/remote access)
- **Permissions:** Read only — cannot modify any data

### c) DBA User
```sql
CREATE USER 'sn_dba'@'localhost' IDENTIFIED BY 'Dba@123';
GRANT ALL PRIVILEGES ON shopnest.* TO 'sn_dba'@'localhost';
```
- **Purpose:** Database administrator — schema changes, migrations
- **Host restriction:** localhost only — never exposed remotely
- **Permissions:** Full control over shopnest database

```sql
FLUSH PRIVILEGES;
```
✅ All privilege changes applied immediately.

---

## TASK 4 — Verification

```sql
SHOW GRANTS FOR 'sn_app'@'localhost';
```
```
GRANT SELECT, INSERT, UPDATE, DELETE ON `shopnest`.* TO `sn_app`@`localhost`
```

```sql
SHOW GRANTS FOR 'sn_readonly'@'%';
```
```
GRANT SELECT ON `shopnest`.* TO `sn_readonly`@`%`
```

```sql
SHOW GRANTS FOR 'sn_dba'@'localhost';
```
```
GRANT ALL PRIVILEGES ON `shopnest`.* TO `sn_dba`@`localhost`
```

✅ All three users have exactly the correct permissions confirmed.

---

## TASK 5 — Revoke & Cleanup

```sql
-- DROP USER automatically removes all privileges.
-- No need to REVOKE separately.
DROP USER 'sn_readonly'@'%';

-- Verify removal
SELECT user, host FROM mysql.user;
```

✅ sn_readonly no longer appears in the user list.

> **Note:** `DROP USER` removes the user AND all their privileges
> in a single command. Running `REVOKE` before `DROP USER` is
> redundant — though not harmful.

---

## TASK 6 — Reflection: Why Not ALL PRIVILEGES for sn_app?

We do **not** grant `ALL PRIVILEGES` to `sn_app` because of the
**Principle of Least Privilege**: a user should have access to
*only* what it needs to do its job, and nothing more.

### Three specific reasons:

**① Security against SQL Injection**
`sn_app` credentials are embedded in application code.
If an attacker exploits a SQL injection vulnerability,
they inherit sn_app's permissions exactly.

- With `SELECT, INSERT, UPDATE, DELETE` → attacker can
  manipulate data. Bad, but recoverable.
- With `ALL PRIVILEGES` → attacker can run
  `DROP DATABASE shopnest`. Everything gone instantly.

**② Accident Prevention**
Application code should never need `DROP TABLE`, `ALTER TABLE`,
or `CREATE USER` at runtime. Those are deliberate DBA operations.
Restricting sn_app means a bug in application code literally
*cannot* accidentally destroy the database structure.

**③ Separation of Concerns**
| User | Role | Permissions |
|------|------|-------------|
| sn_app | Application backend | SELECT, INSERT, UPDATE, DELETE |
| sn_dba | Database administrator | ALL PRIVILEGES |
| sn_readonly | Analytics / reporting | SELECT only |

Mixing these roles into one all-powerful user removes
accountability, auditability, and all safety boundaries.

---

## Module 1 Summary — Concepts Mastered

| # | Concept | Status |
|---|---------|--------|
| 1 | Evolution of Data Storage, DBMS vs RDBMS vs NoSQL | ✅ |
| 2 | ACID Properties (Atomicity, Consistency, Isolation, Durability) | ✅ |
| 3 | MySQL Architecture: Client/Server, InnoDB vs MyISAM | ✅ |
| 4 | Environment Setup: MySQL Server & Workbench | ✅ |
| 5 | User Management: CREATE USER, GRANT, REVOKE, ALTER | ✅ |

