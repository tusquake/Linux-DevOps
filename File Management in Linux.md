# File Management in Linux (Permissions, Ownership, Umask, Zip, Tar)

> **Who is this for?** A backend developer who can create users and groups (Topic 5) and now needs to control **who can read, change, or run each file**, and how to **package and compress files** for backups and transfers. "Permission denied" is the most common error you will ever see on Linux. After this guide, you will know exactly why it happens and how to fix it correctly.

---

## What is it and Why do we need it?

### What is file management here?

This topic covers two related skills:

1. **Access control on files:** permissions (who can do what), ownership (who the file belongs to), and umask (the default permissions for new files).
2. **Packaging and compression:** bundling many files into one archive (`tar`) and shrinking them (`gzip`, `zip`) for backups, deployments, and downloads.

> **Analogy: A Building with Rooms and Moving Boxes**
> Every file and folder is a **room** in a building. **Ownership** says whose room it is. **Permissions** are the lock settings: the owner can enter and rearrange, the owner's team can only look around, and strangers are kept out. **umask** is the building's default lock policy applied to every new room. **tar** is packing the contents of a room into a single cardboard box so it is easy to carry. **gzip/zip** is vacuum-sealing that box so it takes up less space.

### Why do we need it?

| Problem | How this topic solves it |
|---------|--------------------------|
| A hacked web app reads your database password file | Strict permissions (`chmod 600 .env`) so only the right user can read it |
| Developers overwrite each other's work | Group ownership and shared-folder permissions |
| A deploy fails with "Permission denied" | You can diagnose it in seconds with `ls -l` and `namei` |
| You need to back up a project before an upgrade | `tar -czf` creates a single compressed backup file |
| You must send logs to a colleague | Compress them to a fraction of the size |
| SSH refuses your key | SSH requires strict permissions (`600`/`700`); now you will know why |

---

## Core Concepts

### 1. Reading the Permission String

Run `ls -l` and look at the first column:

```text
-rwxr-xr--  1  alice  devs  4096  Jan 10 09:15  deploy.sh
|\_/\_/\_/      |      |
| |  |  |      |      +-- GROUP owner
| |  |  |      +-- USER (owner)
| |  |  +-- permissions for OTHERS (everyone else)
| |  +-- permissions for the GROUP
| +-- permissions for the USER (owner)
+-- file type
```

**First character: file type**

| Symbol | Type |
|--------|------|
| `-` | Regular file |
| `d` | Directory |
| `l` | Symbolic link |
| `c` / `b` | Character / block device |

**The next nine characters: three sets of three**

| Position | Applies to | Symbol meaning |
|----------|------------|----------------|
| 1-3 | **u** (user / owner) | `r` read, `w` write, `x` execute, `-` not allowed |
| 4-6 | **g** (group) | same |
| 7-9 | **o** (others / everyone else) | same |

`a` means **all** three (user, group, and others).

---

### 2. What `r`, `w`, `x` Mean (Files vs. Directories)

This difference trips up many beginners.

| Permission | On a **file** | On a **directory** |
|------------|---------------|--------------------|
| **r** (read) | View the contents | **List** the names inside (`ls`) |
| **w** (write) | Modify the contents | **Create, delete, or rename** entries inside (also requires `x`) |
| **x** (execute) | Run it as a program or script | **Enter** it (`cd`) and access items inside by path |

> **Analogy:** A directory is a **room**. `r` lets you look through the window and read the labels on the shelves. `x` lets you actually walk in. `w` lets you add or remove things on the shelves. Without `x`, the door is locked even if you can see through the window.

Two important consequences:

- To delete a file, you need **write permission on the directory** that contains it, not on the file itself.
- To reach `/var/www/app/file.txt`, you need `x` on **every folder along the path**.

---

### 3. Numeric (Octal) Permissions

Each permission has a number value. Add them up per set.

| Permission | Value |
|------------|-------|
| `r` | 4 |
| `w` | 2 |
| `x` | 1 |
| none | 0 |

| Combination | Calculation | Digit |
|-------------|-------------|-------|
| `rwx` | 4 + 2 + 1 | **7** |
| `rw-` | 4 + 2 | **6** |
| `r-x` | 4 + 1 | **5** |
| `r--` | 4 | **4** |
| `---` | 0 | **0** |

So **`754`** means: user `rwx` (7), group `r-x` (5), others `r--` (4).

**Permissions you will use all the time:**

| Mode | Meaning | Typical use |
|------|---------|-------------|
| `600` | `rw-------` | Private files: SSH private keys, `.env` files |
| `644` | `rw-r--r--` | Normal files everyone can read |
| `640` | `rw-r-----` | Config files readable by a group |
| `700` | `rwx------` | Private directories (`~/.ssh`) |
| `755` | `rwxr-xr-x` | Directories and public scripts/programs |
| `775` | `rwxrwxr-x` | Shared team directories |
| `777` | `rwxrwxrwx` | Everyone can do everything. **Almost always a mistake** |

> **Never "fix" an error with `chmod 777`.** It is the equivalent of removing every lock in the building because one key was lost. Find the real cause instead.

---

### 4. Ownership

Every file has **one owner (user)** and **one group owner**. Permissions are checked in this order:

1. If you are the **owner**, the **user** permissions apply (and nothing else).
2. Otherwise, if you are in the file's **group**, the **group** permissions apply.
3. Otherwise, the **others** permissions apply.
4. **root** bypasses these checks.

> **Analogy:** The security guard checks: "Are you the room's owner? No. Are you on the owner's team? No. Then you get the visitor rules."

---

### 5. umask: The Default Permissions for New Files

When you create a file or folder, the system starts with a maximum default and **subtracts** the umask:

| Item | Starting maximum |
|------|------------------|
| New **file** | `666` (`rw-rw-rw-`); files are never created executable by default |
| New **directory** | `777` (`rwxrwxrwx`) |

The **umask** is a mask of permissions to **remove**.

| umask | New file | New directory | Meaning |
|-------|----------|---------------|---------|
| `022` | `644` | `755` | Others and group cannot write (common default) |
| `002` | `664` | `775` | Group can write too (common on systems with per-user groups, such as Ubuntu for regular users) |
| `077` | `600` | `700` | Only the owner has any access (very secure) |

Quick math with umask `022`: directory `777 - 022 = 755`. File `666 - 022 = 644`.

(Technically it is a bitwise operation rather than plain subtraction, but subtraction gives the right answer in everyday cases.)

> **Analogy:** The umask is the **default lock policy** printed on the form: "Unless told otherwise, new rooms are locked against writes by outsiders."

---

### 6. Special Permissions: setuid, setgid, and the Sticky Bit

A fourth (leading) digit adds three special behaviors:

| Bit | Value | Symbol | On a **file** | On a **directory** | Example |
|-----|-------|--------|---------------|--------------------|---------|
| **setuid** | 4 | `s` in the user execute slot | Program runs with the **owner's** privileges, not the caller's | No effect | `/usr/bin/passwd` (lets normal users update `/etc/shadow`) |
| **setgid** | 2 | `s` in the group execute slot | Program runs with the **group's** privileges | New files **inherit the directory's group** | Shared team folders |
| **sticky bit** | 1 | `t` in the others execute slot | No effect | Users can delete **only their own** files, even if the directory is world-writable | `/tmp` |

Example: `/tmp` shows as `drwxrwxrwt`. Everyone can write there, but the `t` stops you from deleting someone else's files.

> **Analogy:**
> - **setuid** is a **valet key with special power**: the program temporarily acts with the owner's authority.
> - **setgid on a folder** is a **team folder rule**: everything placed inside is automatically labeled with the team's name.
> - **Sticky bit** is a **shared fridge** in an office: everyone can put food in, but you can only throw away your own.

If the symbol shows as a capital `S` or `T`, it means the special bit is set but the underlying execute bit is not.

---

### 7. Archiving vs. Compression (Two Different Ideas)

| Idea | Tool | What it does | Analogy |
|------|------|--------------|---------|
| **Archiving** | `tar` | Bundles many files and folders into **one file**, without shrinking them | Putting many items into one cardboard box |
| **Compression** | `gzip`, `bzip2`, `xz` | Makes a single file **smaller** | Vacuum-sealing the box |
| **Both together** | `zip`; or `tar` with `-z`/`-j`/`-J` | Bundles and compresses | A vacuum-sealed box |

The names tell you what you have:

| Extension | Meaning |
|-----------|---------|
| `.tar` | Archive only |
| `.tar.gz` or `.tgz` | tar archive compressed with gzip (most common on Linux) |
| `.tar.bz2` | tar archive compressed with bzip2 |
| `.tar.xz` | tar archive compressed with xz |
| `.gz` | A single file compressed with gzip |
| `.zip` | Zip archive (common for sharing with Windows users) |

**Compression algorithm comparison:**

| Tool | Speed | Size reduction | Typical use |
|------|-------|----------------|-------------|
| `gzip` | Fast | Good | Default choice, logs, backups |
| `bzip2` | Slower | Better | When size matters more than time |
| `xz` | Slowest | Best | Software distribution, long-term archives |
| `zip` | Fast | Good | Cross-platform sharing |

---

## Practical Examples & Commands

> Practice in a throwaway folder: `mkdir ~/lab && cd ~/lab`

### 1. Inspect Permissions and Ownership

```bash
ls -l
# Long listing: type, permissions, owner, group, size, date, name.

ls -ld /var/www
# -d shows information about the DIRECTORY ITSELF rather than listing its contents.

stat deploy.sh
# Detailed info, including permissions in BOTH numeric (0754) and symbolic form.

stat -c "%a %U:%G %n" deploy.sh
# Compact custom format: numeric mode, owner:group, name. Handy in scripts.

file deploy.sh
# Identify what type of file it really is (script, binary, text...).

namei -l /var/www/myapp/config.yaml
# Show the permissions of EVERY folder along the path. Excellent for "Permission denied" mysteries.

id
# Check which user and groups YOU are acting as.
```

### 2. Change Permissions: `chmod`

#### Symbolic mode (readable: "who, operation, what")

```bash
chmod u+x deploy.sh
# u = user (owner), + add, x execute: "give the owner execute permission."

chmod g-w config.yaml
# Remove write permission from the group.

chmod o-rwx secrets.txt
# Remove ALL permissions from others.

chmod a+r readme.txt
# Add read permission for all (a = user, group, and others).

chmod u=rwx,g=rx,o=r deploy.sh
# Set exact permissions in one go (= replaces whatever was there).

chmod ug+rw,o-rwx shared.txt
# Combine multiple changes with commas.
```

#### Numeric mode (fast, precise)

```bash
chmod 755 deploy.sh
# rwxr-xr-x: the owner can do everything; everyone else can read and run.

chmod 644 index.html
# rw-r--r--: typical for web content files.

chmod 600 ~/.ssh/id_ed25519
# rw-------: required for private keys. SSH refuses to use keys that others can read.

chmod 700 ~/.ssh
# rwx------: only you can enter the .ssh folder.

chmod 400 devops-key.pem
# r--------: read-only for the owner (the AWS key pair recommendation).
```

#### Recursive changes

```bash
chmod -R 755 /var/www/myapp
# -R applies to the folder and EVERYTHING inside. Careful: it also marks every file executable.

# Better: set directories and files differently
find /var/www/myapp -type d -exec chmod 755 {} \;
# Find only directories and set them to 755 (rwxr-xr-x).

find /var/www/myapp -type f -exec chmod 644 {} \;
# Find only regular files and set them to 644 (rw-r--r--).

chmod -R u=rwX,g=rX,o= /var/www/myapp
# Capital X = set execute ONLY on directories (and on files that are already executable).
# A neat one-liner alternative to the two find commands above.
```

### 3. Change Ownership: `chown` and `chgrp`

```bash
sudo chown alice deploy.sh
# Change the owner to alice. (Only root can give files away.)

sudo chown alice:devs deploy.sh
# Change owner AND group in one command (user:group).

sudo chown :devs deploy.sh
# Change only the group (note the colon before the name).

sudo chgrp devs deploy.sh
# Another way to change only the group.

sudo chown -R www-data:www-data /var/www/myapp/uploads
# -R recursively changes everything inside the folder.

sudo chown --reference=file1.txt file2.txt
# Copy the owner and group from one file to another.
```

> **Remember:** `chmod` changes **what is allowed**. `chown` changes **who it belongs to**.

### 4. Special Permissions in Practice

```bash
# setuid: run as the file's owner (rarely needed; a security-sensitive feature)
chmod u+s program        # or: chmod 4755 program
ls -l /usr/bin/passwd    # Shows: -rwsr-xr-x  (note the 's' in the owner slot)

# setgid on a directory: new files inherit the directory's group
sudo mkdir /srv/project
sudo chgrp devs /srv/project
sudo chmod g+s /srv/project      # or: chmod 2775 /srv/project
touch /srv/project/new.txt       # The new file's group is automatically "devs"

# sticky bit on a directory: only owners can delete their own files
sudo chmod +t /srv/shared        # or: chmod 1777 /srv/shared
ls -ld /tmp                      # Shows: drwxrwxrwt  (note the 't')

# 4-digit mode: leading digit is the special bits
chmod 2775 /srv/project          # setgid + rwxrwxr-x
chmod 1777 /srv/shared           # sticky + rwxrwxrwx

# Security audit: list every setuid file on the system
sudo find / -perm -4000 -type f 2>/dev/null
# -perm -4000 = "has the setuid bit". 2>/dev/null hides "permission denied" noise.
```

### 5. umask in Practice

```bash
umask
# Show the current umask (for example 0022).

umask 077
# Change it for THIS shell session only: new files become 600, new folders 700.

touch private.txt && mkdir private_dir
ls -ld private.txt private_dir
# Shows: -rw-------  and  drwx------

umask 022
# Reset to the common default.

umask -S
# Show the umask in symbolic form (u=rwx,g=rx,o=rx).
```

To make a umask permanent for your user, add `umask 027` to `~/.bashrc` or `~/.profile`. System-wide defaults live in `/etc/login.defs` and `/etc/profile`. Services started by systemd can use `UMask=0027` in their unit file.

### 6. Access Control Lists (ACLs): When Basic Permissions Are Not Enough

Standard permissions give you **one** owner and **one** group. ACLs let you grant access to **additional specific users or groups**.

```bash
sudo apt install -y acl
# Install the ACL tools (often already present).

setfacl -m u:bob:rw report.txt
# Give the user bob read+write on report.txt, without changing the owner or group.

setfacl -m g:auditors:r report.txt
# Give the group "auditors" read-only access.

getfacl report.txt
# Show the full ACL. A "+" at the end of the ls -l permission string means an ACL exists.

setfacl -x u:bob report.txt
# Remove bob's entry.

setfacl -b report.txt
# Remove ALL ACL entries.
```

### 7. Protect Files from Accidental Deletion: `chattr`

```bash
sudo chattr +i important.conf
# +i = immutable: nobody (even root) can modify, rename, or delete it until the flag is removed.

lsattr important.conf
# Show the special attributes.

sudo chattr -i important.conf
# Remove the immutable flag so the file can be changed again.
```

### 8. Archive with `tar`

Memorize the main flags:

| Flag | Meaning | Memory aid |
|------|---------|------------|
| `-c` | **C**reate an archive | |
| `-x` | e**X**tract an archive | |
| `-t` | **T**est/list the contents | |
| `-v` | **V**erbose: show files as they are processed | |
| `-f` | **F**ile: the next word is the archive file name (always last in the flag group) | |
| `-z` | Compress with gzip | |
| `-j` | Compress with bzip2 | |
| `-J` | Compress with xz | |
| `-C` | Change to a directory first | |

```bash
tar -cvf project.tar project/
# Create an UNCOMPRESSED archive named project.tar from the folder "project".

tar -czvf project.tar.gz project/
# Create a gzip-compressed archive. The most common backup command: "create, zip, verbose, file".

tar -cjvf project.tar.bz2 project/
# Create a bzip2-compressed archive.

tar -cJvf project.tar.xz project/
# Create an xz-compressed archive (smallest, slowest).

tar -tzf project.tar.gz
# LIST the contents without extracting. Always check before extracting unknown archives!

tar -xzvf project.tar.gz
# EXTRACT into the current directory.

tar -xzvf project.tar.gz -C /tmp/restore
# Extract into a different directory (the folder must already exist: mkdir -p /tmp/restore).

tar -xzf project.tar.gz project/config.yaml
# Extract only ONE file from the archive (use the path exactly as shown by tar -tzf).

tar -czf logs.tar.gz --exclude='*.tmp' --exclude='cache' project/
# Create an archive but skip files that match these patterns.

tar -czf backup-$(date +%F).tar.gz -C /var/www myapp
# Date-stamped backup, e.g. backup-2025-01-10.tar.gz.
# -C /var/www myapp means "go to /var/www and archive 'myapp'", storing clean relative paths.

tar -xzpf backup.tar.gz
# -p preserves the original permissions when extracting (root does this by default).
```

> **Tip:** GNU `tar` also auto-detects compression when extracting: `tar -xf anything.tar.gz` works without `-z`.

### 9. Compress Single Files: `gzip`, `bzip2`, `xz`

```bash
gzip app.log
# Compresses app.log into app.log.gz and DELETES the original.

gzip -k app.log
# -k keeps the original file as well.

gzip -9 app.log
# -9 = maximum compression (slower). Range 1 (fastest) to 9 (smallest).

gunzip app.log.gz
# Decompress (same as: gzip -d app.log.gz).

zcat app.log.gz | head
# View a compressed file WITHOUT extracting it. Also try: zgrep "ERROR" app.log.gz

bzip2 app.log && bunzip2 app.log.bz2
# Compress with bzip2 and decompress it.

xz app.log && unxz app.log.xz
# Compress with xz and decompress it.

ls -lh app.log*
# Compare file sizes.
```

### 10. Zip and Unzip

```bash
sudo apt install -y zip unzip
# Install the tools if they are missing.

zip -r project.zip project/
# Create a zip archive of a folder (-r = recursive, include subfolders).

zip -r project.zip project/ -x "*.log" -x "project/cache/*"
# Exclude files matching patterns.

zip -e secret.zip passwords.txt
# -e encrypts with a password you type. (The classic zip encryption is weak: do not rely on
# it for truly sensitive data. Use GPG or an encrypted store instead.)

unzip -l project.zip
# List the contents without extracting.

unzip project.zip
# Extract into the current directory.

unzip project.zip -d /tmp/restore
# Extract into a specific directory.

unzip -o project.zip
# -o overwrite existing files without prompting.
```

### 11. Verify and Compare Backups

```bash
tar -tzf backup.tar.gz > /dev/null && echo "Archive is readable"
# Reads the whole archive; the message prints only if it succeeded (exit code 0).

sha256sum backup.tar.gz
# Print a checksum (a fingerprint). Compare it before and after transferring to prove the file is intact.

sha256sum backup.tar.gz > backup.tar.gz.sha256
sha256sum -c backup.tar.gz.sha256
# Save the checksum, then later verify it. Prints "backup.tar.gz: OK" if unchanged.

du -sh project/ backup.tar.gz
# Compare original folder size with the archive size.
```

---

## Real-World Use Case

### Scenario: Securing and backing up a production web application

You are deploying `myapp` to `/var/www/myapp`. The code is deployed by a user called `deploy`, served by a web server running as `www-data`, and users upload images to a folder. You must make this **secure** and **recoverable**.

```
/var/www/myapp/
├── src/            code           owner deploy:www-data   dirs 750, files 640
├── .env            secrets        owner deploy:deploy     600  (only deploy reads it)
├── uploads/        user uploads   owner www-data:www-data dirs 2775 (setgid), files 664
└── logs/           app logs       owner www-data:www-data 750
```

**Part 1: Apply least-privilege permissions**

```bash
# Ownership: deploy owns the code; the web server's group may read it
sudo chown -R deploy:www-data /var/www/myapp

# Directories: owner full, group read+enter, others nothing
sudo find /var/www/myapp -type d -exec chmod 750 {} \;

# Files: owner read+write, group read, others nothing
sudo find /var/www/myapp -type f -exec chmod 640 {} \;

# Secrets: only the deploy user may read the environment file
sudo chmod 600 /var/www/myapp/.env

# Uploads: the web server must write here; setgid keeps the group consistent
sudo chown -R www-data:www-data /var/www/myapp/uploads
sudo chmod -R 775 /var/www/myapp/uploads
sudo chmod g+s /var/www/myapp/uploads

# Verify
namei -l /var/www/myapp/.env
```

If the web server cannot reach the files, `namei -l` shows which folder on the path blocks access. The fix is a targeted `chmod` or `chown`, **not** `chmod -R 777`.

**Part 2: A nightly backup script**

```bash
sudo nano /opt/scripts/backup-myapp.sh
```

```bash
#!/bin/bash
# Stop on errors (-e), undefined variables (-u), and failures inside pipes (pipefail)
set -euo pipefail

SRC_PARENT="/var/www"           # Parent folder of the app
APP="myapp"                     # Folder to back up
DEST="/backup"                  # Where backups are stored
STAMP=$(date +%F_%H%M)          # Example: 2025-01-10_0230
FILE="$DEST/${APP}-${STAMP}.tar.gz"

mkdir -p "$DEST"

# 1. Create a compressed archive, skipping logs and cache
tar -czf "$FILE" --exclude="$APP/logs" --exclude="$APP/cache" -C "$SRC_PARENT" "$APP"

# 2. Verify the archive can be read back; fail loudly if not
tar -tzf "$FILE" > /dev/null

# 3. Write a checksum so integrity can be proven later
sha256sum "$FILE" > "$FILE.sha256"

# 4. Lock down the backup: it contains secrets such as .env
chmod 600 "$FILE" "$FILE.sha256"

# 5. Retention: delete backups older than 7 days
find "$DEST" -name "${APP}-*.tar.gz*" -mtime +7 -delete

echo "Backup complete: $FILE"
```

```bash
sudo chmod 750 /opt/scripts/backup-myapp.sh        # Make it executable (for owner and group)
sudo crontab -e                                    # Schedule it (see Topic 4)
# 30 2 * * * /opt/scripts/backup-myapp.sh >> /var/log/backup.log 2>&1
```

**Part 3: Restoring after a bad deployment**

```bash
sudo systemctl stop myapp                              # Stop the app
sudo tar -czf /tmp/broken-$(date +%F).tar.gz -C /var/www myapp   # Keep evidence of the broken state
sudo tar -xzpf /backup/myapp-2025-01-10_0230.tar.gz -C /var/www   # Restore, preserving permissions
sudo systemctl start myapp
```

**Why this matters in production:**

- A leaked `.env` file means leaked database passwords. `600` limits the damage if the web process is compromised.
- Consistent ownership and the `setgid` bit stop "it works for me but not for the web server" problems with uploads.
- Verified, checksummed, access-restricted, rotated backups are the difference between a 10-minute recovery and a career-defining outage.

> **Analogy:** The code is a **display cabinet** (visitors may look but not touch), `.env` is the **safe** in the manager's office, the uploads folder is the **customer suggestion box** (anyone can drop things in, but only staff can open it), and the backup is a **fireproof copy of the blueprints** stored off-site.

---

## Interview Questions & Answers

### Q1. Explain what `chmod 754 file` does, and how permissions differ on files versus directories.

**Answer:**
`754` sets the owner to `7` (read + write + execute), the group to `5` (read + execute), and others to `4` (read only), which is `rwxr-xr--`. The values are r=4, w=2, x=1, added together for each class. On a **file**, `r` lets you read the content, `w` lets you modify it, and `x` lets you run it. On a **directory**, `r` lets you list the names inside, `w` lets you create, delete, or rename entries (it needs `x` as well), and `x` lets you enter it and access items by path. A common gotcha is that deleting a file requires write permission on its **directory**, not on the file.

---

### Q2. What is `umask`? What permissions will new files and directories get with a umask of `022`?

**Answer:**
`umask` defines which permission bits are **removed** from the default when a new file or directory is created. The default starting points are `666` for files and `777` for directories. With a umask of `022`, files are created as `644` (`rw-r--r--`) and directories as `755` (`rwxr-xr-x`). With `077`, they would be `600` and `700`, meaning only the owner has access. The umask can be set for the current shell with `umask 077` and made permanent in `~/.bashrc` or system files such as `/etc/profile` or `/etc/login.defs`.

---

### Q3. What are setuid, setgid, and the sticky bit? Give an example of each.

**Answer:**
They are special permission bits:
- **setuid (4):** on an executable, the program runs with the privileges of the file's **owner** instead of the user who launched it. Example: `/usr/bin/passwd` is owned by root and setuid, so normal users can update their password in the protected `/etc/shadow` file.
- **setgid (2):** on an executable, it runs with the file's **group**; on a **directory**, new files inherit the directory's group. This is ideal for shared team folders.
- **Sticky bit (1):** on a directory, users can delete or rename **only their own files**, even if the directory is world-writable. Example: `/tmp` (`drwxrwxrwt`).

Setuid programs are security-sensitive, so I audit them periodically with `find / -perm -4000 -type f`.

---

### Q4. What is the difference between `tar`, `gzip`, and `zip`? What does `tar -czvf backup.tar.gz folder/` mean?

**Answer:**
`tar` **archives**: it bundles many files and directories into one file without compressing. `gzip` **compresses** a single file. `zip` does both archiving and compression in one format and is popular for cross-platform sharing. In `tar -czvf backup.tar.gz folder/`: `c` = create an archive, `z` = compress it with gzip, `v` = verbose (list files as they are added), `f` = the next argument is the archive file name. To list the contents I use `tar -tzf`, and to extract I use `tar -xzf`, optionally with `-C <dir>` to choose the destination.

---

### Q5. A user reports "Permission denied" when accessing `/var/www/app/config.yaml`. How would you troubleshoot it?

**Answer:**
1. Confirm **who** is failing and which account the process runs as (`id`, or `ps aux` for a service).
2. Inspect the file: `ls -l /var/www/app/config.yaml` to see the owner, the group, and the permission bits.
3. Check the **whole path**, because `x` is needed on every parent directory: `namei -l /var/www/app/config.yaml`.
4. Work out which class applies (owner, group, or others) and whether that class has the needed `r`/`w`/`x`.
5. Fix it **minimally**: `chown` to the right owner or group, or `chmod` to add only the required permission (for example `chmod g+r`), or add the user to the right group. I avoid `chmod 777`.
6. Also consider other layers if basic permissions look correct: ACLs (`getfacl`), immutable flags (`lsattr`), SELinux or AppArmor, and whether the filesystem is mounted read-only.

---

## Quick Recap Cheat Sheet

| Task | Command |
|------|---------|
| View permissions / owner | `ls -l`, `ls -ld dir`, `stat file` |
| Check every folder in a path | `namei -l /path/to/file` |
| Make a script executable | `chmod +x script.sh` |
| Private file (owner only) | `chmod 600 file` |
| Normal file / normal directory | `chmod 644 file`, `chmod 755 dir` |
| Private directory | `chmod 700 dir` |
| Set files and dirs separately | `chmod -R u=rwX,g=rX,o= dir` |
| Change owner and group | `chown user:group file` |
| Change owner recursively | `chown -R user:group dir` |
| Change only group | `chgrp group file` or `chown :group file` |
| Show / set default permissions | `umask`, `umask 027` |
| setuid / setgid / sticky | `chmod u+s`, `chmod g+s`, `chmod +t` |
| Extra user access (ACL) | `setfacl -m u:name:rw file`, `getfacl file` |
| Create tar.gz backup | `tar -czvf name.tar.gz dir/` |
| List archive contents | `tar -tzf name.tar.gz` |
| Extract tar.gz | `tar -xzvf name.tar.gz -C /dest` |
| Compress / decompress one file | `gzip file`, `gunzip file.gz` |
| Zip / unzip | `zip -r a.zip dir/`, `unzip a.zip -d dest` |
| Verify integrity | `sha256sum file`, `sha256sum -c file.sha256` |

**Permission numbers at a glance:** `r = 4`, `w = 2`, `x = 1`. Common modes: `600`, `644`, `700`, `755`, `775`. Special bits: setuid `4`, setgid `2`, sticky `1`.

