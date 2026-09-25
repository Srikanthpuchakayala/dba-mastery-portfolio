# Lab 02 — Backup & Recovery (Part 1: Logical Backup with mysqldump)

**Environment:** AWS EC2 (Linux), MySQL 8.4.10
**Date:** 2026-09-25
**Author:** Srikanth

---

## Part 1 — Concept: Logical Backups

A **logical backup** doesn't copy raw data files — it exports the actual SQL statements needed to recreate the data: `CREATE TABLE` statements, then `INSERT` statements for every row. Running the resulting `.sql` file against an empty server rebuilds the database exactly.

`mysqldump` is MySQL's built-in tool for this:
```
mysqldump -u<username> -p <database_name> > <output_file>.sql
```
- `-u<username>` — account to connect as
- `-p` — prompt for password
- `<database_name>` — database to dump
- `>` — shell redirect: send output to a file instead of the screen

**Key distinction:** `mysqldump` is a **Linux shell command**, not SQL — it must be run from the `[ec2-user@...]$` prompt, never from inside the `mysql>` prompt. (Learned this the hard way — see mistakes log below.)

---

## Part 2 — Taking the backup (actual commands + output)

```bash
[ec2-user@ip-172-31-4-202 ~]$ mysqldump -u root -p employees > employees_backup.sql
Enter password:
[ec2-user@ip-172-31-4-202 ~]$
```

**Verification — never trust a backup just because the command "ran":**

```bash
[ec2-user@ip-172-31-4-202 ~]$ ls -lh employees_backup.sql
-rw-r--r--. 1 ec2-user ec2-user 161M Sep 25 13:37 employees_backup.sql

[ec2-user@ip-172-31-4-202 ~]$ head -30 employees_backup.sql
-- MySQL dump 10.13  Distrib 8.4.10, for Linux (x86_64)
--
-- Host: localhost    Database: employees
-- ------------------------------------------------------
-- Server version 8.4.10
...
DROP TABLE IF EXISTS `current_dept_emp`;
/*!50001 DROP VIEW IF EXISTS `current_dept_emp`*/;
...
```

**Why both checks matter:**
- File size (161MB) is consistent with a real dump of this dataset — not suspiciously small (0 bytes or a few KB would signal a failed/incomplete backup).
- `head -30` confirms the file contains real SQL (`DROP`/`CREATE` statements), not an error message that `mysqldump` might have printed instead of actual data.

---

## Part 3 — Restoring to a test database (the real proof a backup works)

**Concept:** A backup is only proven to work once it's actually been restored somewhere and verified — never restore-test on top of live data. Create a throwaway database, restore into it, compare data, then discard.

**Restore command shape** (note the direction: `<` = input, feeding a file *into* MySQL — opposite of `>` used for taking the backup):
```
mysql -u<user> -p <target_database> < <backup_file>.sql
```
Also a **shell command**, not SQL.

**Actual commands run:**
```sql
mysql> CREATE DATABASE employees_restore_test;
```

```bash
[ec2-user@ip-172-31-4-202 ~]$ mysql -u root -p employees_restore_test < employees_backup.sql
Enter password:
[ec2-user@ip-172-31-4-202 ~]$
```

**Verification — comparing exact row counts against the original:**
```sql
mysql> USE employees_restore_test;
Database changed

mysql> SELECT COUNT(*) FROM employees;
+----------+
| COUNT(*) |
+----------+
|   300024 |
+----------+

mysql> SELECT COUNT(*) FROM salaries;
+----------+
| COUNT(*) |
+----------+
|  2844047 |
+----------+
```

✅ **Both counts match the original `employees` database exactly (300,024 employees, 2,844,047 salaries) — the backup is proven restorable, not just assumed to work.**

**Why exact row counts, not just "tables exist":** a partial/corrupted backup can still successfully run all `CREATE TABLE` statements while missing `INSERT` statements (e.g. if the dump was interrupted by a dropped connection or full disk). Tables existing proves structure restored; matching row counts proves the data restored completely. This is the real-world "false confidence" trap — a backup that runs without error isn't the same as a backup that's actually complete.

---

## Mistakes made and lessons learned (kept intentionally — this is where real learning happens)

1. **Typed `mysqldump`/`mysql` restore commands inside the `mysql>` prompt** instead of the Linux shell, twice. Got `ERROR 1064: You have an error in your SQL syntax` because MySQL tried to parse a shell command as SQL. **Rule:** `mysql>` prompt = SQL only; `[ec2-user@...]$` prompt = shell commands (`mysqldump`, `ls`, `git`, etc.).
2. **`ls - lh` (space after the dash)** failed with "cannot access '-': No such file or directory" — flags must be written as one token (`-lh`), not separated by a space.
3. **Swapped filename and database name** in the restore command on the first attempt, and used an SQL-style trailing semicolon at the shell prompt (not needed there).
4. **Ran the restore before actually creating the target database** — got `ERROR 1049: Unknown database 'employees_restore_test'`. The `CREATE DATABASE` statement had been written but not yet executed inside `mysql>`. Fixed by actually running it, then confirming with `SHOW DATABASES;` before retrying the restore.

---

## Real-world DBA practice: what happens after a backup (beyond just this lab)

A logical backup file sitting on the same server it was taken from is not a real backup strategy — it's a false sense of security. Full real-world practice:

1. **Verification** (done above): file size sanity check, content check, and — the real proof — an actual test restore with row-count comparison.
2. **Off-server storage:** the backup must be copied somewhere separate from the source server — minimum a different volume/server, standard practice is uploading to object storage (AWS S3) so a disk failure, terminated instance, or ransomware event on the DB server doesn't destroy the backup too.
3. **Retention policy:** backups aren't kept forever or deleted immediately — a typical policy: daily backups kept 7–14 days, weekly kept 1–3 months, monthly kept 1+ year for compliance.
4. **Automation:** a real DBA doesn't run `mysqldump` by hand daily. A cron job runs it on a schedule, timestamps each file (so they don't overwrite each other), uploads to S3, prunes old local copies, and logs success/failure with alerting on failure.
5. **Daily verification (this maps to Day 1's morning checklist, Step 8):** did last night's backup job run, did it succeed, is the file size consistent with previous days (a sudden size drop is a red flag), and periodically — actually restore a backup to prove it still works.

---

## Status

- [x] Took a real logical backup with `mysqldump` (161MB, verified)
- [x] Restored it into a separate test database
- [x] Verified restore correctness via exact row-count comparison
- [ ] Point-in-time recovery using binary logs (next session)
- [ ] Percona XtraBackup (physical backup, time permitting)
- [ ] Automate backups with a cron job + off-server storage

**Next:** Point-in-time recovery (PITR) — using the binary log to replay changes made *after* a backup, up to just before a mistake (e.g. an accidental `DELETE`). The single most commonly asked backup/recovery interview question.
