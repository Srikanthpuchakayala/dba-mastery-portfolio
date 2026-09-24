# Commands Reference — Linux & Git (Day 1)

Plain-English explanations of every command used so far, kept here so I actually understand what I'm typing instead of copy-pasting. I'll keep adding to this file as new commands come up in later labs.

---

## Linux / Shell commands

### `df -h`
- `df` = "disk free" — shows how much storage space is used and available on each mounted filesystem.
- `-h` = "human-readable" — shows sizes as `10G`, `500M` instead of raw bytes.
- **Why a DBA runs this daily:** if the disk holding `/var/lib/mysql` fills up, MySQL can't write new data and the server effectively goes down. This is one of the most common real-world causes of a database outage.

### `sudo find /var/log -iname "*mysql*" 2>/dev/null`
- `sudo` — run as superuser/root; needed because log files are often restricted to admin access.
- `find /var/log` — search inside `/var/log` (where Linux keeps system/application log files).
- `-iname "*mysql*"` — match files whose name contains "mysql", case-insensitive (`-i`), using `*` as a wildcard for "anything before/after".
- `2>/dev/null` — redirect error output (stream 2 = stderr) to `/dev/null` (discard it), so permission-denied noise doesn't clutter the screen.
- **Why:** the exact error log path differs across Linux distributions, so you search for it instead of guessing.

### `sudo tail -100 /var/log/mysqld.log`
- `tail` — show the end of a file (opposite of `head`, which shows the start).
- `-100` — show the last 100 lines.
- **Why:** log files are chronological; the newest and most relevant entries are at the bottom, so you read from there, not the top.

### `ssh -i <key>.pem <user>@<ip>`
- `ssh` — Secure Shell; opens an encrypted remote terminal session on another machine.
- `-i <key>.pem` — "identity file"; the private key that proves who you are (like a password, but cryptographic).
- `<user>@<ip>` — which account to log in as, on which machine (by IP address).
- **Why:** the EC2 server has no keyboard or screen attached — SSH is how you get a command line on it remotely.

---

## Git commands

### `git init`
- Creates a new, empty Git repository in the current folder (a hidden `.git` folder that tracks history).
- **Why:** this is what turns a plain folder into something Git can version-control.

### `git add .`
- Stages changes — tells Git "include these files in my next commit."
- `.` means "everything in and under the current folder."
- **Why:** Git doesn't auto-save everything; you choose what to include, which avoids accidentally committing secrets or junk files.

### `git commit -m "message"`
- Saves everything staged as a permanent snapshot in the repo's history.
- `-m "message"` — a short description of what changed.
- **Why:** commits are checkpoints you can return to if something breaks later.

### `git branch -M main`
- Renames the current branch to `main`.
- `-M` — force-rename, even if a branch with that name already exists.
- **Why:** GitHub's default branch name is `main`; older Git installs default to `master`, so this aligns the two.

### `git remote add origin <url>`
- Registers a remote server (GitHub) under the nickname `origin`, pointing at that URL.
- **Why:** a local repo doesn't know about GitHub until it's told where GitHub is.

### `git push -u origin main`
- Uploads local commits to the `main` branch on the `origin` remote.
- `-u` — sets upstream tracking, so future `git push`/`git pull` know where to go automatically.
- **Why:** this is the actual "publish to GitHub" step; everything before it only existed locally.

### `git pull origin main --allow-unrelated-histories --no-rebase`
- `git pull` — fetch changes from the remote and merge them into the local branch.
- `--allow-unrelated-histories` — overrides Git's default refusal to merge two repos that don't share a common starting commit (needed because GitHub had already created content before the first local push).
- `--no-rebase` — combine the two histories with a merge commit, instead of replaying local commits on top of the remote ones.
- **Why it was needed:** GitHub's repo wasn't truly empty (it had its own initial content), so the local and remote histories had "diverged" and Git needed explicit instructions on how to reconcile them.

### `git checkout --ours README.md`
- During a merge conflict, keeps *this side's* (local) version of the named file and discards the incoming one.
- **Why:** two different `README.md` files existed (local vs. GitHub's), and this told Git which one to keep.

---

## Interview angle

**Likely question:** *"Walk me through how you'd push a config or script change using Git."*

**A real, defensible answer (not memorized — actually lived through today):**
> "I stage my changes with `git add`, commit them with a clear message, and push to the remote. If the remote has changes I don't have locally, I pull first — usually as a merge — resolve any conflicts by choosing the correct version of each file, then push again."

---

*This file grows as new commands come up in later labs — treat it as a running personal command reference, not a one-time document.*
