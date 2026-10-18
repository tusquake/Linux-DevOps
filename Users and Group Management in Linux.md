# Users and Group Management in Linux

> **Who is this for?** A backend developer who has been logging into a server with a single default account (like `ubuntu`) and now needs to understand **who is allowed to do what** on a shared machine. This is the foundation of Linux security and of every access-control decision a DevOps engineer makes.

---

## What is it and Why do we need it?

### What is user and group management?

Linux was designed from day one as a **multi-user** system: many people and many programs can use the same machine at the same time without interfering with each other. To make this safe, every action on the system is performed by a specific **user**, and users can be organized into **groups**.

- A **user** is an identity (a person, or a program) that owns files and runs processes.
- A **group** is a named collection of users who share the same permissions.
- **Privileges** decide what each user or group is allowed to read, change, or run.

> **Analogy: An Office Building with Key Cards**
> Every employee has a personal **key card** (a user account). Cards are given **access levels**: the finance team's cards open the finance floor (a group). The cleaners' cards open storage rooms. The building manager has a **master key** (root). When something goes wrong, the access log tells you exactly whose card opened which door.

### Why do we need it?

| Need | How users and groups help |
|------|---------------------------|
| **Security: least privilege** | Give each person or program only the access they need, nothing more |
| **Accountability** | Logs show *who* did what; a shared login makes that impossible |
| **Isolation** | If your web app is hacked, the attacker only has that app's limited powers, not the whole server |
| **Collaboration** | Teams can share folders safely through groups |
| **Easy onboarding / offboarding** | Add a person to a group to grant access; remove them to revoke it |

**The DevOps rule of thumb:** *one person = one account, one application = one service account, never share logins, and never run applications as root.*

---

## Core Concepts

### 1. Types of Users

| Type | UID range | Purpose | Example |
|------|-----------|---------|---------|
| **root (superuser)** | `0` | Full control of everything. Can bypass all permission checks | `root` |
| **System / service users** | typically `1` to `999` | Used by programs and services; usually cannot log in interactively | `www-data`, `postgres`, `nginx` |
| **Regular users** | `1000` and up | Real people with login access and a home folder | `alice`, `bob` |

> **Analogy:** `root` is the building owner with a master key. **System users** are contractors (the elevator company, the cleaning company) who have access only to their own equipment. **Regular users** are the employees.

The kernel does not care about names. It only uses the **UID** (User ID number). Names such as `alice` are friendly labels mapped to numbers.

---

### 2. Groups

Every user belongs to:

| Group type | Meaning | Analogy |
|------------|---------|---------|
| **Primary group** | The default group assigned to files the user creates. Exactly one per user. Usually named after the user | Your main department on your ID badge |
| **Secondary (supplementary) groups** | Extra groups that grant additional access. A user can have many | Project teams or committees you also sit on |

Groups are identified by a **GID** (Group ID).

> **Why groups matter:** Instead of giving permission to 20 developers one by one, you give permission to the group `devs` **once**, then add or remove people from that group.

---

### 3. Where the Information Lives: The Four Key Files

| File | Contains | Readable by everyone? |
|------|----------|-----------------------|
| `/etc/passwd` | List of users (name, UID, GID, home, shell) | Yes |
| `/etc/shadow` | Encrypted (hashed) passwords and password aging settings | **No**, only root |
| `/etc/group` | List of groups and their members | Yes |
| `/etc/gshadow` | Group passwords and group administrators (rarely used) | **No**, only root |

#### `/etc/passwd`: one line per user

```text
alice:x:1001:1001:Alice Smith:/home/alice:/bin/bash
  |   |  |    |       |            |          |
  |   |  |    |       |            |          +-- login shell
  |   |  |    |       |            +-- home directory
  |   |  |    |       +-- comment / full name (GECOS field)
  |   |  |    +-- primary group ID (GID)
  |   |  +-- user ID (UID)
  |   +-- "x" means the password hash is stored in /etc/shadow
  +-- username
```

#### `/etc/shadow`: one line per user

```text
alice:$y$j9T$abc...xyz:19800:0:90:7:14::
  |          |           |   | |  |  |
  |          |           |   | |  |  +-- days after expiry before account is disabled
  |          |           |   | |  +-- warn days before password expires
  |          |           |   | +-- maximum password age (days)
  |          |           |   +-- minimum password age (days)
  |          |           +-- last password change (days since 1970-01-01)
  |          +-- the password HASH (a one-way scrambled form; never the real password)
  +-- username
```

> **Why two files?** `/etc/passwd` must be readable by everyone because many programs need to translate UIDs to names (for example, `ls -l`). If hashes lived there too, anyone could copy them and try to crack them. Splitting them keeps the public information public and the secrets locked away.
>
> **Analogy:** The staff directory is posted in the lobby (`passwd`), but the safe containing everyone's secret codes is in a locked room (`shadow`).

A **hash** is a one-way scrambling of the password. The system stores the scrambled result and compares it on login; it never stores the original password.

#### `/etc/group`: one line per group

```text
devs:x:1500:alice,bob
  |  |  |      |
  |  |  |      +-- members (secondary members only)
  |  |  +-- group ID (GID)
  |  +-- password placeholder (unused in practice)
  +-- group name
```

---

### 4. Privilege Escalation: `su` and `sudo`

| Tool | Meaning | Requires | Typical use |
|------|---------|----------|-------------|
| `su` | **S**witch **U**ser: become another user entirely | The **target** user's password | `su - alice` |
| `sudo` | **S**uper**u**ser **do**: run a single command as another user (usually root) | **Your own** password, and permission in sudoers | `sudo apt update` |

`sudo` is preferred because:

- You never need to know or share the root password.
- It records who ran what (in `/var/log/auth.log`).
- Permissions can be limited to specific commands.

Who may use `sudo` is defined in `/etc/sudoers` (always edited safely with the `visudo` command, never with a plain text editor). On Ubuntu/Debian, members of the `sudo` group get full sudo access. On Red Hat-family systems, the equivalent group is `wheel`.

> **Analogy:** `su` is handing over your entire identity to become someone else. `sudo` is borrowing a **temporary signed permission slip** for one specific task, with the security desk keeping a copy.

---

### 5. Account Management Concepts

| Concept | Meaning |
|---------|---------|
| **Home directory** | Each regular user's private folder, normally `/home/<username>` |
| **Login shell** | The program started at login (for example `/bin/bash`). A special shell such as `/usr/sbin/nologin` **blocks interactive logins** |
| **Skeleton directory** `/etc/skel` | A template folder whose contents are copied into every new user's home |
| **Locked account** | The password is disabled, so the user cannot log in with it |
| **Password aging** | Rules that force password changes (maximum age, warning period) |
| **Service account** | A non-human account created for one application, with a `nologin` shell and minimal permissions |

---

### 6. `useradd` vs. `adduser` (A Very Common Source of Confusion)

| Command | Nature | Behavior |
|---------|--------|----------|
| `useradd` | Low-level, available on **every** distro | Does only what the options say. On Debian/Ubuntu it does **not** create a home folder or set a password unless you pass `-m` and run `passwd` |
| `adduser` | Friendly wrapper script on **Debian/Ubuntu** | Interactive: creates the home folder, copies `/etc/skel`, prompts for a password and details |

> **Analogy:** `useradd` is a bare form where you fill in every box yourself. `adduser` is a receptionist who asks you questions and handles the paperwork.

In scripts and automation, use `useradd` (works everywhere, no prompts). Interactively on Ubuntu, `adduser` is easier.

---

## Practical Examples & Commands

> Most of these commands need `sudo`. Practice on a throwaway VM, never on a production server first.

### 1. Find Out Who You Are and Who Is Logged In

```bash
whoami
# Prints your current username.

id
# Shows your UID, primary GID, and all groups.
# Example: uid=1001(alice) gid=1001(alice) groups=1001(alice),27(sudo),1500(devs)

id bob
# Show the same information for another user.

groups
# List the groups you belong to.

groups bob
# List the groups that bob belongs to.

who
# Who is logged in right now (user, terminal, login time).

w
# Who is logged in AND what they are currently running.

last -n 10
# The last 10 logins (from /var/log/wtmp). Great for audits.

lastlog
# The most recent login time of every user.

sudo lastb | head
# Recent FAILED login attempts (from /var/log/btmp). Shows brute-force attacks.
```

### 2. Look Up Accounts and Groups

```bash
cat /etc/passwd
# List all users (including system users).

getent passwd alice
# Look up alice in ALL account sources (local files, LDAP, etc.). Better than grep on /etc/passwd.

cut -d: -f1 /etc/passwd
# Print just the usernames (split on ":", take field 1).

awk -F: '$3 >= 1000 && $3 < 65534 {print $1}' /etc/passwd
# List only regular (human) users: UID 1000 and above, excluding the special "nobody" user.

getent group devs
# Show the group named devs and its members.

sudo cat /etc/shadow
# View password hashes (root only). Be careful: never paste this output anywhere.
```

### 3. Create a User

```bash
sudo adduser alice
# Debian/Ubuntu friendly way. It asks for a password and details,
# creates /home/alice, and copies the template files from /etc/skel.

sudo useradd -m -s /bin/bash -c "Bob Jones" bob
# The portable way:
#   -m            create the home directory (/home/bob)
#   -s /bin/bash  set the login shell
#   -c "..."      comment / full name
#   bob           the username

sudo passwd bob
# Set (or change) bob's password. You will be prompted twice. Required after plain useradd.

sudo useradd -m -s /bin/bash -G sudo,devs carol
# -G sets SECONDARY groups at creation time (the groups must already exist).
```

### 4. Modify a User: `usermod`

```bash
sudo usermod -aG devs alice
# ADD alice to the group "devs".
# -a = append, -G = supplementary groups.
# WARNING: without -a, alice is REMOVED from all other groups and put only in devs!

sudo usermod -aG sudo alice
# Give alice administrator access on Ubuntu/Debian (use "wheel" on RHEL/Rocky/Amazon Linux).

sudo usermod -s /bin/zsh alice
# Change alice's login shell.

sudo usermod -c "Alice Smith - Platform Team" alice
# Change the comment (full name).

sudo usermod -l alicia alice
# Rename the login from alice to alicia (the home folder name does not change automatically).

sudo usermod -d /home/alicia -m alicia
# Move the home directory to a new path (-m moves the contents).

sudo usermod -L alice
# LOCK the account (disables password login).

sudo usermod -U alice
# UNLOCK the account.

sudo usermod -e 2025-12-31 alice
# Set an account expiry date. Perfect for contractors and temporary staff.
```

> **Note:** A user must log out and back in before a new group membership takes effect in their active sessions. Check with `id alice`.

### 5. Passwords and Password Policy: `passwd`, `chage`

```bash
sudo passwd alice
# Set a new password for alice.

passwd
# Change YOUR OWN password.

sudo passwd -l alice
# Lock the password.   (-u unlocks)

sudo passwd -e alice
# Expire the password immediately: alice must choose a new one at next login.
# The standard way to hand over a new account with a temporary password.

sudo passwd -S alice
# Status: shows if the password is set, locked, and when it last changed.

sudo chage -l alice
# List alice's password aging details.

sudo chage -M 90 -m 7 -W 14 alice
# Password policy:
#   -M 90  maximum 90 days before a change is forced
#   -m 7   minimum 7 days between changes
#   -W 14  warn 14 days before expiry

sudo chage -d 0 alice
# Force a password change at the next login.
```

### 6. Delete a User: `userdel`

```bash
sudo userdel alice
# Delete the account but KEEP the home directory and files.

sudo userdel -r alice
# Delete the account AND remove the home directory and mail spool.
# Back up first if you may need the data!

sudo find / -user 1001 -ls 2>/dev/null
# Find leftover files still owned by a deleted user's old UID (here 1001).
```

### 7. Groups: `groupadd`, `groupmod`, `groupdel`, `gpasswd`

```bash
sudo groupadd devs
# Create a group named devs.

sudo groupadd -g 1500 ops
# Create a group with a specific GID (useful to keep IDs consistent across servers).

sudo gpasswd -a alice devs
# Add alice to devs. (Equivalent in effect to: usermod -aG devs alice)

sudo gpasswd -d alice devs
# Remove alice from devs.

sudo groupmod -n developers devs
# Rename the group devs to developers.

sudo groupdel developers
# Delete a group (it cannot be the primary group of an existing user).

getent group devs
# Verify the members.

newgrp devs
# Start a new shell with devs as your active primary group (rarely needed).
```

### 8. Switching Users: `su` and `sudo`

```bash
su - alice
# Become alice with her full environment (the "-" gives a login shell). Needs alice's password.

sudo su - alice
# Become alice using YOUR sudo rights instead (common when you are an admin).

sudo -i
# Open a root login shell using your own password (the safe alternative to "su -").

sudo -u postgres psql
# Run a single command (psql) as the user "postgres".

sudo -l
# List what YOU are allowed to run with sudo.

sudo -l -U alice
# List what alice is allowed to run with sudo (as root).

exit
# Leave the switched shell and go back to your own user.
```

### 9. Controlling Sudo Access: `visudo` and `sudoers.d`

```bash
sudo visudo
# Safely edit /etc/sudoers. It checks the syntax before saving, so a typo cannot lock you
# out of sudo. NEVER edit /etc/sudoers with a normal editor.
```

The format of a sudoers line is:

```text
alice   ALL=(ALL:ALL)   ALL
  |      |    |    |     |
  |      |    |    |     +-- which COMMANDS are allowed
  |      |    |    +-- run as which GROUP
  |      |    +-- run as which USER
  |      +-- on which HOSTS
  +-- who (a user name, or %groupname for a group)
```

Better practice: put each rule in its own file under `/etc/sudoers.d/`:

```bash
sudo visudo -f /etc/sudoers.d/devs
```

```text
# Members of the group "devs" can restart ONLY the myapp service, without a password
%devs ALL=(root) NOPASSWD: /usr/bin/systemctl restart myapp, /usr/bin/systemctl status myapp

# Members of "ops" get full sudo, but must enter their own password
%ops  ALL=(ALL:ALL) ALL
```

```bash
sudo visudo -c
# Check ALL sudoers files for syntax errors.

ls -l /etc/sudoers.d/
# Files here should be owned by root with permissions 0440. visudo sets this for you.
```

> **Principle of least privilege:** Give people `sudo` for **only the commands they need**, instead of full root, wherever possible.

### 10. Create a Service Account for an Application

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin myapp
# --system          creates a system account (low UID, no password aging)
# --no-create-home  no /home/myapp folder is needed
# --shell nologin   nobody can log in as this user; it exists only to run the app

sudo mkdir -p /opt/myapp
sudo chown -R myapp:myapp /opt/myapp
# Give the app ownership of its own folder (file ownership is covered in Topic 6).

id myapp
# Confirm: uid=998(myapp) gid=998(myapp) groups=998(myapp)
```

Then in the systemd unit file (Topic 4) you set `User=myapp`, so the application runs with minimal privileges.

### 11. Allow a New User to Log In with SSH Keys

```bash
# Run as an admin. The user "alice" already exists.

sudo mkdir -p /home/alice/.ssh
# Create the folder where SSH looks for authorised keys.

echo "ssh-ed25519 AAAAC3Nza...rest-of-alices-PUBLIC-key... alice@laptop" \
  | sudo tee -a /home/alice/.ssh/authorized_keys
# Add alice's PUBLIC key (she sends you the .pub file; never ask for her private key).
# tee -a appends the line using sudo privileges.

sudo chown -R alice:alice /home/alice/.ssh
# Make alice the owner.

sudo chmod 700 /home/alice/.ssh
# Only alice can enter this folder. (SSH refuses to work if permissions are too open.)

sudo chmod 600 /home/alice/.ssh/authorized_keys
# Only alice can read or write the key file.
```

Alice can now log in with `ssh alice@server-ip` without any password.

### 12. A Shared Project Folder for a Group

```bash
sudo groupadd devs                       # Create the group (skip if it exists)
sudo usermod -aG devs alice              # Add members
sudo usermod -aG devs bob

sudo mkdir -p /srv/project               # Create the shared folder
sudo chgrp devs /srv/project             # Make "devs" the group owner of the folder
sudo chmod 2775 /srv/project             # Set permissions (explained below)
```

What `2775` means (permissions are covered fully in Topic 6):

- `775` = owner and group can read, write, and enter; everyone else can read and enter.
- The leading `2` is the **setgid** bit: any new file created inside automatically **inherits the group `devs`**, so teammates can always edit each other's files.

### 13. Offboarding a Departing Employee (Safe Order of Steps)

```bash
sudo usermod -L -e 1 alice
# 1. Lock the account and expire it right away (the user can no longer log in).

sudo pkill -u alice
# 2. End any processes still running as alice.

sudo tar -czf /backup/alice-home-$(date +%F).tar.gz /home/alice
# 3. Back up her home folder (tar is covered in Topic 6).

sudo rm -f /home/alice/.ssh/authorized_keys
# 4. Remove her SSH keys (also remove them from any shared or other servers' lists).

sudo rm -f /etc/sudoers.d/alice
# 5. Remove any custom sudo rules (if one exists).

sudo crontab -u alice -r
# 6. Remove her scheduled cron jobs.

sudo userdel -r alice
# 7. Finally, delete the account and home directory (after the backup is confirmed).
```

---

## Real-World Use Case

### Scenario: Setting up access for a growing engineering team

Your company has three kinds of people and one application on a shared Linux server:

| Who | What they need |
|-----|----------------|
| **Developers** (`alice`, `bob`) | Read logs, deploy code to `/srv/project`; restart the app; **no** full root |
| **Operations** (`carol`) | Full administrative access |
| **The application** (`myapp`) | Run the API; access only its own folder; **no** login |

```
                      +-----------------------------+
                      |         Linux Server        |
                      |                             |
   alice  ---+        |  group "devs"               |
   bob    ---+--------|    - read /var/log/myapp    |
                      |    - write /srv/project     |
                      |    - sudo: restart myapp    |
                      |                             |
   carol  ------------|  group "ops" (sudo)         |
                      |    - full admin             |
                      |                             |
   myapp (nologin) ---|  runs the API process       |
                      |    - owns /opt/myapp only   |
                      +-----------------------------+
```

**Implementation:**

```bash
# 1. Groups
sudo groupadd devs
sudo groupadd ops

# 2. Human users (each person has their own account: accountability)
sudo useradd -m -s /bin/bash -G devs alice
sudo useradd -m -s /bin/bash -G devs bob
sudo useradd -m -s /bin/bash -G ops,sudo carol
sudo passwd -e alice        # Force each user to choose their own password at first login
sudo passwd -e bob          # (set an initial password with "sudo passwd <user>" first)
sudo passwd -e carol

# 3. Service account for the application
sudo useradd --system --no-create-home --shell /usr/sbin/nologin myapp
sudo chown -R myapp:myapp /opt/myapp

# 4. Developers can restart ONLY the app, not everything
echo '%devs ALL=(root) NOPASSWD: /usr/bin/systemctl restart myapp' \
  | sudo tee /etc/sudoers.d/devs
sudo chmod 440 /etc/sudoers.d/devs
sudo visudo -c              # Validate the syntax

# 5. Shared project folder
sudo mkdir -p /srv/project
sudo chgrp devs /srv/project
sudo chmod 2775 /srv/project

# 6. Audit regularly
last -n 20                  # Who logged in recently?
sudo lastb | head           # Any failed login attempts?
sudo grep sudo /var/log/auth.log | tail   # Who used sudo and what did they run?
```

**Why this matters in production:**

- If the app is compromised, the attacker is `myapp`, a user with no login and no access to other files, instead of `root`.
- If a developer leaves, you lock **one** account and nothing else breaks. With a shared login, you would have to change the password for everyone.
- Auditors and incident responders can answer "who ran this command at 14:03?" because every action is tied to a named account.
- Access is granted by **group membership**, so onboarding is one command (`usermod -aG devs newperson`) rather than dozens of per-file changes.

> **Analogy:** It's the difference between a hotel where every guest gets a key card for their own room (with a log of every entry) and one where everyone shares a single key left under the doormat.

---

## Interview Questions & Answers

### Q1. What is the difference between `useradd` and `adduser`?

**Answer:**
`useradd` is a low-level command available on all Linux distributions. It only does what you tell it through options; on Debian and Ubuntu it does not create a home directory or set a password unless you pass `-m` and later run `passwd`. `adduser` is a friendlier, interactive Perl wrapper script on Debian-based systems that creates the home directory, copies files from `/etc/skel`, and prompts for the password and details. In scripts and automation I use `useradd` because it is portable and non-interactive.

---

### Q2. Why are user details stored in `/etc/passwd` but passwords in `/etc/shadow`?

**Answer:**
`/etc/passwd` has to be readable by every user, because many programs need to map UIDs to usernames (for example, `ls -l` showing file owners). If the password hashes were stored there too, any user could copy them and attempt to crack them offline. Moving the hashes to `/etc/shadow`, which only root can read, keeps the public account information available while protecting the sensitive data. The `x` in the second field of `/etc/passwd` indicates that the hash is in `/etc/shadow`. `/etc/shadow` also stores password aging information.

---

### Q3. What is the difference between a primary and a secondary group, and why do we use `usermod -aG` instead of `usermod -G`?

**Answer:**
The **primary group** is the user's default group. It is recorded in `/etc/passwd`, and new files created by the user are owned by it. **Secondary groups** are additional groups, recorded in `/etc/group`, that grant extra access. With `usermod -G`, the supplied list **replaces** all existing secondary groups, so a user could accidentally lose important memberships such as `sudo`. The `-a` (append) flag, as in `usermod -aG devs alice`, **adds** the group while keeping the existing ones.

---

### Q4. How do you give a user administrator rights safely? What is `visudo` and why use it?

**Answer:**
The simplest way is to add the user to the admin group: `sudo` on Debian/Ubuntu or `wheel` on RHEL-based systems (`usermod -aG sudo alice`). For finer control, I create a rule file in `/etc/sudoers.d/` that grants only specific commands, following the principle of least privilege. I always edit sudoers files with `visudo` (or `visudo -f /etc/sudoers.d/<file>`), because it locks the file against simultaneous edits and **validates the syntax before saving**. A syntax mistake in a sudoers file written with a normal editor can break `sudo` for everyone and lock you out of administrative access.

---

### Q5. An employee has left the company. How do you handle their account, and how do you create a safe account for an application?

**Answer:**
**Offboarding:** First **lock and expire** the account (`usermod -L -e 1 user`) so they can't log in, then end their running processes, back up their home directory, remove their SSH keys, sudo rules, and cron jobs, and finally delete the account with `userdel -r`. I also search for orphaned files owned by their old UID.

**Application accounts:** I create a dedicated **service account** with `useradd --system --no-create-home --shell /usr/sbin/nologin appname`. It cannot be used for interactive login, owns only the app's own directories, and the service runs under it (`User=appname` in the systemd unit). That way, if the application is compromised, the attacker is limited to that account's small set of permissions rather than getting root.

---

## Quick Recap Cheat Sheet

| Task | Command |
|------|---------|
| Who am I / my groups | `whoami`, `id`, `groups` |
| Who is logged in / history | `who`, `w`, `last`, `lastlog` |
| Create user (Debian/Ubuntu, interactive) | `sudo adduser name` |
| Create user (portable) | `sudo useradd -m -s /bin/bash name` |
| Set password | `sudo passwd name` |
| Add to group (keep others) | `sudo usermod -aG group name` |
| Lock / unlock account | `sudo usermod -L name` / `-U` |
| Force password change | `sudo passwd -e name` or `chage -d 0 name` |
| Password aging policy | `sudo chage -M 90 -m 7 -W 14 name` |
| Delete user + home | `sudo userdel -r name` |
| Create / delete group | `sudo groupadd g` / `sudo groupdel g` |
| Add / remove group member | `sudo gpasswd -a name g` / `-d` |
| Become another user | `su - name`, `sudo -u name cmd` |
| Root shell via sudo | `sudo -i` |
| Edit sudo rules safely | `sudo visudo`, `visudo -f /etc/sudoers.d/file` |
| Service account | `useradd --system --no-create-home --shell /usr/sbin/nologin name` |
| Key files | `/etc/passwd`, `/etc/shadow`, `/etc/group`, `/etc/sudoers` |
