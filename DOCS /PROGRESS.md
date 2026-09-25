# Progress Summary — DBA Mastery Journey

A running, honest record of everything covered so far: what was done, what was learned, and what mistakes taught the most. Updated as each lab completes.

---

## Day 1 (2026-09-24)

**Setup**
- Built this GitHub portfolio structure (labs, tickets, daily logs, JD tracker)
- Connected to a real AWS EC2 Linux instance running MySQL 8.4.10 (native install — not Docker, not RDS) — closer to real on-prem/self-managed DBA work than a laptop install

**Lab 01 — MySQL Setup, Architecture & User Management** → [`01-mysql-core/lab-01-setup-architecture-users.md`](01-mysql-core/lab-01-setup-architecture-users.md)
- Explored server internals: `datadir`, buffer pool size, binary logging, storage engines, `max_connections`, active connections
- Established a full baseline health check and closed **JIRA-101**
- Cleaned up stale practice databases for a documented clean slate
- Loaded the real `employees` sample dataset (300,024 employees, 2,844,047 salary rows)
- Created least-privilege users (`app_user`, `report_user`) and **proved** the permissions work — not just configured them, but logged in as `report_user` and confirmed `SELECT` succeeded while `DELETE` was denied (`ERROR 1142`)
- Built MySQL 8 roles (`read_only`, `read_write`), attached one to a real user, and verified the grant
- Ran everyday admin tasks: locking an account, revoking a privilege, setting password expiry
- **Security catch:** the `mysql.user` audit query surfaced an unrecognized leftover account — a real example of what this kind of review is meant to find

**Fundamentals learned:**
- Reading a MySQL error log: `[System]` vs `[Warning]` vs `[ERROR]`, and how to tell a clean shutdown from a crash (shutdown message present vs. absent + crash-recovery lines on next start)
- `df -h`, `find`, `tail`, `ssh` — what each flag does and why
- Full Git workflow: `init`, `add`, `commit`, `push`, `pull` with `--allow-unrelated-histories --no-rebase`, resolving a merge conflict with `checkout --ours`
- The critical distinction between the `mysql>` prompt (SQL only) and the Linux shell prompt (`mysqldump`, `git`, `ls`, etc.) — a mistake made and caught multiple times, which is exactly how it becomes permanent knowledge

**Career/prep work:**
- Reviewed and critiqued the resume (found inflated language, repeated bullets, and inconsistencies to fix)
- Settled the career path decision: DBA first, Data Engineering as the natural next step
- Built a **JD tracker** to log real job postings against the roadmap

---

## Day 2 (2026-09-25)

**JD Tracker — 4 real postings logged** → [`jd-tracker.md`](jd-tracker.md)
- SQL Server + PowerShell (DBA-type)
- SQL Server + T-SQL/SSIS/SSRS (Database Developer-type — a different role category, flagged)
- MySQL/PostgreSQL/Oracle multi-engine — **strongly validates the current roadmap**, no changes needed
- Oracle OCI cloud specialist — niche, later-career track, not prioritized now
- **New tool names surfaced for later:** PMM (Percona Monitoring & Management), Zabbix, RMAN/OEM (Oracle-native)

**Lab 02 (Part 1) — Backup & Recovery with mysqldump** → [`01-mysql-core/lab-02-backup-and-recovery.md`](01-mysql-core/lab-02-backup-and-recovery.md)
- Took a real logical backup of `employees` with `mysqldump` (161MB) and verified it two ways: file size sanity check, and inspecting real SQL content with `head`
- Restored the backup into a separate test database (`employees_restore_test`) — never testing on top of live data
- **Proved** the restore was complete and correct by comparing exact row counts against the original (300,024 / 2,844,047 — exact match)
- Learned why row-count comparison matters more than "tables exist": a partial/corrupted dump can still create table structures successfully while missing data
- Covered the full real-world picture beyond the lab: off-server storage (S3), retention policy, cron-based automation, and daily backup verification as part of the morning checklist

**Format change (self-directed):** switched from "here are 6 commands, run them" to concept-first: explain the idea, write the command myself, get corrected, then run it — genuinely more hands-on and better retention.

---

## Skills demonstrated so far (résumé/interview-ready, because they were actually done)

- MySQL server architecture and configuration inspection
- Health-check methodology (the same 10-step checklist a working DBA uses every morning)
- Error log interpretation (crash vs. clean shutdown)
- Least-privilege user and role management, with proof, not just claims
- Logical backup and restore with `mysqldump`, with real verification methodology
- Git version control workflow, including conflict resolution
- Linux fundamentals: disk checks, log searching, SSH

## Up next
- Point-in-time recovery (PITR) using binary logs — the most commonly asked backup/recovery interview question
- Percona XtraBackup (physical backup)
- Automating backups with cron + off-server storage
- Lab 03: Replication
