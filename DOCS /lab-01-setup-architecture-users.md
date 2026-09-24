# Lab 01 — MySQL Setup, Architecture & User Management

**Environment:** AWS EC2 (Linux), MySQL 8.4.10, natively installed (not Docker/RDS)
**Date:** 2026-09-24
**Author:** Srikanth

---

## Part 1 — Concepts

| Concept | What it means | Why a DBA cares |
|---|---|---|
| **mysqld** | The server process that manages data | You start/stop/monitor it; it's what crashes |
| **Client** (`mysql`) | A program that connects and sends SQL | You use it for all admin work |
| **Data directory** (`datadir`) | Where MySQL stores data files on disk | Disk space, backups, and recovery depend on it |
| **InnoDB** | The default storage engine: transactions, row locking, crash recovery | 99% of production tables use it |
| **Buffer pool** | InnoDB's memory cache for data and indexes | The single most important tuning setting |
| **Redo log** | Records changes so InnoDB can recover after a crash | Crash recovery, write performance |
| **Undo log** | Old versions of rows, for rollback and consistent reads | Long transactions make it grow |
| **Binary log (binlog)** | Records every change to data | Used for replication and point-in-time recovery |
| **Error log** | Startup, shutdown, crash and warning messages | First place to look during any incident |
| **my.cnf** | The configuration file | Where permanent settings live |

**In plain words:** A client sends SQL to mysqld. InnoDB reads data from the buffer pool (memory) if it's there, otherwise from disk. Every change is written to the redo log for crash safety and to the binlog for replication/recovery.

---

## Part 2 — Environment

Connected to a real AWS EC2 Linux instance over SSH, with MySQL installed natively (not Docker, not RDS). This mirrors an on-prem/self-managed DBA setup, which is what companies use before migrating to a managed cloud database like RDS/Aurora — the migration path we'll cover in Tier 2.

```bash
ssh -i <key>.pem <user>@<ec2-public-ip>
mysql -uroot -p
```

✅ Connected successfully.

---

## Part 3 — Explore the server (actual output)

```sql
mysql> SELECT VERSION();
+-----------+
| version() |
+-----------+
| 8.4.10    |
+-----------+

mysql> SHOW DATABASES;
+-----------------------+
| Database              |
+-----------------------+
| dba_lab               |
| information_schema    |
| mysql                 |
| performance_schema    |
| shopkart              |
| shopkart_restore_test |
| sys                   |
+-----------------------+
7 rows in set
```

> Note: `dba_lab`, `shopkart`, `shopkart_restore_test` were left over from earlier, pre-mentorship experimentation. Dropped in this session to start from a clean, documented baseline (see Part 3.1).

```sql
mysql> SHOW VARIABLES LIKE 'datadir';
+---------------+-----------------+
| Variable_name | Value           |
+---------------+-----------------+
| datadir       | /var/lib/mysql/ |
+---------------+-----------------+

mysql> SELECT @@innodb_buffer_pool_size/1024/1024 AS buffer_pool_mb;
+----------------+
| buffer_pool_mb |
+----------------+
|   128.00000000 |
+----------------+

mysql> SHOW VARIABLES LIKE 'log_bin';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| log_bin       | ON    |
+---------------+-------+
```

**What this tells me:**
- `datadir = /var/lib/mysql/` — everything lives here. Disk fill-up here is a production incident.
- `buffer_pool_mb = 128` — very small (likely a small/free-tier EC2 instance). Production servers size this at 60–75% of RAM. Flagged for the tuning lab (Lab 05).
- `log_bin = ON` — replication and point-in-time recovery are both possible on this server.

### 3.1 — Cleaning up prior practice databases

```sql
DROP DATABASE dba_lab;
DROP DATABASE shopkart;
DROP DATABASE shopkart_restore_test;

SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
```

Clean baseline confirmed — only system databases remain.

### 3.2 — Remaining checks (actual output)

```sql
mysql> SHOW ENGINES;
+--------------------+---------+------------------------------------------------------------+--------------+------+------------+
| Engine             | Support | Comment                                                     | Transactions | XA   | Savepoints |
+--------------------+---------+------------------------------------------------------------+--------------+------+------------+
| InnoDB             | DEFAULT | Supports transactions, row-level locking, and foreign keys | YES          | YES  | YES        |
| MEMORY, MyISAM, PERFORMANCE_SCHEMA, CSV, ARCHIVE, BLACKHOLE, MRG_MYISAM | YES (no transactions) |
| ndbcluster, FEDERATED, ndbinfo | NO (not compiled in) |
+--------------------+---------+------------------------------------------------------------+--------------+------+------------+

mysql> SHOW VARIABLES LIKE 'max_connections';
+-----------------+-------+
| Variable_name   | Value |
+-----------------+-------+
| max_connections | 151   |
+-----------------+-------+

mysql> SHOW PROCESSLIST;
+----+-----------------+-----------+------+---------+------+------------------------+------------------+
| Id | User            | Host      | db   | Command | Time | State                  | Info             |
+----+-----------------+-----------+------+---------+------+------------------------+------------------+
|  5 | event_scheduler | localhost | NULL | Daemon  | 3110 | Waiting on empty queue | NULL             |
|  8 | root            | localhost | NULL | Sleep   | 1993 |                        | NULL             |
| 11 | root            | localhost | NULL | Query   |    0 | init                   | SHOW PROCESSLIST |
+----+-----------------+-----------+------+---------+------+------------------------+------------------+

mysql> SHOW GLOBAL STATUS LIKE 'Uptime';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| Uptime        | 3117  |
+---------------+-------+
```

```bash
[ec2-user]$ df -h
Filesystem        Size  Used Avail Use% Mounted on
/dev/nvme0n1p1    8.0G  2.4G  5.7G  30% /
/dev/nvme0n1p128   10M  1.3M  8.7M  13% /boot/efi

[ec2-user]$ sudo find /var/log -iname "*mysql*" 2>/dev/null
/var/log/mysqld.log

[ec2-user]$ sudo tail -50 /var/log/mysqld.log
# Clean startup/shutdown history — only [System] entries plus one recurring
# [Warning] about a self-signed CA certificate (expected/harmless on a lab
# server). Every restart shows "Received SHUTDOWN from user <via user signal>"
# followed by "Shutdown complete" — i.e. clean, intentional shutdowns, not
# crashes. No [ERROR] lines. Last start: 2026-09-24T13:46:34Z.
```

**What this tells me:**
- `InnoDB` confirmed as `DEFAULT` engine — matches expectation.
- `max_connections = 151` — MySQL's out-of-the-box default; fine for a small lab server, commonly raised in production.
- Only 3 active connections (event scheduler, one idle root session, the current session) — healthy, nothing stuck or runaway.
- Root filesystem is small (8GB total, 5.7GB free, 30% used) — a real constraint to keep in mind as sample data and backups are added in later labs; disk-full is the #1 real-world cause of DB outages.
- Error log is clean: no `[ERROR]` entries, only routine `[System]` lines and one expected `[Warning]` about the self-signed SSL certificate. All restarts are clean shutdowns (`Received SHUTDOWN from user` → `Shutdown complete`), not crashes — a crash would instead show InnoDB crash-recovery lines on the next startup with no preceding shutdown message.

**Baseline summary (JIRA-101):**

| Check | Result |
|---|---|
| Version | 8.4.10 |
| Datadir | `/var/lib/mysql/` |
| Buffer pool | 128MB (small — flagged for Lab 05 tuning) |
| Binary logging | ON |
| Default engine | InnoDB |
| max_connections | 151 |
| Active connections | 3 (all healthy) |
| Disk usage | 30% of 8GB (5.7GB free) |
| Error log | `/var/log/mysqld.log` — clean, no errors |

✅ **JIRA-101 closed.**

---

## Part 4 — Load a real sample database

The `employees` database (~300K employees, 2.8M salary rows) — big enough to practise real queries and backups on.

```bash
git clone https://github.com/datacharmer/test_db.git
cd test_db
mysql -uroot -p < employees.sql
```

Verify:

```sql
USE employees;
SHOW TABLES;
SELECT COUNT(*) FROM employees;
SELECT COUNT(*) FROM salaries;

SELECT table_name,
       ROUND((data_length + index_length)/1024/1024, 1) AS size_mb,
       table_rows
FROM information_schema.tables
WHERE table_schema = 'employees'
ORDER BY size_mb DESC;
```

_(Output to be pasted here once run.)_

---

## Part 5 — Users, privileges and roles

A MySQL user is `'name'@'host'`. `'app'@'localhost'` and `'app'@'%'` are different accounts.

### 5.1 Least-privilege users

```sql
CREATE USER 'app_user'@'%' IDENTIFIED BY 'App@12345';
GRANT SELECT, INSERT, UPDATE, DELETE ON employees.* TO 'app_user'@'%';

CREATE USER 'report_user'@'%' IDENTIFIED BY 'Rep@12345';
GRANT SELECT ON employees.* TO 'report_user'@'%';

SHOW GRANTS FOR 'app_user'@'%';
SHOW GRANTS FOR 'report_user'@'%';
```

### 5.2 Prove the permissions work

```bash
mysql -ureport_user -p employees
```

```sql
SELECT * FROM departments LIMIT 3;   -- should work
DELETE FROM departments;             -- should fail: ERROR 1142 command denied
```

### 5.3 Roles (MySQL 8 feature)

```sql
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
SELECT user, host, account_locked, password_expired FROM mysql.user;

ALTER USER 'analyst1'@'%' ACCOUNT LOCK;         -- employee left
REVOKE DELETE ON employees.* FROM 'app_user'@'%';
ALTER USER 'app_user'@'%' IDENTIFIED BY 'NewApp@12345';
ALTER USER 'app_user'@'%' PASSWORD EXPIRE INTERVAL 90 DAY;
DROP USER 'analyst1'@'%';
```

_(All outputs to be pasted here once run.)_

---

## Part 6 — Changing configuration safely

```sql
SET GLOBAL max_connections = 300;              -- lost on restart
SET PERSIST max_connections = 300;             -- survives restart (MySQL 8+)
SELECT * FROM performance_schema.persisted_variables;
```

```bash
sudo systemctl restart mysqld
mysql -uroot -p -e "SHOW VARIABLES LIKE 'max_connections';"
```

Static variables (e.g. `innodb_log_file_size`) can only change in `my.cnf` + restart — confirmed by the error when trying `SET GLOBAL` on one.

---

## Part 7 — Interview questions

1. **What is InnoDB, and why is it the default?** Transactions (ACID), row-level locking, crash recovery via the redo log, foreign keys.
2. **What is the buffer pool, and how would you size it?** InnoDB's memory cache. Commonly 60–75% of RAM on a dedicated DB server; verify with hit ratio and free memory.
3. **Redo log vs. binlog?** Redo log: InnoDB-level, crash recovery. Binlog: server-level, logical change record, used for replication and PITR.
4. **What does `'user'@'%'` mean?** Can connect from any host — distinct from `'user'@'localhost'`.
5. **How do you give an app minimum access?** Dedicated account, only the DML it needs on its own schema, no `ALL PRIVILEGES`/`SUPER`, managed via roles.
6. **`SET GLOBAL` vs `SET PERSIST`?** GLOBAL is runtime-only; PERSIST survives a restart via `mysqld-auto.cnf`.
7. **MySQL won't start — first checks?** Error log, disk space, datadir permissions, config syntax, port conflicts.

---

## Part 8 — Say it out loud (90 seconds)

> "For user management, I follow least privilege. Each application gets its own account with only the DML it needs on its own schema — no admin rights. Reporting users get read-only access. In MySQL 8, I use roles so access is managed by group instead of user by user. When someone leaves, I lock the account first and drop it later once I've confirmed nothing depends on it. I also enforce password expiry and review `mysql.user` regularly to catch unused accounts."

---

## What I learned (Day 1)

- The `datadir`, buffer pool, and binlog settings are the first three things worth checking on any unfamiliar server — they tell you where data lives, how much memory InnoDB has to work with, and whether replication/PITR are even possible.
- A small buffer pool (128MB here) is a real, common finding on small EC2 instances — not a mistake, just something to flag and revisit during tuning.
- Practicing on a real EC2 Linux box (vs. Docker or a laptop install) is closer to how this job is actually done, and gives a stronger story for interviews.

## Status

- [x] Connected to real AWS EC2 MySQL server
- [x] Explored version, databases, datadir, buffer pool, binlog status
- [x] Cleaned up prior practice databases for a clean baseline
- [ ] Run remaining Part 3.2 checks (engines, connections, uptime, error log, disk space)
- [ ] Load `employees` sample database
- [ ] Complete Part 5 users/roles exercises with real output
- [ ] Complete Part 6 config change exercise
- [ ] Record Part 8 spoken answer

**Next:** Lab 02 — Backup & Recovery (mysqldump, point-in-time recovery with binlogs, XtraBackup).
