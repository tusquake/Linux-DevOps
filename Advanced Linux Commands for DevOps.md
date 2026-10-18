# Advanced Linux Commands for DevOps

> **Who is this for?** A backend developer who is comfortable moving around a Linux server and reading files (Topic 3) and now needs to **manage what is running on it**: processes, background services, logs, scheduled jobs, and resource usage. This is where "using Linux" becomes "operating Linux".

---

## What is it and Why do we need it?

### What are "advanced" commands?

In Topic 3 you learned to work with **files**. Advanced commands deal with **running things**:

- **Processes:** programs that are currently running.
- **Services:** programs designed to run continuously in the background (a web server, a database).
- **Logs:** the diary that every service writes about what it is doing.
- **Scheduled jobs:** tasks that run automatically at set times.
- **Resource usage:** how much CPU, memory, and disk the machine is using.

> **Analogy: Running a Restaurant, Not Just Cooking in It**
> Basic commands are like knowing where the pots and ingredients are. Advanced commands are like being the **restaurant manager**: seeing which cooks are busy (processes), making sure the ovens stay on all day (services), reading the daily activity logbook (logs), setting the cleaning crew's schedule (cron), and watching whether the kitchen is running out of gas or space (resources).

### Why do we need them?

Your application is not "just a file". On a server it is a **process** that must start at boot, restart when it crashes, write logs you can read, and share the machine's limited CPU and memory fairly with everything else. When something goes wrong in production, you need to answer questions like:

- *Is my app actually running?*
- *Which process is eating all the memory?*
- *Why did the service crash at 3 AM?*
- *How do I make this script run every night?*

These commands answer exactly those questions.

---

## Core Concepts

### 1. What is a Process?

A **process** is a running instance of a program. Every process has:

| Property | Meaning | Analogy |
|----------|---------|---------|
| **PID** (Process ID) | A unique number identifying the process | A ticket number at a deli counter |
| **PPID** (Parent PID) | The PID of the process that started it | The person who gave the order |
| **Owner (user)** | Which user it runs as | Which employee is doing the job |
| **State** | What it is doing right now | Working, waiting, or on break |
| **CPU / Memory usage** | Resources it is consuming | Hours worked and desk space used |

> **Analogy:** A **program** is a recipe in a cookbook. A **process** is someone actually cooking that recipe right now. You can have the same recipe cooked by several people at once (several processes of one program).

**Common process states** (shown in the `STAT` column of `ps`):

| Code | State | Meaning |
|------|-------|---------|
| `R` | Running | Using the CPU right now or ready to |
| `S` | Sleeping | Waiting for something (most processes most of the time) |
| `D` | Uninterruptible sleep | Waiting on disk or network I/O; cannot be killed easily |
| `T` | Stopped | Paused (for example with `Ctrl + Z`) |
| `Z` | Zombie | Finished, but its parent has not collected its exit status yet |

> **Zombie analogy:** A zombie process is like an employee who has already left the company but whose name is still on the payroll list because HR (the parent process) hasn't removed it. It uses almost no resources, but too many suggest a buggy parent program.

---

### 2. Signals: How You "Talk" to a Process

A **signal** is a small message sent to a process. The `kill` command is misleadingly named: it *sends signals*, and only some of them end a process.

| Signal | Number | Meaning | Analogy |
|--------|--------|---------|---------|
| `SIGTERM` | 15 | "Please shut down cleanly" (the default) | Politely asking a guest to leave and take their things |
| `SIGKILL` | 9 | "Die immediately", cannot be ignored | Cutting the power to the building |
| `SIGHUP` | 1 | "Reload your configuration" (for many daemons) | Handing the chef an updated menu mid-service |
| `SIGINT` | 2 | Interrupt (what `Ctrl + C` sends) | Tapping someone on the shoulder to stop |
| `SIGSTOP` / `SIGCONT` | 19 / 18 | Pause / resume | Pause and play buttons |

**Always try `SIGTERM` (15) first.** It lets the program save data and close connections. Use `SIGKILL` (9) only as a last resort because the program gets no chance to clean up.

---

### 3. Foreground vs. Background Jobs

| Mode | Behavior | How |
|------|----------|-----|
| **Foreground** | Occupies your terminal until it finishes | Default |
| **Background** | Runs while you keep using the terminal | Add `&` at the end |
| **Detached (survives logout)** | Keeps running after you disconnect from SSH | `nohup`, `tmux`, `screen`, or a systemd service |

> **Analogy:** Foreground is cooking while standing at the stove, unable to do anything else. Background is putting the dish in the oven and getting on with other tasks. A systemd service is hiring a dedicated cook who works even when you go home.

---

### 4. Services and systemd

As introduced in Topic 2, a **service** (daemon) is a long-running background program. **systemd** is the manager that starts services at boot, supervises them, and restarts them if they crash. You control it with `systemctl`.

A service has two independent properties, and beginners often confuse them:

| Question | Command | Meaning |
|----------|---------|---------|
| Is it running **right now**? | `systemctl start / stop` | Controls the current state |
| Will it start **automatically at boot**? | `systemctl enable / disable` | Controls the future state |

> **Analogy:** A shop light has a switch (on or off right now) and a timer (turns on automatically at 6 PM every day). `start` flips the switch now. `enable` sets the timer. You need both to have a light that is on now *and* comes on tomorrow.

A **unit file** is the small configuration file that tells systemd how to run a service. Unit files live in `/etc/systemd/system/` (your own) and `/lib/systemd/system/` (installed by packages).

---

### 5. Logs

Logs are records of events. Two main sources on modern Linux:

| Source | Where | How to read |
|--------|-------|-------------|
| **Traditional log files** | `/var/log/` (for example `syslog`, `auth.log`, `nginx/error.log`) | `tail`, `less`, `grep` |
| **systemd journal** | Binary database managed by `journald` | `journalctl` |

Important log files on Ubuntu/Debian:

| File | Contents |
|------|----------|
| `/var/log/syslog` | General system messages |
| `/var/log/auth.log` | Logins, `sudo` usage, SSH attempts |
| `/var/log/kern.log`, `dmesg` | Kernel messages (hardware, disk errors, out-of-memory kills) |
| `/var/log/nginx/` | Web server access and error logs |
| `/var/log/apt/` | Package install history |

On Red Hat-family systems, `/var/log/messages` and `/var/log/secure` play the roles of `syslog` and `auth.log`.

**Log rotation** (`logrotate`) automatically compresses and deletes old logs on a schedule so that logs don't fill the disk.

> **Analogy:** Logs are the **black box flight recorder** of your server. **Log rotation** is archiving last month's notebook into a box so the desk doesn't overflow.

---

### 6. Load Average and Resource Basics

`uptime` and `top` show three **load average** numbers: the average number of processes waiting for or using the CPU over the last **1, 5, and 15 minutes**.

```text
load average: 0.52, 0.80, 1.10
              |     |     |
              1 min 5 min 15 min
```

How to read it: **compare the load to the number of CPU cores** (`nproc`).

| Cores | Load of ... | Interpretation |
|-------|-------------|----------------|
| 4 | 1.0 | Comfortable (25% busy) |
| 4 | 4.0 | Fully used |
| 4 | 8.0 | Overloaded: processes are queueing |

> **Analogy: Checkout Counters.** With 4 counters (cores), a load of 4 means every counter has exactly one customer. A load of 8 means 4 customers are being served and 4 are standing in line.

If the 1-minute number is higher than the 15-minute number, the load is **rising**; if lower, it is **settling down**.

---

### 7. Scheduling with Cron

**cron** is the built-in scheduler. You write jobs in a table called a **crontab**; the `cron` daemon runs them at the specified times.

A cron schedule has **five time fields** followed by the command:

```
 ┌───────────── minute        (0 - 59)
 │ ┌─────────── hour          (0 - 23)
 │ │ ┌───────── day of month  (1 - 31)
 │ │ │ ┌─────── month         (1 - 12)
 │ │ │ │ ┌───── day of week   (0 - 7, where 0 and 7 are Sunday)
 │ │ │ │ │
 * * * * *   command-to-run
```

| Symbol | Meaning |
|--------|---------|
| `*` | Every value |
| `,` | A list (`1,15` = 1st and 15th) |
| `-` | A range (`1-5` = Monday to Friday) |
| `/` | A step (`*/10` = every 10) |

> **Analogy:** Cron is the **alarm clock and to-do list of the server**: "Every night at 2 AM, take out the trash (clean temp files)."

---

### 8. Combining Commands: Exit Codes and Operators

Every command finishes with an **exit code**: `0` means success, any other number means failure. You can see the last one with `echo $?`.

| Operator | Meaning | Example |
|----------|---------|---------|
| `;` | Run next command regardless | `cmd1 ; cmd2` |
| `&&` | Run next only if the previous **succeeded** | `mkdir x && cd x` |
| `\|\|` | Run next only if the previous **failed** | `ping -c1 host \|\| echo "down"` |
| `&` | Run the command in the background | `./job.sh &` |

---

## Practical Examples & Commands

### 1. Viewing Processes: `ps`, `top`, `htop`, `pgrep`

```bash
ps aux
# Snapshot of ALL processes.
#   a = processes of all users, u = user-friendly format, x = include those without a terminal
# Columns: USER, PID, %CPU, %MEM, VSZ, RSS, TTY, STAT, START, TIME, COMMAND

ps -ef
# Another common format. Shows PPID (parent process) too.

ps aux --sort=-%mem | head -n 6
# Top 5 memory consumers (plus the header line). "-" means descending order.

ps aux --sort=-%cpu | head -n 6
# Top 5 CPU consumers.

ps aux | grep nginx
# Find processes whose line contains "nginx".

pgrep -a nginx
# Cleaner way: list PIDs (and full command with -a) of processes matching "nginx".

pstree -p
# Show processes as a family tree with PIDs.

top
# Live, refreshing view of processes and system load.
#   Inside top:  M = sort by memory,  P = sort by CPU,  1 = show each CPU core,
#                k = kill a process (asks for PID),  q = quit

htop
# A friendlier, colorful version of top (install with: sudo apt install -y htop).
#   Use arrow keys, F6 to sort, F9 to kill, F10 to quit.
```

### 2. Stopping Processes: `kill`, `pkill`, `killall`

```bash
kill 1234
# Send SIGTERM (15) to the process with PID 1234: a polite shutdown request.

kill -9 1234
# Send SIGKILL: forceful. Use only if the process ignores a normal kill.

kill -HUP 1234
# Send SIGHUP: many daemons (like nginx) re-read their config when they receive it.

kill -l
# List all available signals.

pkill nginx
# Send SIGTERM to every process whose NAME matches "nginx".

pkill -f "python3 app.py"
# -f matches against the FULL command line, not just the program name.

killall node
# Send SIGTERM to all processes named exactly "node".
```

> **Tip:** Before killing, **verify the PID** with `ps -p 1234 -o pid,user,cmd` so you do not end the wrong process.

### 3. Foreground, Background, and Jobs

```bash
sleep 300 &
# The "&" runs sleep (a 5-minute wait) in the BACKGROUND. The shell prints a job number and PID.

jobs
# List background/stopped jobs started from THIS terminal session.

sleep 300
# Run in the foreground, then press Ctrl + Z to PAUSE (suspend) it.

bg %1
# Resume job number 1 in the background.

fg %1
# Bring job number 1 back to the foreground.

nohup ./long-task.sh > task.log 2>&1 &
# nohup = "no hang up": the process keeps running even if you close the SSH session.
#   > task.log 2>&1  -> save both normal output and errors into task.log
#   &                -> run in the background

disown -h %1
# Detach an already-running job from the shell so it survives logout.
```

### 4. Process Priority: `nice` and `renice`

```bash
nice -n 10 ./backup.sh
# Start a script with LOWER priority (nice value 10), so it does not slow other programs.
# Range: -20 (highest priority) to 19 (lowest). Default is 0.

sudo renice -n 5 -p 1234
# Change the priority of an already-running process (PID 1234).
```

> **Analogy:** "Nice" is how polite a process is. A very nice process (high number) lets others go first, like a customer waving others ahead in a queue.

### 5. Managing Services: `systemctl`

```bash
sudo systemctl status nginx
# Show if the service is running, its PID, memory use, and the most recent log lines.

sudo systemctl start nginx        # Start it now
sudo systemctl stop nginx         # Stop it now
sudo systemctl restart nginx      # Stop then start (brief downtime)
sudo systemctl reload nginx       # Re-read config WITHOUT dropping connections (if supported)

sudo systemctl enable nginx       # Start automatically at boot
sudo systemctl disable nginx      # Do NOT start at boot
sudo systemctl enable --now nginx # Enable at boot AND start immediately (very handy)

systemctl is-active nginx         # Prints "active" or "inactive"
systemctl is-enabled nginx        # Prints "enabled" or "disabled"

systemctl list-units --type=service --state=running
# List all currently running services.

systemctl --failed
# List services that have FAILED. Your first stop when something is broken after a reboot.

sudo systemctl daemon-reload
# Tell systemd to re-read unit files after you create or edit one.
```

### 6. Create Your Own Service (Unit File)

Suppose you have a Node.js API at `/opt/myapp/server.js`. Create the file:

```bash
sudo nano /etc/systemd/system/myapp.service
```

```ini
[Unit]
# Human-readable description shown in "systemctl status"
Description=My Node.js API
# Start only after the network is available
After=network.target

[Service]
# Run the app as this unprivileged user (never as root unless necessary)
User=appuser
# Folder the process starts in
WorkingDirectory=/opt/myapp
# Environment variables for the app
Environment=NODE_ENV=production
Environment=PORT=3000
# The exact command to start the app (use absolute paths)
ExecStart=/usr/bin/node /opt/myapp/server.js
# Restart automatically if it crashes (not if you stop it manually)
Restart=on-failure
# Wait 5 seconds between restart attempts
RestartSec=5

[Install]
# Start this service during normal multi-user boot
WantedBy=multi-user.target
```

Then activate it:

```bash
sudo systemctl daemon-reload          # Make systemd notice the new file
sudo systemctl enable --now myapp     # Start now and on every boot
sudo systemctl status myapp           # Confirm it is "active (running)"
```

> **Analogy:** A unit file is a **job description** for the building manager (systemd): who the employee is, where they work, what to do, and what to do if they faint (restart).

### 7. Reading Logs: `journalctl`, `dmesg`, `/var/log`

```bash
journalctl -u myapp
# All logs for the "myapp" service (-u = unit).

journalctl -u myapp -f
# FOLLOW the logs live, like "tail -f".

journalctl -u myapp -n 50
# Only the last 50 lines.

journalctl -u myapp --since "1 hour ago"
# Logs from the last hour. Also try: --since "2025-01-10 09:00" --until "2025-01-10 10:00"

journalctl -p err -b
# Only messages of priority "error" or worse (-p err) since the current boot (-b).

journalctl --disk-usage
# How much disk space the journal is using.

dmesg -T | tail -n 20
# Last 20 KERNEL messages with human-readable timestamps (-T).
# Check here for disk errors or "Out of memory: Killed process ..." messages.

sudo tail -f /var/log/auth.log
# Watch login and sudo activity live (who is logging in? any failed attempts?).

sudo grep "Failed password" /var/log/auth.log | tail
# Show recent failed SSH password attempts.
```

### 8. Log Rotation: `logrotate`

Rules live in `/etc/logrotate.d/`. Example for your own app:

```bash
sudo nano /etc/logrotate.d/myapp
```

```text
/var/log/myapp/*.log {
    daily               # Rotate once a day
    rotate 14           # Keep 14 old logs, delete older ones
    compress            # Gzip old logs to save space
    missingok           # No error if the log file is missing
    notifempty          # Do not rotate empty files
    copytruncate        # Copy then empty the file so the app keeps writing to it
}
```

```bash
sudo logrotate -d /etc/logrotate.d/myapp
# -d = dry run: shows what WOULD happen without changing anything. Always test first.
```

### 9. Resource Monitoring: `uptime`, `free`, `vmstat`, `df`, `du`, `lsof`

```bash
uptime
# Shows time up and the 1/5/15-minute load averages.

nproc
# Number of CPU cores, used to judge the load average.

free -h
# Memory: total, used, free, and "available" (the number that really matters).
# "available" = memory that can be given to programs, including reclaimable cache.

vmstat 2 5
# Print system statistics every 2 seconds, 5 times.
#   si/so = swap in/out (anything above 0 for long means memory pressure)
#   us/sy/id/wa = CPU time in user/system/idle/waiting-for-disk

df -h
# Disk usage per filesystem (look at "Use%").

df -i
# INODE usage. A disk can be "full" because it ran out of inodes (file slots)
# even when space remains: typical when millions of tiny files pile up.

du -sh /var/log/* | sort -h | tail -n 5
# The 5 biggest items in /var/log. Great for finding what is filling the disk.

lsof -i :3000
# Which process is using (listening on) port 3000? Needs: sudo apt install lsof

sudo lsof -p 1234
# All files and network connections that process 1234 has open.

sudo lsof | grep deleted
# Files that were deleted but are still held open by a process.
# Disk space is NOT freed until the process closes them or is restarted.

watch -n 2 'df -h'
# Re-run a command every 2 seconds and show the result live. Ctrl + C to stop.
```

### 10. Scheduling: `crontab`

```bash
crontab -e
# Open YOUR crontab in an editor (choose nano if asked).

crontab -l
# List your scheduled jobs.

sudo crontab -u appuser -l
# List another user's jobs.
```

Example lines to put inside the crontab:

```cron
# Every day at 02:30, run a backup script and log its output
30 2 * * * /opt/scripts/backup.sh >> /var/log/backup.log 2>&1

# Every 5 minutes, check the app's health endpoint
*/5 * * * * curl -fsS http://localhost:3000/health > /dev/null || echo "App down" >> /var/log/health.log

# Every Monday at 09:00, delete temp files older than 7 days
0 9 * * 1 find /tmp/uploads -type f -mtime +7 -delete

# At every reboot
@reboot /opt/scripts/on-boot.sh
```

Tips for reliable cron jobs:

```bash
# 1. Use ABSOLUTE paths. Cron runs with a very small PATH and a different working directory.
# 2. Always redirect output (>> file 2>&1) or cron will try to email it and you will never see errors.
# 3. Test the command manually first, as the same user.
# 4. Check execution history:
grep CRON /var/log/syslog | tail        # Debian/Ubuntu
journalctl -u cron --since today         # Alternative on systemd systems
```

### 11. Power Tools for Text and Pipelines

```bash
sort names.txt                     # Sort lines alphabetically
sort -n numbers.txt                # Sort numerically
sort -h sizes.txt                  # Sort human-readable sizes (1K, 2M, 1G)
sort file.txt | uniq               # Remove adjacent duplicates (data must be sorted first)
sort file.txt | uniq -c            # Count how many times each line occurs
sort file.txt | uniq -c | sort -nr | head -n 5   # Top 5 most frequent lines

cut -d: -f1 /etc/passwd            # Split each line on ":" and print the 1st field (usernames)
tr 'a-z' 'A-Z' < file.txt          # Translate lowercase to uppercase
diff old.conf new.conf             # Show differences between two files
diff -u old.conf new.conf          # Same, in the familiar "unified" format used by Git

echo "line" | tee output.txt       # Print to the screen AND save to a file (tee = T-junction)
echo "line" | tee -a output.txt    # Same but append
echo "key=1" | sudo tee /etc/myapp.conf > /dev/null
# Common trick: "sudo echo ... > file" fails because the redirect runs as YOU, not root.
# Piping into "sudo tee" fixes it.

find /tmp -name "*.tmp" -print0 | xargs -0 rm -f
# xargs turns the output of one command into arguments for another.
# -print0 / -0 safely handle file names containing spaces.

time ./script.sh                   # Measure how long a command takes
timeout 10 curl http://slow-site   # Kill a command automatically if it runs longer than 10 s
```

### 12. Exit Codes and Chaining in Practice

```bash
ls /etc/hosts
echo $?                            # 0 = success

ls /nonexistent
echo $?                            # Non-zero (2) = failure

mkdir -p /tmp/demo && cd /tmp/demo && echo "ready"
# Each step runs only if the previous one succeeded.

systemctl is-active --quiet nginx || sudo systemctl start nginx
# If nginx is NOT active, start it.
```

### 13. Package Management Beyond Install (Debian/Ubuntu)

```bash
apt list --installed | grep nginx      # Is a package installed? Which version?
apt list --upgradable                  # What updates are pending?
apt show nginx                         # Details about a package
dpkg -l | grep nginx                   # Low-level list of installed packages
dpkg -L nginx                          # List all files a package installed
dpkg -S /usr/sbin/nginx                # Which package owns this file?
sudo apt-mark hold nginx               # Prevent a package from being upgraded
sudo apt-mark unhold nginx             # Allow upgrades again
sudo apt autoremove -y                 # Remove dependencies no longer needed
```

---

## Real-World Use Case

### Scenario: Deploying an API as a managed service, and responding to a 3 AM incident

**Part 1: Deploy it properly**

A backend team hands you a Node.js API. Running `node server.js` in an SSH window is not acceptable in production because it dies when you log out or the app crashes. You do the following:

1. Create a dedicated **unprivileged user** (`appuser`) to run the app (security).
2. Write the **systemd unit file** (see example 6) with `Restart=on-failure`.
3. Run `sudo systemctl enable --now myapp` so it starts at boot and now.
4. Add a **logrotate** rule so logs don't fill the disk.
5. Add a **cron job** for a nightly backup and temp-file cleanup.

```
                   +---------------------+
   boot ---------> |   systemd           |
                   |   (manager)         |
                   +----------+----------+
                              | starts + supervises
                              v
                   +---------------------+      writes      +----------------+
                   |  myapp.service      | ---------------> |  journal/logs  |
                   |  (your Node app)    |                  +----------------+
                   +---------------------+
                              ^
                   cron (02:30 daily) runs backup.sh
```

**Part 2: The 3 AM incident**

An alert says the API is slow. You SSH in and work through it logically:

```bash
# 1. Is the machine overloaded?
uptime                      # Load 9.5 on a 2-core machine: heavily overloaded
nproc                       # 2

# 2. Who is using the CPU and memory?
top                         # Press M (sort by memory). One "java" or "node" process uses 90% RAM.
ps aux --sort=-%mem | head -n 5

# 3. Is the service healthy?
systemctl status myapp
journalctl -u myapp --since "1 hour ago" -p err

# 4. Did the kernel kill something (out-of-memory)?
dmesg -T | grep -i "killed process"

# 5. Is the disk full?
df -h
du -sh /var/log/* | sort -h | tail -n 3

# 6. Take action: restart the service gracefully
sudo systemctl restart myapp
systemctl is-active myapp

# 7. If a rogue process will not stop:
kill 4321                   # Try the polite signal first
sleep 5; ps -p 4321         # Still there?
kill -9 4321                # Last resort

# 8. Confirm recovery and keep watching
journalctl -u myapp -f
```

**Why this matters:** In production you rarely "fix" anything by guessing. You follow a fixed order: **load, then processes, then service status, then logs, then disk and memory, then action, then verification**. These commands give you the evidence at each step. And because the service is managed by systemd with auto-restart, many crashes heal themselves before anyone is paged.

> **Analogy:** A hospital triage: check vital signs (`uptime`, `free`, `df`), find the problem organ (`top`, `ps`), read the patient chart (`journalctl`, logs), treat (`restart`, `kill`), and monitor recovery (`journalctl -f`).

---

## Interview Questions & Answers

### Q1. What is the difference between `kill` and `kill -9`?

**Answer:**
`kill <PID>` sends **SIGTERM (15)**, a polite request to terminate, which lets the process finish its work, close files and connections, and clean up. `kill -9 <PID>` sends **SIGKILL**, which the kernel enforces immediately; the process cannot catch or ignore it, so it gets no chance to clean up and may leave temp files, locks, or corrupted data. Best practice: try SIGTERM first, wait a few seconds, and use SIGKILL only if the process is unresponsive.

---

### Q2. What is the difference between `systemctl start` and `systemctl enable`?

**Answer:**
`start` runs the service **right now** but does not affect what happens after a reboot. `enable` configures the service to **start automatically at boot** but does not start it now. To do both in one step, use `systemctl enable --now <service>`. A common mistake is starting a service, forgetting to enable it, and then finding it down after a server reboot.

---

### Q3. What does "load average" mean, and how do you interpret it?

**Answer:**
Load average is the average number of processes that are either running on the CPU or waiting to run, shown for the last 1, 5, and 15 minutes (via `uptime` or `top`). You must compare it with the **number of CPU cores** (`nproc`): a load of 4 on a 4-core machine means fully used, while a load of 8 means processes are queueing and the system is overloaded. If the 1-minute value is higher than the 15-minute value, the load is increasing; if lower, it is easing. Also note that processes waiting on slow disk I/O count towards load, so high load with low CPU usage often points to a disk problem.

---

### Q4. What is a zombie process, and how do you deal with it?

**Answer:**
A zombie is a process that has **finished executing** but still has an entry in the process table because its **parent hasn't read its exit status** (shown as `Z` in `ps`). It uses essentially no CPU or memory, and you cannot kill it because it is already dead. The fix is to make the **parent** process reap it: send the parent a signal if it supports one, or restart or kill the parent so the zombie is adopted by `init`/systemd and cleaned up. A few zombies are harmless; a growing number indicates a bug in the parent program.

---

### Q5. How would you run a script every day at 2:30 AM, and how would you troubleshoot it if it didn't run?

**Answer:**
Edit the crontab with `crontab -e` and add: `30 2 * * * /opt/scripts/backup.sh >> /var/log/backup.log 2>&1`. The fields are minute, hour, day of month, month, and day of week. If it does not run, I would check: (1) the script is **executable** (`chmod +x`) and I'm using **absolute paths**, because cron runs with a minimal environment; (2) the output log for errors; (3) the cron logs (`grep CRON /var/log/syslog` or `journalctl -u cron`) to see whether cron even attempted it; (4) that the cron service is running (`systemctl status cron`); and (5) that the command works when run manually as the same user.

---

## Quick Recap Cheat Sheet

| Task | Command |
|------|---------|
| List all processes | `ps aux` |
| Live process monitor | `top`, `htop` |
| Find a process by name | `pgrep -a name` |
| Stop politely / forcefully | `kill PID` / `kill -9 PID` |
| Stop by name | `pkill name` |
| Run in background, survive logout | `nohup cmd > out.log 2>&1 &` |
| Service status | `systemctl status svc` |
| Start now and at boot | `systemctl enable --now svc` |
| List failed services | `systemctl --failed` |
| Reload unit files | `systemctl daemon-reload` |
| Live service logs | `journalctl -u svc -f` |
| Kernel messages | `dmesg -T` |
| Load and cores | `uptime`, `nproc` |
| Memory / disk / inodes | `free -h`, `df -h`, `df -i` |
| Biggest folders | `du -sh * \| sort -h` |
| What is using a port | `lsof -i :PORT` |
| Edit / list cron jobs | `crontab -e`, `crontab -l` |
| Save and display output | `cmd \| tee file` |
| Last exit code | `echo $?` |
