# The Real DBA Morning Checklist (5–6 Years Experience Level)

This is what an experienced MySQL DBA actually checks, in order, before doing anything else — every single morning, whether on-prem, EC2, or RDS. Each check answers one question: **"Is anything broken, and do I need to act before my coffee gets cold?"**

Run these **in this order** — each one either clears you to move to the next, or stops you to investigate immediately.

---

## 1. Is the server even up?

```bash
# SSH in first
ssh shopkart

# Is the MySQL service running?
sudo systemctl status mysqld
```

**Read the output:** look for `Active: active (running)` in green. If it says `inactive (dead)` or `failed`, MySQL is down — that's your #1 priority, before checking anything else.

**If it's down, the immediate next step is:**
```bash
sudo systemctl start mysqld
sudo systemctl status mysqld    # confirm it started
sudo tail -50 /var/log/mysqld.log    # see WHY it was down
```

---

## 2. Can you actually connect?

```bash
mysql -uroot -p -e "SELECT 1;"
```

A service can show "running" but still refuse connections (e.g. too many connections, disk full, corrupted socket). This proves it's actually usable, not just alive.

---

## 3. Was there an unexpected restart overnight?

```sql
SHOW GLOBAL STATUS LIKE 'Uptime';
```

Convert seconds to time (e.g. 3600 = 1 hour). **If uptime is suspiciously short** (a few minutes/hours, and you didn't restart it yourself), that means it crashed or was restarted — go straight to the error log (next step) before anything else.

---

## 4. Error log — anything overnight?

```bash
sudo tail -100 /var/log/mysqld.log
```

**What you're scanning for:**
- Any `[ERROR]` tag — investigate immediately.
- `[Warning]` tags — note them, usually not urgent (e.g. self-signed cert warning is fine).
- A shutdown with **no** matching "Shutdown complete" message, followed by InnoDB **crash recovery** lines on the next startup — this means the server actually crashed, not a clean stop. This is a real incident to document and explain.

---

## 5. Disk space — the #1 real-world outage cause

```bash
df -h
```

**Rule of thumb:** anything over 80% used on the partition holding `/var/lib/mysql` needs a plan (cleanup, or request more storage) *before* it becomes an emergency. At 100%, MySQL cannot write new data — the database effectively stops.

Also check log growth specifically — binlogs and slow query logs can silently eat disk over time:
```bash
sudo du -sh /var/lib/mysql/*
```
(`du -sh` = "disk usage, human-readable, summarized" — shows size of each file/folder inside the data directory.)

---

## 6. Connections — is something eating all the slots?

```sql
SHOW STATUS LIKE 'Threads_connected';
SHOW VARIABLES LIKE 'max_connections';
SHOW PROCESSLIST;
```

**What you're looking for:**
- `Threads_connected` close to `max_connections` → you're near the connection limit; new connections will start failing.
- In `PROCESSLIST`, any query stuck in `State: Sleep` or `Locked` for a long `Time` → a hung or blocking query that may need to be killed:
```sql
KILL <process_id>;
```

---

## 7. Replication health (once a replica exists — from Lab 03 onward)

```sql
-- On the replica
SHOW REPLICA STATUS\G
```

**The two fields that matter most:**
- `Replica_IO_Running: Yes` and `Replica_SQL_Running: Yes` — both must say Yes. If either says `No`, replication is broken.
- `Seconds_Behind_Source` — how far behind the replica is. `0` is ideal; a large or growing number means the replica can't keep up (replication lag), which risks stale reads on that replica.

---

## 8. Last backup — did it actually succeed? (once automated backups exist — from Lab 02 onward)

```bash
ls -lh /path/to/backups/
```
Check the timestamp and file size of the most recent backup file. A backup that "ran" but produced a 0-byte or truncated file is worse than no backup — it gives false confidence. This is why backups get **tested by restoring them periodically**, not just taken and trusted.

---

## 9. Slow queries — anything degrading performance?

```sql
SHOW VARIABLES LIKE 'slow_query_log%';
```
If enabled, check the slow query log file for new entries — recurring slow queries are an early warning before users start complaining.

---

## 10. Security sanity check (periodic, not strictly every single morning, but a real DBA habit)

```sql
SELECT user, host, account_locked, password_expired FROM mysql.user;
```
Watch for: unexpected new accounts, accounts with `%` host access that shouldn't have it, or accounts that should have been locked/dropped after someone left.

---

## The one-line mental model

> **Server up → reachable → no crash → disk OK → connections OK → replication OK → backups OK → performance OK → access OK.**

Each step only matters if the one before it passed. If step 1 fails, you don't move to step 2 — you fix step 1 first.

---

## Turning this into a script (end-of-week goal)

Once you're comfortable running these manually and understanding *why* each one matters, the next step is combining steps 1–6 into a single shell script (`health_check.sh`) that prints a clean summary. That's a genuinely useful artifact for your GitHub portfolio and a common real interview ask: *"Have you automated your own monitoring?"*
