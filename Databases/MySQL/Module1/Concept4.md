📍 **Module 1 → Concept 4 of 5: Environment Setup — Installing MySQL Server & MySQL Workbench**

---

## 📖 THEORY — What are we installing and why?

You need two things to work with MySQL:

**① MySQL Server** — the actual database engine (`mysqld`). This is the background process that stores data, processes queries, and manages connections. Without this, there's no database.

**② MySQL Workbench** — a free, official GUI tool by Oracle/MySQL. It lets you write and run SQL queries, visualize table structures, manage users, and inspect query performance — all through a visual interface instead of a terminal.

Think of it this way:
- **MySQL Server** = the kitchen (where all cooking/data processing happens)
- **MySQL Workbench** = the restaurant counter (where you place orders and receive results)

You can also use the **command-line client** (`mysql`) — a terminal-based client that ships with MySQL Server. We'll use both.

---

## 🔤 SYNTAX / SETUP — Step by Step by Operating System

---

### 🪟 Windows Installation

**Step 1: Download MySQL Installer**
Go to the official MySQL site and download **MySQL Installer for Windows** (the ~450MB full installer, not the web installer).

**Step 2: Run the Installer**
Choose **"Developer Default"** setup type — this installs:
- MySQL Server
- MySQL Workbench
- MySQL Shell
- Sample databases

**Step 3: Configuration wizard will ask:**
```
Config Type      → Development Computer
Port             → 3306 (leave default)
Authentication   → Use Strong Password Encryption (recommended)
Root Password    → Set a strong password — WRITE THIS DOWN
Windows Service  → ✅ Start MySQL Server at System Startup
```

**Step 4: Complete installation → MySQL Server starts automatically as a Windows Service.**

---

### 🍎 macOS Installation

**Option A — MySQL Installer (GUI, recommended for beginners):**
Download the `.dmg` installer from MySQL's official site. Run it, follow the wizard, set your root password when prompted.

After install, start/stop MySQL from:
`System Preferences → MySQL → Start/Stop`

**Option B — Homebrew (recommended for developers):**
```bash
# Install Homebrew first if you don't have it, then:
brew install mysql

# Start MySQL service
brew services start mysql

# Run the security setup script (sets root password, removes test data)
mysql_secure_installation
```

**Install Workbench separately on Mac:**
Download **MySQL Workbench** `.dmg` from MySQL's official site and drag to Applications.

---

### 🐧 Linux (Ubuntu/Debian)

```bash
# Update package list
sudo apt update

# Install MySQL Server
sudo apt install mysql-server -y

# Check it's running
sudo systemctl status mysql

# Run the security hardening script
sudo mysql_secure_installation
# This will ask you to:
# → Set root password
# → Remove anonymous users      → Y
# → Disallow remote root login  → Y
# → Remove test database        → Y
# → Reload privilege tables     → Y

# Install MySQL Workbench (Ubuntu)
sudo apt install mysql-workbench -y
```

---

### ✅ Verify Installation Works

After any OS installation, verify everything works:

```bash
# Connect to MySQL as root via command line
mysql -u root -p
# Enter your root password when prompted

# You should see:
# Welcome to the MySQL monitor.
# mysql>

# Run your first command inside MySQL:
mysql> SHOW DATABASES;
```

Expected output:
```
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set
```

These 4 databases are **system databases** created automatically:

| Database | Purpose |
|---|---|
| `mysql` | Stores user accounts, privileges, server config |
| `information_schema` | Read-only metadata about all databases/tables |
| `performance_schema` | Runtime performance monitoring data |
| `sys` | Human-readable views over performance_schema |

**Never delete or manually edit these.** They're MySQL's internal operating system.

---

### 🖥️ Connecting via MySQL Workbench

After installing Workbench:

```
1. Open MySQL Workbench
2. Click the [+] icon next to "MySQL Connections"
3. Fill in:
   Connection Name  → ShopNest Local
   Hostname         → 127.0.0.1  (your own machine)
   Port             → 3306
   Username         → root
   Password         → [Store in Vault → enter your root password]
4. Click "Test Connection" → should say "Successfully made the MySQL connection"
5. Click OK → double-click the connection to open it
```

You now have a full SQL editor. The panel on the left shows your databases and tables. The top panel is where you write SQL. Hit **Ctrl+Enter** (or **Cmd+Enter** on Mac) to run a query.

---

## 💡 EXAMPLE — Your First ShopNest Commands

Once connected (via Workbench or command line), run these:

```sql
-- Create our ShopNest database (we'll use this entire course)
CREATE DATABASE shopnest;

-- Switch into it
USE shopnest;

-- Confirm you're inside it
SELECT DATABASE();
-- Output: shopnest

-- Check MySQL version
SELECT VERSION();
-- Output: 8.0.xx  (or similar)
```

---

## ⚠️ GOTCHAS — Common Setup Problems

**1. "Access denied for user 'root'@'localhost'"**
On Linux, MySQL 8 uses `auth_socket` by default — root authenticates via the OS user, not a password. Fix:
```bash
# Connect without password first (Linux only, right after install)
sudo mysql

# Then change root to use password auth:
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'YourPassword';
FLUSH PRIVILEGES;
EXIT;

# Now normal login works:
mysql -u root -p
```

**2. Port 3306 already in use**
Another MySQL instance or service is using the port. Check with:
```bash
# Linux/Mac
sudo lsof -i :3306

# Windows (Command Prompt)
netstat -ano | findstr :3306
```

**3. Workbench connects but shows no databases**
You're connected but the root user may lack permissions to see them. Run `SHOW DATABASES;` in the query panel — if it works there, it's a Workbench display refresh issue. Click the refresh icon in the schemas panel.

**4. "MySQL Server has gone away" error**
The server timed out your idle connection. Run `SET SESSION wait_timeout = 28800;` to extend it, or simply reconnect.

**5. Forgetting the root password**
Happens to everyone once. Recovery process exists but is painful (requires restarting MySQL in `--skip-grant-tables` mode). Write your root password somewhere safe right now.

**6. Windows: MySQL not starting**
Open **Services** (Win+R → `services.msc`) → find **MySQL80** → right-click → Start. If it fails, check the error log at `C:\ProgramData\MySQL\MySQL Server 8.0\Data\` for a `.err` file.

---

✅ **Quick Check:**
After a fresh MySQL installation, you run `SHOW DATABASES;` and see four databases: `mysql`, `information_schema`, `performance_schema`, and `sys`. Your colleague says you should delete `information_schema` to "clean up" the server. Why is this a bad idea — and what specifically does `information_schema` contain?
