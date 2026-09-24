# JIRA-101 — Set up baseline monitoring checklist and establish server health baseline

**Priority:** P2
**Sprint:** Onboarding
**Assigned by:** Mentor
**Status:** Closed

## Description
New DBA joining the team. Before any production work, document current server state as a baseline for future comparison.

## The daily DBA morning health check

1. Server up and reachable — `SELECT VERSION();`, `SHOW VARIABLES LIKE 'hostname';`
2. Uptime — did it restart overnight unexpectedly? `SHOW GLOBAL STATUS LIKE 'Uptime';`
3. Error log — anything overnight? `sudo tail -100 /var/log/mysqld.log`
4. Disk space — the #1 cause of DB outages. `df -h`
5. Active connections vs. limit — `SHOW STATUS LIKE 'Threads_connected';` / `SHOW VARIABLES LIKE 'max_connections';`
6. Replication status (once a replica exists — Lab 03) — `SHOW REPLICA STATUS\G`
7. Last backup succeeded (once automated backups exist — Lab 02)

## Work done
- Connected to EC2 MySQL server, confirmed version 8.4.10
- Captured baseline: datadir, buffer pool size, binlog status
- Dropped stale practice databases (`dba_lab`, `shopkart`, `shopkart_restore_test`) to start from a clean, documented state

- Ran items 3–5: error log clean (only routine `[System]` lines + 1 expected self-signed cert `[Warning]`, no `[ERROR]`), disk 30% used (5.7GB free of 8GB), 3 active connections (all healthy)
- Confirmed error log path for this server: `/var/log/mysqld.log`
- Full baseline table recorded in the linked lab doc

## Remaining
- [ ] Turn checklist into a single reusable shell script by end of week (carried to next sprint)

**Linked doc:** `01-mysql-core/lab-01-setup-architecture-users.md`
