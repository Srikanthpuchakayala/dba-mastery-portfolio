# MySQL Architecture, Users, Roles & Security — Complete Beginner-to-Pro Guide

A permanent reference document. If someone with zero database background reads this start to finish, they should come away actually understanding how MySQL is built and how access control works — not just memorized commands.

---

# PART 1 — What Even Is a Database Server?

## The simplest possible picture

Imagine a **library**.

- The **library building** is the server (a computer).
- The **librarian** is the database software (MySQL) — the only one allowed to touch the books directly.
- The **books** are your data.
- **You** (a person wanting information) never walk into the shelves yourself. You ask the librarian: "Get me this book" or "Add this new book." The librarian fetches it, updates records, and hands you what you asked for.

This is exactly how a database works:
- You never touch the raw data files directly.
- You send a **request** (called a query, written in SQL — Structured Query Language) to the database software.
- The database software reads/writes the actual files on disk and gives you the answer.

## The two main things involved

| Real-world word | Database word | What it means |
|---|---|---|
| The librarian (the worker) | **`mysqld`** (the MySQL server process) | The actual running program that manages everything. If this stops, the whole library is "closed" — nobody can be served. |
| A visitor asking the librarian something | **`mysql` client** | The program YOU use to type requests and send them to `mysqld`. |

**Command you already ran to prove this:**
```bash
sudo systemctl status mysqld
```
This checks: "Is the librarian (mysqld) currently working, or on a break (stopped)?"

---

# PART 2 — Where Does the Data Actually Live?

## Simple picture

Continuing the library analogy: the **shelves and the books themselves** are stored in one physical location in the building.

In MySQL, that physical location is called the **data directory**, usually:
```
/var/lib/mysql/
```

Every database you create, every table, every row of data — physically lives inside this one folder on the server's hard disk.

**Why this matters so much:**
- If this folder's disk runs out of space, the librarian (mysqld) literally cannot add any new books (data) — the whole database stops accepting changes. This is the single most common real-world cause of database outages.
- If this folder gets deleted or corrupted and you have no backup, your data is gone. Forever.

**Command you already ran:**
```sql
SHOW VARIABLES LIKE 'datadir';
```
```bash
df -h    -- checks how much disk space is used vs. free where this folder lives
```

---

# PART 3 — The "Filing System" Inside: Storage Engines & InnoDB

## Simple picture

Imagine the library has different **filing systems** for different sections — some sections use strict card catalogs with rules (checked in, checked out, tracked), other sections are just loose piles with no rules.

MySQL calls these different filing systems **storage engines**. The one almost everyone uses, and the one that has "rules and safety," is called **InnoDB**.

## What makes InnoDB special (in plain words)

| InnoDB feature | Plain-word meaning |
|---|---|
| **Transactions** | A group of changes either ALL happen, or NONE happen. Like transferring money between two bank accounts — you never want it to deduct from one account but fail to add to the other. InnoDB guarantees this. |
| **Row-level locking** | If two people try to edit the *same single row* of data at the same exact moment, InnoDB manages that safely so nothing gets corrupted. It only locks the specific row being changed, not the whole table — so other people can still work on different rows at the same time. |
| **Crash recovery** | If the server suddenly loses power mid-operation, InnoDB can figure out exactly what was "in progress" and either finish it safely or undo it cleanly when it restarts — instead of leaving your data in a broken, half-changed state. |

**Command you already ran:**
```sql
SHOW ENGINES;
```

---

# PART 4 — Memory: The Buffer Pool

## Simple picture

Imagine the librarian doesn't walk to the far shelves every single time someone asks for a popular book. Instead, they keep the **most frequently requested books on a cart right next to their desk** — much faster to grab.

That "cart next to the desk" is the **buffer pool** — a chunk of the computer's RAM (memory) where MySQL keeps recently/frequently used data, so it doesn't have to read from the slow physical disk every single time.

**Why it matters:** RAM is dramatically faster than disk. The bigger and smarter this "cart" is, the faster your database feels overall. This is the single most important performance setting in MySQL.

**Command you already ran:**
```sql
SELECT @@innodb_buffer_pool_size/1024/1024 AS buffer_pool_mb;
```

---

# PART 5 — The "Safety Notebooks": Redo Log, Undo Log, Binary Log

These three are often confused, so here's each one with its own clear analogy.

## Redo Log — "the recovery notebook"

Imagine every time the librarian makes a change (adds a book, moves a book), they FIRST jot a quick note in a small notebook: *"about to add book X to shelf 5."* Only after writing that note do they actually do the physical action.

**Why:** if the power goes out mid-action, when the librarian comes back, they can read the notebook and know exactly what was supposed to happen, and finish it or redo it. This is InnoDB's **crash recovery** mechanism — it's called the redo log because it lets InnoDB "redo" work after a crash.

## Undo Log — "the undo button"

Imagine the librarian also keeps a separate notebook of **the old version of anything they change** — like a "before" photo. If someone says "actually, cancel that, put it back the way it was," the librarian checks this notebook to know exactly what to restore.

**Why:** this is what makes `ROLLBACK` possible (canceling a transaction partway through), and it's also used so that someone reading data doesn't see someone else's half-finished changes.

## Binary Log (binlog) — "the master diary of everything that ever happened"

This is different from the other two — it's not about crash safety, it's a **complete, ongoing diary of every single change** made to the data, in the order it happened, kept indefinitely (until you clean it up on purpose).

**Why it matters — two huge uses:**
1. **Replication:** you can hand this "diary" to a second server and say "replay everything in this diary," and that second server ends up as an exact copy — always staying in sync. This is how replicas/copies of a database are kept up to date in real time.
2. **Point-in-time recovery (PITR):** if you have a backup from last night, and something bad happened at 2pm today (like an accidental `DELETE`), you can take last night's backup AND replay the diary entries from midnight up to just before the bad delete at 2pm — recovering everything except the mistake itself.

**Command you already ran:**
```sql
SHOW VARIABLES LIKE 'log_bin';
```

## Quick comparison table

| Log | What it tracks | Main purpose | Lives how long |
|---|---|---|---|
| Redo log | Changes about to happen, InnoDB-level | Crash recovery | Short — recycled constantly |
| Undo log | The "before" version of a row | Rollback, consistent reads | As long as needed for that transaction |
| Binary log (binlog) | Every completed change, server-level | Replication, point-in-time recovery | Kept long-term until purged |

---

# PART 6 — Users: Who's Allowed In The Building At All?

## Simple picture

Think of a **hotel**. Before anyone can even walk in the front door, they need to be a **registered guest** with a room key. Being a registered guest just means "you're allowed inside" — it says nothing yet about which rooms you can enter.

A MySQL **user** is exactly this: a registered identity that's allowed to connect to the server at all.

## The weird-but-important part: `'name'@'host'`

In MySQL, a user isn't just a name — it's a name PLUS **where they're allowed to connect from**. This is written as:
```
'username'@'host'
```

Think of it like a hotel guest's registration also specifying *which entrance* they're allowed to use:

| Example | Plain meaning |
|---|---|
| `'app_user'@'localhost'` | Can only connect if they are physically ON the same machine as the server |
| `'app_user'@'%'` | The `%` is a wildcard meaning "any location" — can connect from anywhere |
| `'app_user'@'192.168.1.50'` | Can only connect from that one specific computer |

**Critically:** `'app_user'@'localhost'` and `'app_user'@'%'` are treated as **two completely separate, unrelated accounts**, even though they share a name — like two different hotel guests who happen to share the same first name.

**Commands you already ran:**
```sql
CREATE USER 'app_user'@'%' IDENTIFIED BY 'App@12345';
```
This means: "Register a new guest named `app_user`, allowed to enter from anywhere, and their room key password is `App@12345`."

```sql
SELECT user, host, account_locked, password_expired FROM mysql.user;
```
This is checking the hotel's entire guest registry — every single account that exists, whether their key is currently deactivated (`account_locked`), and whether their password needs resetting (`password_expired`).

---

# PART 7 — Privileges: Once Inside, What Are You Allowed To Do?

## Simple picture

Being a registered hotel guest gets you in the front door. But that alone doesn't mean you can walk into the kitchen, the manager's office, or someone else's room. **Privileges** are the specific permissions for specific rooms/actions.

## The four privileges you've already used, in plain words

| Privilege | Plain meaning |
|---|---|
| `SELECT` | "Look at" — read/view data, but can't change anything |
| `INSERT` | "Add new" — create brand new rows of data |
| `UPDATE` | "Change existing" — modify data that's already there |
| `DELETE` | "Remove" — delete existing rows |

There are many more (like `CREATE`, `DROP`, `ALTER` — which control changing the actual *structure* of tables, not just the data inside them), but these four cover almost all day-to-day application access.

## GRANT — the act of handing out a specific permission

`GRANT` is the command that says "give this specific permission, on this specific thing, to this specific person."

**The full sentence structure, in plain English:**
> GRANT [what they can do] ON [which database/table] TO [which user]

**Command you already ran:**
```sql
GRANT SELECT, INSERT, UPDATE, DELETE ON employees.* TO 'app_user'@'%';
```
Read this literally left to right:
- **GRANT** (give permission)
- **SELECT, INSERT, UPDATE, DELETE** (to look at, add, change, and remove data)
- **ON employees.\*** (inside the `employees` database, on ALL tables — the `*` is a wildcard meaning "everything")
- **TO 'app_user'@'%'** (to this specific user)

Compare this to the read-only user:
```sql
GRANT SELECT ON employees.* TO 'report_user'@'%';
```
This user can ONLY look, never change anything.

## REVOKE — the opposite: taking a permission away

**Command you already ran:**
```sql
REVOKE DELETE ON employees.* FROM 'app_user'@'%';
```
Exact mirror of `GRANT`, just removing instead of adding. `app_user` keeps `SELECT, INSERT, UPDATE` but loses the ability to delete anything.

## Proving privileges actually work — don't just trust the configuration

You didn't just set this up and assume it worked — you **logged in as `report_user` and tried both actions**:

```sql
SELECT * FROM departments LIMIT 3;   -- WORKED
DELETE FROM departments;              -- FAILED: ERROR 1142 command denied
```

This is the single most important habit in this entire document: **never trust that a permission setup works just because you typed the command correctly — always test it as the actual restricted user.**

## The principle behind all of this: "Least Privilege"

**In one sentence:** give every person/application the absolute minimum access they need to do their job — nothing more.

**Why:** if an application only ever needs to read and write data (never delete anything, never change table structure), and its account is compromised or has a bug, the damage it can do is limited to exactly what it was allowed to do. If that same account had full admin rights "just in case," a single mistake or breach could destroy everything.

---

# PART 8 — Roles: Managing Permissions in Groups Instead of One-by-One

## Simple picture

Imagine a hotel with 200 staff, all needing the exact same set of room-access permissions (housekeeping needs access to guest rooms and supply closets, managers need access to everything, etc.). Instead of the manager individually programming each of 200 keycards one at a time, they create **keycard templates**: "Housekeeping Template," "Manager Template." New staff just get handed the right template, instantly getting all those permissions at once.

A MySQL **role** is exactly this template.

## How it works, step by step (with the commands you ran)

**Step 1 — Create the template (role):**
```sql
CREATE ROLE 'read_only', 'read_write';
```
This just creates two empty templates, named `read_only` and `read_write`. They don't do anything yet.

**Step 2 — Define what the template includes:**
```sql
GRANT SELECT ON employees.* TO 'read_only';
GRANT SELECT, INSERT, UPDATE, DELETE ON employees.* TO 'read_write';
```
Now the `read_only` template includes "can look, nothing else." The `read_write` template includes full data access.

**Step 3 — Create a real person (user):**
```sql
CREATE USER 'analyst1'@'%' IDENTIFIED BY 'Ana@12345';
```

**Step 4 — Hand them a template (attach the role):**
```sql
GRANT 'read_only' TO 'analyst1'@'%';
```
Instead of manually re-typing `GRANT SELECT ON employees.* TO 'analyst1'@'%'` individually, `analyst1` just inherits everything the `read_only` template already has.

**Step 5 — Make the template active automatically on login:**
```sql
SET DEFAULT ROLE 'read_only' TO 'analyst1'@'%';
```
This is a genuinely easy-to-miss detail: in MySQL, just being *handed* a role doesn't mean it's automatically switched on the moment you log in — this command says "activate this role every time this person logs in, automatically."

## Why this matters at real companies

If you have 50 people who all need identical "reporting analyst" access, and the requirements change later (say, they now also need access to a new table), you update the **role definition once**, and all 50 people's access updates automatically. Without roles, you'd have to individually re-run `GRANT`/`REVOKE` 50 separate times and risk missing someone.

---

# PART 9 — Passwords & Account Lifecycle (The Real, Everyday DBA Work)

## Locking an account — "deactivate the keycard, don't destroy it yet"

**Command you already ran:**
```sql
ALTER USER 'analyst1'@'%' ACCOUNT LOCK;
```
This is the very first thing a real DBA does the moment someone leaves a company: **lock immediately** (they can't log in anymore, instantly), but **don't delete yet** — because deleting is permanent, and you first want to confirm nothing else (a scheduled script, another service) secretly depends on that account still existing.

Only once you've confirmed it's safe do you actually remove it:
```sql
DROP USER 'analyst1'@'%';
```

## Password expiry — forcing regular password changes

**Command you already ran:**
```sql
ALTER USER 'app_user'@'%' PASSWORD EXPIRE INTERVAL 90 DAY;
```
This means: "this account's password automatically becomes invalid after 90 days, and it must be changed." This is a common compliance/security requirement (often required by standards like SOC2 or ISO 27001) so that even if a password leaked at some point, it has a limited useful lifetime for an attacker.

## Changing a password directly

```sql
ALTER USER 'app_user'@'%' IDENTIFIED BY 'NewApp@12345';
```
Straightforward — sets a new password immediately, without waiting for expiry.

## The full audit query — "who exists, and what's their status right now?"

```sql
SELECT user, host, account_locked, password_expired FROM mysql.user;
```
This is one of the most important habits for ongoing security: periodically list every single account on the server and check — is this account still needed? Is it locked when it should be? Has its password expired and not been renewed? This exact query is how you'd catch, for example, a forgotten test account (`rahul`@`localhost`, in this project's own real history) that shouldn't still exist.

---

# PART 10 — Glossary (Quick Lookup)

| Term | One-line plain meaning |
|---|---|
| **mysqld** | The actual running database server program |
| **mysql client** | The tool you use to send commands to the server |
| **datadir** | The folder on disk where all data physically lives |
| **InnoDB** | The default storage engine — safe, transaction-supporting |
| **Buffer pool** | RAM cache MySQL uses to avoid slow disk reads |
| **Redo log** | Crash-recovery notebook — lets InnoDB finish/undo work after a crash |
| **Undo log** | "Before" version of data — enables rollback |
| **Binary log (binlog)** | Master diary of every change — used for replication and point-in-time recovery |
| **User** | A registered login identity (`'name'@'host'`) |
| **Host (in `'user'@'host'`)** | Where that user is allowed to connect from |
| **Privilege** | A specific permission (SELECT, INSERT, UPDATE, DELETE, etc.) |
| **GRANT** | Command to give a privilege or role to a user |
| **REVOKE** | Command to take away a privilege |
| **Role** | A reusable bundle/template of privileges, assignable to many users |
| **Least privilege** | The principle: give the minimum access needed, nothing more |
| **Account lock** | Disables login without deleting the account |
| **Password expiry** | Forces a password reset after a set time period |

---

# PART 11 — The One-Paragraph Version (for memorizing before an interview)

> "MySQL is a server program (`mysqld`) that manages data stored on disk in a data directory, using InnoDB as the default storage engine for safe, transactional reads and writes. It keeps frequently used data in memory (the buffer pool) for speed, and protects against crashes and enables recovery using the redo log, undo log, and binary log — the binary log specifically also powers replication and point-in-time recovery. Access is controlled through users, identified as `'name'@'host'`, who are granted specific privileges (like SELECT, INSERT, UPDATE, DELETE) on specific databases using GRANT, following the principle of least privilege. Roles let you bundle privileges into reusable templates instead of managing each user individually. Ongoing account hygiene — locking unused accounts, revoking unnecessary privileges, and enforcing password expiry — is a core, everyday part of the DBA job, not a one-time setup task."

---

*This document is permanent reference material — re-read it whenever a concept feels fuzzy, and add to it as new topics (replication, HA, performance tuning) are covered in later labs.*
