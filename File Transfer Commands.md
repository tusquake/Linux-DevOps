# File Transfer Commands (SCP, rsync)

> **Who is this for?** A backend developer who can now work on a single Linux server and needs to **move files between machines**: upload a build to production, download logs for analysis, back up data to another server, or migrate an application to a new host. This guide covers the secure, standard tools for the job: `scp`, `sftp`, and `rsync`.

---

## What is it and Why do we need it?

### What are file transfer commands?

They are command-line tools that **copy files from one computer to another over a network**, usually over **SSH** (the secure connection you learned about in Topics 1 and 2). Because they ride on SSH, everything is **encrypted** in transit and uses the **same login credentials or keys** you already use to connect to the server.

| Tool | What it is | Best for |
|------|-----------|----------|
| `scp` | **S**ecure **C**o**p**y: a simple "copy over SSH" command | Quick, one-off copies of a few files |
| `sftp` | **S**SH **F**ile **T**ransfer **P**rotocol: an interactive file browser over SSH | Browsing a remote server and picking files by hand |
| `rsync` | **R**emote **sync**hronization: copies only the **differences** between two locations | Backups, deployments, migrations, large or repeated transfers |

> **Analogy: Moving Things Between Two Houses**
> - **`scp`** is **carrying a box to the other house in your car**: simple and direct. If the trip is interrupted, you start the whole trip again.
> - **`sftp`** is **walking through the other house** with a guide, picking up or dropping off specific items.
> - **`rsync`** is a **smart mover who compares both houses' inventory lists** and carries over only the new or changed items, and can even remove things that no longer belong. The second move takes a fraction of the time because nothing is repeated.

### Why do we need them?

Real DevOps work is full of file movement:

- Copying an application build or a configuration file to a server.
- Pulling logs from production to investigate an incident.
- Backing up data to a second machine so that one failure cannot destroy everything.
- Migrating a site from an old server to a new one.
- Keeping two directories in sync (for example, an uploads folder shared across servers).

Doing this with email, USB drives, or insecure FTP is slow, risky, and not automatable. SSH-based tools are secure, scriptable, and available on virtually every Linux server.

---

## Core Concepts

### 1. How These Tools Work (The SSH Foundation)

All three tools open an **SSH connection** to the remote machine and move data through it.

```
 Your laptop                                   Remote server
 +-----------+      encrypted SSH tunnel       +-----------+
 |  file.txt | ==============================> | /home/... |
 +-----------+     (port 22 by default)        +-----------+
```

So the requirements are the same as for logging in with SSH:

- The remote server must be reachable (network + firewall allow port 22).
- You need a valid **username** and either a **password** or an **SSH key**.
- The remote user must have **permission** to read or write the files involved (Topic 6).

---

### 2. The Remote Path Syntax

Locations on another machine are written as:

```
user@host:/path/on/remote
  |    |        |
  |    |        +-- the path ON THE REMOTE machine
  |    +-- hostname or IP address of the remote machine
  +-- the remote user name
```

A path **without** a `host:` part is a local path. The rule of thumb: **the colon `:` marks the remote side.**

| Direction | Pattern | Description |
|-----------|---------|-------------|
| **Upload** (local to remote) | `tool  local-file  user@host:/remote/dir/` | Push |
| **Download** (remote to local) | `tool  user@host:/remote/file  local-dir/` | Pull |

> **Analogy:** It's like addressing a parcel: `user@host:` is the street address; everything after the colon is the room inside the building.

---

### 3. `scp`: Strengths and Limits

| Strengths | Limits |
|-----------|--------|
| Very simple syntax | Always copies the **whole file**, even if only one line changed |
| Installed everywhere SSH is | Cannot **resume** an interrupted transfer |
| Fine for small, occasional copies | No built-in way to **exclude** files or **delete** extras |
| | Poor feedback and no dry-run option |

Modern OpenSSH documentation describes the original `scp` protocol as outdated and recommends `sftp` or `rsync` for new workflows. Recent versions of the `scp` command still work with the same syntax (they now use the SFTP protocol behind the scenes), so everyday usage does not change.

---

### 4. `rsync`: The Delta-Transfer Idea

`rsync` compares the **source** and **destination**, and transfers only what is **new or changed**. It checks, by default, the file's **size and modification time**. For files that did change, its algorithm sends only the changed portions.

| Feature | Benefit |
|---------|---------|
| **Delta transfer** | Repeated syncs are very fast |
| **Preserves metadata** (`-a`) | Keeps permissions, timestamps, symlinks, owner, and group |
| **Resume support** (`--partial`) | Continue large transfers after an interruption |
| **Include / exclude rules** | Skip logs, caches, `node_modules`, and so on |
| **Mirror mode** (`--delete`) | Make the destination an exact copy of the source |
| **Dry run** (`-n`) | Preview exactly what would happen before doing it |
| **Compression** (`-z`) | Saves bandwidth on slow links |

Requirement: `rsync` must be installed on **both** machines (`sudo apt install rsync` on Debian/Ubuntu, `sudo dnf install rsync` on Red Hat family).

---

### 5. The Trailing Slash Rule (The Most Common rsync Gotcha)

A trailing `/` on the **source** changes the meaning:

| Command | Result |
|---------|--------|
| `rsync -a myapp/  /backup/` | Copies the **contents** of `myapp` **into** `/backup/` (files appear directly in `/backup/`) |
| `rsync -a myapp   /backup/` | Copies the **folder itself**, so you get `/backup/myapp/` |

The slash on the **destination** does not change the meaning; it only emphasizes that it's a directory.

> **Analogy:** "Move **the contents of** the box onto the shelf" (`box/`) versus "Move **the box** onto the shelf" (`box`).

---

### 6. Make SSH Easier: Keys, Ports, and the Config File

Typing `-i key.pem -P 2222 user@203.0.113.25` repeatedly is tedious and error-prone. Define a **Host alias** once in `~/.ssh/config`:

```text
Host prod-web
    HostName 203.0.113.25
    User deploy
    Port 22
    IdentityFile ~/.ssh/prod-key.pem
```

After this, `ssh prod-web`, `scp file prod-web:/tmp/`, and `rsync -av dir/ prod-web:/srv/dir/` all work with no extra flags. The file must have strict permissions: `chmod 600 ~/.ssh/config`.

---

### 7. A Warning About Port Flags

Different tools use different letters for the **port**. This is a classic trap:

| Tool | Port flag | Example |
|------|-----------|---------|
| `ssh` | `-p` (lowercase) | `ssh -p 2222 user@host` |
| `scp` | `-P` (**uppercase**) | `scp -P 2222 file user@host:/tmp/` |
| `sftp` | `-P` (**uppercase**) | `sftp -P 2222 user@host` |
| `rsync` | via `-e "ssh -p 2222"` | `rsync -e "ssh -p 2222" ...` |

---

## Practical Examples & Commands

> Replace `203.0.113.25` with your server's IP, `ubuntu` with your username, and `~/.ssh/devops-key.pem` with your key. Create a practice folder first: `mkdir -p ~/lab/site && cd ~/lab`.

### 1. `scp`: Upload (Local to Remote)

```bash
scp report.txt ubuntu@203.0.113.25:/home/ubuntu/
# Copy ONE local file to the remote user's home directory.
#   report.txt                        -> the local file
#   ubuntu@203.0.113.25:/home/ubuntu/ -> remote user, host, and destination folder

scp -i ~/.ssh/devops-key.pem report.txt ubuntu@203.0.113.25:/home/ubuntu/
# -i points to your private key file (as on AWS EC2 servers).

scp -P 2222 report.txt ubuntu@203.0.113.25:/home/ubuntu/
# -P (UPPERCASE) specifies a non-standard SSH port.

scp a.txt b.txt c.txt ubuntu@203.0.113.25:/home/ubuntu/backup/
# Copy several files at once; the LAST argument is always the destination.

scp *.log ubuntu@203.0.113.25:/tmp/logs/
# Wildcards expand locally, so all local .log files are copied.

scp -r site/ ubuntu@203.0.113.25:/home/ubuntu/
# -r = recursive: required to copy a whole DIRECTORY and its contents.

scp -p report.txt ubuntu@203.0.113.25:/home/ubuntu/
# -p preserves the original modification times and permissions.

scp -C bigfile.sql ubuntu@203.0.113.25:/tmp/
# -C compresses data in transit (helps on slow links, text and SQL; little use for already-compressed files).

scp -l 2000 bigfile.iso ubuntu@203.0.113.25:/tmp/
# -l limits bandwidth, in Kbit/s (here about 2 Mbit/s), so you don't saturate a shared connection.

scp -v report.txt ubuntu@203.0.113.25:/tmp/
# -v shows verbose debugging output; invaluable when a connection fails.
```

### 2. `scp`: Download (Remote to Local)

```bash
scp ubuntu@203.0.113.25:/var/log/nginx/error.log .
# Copy a remote file into the CURRENT local directory (the dot "." means "here").

scp ubuntu@203.0.113.25:/var/log/nginx/error.log ~/Downloads/server1-error.log
# Download and rename it at the same time.

scp -r ubuntu@203.0.113.25:/var/www/myapp ~/backups/
# Download a whole remote directory.

scp 'ubuntu@203.0.113.25:/var/log/myapp/*.log' ./logs/
# Quote remote wildcards so YOUR shell does not try to expand them locally.

scp "ubuntu@203.0.113.25:/home/ubuntu/My Documents/file.txt" .
# Quote paths that contain spaces.
```

### 3. `scp`: Between Two Remote Servers and Through a Jump Host

```bash
scp ubuntu@server1:/data/export.sql ubuntu@server2:/data/
# Copy from one remote server to another. In modern OpenSSH, the data flows through
# YOUR machine by default, so make sure you have enough bandwidth.

scp -J ubuntu@bastion.example.com ubuntu@10.0.1.15:/var/log/app.log .
# -J = jump host. Many production servers have only private IPs and can be reached
# only via a "bastion" (jump) server.
```

### 4. `sftp`: Interactive Transfer Session

```bash
sftp ubuntu@203.0.113.25
# Open an interactive session. You will see a prompt:  sftp>
```

Commands you type inside the `sftp>` prompt:

```text
ls                      List files on the REMOTE server
lls                     List files on your LOCAL machine
pwd                     Show the remote directory
lpwd                    Show the local directory
cd /var/log             Change the remote directory
lcd ~/Downloads         Change the local directory
get error.log           Download a file
get -r logs             Download a directory recursively
put report.txt          Upload a file
put -r site             Upload a directory recursively
mkdir backup            Create a remote directory
rm oldfile.txt          Delete a remote file
help                    Show all commands
exit                    Leave the session (or: bye)
```

### 5. `rsync`: The Essential Forms

```bash
rsync -av site/ /tmp/site-copy/
# LOCAL copy of one folder to another.
#   -a  archive mode: recursive + preserves permissions, timestamps, symlinks, owner, group
#   -v  verbose: lists the files being transferred

rsync -avz site/ ubuntu@203.0.113.25:/home/ubuntu/site/
# UPLOAD to a remote server.
#   -z  compress data during transfer (good for text; skip it for already-compressed files)

rsync -avz ubuntu@203.0.113.25:/var/www/myapp/ ~/backups/myapp/
# DOWNLOAD from a remote server.

rsync -avzh --progress site/ ubuntu@203.0.113.25:/home/ubuntu/site/
# -h  human-readable sizes (KB/MB/GB)
# --progress  show per-file progress and speed

rsync -avzP site/ ubuntu@203.0.113.25:/home/ubuntu/site/
# -P is a shortcut for: --partial --progress
# "--partial" keeps half-transferred files so an interrupted transfer can RESUME
# instead of restarting from zero. Perfect for large files on unstable connections.
```

### 6. `rsync`: Always Preview First with a Dry Run

```bash
rsync -avzn site/ ubuntu@203.0.113.25:/home/ubuntu/site/
# -n (or --dry-run): shows what WOULD be transferred without changing anything.

rsync -avzn --itemize-changes site/ ubuntu@203.0.113.25:/home/ubuntu/site/
# -i / --itemize-changes: prints a short code per file explaining WHY it would be transferred
# (new file, changed size, changed time, changed permissions, and so on).
```

### 7. `rsync`: Excluding and Including Files

```bash
rsync -avz --exclude='*.log' --exclude='node_modules/' --exclude='.git/' \
  myapp/ ubuntu@203.0.113.25:/var/www/myapp/
# Skip logs, dependencies, and Git history. A trailing slash matches only directories.
# The backslash \ at the end of a line continues the command on the next line.

rsync -avz --exclude-from='exclude-list.txt' myapp/ ubuntu@203.0.113.25:/var/www/myapp/
# Keep many exclusion patterns in a file (one per line).

rsync -avz --include='*.conf' --exclude='*' /etc/myapp/ ~/conf-backup/
# Copy ONLY .conf files: include rules come first, then exclude everything else.
```

### 8. `rsync`: Mirroring with `--delete` (Powerful and Dangerous)

```bash
rsync -avzn --delete site/ ubuntu@203.0.113.25:/var/www/site/
# STEP 1: always dry-run first. Lines starting with "deleting" are files that would be REMOVED.

rsync -avz --delete site/ ubuntu@203.0.113.25:/var/www/site/
# STEP 2: make the destination an EXACT mirror of the source.
# Files that exist at the destination but NOT in the source are deleted.
```

> **WARNING:** `--delete` removes data at the destination. A wrong source path, a missing trailing slash, or an unmounted source folder can wipe your destination. **Always run with `-n` first**, and double-check both paths. Consider `--backup --backup-dir=/path/to/trash` to keep deleted or overwritten files in a safety folder.

### 9. `rsync`: Custom SSH Port, Key, and Remote Sudo

```bash
rsync -avz -e "ssh -i ~/.ssh/devops-key.pem -p 2222" site/ ubuntu@203.0.113.25:/home/ubuntu/site/
# -e tells rsync which remote shell to use and with which options (key, port).

rsync -avz -e "ssh -J ubuntu@bastion.example.com" site/ ubuntu@10.0.1.15:/home/ubuntu/site/
# Reach a private server through a bastion host.

rsync -avz --rsync-path="sudo rsync" config/ ubuntu@203.0.113.25:/etc/myapp/
# Run rsync as root on the REMOTE side so it can write to root-owned folders such as /etc.
# The remote user needs passwordless sudo permission for rsync.

rsync -avz site/ prod-web:/srv/site/
# If you defined "Host prod-web" in ~/.ssh/config, no extra flags are needed.
```

### 10. `rsync`: Limiting, Verifying, and Cleaning Up

```bash
rsync -avz --bwlimit=5000 bigdata/ ubuntu@203.0.113.25:/data/
# Limit bandwidth to about 5000 KB/s so you do not choke production traffic.

rsync -avz --checksum site/ ubuntu@203.0.113.25:/home/ubuntu/site/
# -c compares file CHECKSUMS rather than size and time. Slower but thorough.

rsync -av --remove-source-files outbox/ ubuntu@203.0.113.25:/data/inbox/
# After each file transfers successfully, delete it from the SOURCE (a "move" operation).

rsync -av --stats site/ ubuntu@203.0.113.25:/home/ubuntu/site/
# Print a summary: number of files, bytes transferred, and speed-up.

rsync -au source/ dest/
# -u skips files that are NEWER at the destination, to avoid overwriting recent edits.
```

### 11. `rsync`: Space-Efficient Incremental Backups with `--link-dest`

```bash
rsync -a --link-dest=/backup/2025-01-09 /var/www/myapp/ /backup/2025-01-10/
# Create today's backup folder as a full-looking snapshot.
# Files UNCHANGED since the previous backup are hard-linked (they take no extra disk space);
# only changed files are copied. Every day looks like a full backup, but costs only the changes.
```

### 12. Streaming a Directory over SSH with `tar` (When rsync Is Not Installed)

```bash
tar -czf - myapp/ | ssh ubuntu@203.0.113.25 "tar -xzf - -C /var/www/"
# Left side: tar creates a compressed archive and writes it to STANDARD OUTPUT ("-").
# The pipe sends it through SSH. Right side: the remote tar reads from STANDARD INPUT ("-")
# and extracts into /var/www/. No temporary archive file is needed on either machine.
```

### 13. Set Up Passwordless Transfers with SSH Keys

```bash
ssh-keygen -t ed25519 -C "alice@laptop"
# Create a modern key pair. Accept the defaults. Use a passphrase for protection.
# Creates ~/.ssh/id_ed25519 (PRIVATE: never share) and ~/.ssh/id_ed25519.pub (public).

ssh-copy-id ubuntu@203.0.113.25
# Install your PUBLIC key into the server's ~/.ssh/authorized_keys.
# After this, scp, sftp, and rsync work without typing a password.

ssh ubuntu@203.0.113.25 "echo connected as \$(whoami)"
# Test that key-based login works.
```

### 14. Verify the Transfer Arrived Intact

```bash
sha256sum bigfile.iso
# Print a checksum on the source machine.

ssh ubuntu@203.0.113.25 "sha256sum /tmp/bigfile.iso"
# Print the checksum of the copy on the destination. The two long strings must match exactly.

ls -lh bigfile.iso
ssh ubuntu@203.0.113.25 "ls -lh /tmp/bigfile.iso"
# A quick sanity check: the sizes should be identical.
```

### 15. Troubleshooting Common Errors

| Error | Likely cause | Fix |
|-------|--------------|-----|
| `Permission denied (publickey)` | Wrong key, wrong user, or bad key permissions | Use `-i`, check the username, run `chmod 600` on the key |
| `Permission denied` on a remote path | The remote user cannot write there | Choose a path you own (such as `/tmp` or home), or fix permissions (Topic 6) |
| `Connection timed out` | Firewall or security group blocks port 22, or wrong IP | Check the cloud security group, firewall, and IP address |
| `Connection refused` | SSH service is not running, or wrong port | `systemctl status ssh` on the server; check the port |
| `No such file or directory` | Typo in a path, or the destination directory does not exist | Verify with `ssh host ls /path`; create the folder first |
| `rsync: command not found` | rsync is not installed on one side | `sudo apt install rsync` on the remote machine |
| `Host key verification failed` | The server's fingerprint changed (rebuilt server, or possible attack) | Verify it is legitimate, then `ssh-keygen -R <host>` |

---

## Real-World Use Case

### Scenario: Migrating a live application to a new server with minimal downtime

Your company is moving `myapp` (code plus 50 GB of user uploads) from an old server (`old-srv`) to a new, larger one (`new-srv`). You want users to experience only a few minutes of downtime, not hours.

```
    old-srv (live)                              new-srv (empty)
   +------------------+                        +------------------+
   | /var/www/myapp   |  1) rsync (app LIVE)   | /var/www/myapp   |
   | 50 GB uploads    | ---------------------> | (bulk copy)      |
   |                  |  2) stop app           |                  |
   |                  |  3) rsync FINAL sync   |                  |
   |                  | ---------------------> | (only changes)   |
   +------------------+  4) start new + DNS    +------------------+
```

**Step 1: Bulk copy while the old server is still serving users**

```bash
# Run from the NEW server, pulling from the old one, inside tmux or screen so it survives disconnects
rsync -avzP --exclude='logs/' --exclude='cache/' \
  -e "ssh -i ~/.ssh/migrate-key.pem" \
  ubuntu@old-srv:/var/www/myapp/ /var/www/myapp/
# This may take hours for 50 GB, but the live site keeps running. Nothing is disrupted.
```

**Step 2: Repeat the sync to shrink the remaining differences**

```bash
# Run again (even several times). Only files changed since the last run are transferred,
# so each pass is faster.
rsync -avzP --exclude='logs/' --exclude='cache/' \
  -e "ssh -i ~/.ssh/migrate-key.pem" \
  ubuntu@old-srv:/var/www/myapp/ /var/www/myapp/
```

**Step 3: Maintenance window. Freeze the old app and do a final sync**

```bash
ssh old-srv "sudo systemctl stop myapp"     # Stop writes on the old server

# FINAL sync: dry-run first, then real, now with --delete so the copy is exact
rsync -avzn --delete --exclude='logs/' --exclude='cache/' \
  -e "ssh -i ~/.ssh/migrate-key.pem" ubuntu@old-srv:/var/www/myapp/ /var/www/myapp/
rsync -avz  --delete --exclude='logs/' --exclude='cache/' \
  -e "ssh -i ~/.ssh/migrate-key.pem" ubuntu@old-srv:/var/www/myapp/ /var/www/myapp/
# This final pass moves only the last few minutes of changes: it takes seconds or minutes.
```

**Step 4: Verify, fix ownership, and go live**

```bash
du -sh /var/www/myapp                                  # Compare sizes with the old server
ssh old-srv "du -sh /var/www/myapp"
sudo chown -R deploy:www-data /var/www/myapp           # Ensure correct ownership (Topic 6)
sudo systemctl start myapp                             # Start the app on the new server
curl -I http://localhost:3000/health                   # Smoke test
# Finally, update DNS (Topic 1) to point to the new server's IP.
```

**Ongoing: Nightly offsite backup with rsync and cron**

```bash
# crontab entry (see Topic 4): every night at 03:00
0 3 * * * rsync -az --delete -e "ssh -i /home/backup/.ssh/backup-key" /var/www/myapp/ backup@backup-srv:/backups/myapp/ >> /var/log/rsync-backup.log 2>&1
```

**Why this matters in production:**

- **Minimal downtime:** the heavy lifting happens while the site is up; the outage only covers the last small delta.
- **Safety:** `--dry-run`, checksums, and verification steps catch mistakes before they become incidents.
- **Resumability:** `--partial` means a dropped connection at 90% does not cost you hours.
- **Efficiency:** repeated syncs and nightly backups only move what changed, saving bandwidth and time.

> **Analogy:** Moving a restaurant to a bigger building. You carry over the heavy equipment during the week while the restaurant still serves customers. On the closing night, you only move the last perishable items, then open the doors in the new place the next morning.

---

## Interview Questions & Answers

### Q1. What is the difference between `scp` and `rsync`, and when would you choose each?

**Answer:**
Both transfer files securely over SSH. `scp` is simple and always copies the **entire file**, so I use it for quick, one-off copies of a few files. `rsync` uses **delta transfer**: it compares source and destination and sends only new or changed data, supports **resuming** interrupted transfers (`--partial`), **include/exclude** rules, **dry runs** (`-n`), **mirroring** (`--delete`), and **preserving metadata** (`-a`). I choose `rsync` for backups, deployments, large directories, migrations, and anything repeated. The trade-off is that `rsync` must be installed on both ends and has more options to understand.

---

### Q2. What is the difference between `rsync -a src/ dest/` and `rsync -a src dest/`?

**Answer:**
The trailing slash on the **source** matters. With `src/`, rsync copies the **contents** of `src` directly into `dest/`. Without the slash, it copies the **directory itself**, creating `dest/src/`. Getting this wrong can produce an extra nested folder or, combined with `--delete`, unintentionally remove files. A trailing slash on the destination does not change the behavior.

---

### Q3. What does `rsync -avz --delete src/ user@host:/dest/` do, and how do you make it safe to run?

**Answer:**
`-a` is archive mode (recursive; preserves permissions, timestamps, symlinks, owner, and group), `-v` is verbose, `-z` compresses data in transit, and `--delete` removes files from the destination that no longer exist in the source, producing an exact mirror. Because `--delete` can destroy data if a path is wrong, I **always run it first with `-n` (`--dry-run`)** and review the output, confirm the trailing slashes and both paths, and often add `--backup --backup-dir=...` so deleted or overwritten files go to a safety folder. For critical systems I also keep separate versioned backups.

---

### Q4. How do you use a specific SSH key and a non-standard port with `scp` and with `rsync`?

**Answer:**
For `scp`: `scp -i ~/.ssh/key.pem -P 2222 file user@host:/path/`. Note the **uppercase `-P`** for the port, whereas `ssh` uses lowercase `-p`. For `rsync`, the SSH options are passed through the `-e` flag: `rsync -avz -e "ssh -i ~/.ssh/key.pem -p 2222" src/ user@host:/path/`. A cleaner, long-term approach is to define a `Host` entry in `~/.ssh/config` with `HostName`, `User`, `Port`, and `IdentityFile`, after which `scp file myserver:/path/` and `rsync -av src/ myserver:/path/` need no extra flags.

---

### Q5. A 100 GB transfer keeps getting interrupted by network drops. How would you complete it reliably and confirm the data is intact?

**Answer:**
I would use `rsync -avzP` (or `rsync -av --partial --progress`) instead of `scp`, because `--partial` keeps the partly transferred files so that re-running the **same command** resumes where it left off rather than restarting. I would run it inside `tmux` or `screen` (or with `nohup`) so that closing my SSH session does not kill it, and possibly use `--bwlimit` to avoid saturating the link. Once finished, I'd re-run `rsync -avn --checksum` (a dry run that compares checksums) and expect no files to be listed, and/or compare `sha256sum` outputs and file sizes on both machines to confirm the data arrived intact.

---

## Quick Recap Cheat Sheet

| Task | Command |
|------|---------|
| Upload a file | `scp file user@host:/path/` |
| Download a file | `scp user@host:/path/file .` |
| Copy a directory | `scp -r dir user@host:/path/` |
| Use key / port with scp | `scp -i key.pem -P 2222 file user@host:/path/` |
| Copy via jump host | `scp -J user@bastion file user@private:/path/` |
| Interactive transfer | `sftp user@host` (then `get`, `put`, `ls`, `lls`) |
| Sync a folder (remote) | `rsync -avz src/ user@host:/dest/` |
| Resume large transfers | `rsync -avzP src/ user@host:/dest/` |
| Preview changes | `rsync -avzn src/ user@host:/dest/` |
| Mirror exactly | `rsync -avz --delete src/ user@host:/dest/` |
| Exclude files | `rsync -avz --exclude='*.log' src/ dest/` |
| Key and port with rsync | `rsync -avz -e "ssh -i key.pem -p 2222" src/ user@host:/dest/` |
| Write to root-owned remote path | `rsync -avz --rsync-path="sudo rsync" src/ user@host:/dest/` |
| Limit bandwidth | `rsync -avz --bwlimit=5000 src/ dest/` |
| Incremental snapshots | `rsync -a --link-dest=/prev src/ /new/` |
| Passwordless login | `ssh-keygen -t ed25519`, `ssh-copy-id user@host` |
| Verify integrity | `sha256sum file` (compare on both machines) |

**Remember:**
- `scp` and `sftp` use **`-P`** (uppercase) for the port; `ssh` uses **`-p`**.
- A trailing slash on the rsync **source** means "the contents of".
- **Dry-run first** (`-n`) whenever you use `--delete`.
