# Lab 01 — MySQL 8.0 Setup, Architecture & User Management

**Goal:** Run MySQL 8.0 in Docker, load a real sample database, explore how the server is built, and manage users, roles and privileges the way a DBA does in production.

**Time:** 2–3 hours
**Prerequisites:** Docker Desktop installed (Windows/Mac) or Docker Engine (Linux), Git.

---

## Part 1 — Concepts (read first)

| Concept | What it means | Why a DBA cares |
|---|---|---|
| **mysqld** | The server process that manages data | You start/stop/monitor it; it's what crashes |
| **Client** (`mysql`) | A program that connects and sends SQL | You use it for all admin work |
| **Data directory** (`datadir`) | Where MySQL stores data files on disk | Disk space, backups, and recovery depend on it |
| **InnoDB** | The default storage engine: transactions, row locking, crash recovery | 99% of production tables use it |
| **Buffer pool** | InnoDB's memory cache for data and indexes | The single most important tuning setting |
| **Redo log** | Records changes so InnoDB can recover after a crash | Crash recovery, write performance |
| **Undo log** | Old versions of rows, for rollback and consistent reads | Long transactions make it grow |
| **Binary log (binlog)** | Records every change to data | Used for **replication** and **point-in-time recovery** |
| **Error log** | Startup, shutdown, crash and warning messages | First place to look during any incident |
| **my.cnf** | The configuration file | Where permanent settings live |

**Say it simply:** A client sends SQL to mysqld. InnoDB reads data from the buffer pool (memory) if it's there, otherwise from disk. Every change is written to the redo log for crash safety and to the binlog for replication and recovery.

---

## Part 2 — Install MySQL 8.0 with Docker

```bash
# Start MySQL 8.0 in a container
docker run --name mysql-lab \
  -e MYSQL_ROOT_PASSWORD='Lab@12345' \
  -p 3306:3306 \
  -d mysql:8.0

# Check it is running
docker ps

# Read the error log (in Docker, it goes to container logs)
docker logs mysql-lab

# Connect as root
docker exec -it mysql-lab mysql -uroot -p
```

✅ **Check:** You see the `mysql>` prompt.

---

## Part 3 — Explore the server

Run these inside the `mysql>` prompt and **note each answer in your write-up**.

```sql
SELECT VERSION();
SHOW DATABASES;

-- Where is the data stored?
SHOW VARIABLES LIKE 'datadir';

-- How big is the buffer pool? (bytes)
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
SELECT @@innodb_buffer_pool_size/1024/1024 AS buffer_pool_mb;

-- Is binary logging on? (it is ON by default in 8.0)
SHOW VARIABLES LIKE 'log_bin';
SHOW BINARY LOGS;

-- Which storage engines exist, and which is default?
SHOW ENGINES;

-- Connection limit and current connections
SHOW VARIABLES LIKE 'max_connections';
SHOW PROCESSLIST;

-- How long has the server been up? (seconds)
SHOW GLOBAL STATUS LIKE 'Uptime';
```

Now look at the config file from outside MySQL:

```bash
docker exec -it mysql-lab cat /etc/my.cnf
docker exec -it mysql-lab ls -lh /var/lib/mysql
```

In the data directory, find: the `ibdata1` file, the `#innodb_redo` folder, the `binlog.00000X` files, and one folder per database.

---

## Part 4 — Load a real sample database

The `employees` database has about 300,000 employees and 2.8 million salary rows, big enough to practise on properly.

```bash
git clone https://github.com/datacharmer/test_db.git
docker cp test_db mysql-lab:/tmp/test_db
docker exec -it mysql-lab bash -c "cd /tmp/test_db && mysql -uroot -p'Lab@12345' < employees.sql"
```

Verify:

```sql
USE employees;
SHOW TABLES;
SELECT COUNT(*) FROM employees;
SELECT COUNT(*) FROM salaries;

-- Size of each table in MB (a very common DBA query)
SELECT table_name,
       ROUND((data_length + index_length)/1024/1024, 1) AS size_mb,
       table_rows
FROM information_schema.tables
WHERE table_schema = 'employees'
ORDER BY size_mb DESC;
```

---

## Part 5 — Users, privileges and roles

In MySQL, a user is **`'name'@'host'`**. `'app'@'localhost'` and `'app'@'%'` are two different users.

### 5.1 Create users with least privilege

```sql
-- Application user: can read and change data, cannot change structure
CREATE USER 'app_user'@'%' IDENTIFIED BY 'App@12345';
GRANT SELECT, INSERT, UPDATE, DELETE ON employees.* TO 'app_user'@'%';

-- Reporting user: read-only
CREATE USER 'report_user'@'%' IDENTIFIED BY 'Rep@12345';
GRANT SELECT ON employees.* TO 'report_user'@'%';

SHOW GRANTS FOR 'app_user'@'%';
SHOW GRANTS FOR 'report_user'@'%';
```

### 5.2 Prove the permissions work

```bash
docker exec -it mysql-lab mysql -ureport_user -p'Rep@12345' employees
```

```sql
SELECT * FROM departments LIMIT 3;   -- works
DELETE FROM departments;             -- fails: ERROR 1142 ... command denied
```

📸 Screenshot the error message for your write-up.

### 5.3 Roles (MySQL 8.0 feature)

```sql
-- As root
CREATE ROLE 'read_only', 'read_write';
GRANT SELECT ON employees.* TO 'read_only';
GRANT SELECT, INSERT, UPDATE, DELETE ON employees.* TO 'read_write';

CREATE USER 'analyst1'@'%' IDENTIFIED BY 'Ana@12345';
GRANT 'read_only' TO 'analyst1'@'%';
SET DEFAULT ROLE 'read_only' TO 'analyst1'@'%';

SHOW GRANTS FOR 'analyst1'@'%' USING 'read_only';
```

### 5.4 Everyday user administration

```sql
-- Who exists?
SELECT user, host, account_locked, password_expired FROM mysql.user;

-- Lock an account (employee left the company)
ALTER USER 'analyst1'@'%' ACCOUNT LOCK;

-- Remove a privilege
REVOKE DELETE ON employees.* FROM 'app_user'@'%';

-- Reset a password
ALTER USER 'app_user'@'%' IDENTIFIED BY 'NewApp@12345';

-- Force password expiry every 90 days
ALTER USER 'app_user'@'%' PASSWORD EXPIRE INTERVAL 90 DAY;

-- Remove a user completely
DROP USER 'analyst1'@'%';
```

---

## Part 6 — Changing configuration safely

```sql
-- Change at runtime (lost on restart)
SET GLOBAL max_connections = 300;

-- Change at runtime AND keep after restart (MySQL 8.0)
SET PERSIST max_connections = 300;

-- See where persisted settings are saved
SELECT * FROM performance_schema.persisted_variables;
```

Restart and confirm the value survived:

```bash
docker restart mysql-lab
docker exec -it mysql-lab mysql -uroot -p'Lab@12345' -e "SHOW VARIABLES LIKE 'max_connections';"
```

Some settings are **static**: they can only change in `my.cnf` followed by a restart. Try `SET GLOBAL innodb_log_file_size = ...;` and read the error.

---

## Part 7 — Interview questions

1. **What is InnoDB, and why is it the default?** Transactions (ACID), row-level locking, crash recovery through the redo log, foreign keys.
2. **What is the buffer pool, and how would you size it?** InnoDB's memory cache. On a dedicated DB server, commonly 60–75% of RAM; then check hit ratio and free memory.
3. **Difference between the redo log and the binlog?** Redo log: InnoDB-level, for crash recovery. Binlog: server-level, logical record of changes, for replication and point-in-time recovery.
4. **What does `'user'@'%'` mean?** A user who can connect from any host. `'user'@'localhost'` is a separate account.
5. **How do you give an app the minimum access it needs?** A dedicated user with only DML on its own schema, no `ALL PRIVILEGES`, no `SUPER`, no global grants. Use roles to manage groups.
6. **`SET GLOBAL` vs `SET PERSIST`?** GLOBAL is lost on restart; PERSIST is saved to `mysqld-auto.cnf` and survives.
7. **MySQL won't start. What do you check first?** The error log, then disk space, permissions on the data directory, config file errors, and port conflicts.

---

## Part 8 — Say it out loud (90 seconds)

Practise until it sounds natural, not memorised:

> "For user management, I follow least privilege. Each application gets its own account with only SELECT, INSERT, UPDATE and DELETE on its own schema, with no admin rights. Reporting users get read-only access. In MySQL 8, I use roles, so access is managed by group instead of user by user. When someone leaves, I lock the account first and drop it later after confirming nothing depends on it. I also enforce password expiry and review the `mysql.user` table regularly to catch unused accounts."

---

## Deliverables for GitHub

- [ ] This file, with your own output pasted under each command
- [ ] Screenshots: `docker ps`, table sizes query, the "command denied" error, `SHOW GRANTS`
- [ ] A short "What I learned" section in your own words
- [ ] Record yourself giving the Part 8 answer once

**Next:** Lab 02 — Backup & Recovery (mysqldump, point-in-time recovery with binlogs, XtraBackup).
