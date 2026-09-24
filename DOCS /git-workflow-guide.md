# Git Workflow — How This Actually Works Day to Day

This explains the **real reason** behind each Git step, not just commands to memorize. Read this once fully, then use the "Daily Routine" section at the bottom as your quick reference every day.

---

## The core idea you're missing: three places your code can be

1. **Working directory** — the actual files you're editing on your Mac.
2. **Local repository** — your `.git` folder's history, on your Mac, that nobody else can see.
3. **Remote repository (GitHub)** — the shared copy that others (or "future you" from another laptop) can see and pull from.

A `commit` moves changes from (1) into (2). A `push` moves changes from (2) into (3). A `pull` brings changes from (3) down into (1) and (2). Every Git command is really just moving work between these three places.

---

## Why "just git push" isn't enough on a real team

If you're the only person working on this repo (which you are, right now), a simple pattern works fine:
```bash
git pull          # get any changes I made from another machine
git add .
git commit -m "..."
git push
```

But **on a real team with multiple DBAs/engineers**, several people touch the same repo. If you skip pulling first, you might push changes that conflict with someone else's work, or overwrite something. That's why professional teams use **branches**.

---

## What a branch actually is

Think of `main` as the **official, working, trusted version** of the code/docs. A **branch** is a parallel copy where you can make changes safely, without touching `main`, until your work is reviewed and ready.

```
main:     A---B---C-------------F   (always stable)
                   \           /
feature: (you)      D---E-----    (your work in progress)
```

You create a branch, do your work there, and only merge it into `main` once it's done and correct.

---

## The real daily workflow (what a DBA on a team actually does)

### Step 1 — Start of day: sync with the team's latest work
```bash
git checkout main
git pull origin main
```
- `git checkout main` — switch to the `main` branch (make sure you're on it).
- `git pull origin main` — download and merge in anything a teammate pushed since you last checked.
- **Why first:** you need to know what changed before you start today's work, so you're not duplicating effort or missing context.

### Step 2 — Create a feature branch for today's work
```bash
git checkout -b day2-backup-recovery-lab
```
- `checkout -b <name>` — creates a **new branch** and switches to it in one step.
- **Naming convention:** short, descriptive, often tied to a ticket — e.g. `day2-backup-recovery-lab`, `jira-102-backup-lab`, `fix-replication-lag`.
- **Why a new branch instead of working directly on `main`:** it keeps `main` always stable. If your work is incomplete or wrong, it never affects the trusted copy.

### Step 3 — Do your actual work
Edit files, run labs, write documentation — same as before.

### Step 4 — Check what changed
```bash
git status
```
Shows which files are modified, new, or staged. Always run this before adding — it prevents accidentally committing something you didn't mean to (like a `.pem` key file).

### Step 5 — Stage and commit
```bash
git add .
git commit -m "Day 2: backup and recovery lab — mysqldump and PITR"
```
- Write commit messages that describe **what** changed, so anyone (including future-you) can scan the history and understand progress without opening every file.

### Step 6 — Push your branch (not `main`) to GitHub
```bash
git push -u origin day2-backup-recovery-lab
```
- This uploads your **branch**, not `main`. Your work is now backed up and visible, but hasn't touched the official `main` branch yet.

### Step 7 — Merge into main (on a real team, this happens via a "Pull Request")
On a real team, you'd open a **Pull Request (PR)** on GitHub — a request asking a teammate to review your branch before it merges into `main`. They comment, you fix anything needed, then it merges.

**Since you're solo right now**, you can merge directly:
```bash
git checkout main
git pull origin main          # make sure main is current
git merge day2-backup-recovery-lab
git push origin main
```

### Step 8 — Clean up the finished branch (optional but tidy)
```bash
git branch -d day2-backup-recovery-lab              # delete locally
git push origin --delete day2-backup-recovery-lab   # delete on GitHub
```

---

## What happens when two changes conflict

If `main` and your branch both changed the same lines of the same file, Git can't automatically decide which to keep — this is a **merge conflict** (you already hit one with `README.md` earlier). Git marks the conflicting section in the file itself, and you manually decide what the final version should look like, then:
```bash
git add <the resolved file>
git commit
```

---

## Daily Routine — Quick Reference (once you're comfortable with the "why" above)

**Morning, before starting work:**
```bash
git checkout main
git pull origin main
git checkout -b <today's-branch-name>
```

**After finishing today's lab/log/ticket:**
```bash
git status
git add .
git commit -m "<clear description of today's work>"
git push -u origin <today's-branch-name>
git checkout main
git pull origin main
git merge <today's-branch-name>
git push origin main
```

**Simple solo shortcut (fine while it's just you, no team review needed):**
```bash
git add .
git commit -m "<description>"
git push
```

---

## Moving files from Downloads into your portfolio folder (the part you asked about specifically)

Whenever I send you a file, it lands in your Mac's Downloads folder. To get it into your tracked portfolio and pushed to GitHub:

```bash
# 1. Move the file(s) from Downloads into the right folder in your portfolio
mv ~/Downloads/<filename>.md "/Users/srikanthpuchakayala/Desktop/DBA /dba-mastery-portfolio/<destination-folder>/"

# 2. Go into the portfolio folder
cd "/Users/srikanthpuchakayala/Desktop/DBA /dba-mastery-portfolio"

# 3. Check what's new/changed
git status

# 4. Stage, commit, push
git add .
git commit -m "<describe what this file/update is>"
git push
```

**Example, using today's files:**
```bash
mv ~/Downloads/commands-reference.md "/Users/srikanthpuchakayala/Desktop/DBA /dba-mastery-portfolio/"
mv ~/Downloads/lab-01-setup-architecture-users.md "/Users/srikanthpuchakayala/Desktop/DBA /dba-mastery-portfolio/01-mysql-core/"
mv ~/Downloads/JIRA-101-baseline-monitoring-checklist.md "/Users/srikanthpuchakayala/Desktop/DBA /dba-mastery-portfolio/tickets/"
mv ~/Downloads/daily-dba-checklist.md "/Users/srikanthpuchakayala/Desktop/DBA /dba-mastery-portfolio/daily-logs/"
mv ~/Downloads/git-workflow-guide.md "/Users/srikanthpuchakayala/Desktop/DBA /dba-mastery-portfolio/"

cd "/Users/srikanthpuchakayala/Desktop/DBA /dba-mastery-portfolio"
git status
git add .
git commit -m "Day 1 complete: baseline checklist, DBA morning routine, Git workflow guide"
git push
```

**Note:** if a file already exists at the destination (like `commands-reference.md` does), `mv` will silently overwrite it with the Downloads version — which is what you want, since these are updated versions.
