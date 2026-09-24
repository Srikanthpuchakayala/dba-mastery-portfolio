# Srikanth's DBA Mastery Portfolio

Hands-on database administration labs, daily work logs, and real-world practice tickets — built while training toward a Database Administrator role.

**Environment:** AWS EC2 (Linux) running MySQL 8.4.10, with AWS RDS/Aurora, PostgreSQL, and automation tooling added as the plan progresses.

## Structure

| Folder | Contents |
|---|---|
| `01-mysql-core/` | MySQL setup, users/privileges, backup & recovery, replication, HA, performance tuning |
| `02-aws-cloud/` | RDS/Aurora, migrations (DMS/SCT), zero-downtime cutover |
| `03-postgresql/` | PostgreSQL basics, VACUUM/WAL, streaming replication, Patroni |
| `04-automation-ops/` | Terraform/Ansible, Prometheus/Grafana, Oracle & SQL Server essentials |
| `daily-logs/` | Day-by-day log of what was checked, learned, and worked on |
| `tickets/` | Simulated Jira-style tickets used to drive each lab, with resolution notes |

## Roadmap

**Tier 1 — MySQL Core (Weeks 1–4):** setup & users → backup/recovery → replication → HA → performance tuning
**Tier 2 — AWS Cloud & Migrations (Weeks 5–7):** RDS/Aurora → DMS/SCT migrations → zero-downtime cutover
**Tier 3 — PostgreSQL (Weeks 8–9):** basics/VACUUM/WAL → streaming replication/Patroni
**Tier 4 — Automation & Ops (Weeks 10–12):** Terraform/Ansible → Prometheus/Grafana → Oracle & SQL Server essentials

## How each lab is documented

Every lab file includes: concept notes, hands-on commands run against a real server, my actual output, interview Q&A, and a spoken practice answer.

## How to publish this to GitHub

```bash
cd dba-mastery-portfolio
git init
git add .
git commit -m "Day 1: MySQL setup, architecture, and user management"
git branch -M main
git remote add origin https://github.com/<your-username>/dba-mastery-portfolio.git
git push -u origin main
```
