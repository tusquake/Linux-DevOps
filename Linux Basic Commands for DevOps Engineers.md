# Linux Basic Commands for DevOps Engineers

> **Who is this for?** A backend developer who can now log into a Linux server (Topic 2) and needs to move around, create and read files, and find their way without a graphical interface. These commands are the "alphabet" of Linux. Everything advanced is built from them.

---

## What is it and Why do we need it?

### What are Linux commands?

A **command** is a short text instruction you type into the shell (Bash) to make the computer do something: list files, create a folder, read a log, copy a configuration. Each command is usually a tiny program that does exactly one job well.

The general shape of nearly every command is:

```
command  [options]  [arguments]
   |         |           |
   |         |           +-- WHAT to act on (a file, a folder, a name)
   |         +-- HOW to behave (usually starts with - or --)
   +-- the action to perform
```

Example: `ls -l /etc` means "list (`ls`), in long format (`-l`), the folder `/etc`".

> **Analogy: Giving Instructions to a Very Literal Assistant**
> Imagine an assistant who does exactly what you say and nothing more. "Open" is the **command**. "Quickly" is the **option**. "The red folder" is the **argument**. Because the assistant never guesses, your instructions must be precise, but the same precise instruction works every time. That is why commands can be automated.

### Why do we need them?

- Servers have **no mouse and no desktop**. The command line is the only interface.
- Commands are **fast**: one line can do what takes many clicks.
- Commands are **repeatable and scriptable**: you can save them in a file and run them on 100 servers. You cannot script a mouse click easily.
- Commands are **the language of troubleshooting**: when a site is down at 2 AM, you will be reading logs and checking files with exactly these tools.

---

## Core Concepts

### 1. Absolute Paths vs. Relative Paths

A **path** is the address of a file or folder.

| Type | Starts with | Meaning | Example |
|------|-------------|---------|---------|
| **Absolute path** | `/` | Full address from the root of the system | `/var/log/syslog` |
| **Relative path** | anything else | Address starting from where you are now | `logs/app.log` |

Special shortcuts:

| Symbol | Meaning |
|--------|---------|
| `.` | The current directory |
| `..` | The parent directory (one level up) |
| `~` | Your home directory (for example `/home/ubuntu`) |
| `-` | The previous directory you were in (used with `cd -`) |

> **Analogy: Giving Directions**
> An absolute path is a full postal address: "India, Kolkata, 12 Park Street, Flat 4". A relative path is "two doors down on the left", which only makes sense if you know where the person is standing.

---

### 2. Command Anatomy: Options and Arguments

- **Short options** use one dash and one letter: `ls -l`
- **Long options** use two dashes and a word: `ls --all`
- Short options can be combined: `ls -l -a -h` is the same as `ls -lah`
- Commands are **case-sensitive**: `File.txt` and `file.txt` are different files.
- Linux shows **no confirmation** for most actions and **no recycle bin**. When you delete something, it is gone.

---

### 3. Wildcards (Pattern Matching)

Wildcards let one command act on many files.

| Wildcard | Matches | Example |
|----------|---------|---------|
| `*` | Any number of any characters | `*.log` matches `app.log`, `error.log` |
| `?` | Exactly one character | `file?.txt` matches `file1.txt`, not `file10.txt` |
| `[abc]` | One character from the set | `file[123].txt` matches `file1.txt`, `file2.txt`, `file3.txt` |

> **Analogy:** A search with "blanks" in a crossword puzzle. `?` is one missing letter; `*` is "anything goes here".

---

### 4. Input, Output, and Redirection

Every command has three standard "streams":

| Stream | Meaning | Number | Default |
|--------|---------|--------|---------|
| **stdin** | Input going in | 0 | Your keyboard |
| **stdout** | Normal output coming out | 1 | Your screen |
| **stderr** | Error messages coming out | 2 | Your screen |

**Redirection** changes where these go:

| Operator | What it does | Example |
|----------|--------------|---------|
| `>` | Send output to a file, **overwriting** it | `echo "hi" > note.txt` |
| `>>` | Send output to a file, **appending** to the end | `echo "more" >> note.txt` |
| `2>` | Send only **errors** to a file | `ls /fake 2> errors.txt` |
| `&>` | Send both output and errors to a file | `command &> all.txt` |
| `<` | Read input from a file | `sort < names.txt` |

> **Analogy: Pipes and Buckets in a Factory**
> Each machine has an input hose, an output hose, and a waste hose (errors). By default, output flows to a window (your screen). Redirection re-routes that hose into a bucket (a file) instead.

---

### 5. Pipes: Chaining Commands Together

The pipe `|` takes the **output of one command** and feeds it as the **input of the next**.

```
command1  |  command2  |  command3
 (produce)    (filter)     (count)
```

> **Analogy: An Assembly Line**
> One station washes the vegetables, the next chops them, the next cooks them. Each station does one simple job. The pipe is the conveyor belt that moves the work along. This "small tools chained together" idea is the core philosophy of Linux.

---

### 6. Command Categories (the Big Picture)

| Category | What you do | Key commands |
|----------|-------------|--------------|
| **Navigate** | Know where you are and move | `pwd`, `ls`, `cd` |
| **Create** | Make files and folders | `touch`, `mkdir`, `echo`, `nano` |
| **Copy / Move / Delete** | Reorganize | `cp`, `mv`, `rm`, `rmdir` |
| **View** | Read file contents | `cat`, `less`, `head`, `tail`, `wc` |
| **Info** | Learn about the system and commands | `whoami`, `uname`, `date`, `man`, `which` |
| **Disk / Memory** | Quick resource checks | `df`, `du`, `free` |
| **Edit** | Change file content | `nano`, `vim` |
| **Convenience** | Work faster | `history`, `alias`, `clear`, Tab completion |

---

### 7. Links: Shortcuts to Files

| Type | Command | Behavior | Analogy |
|------|---------|----------|---------|
| **Symbolic (soft) link** | `ln -s target linkname` | A pointer to another path. If the original is deleted, the link breaks | A desktop shortcut |
| **Hard link** | `ln target linkname` | A second name for the *same data*. Data stays until all names are deleted | Two people sharing one house key copy; the house remains while either exists |

Symbolic links are far more common in DevOps (for example, pointing `current` to the latest release folder).

---

## Practical Examples & Commands

> Practice in a throwaway folder. Run `mkdir ~/practice && cd ~/practice` before you start.

### 1. Know Where You Are: `pwd`, `whoami`, `hostname`

```bash
pwd                # "Print Working Directory": shows your current location, e.g. /home/ubuntu
whoami             # Shows which user you are logged in as
hostname           # Shows the name of this machine
id                 # Shows your user ID and the groups you belong to
```

### 2. Look Around: `ls`

```bash
ls                 # List files and folders in the current directory
ls -l              # Long format: permissions, owner, size, date, name
ls -a              # Show ALL files, including hidden ones (names starting with .)
ls -lh             # Long format with human-readable sizes (KB, MB, GB)
ls -lah            # Long + all + human-readable (the most commonly used combination)
ls -lt             # Sort by modification time, newest first
ls -ltr            # Same, but reversed: newest LAST (great for finding recent logs)
ls -R              # List recursively (folders and all subfolders)
ls /etc            # List a different folder without moving into it
ls *.log           # List only files ending in .log (uses a wildcard)
```

Reading `ls -l` output:

```text
-rw-r--r-- 1 ubuntu ubuntu 1024 Jan 10 09:15 app.log
|           |  |      |      |    |           |
|           |  |      |      |    |           +-- file name
|           |  |      |      |    +-- last modified date/time
|           |  |      |      +-- size in bytes
|           |  |      +-- group owner
|           |  +-- user owner
|           +-- number of links
+-- file type (- = file, d = directory, l = link) + permissions (covered in Topic 6)
```

### 3. Move Around: `cd`

```bash
cd /var/log        # Go to an absolute path
cd ..              # Go up one level (to the parent folder)
cd ../..           # Go up two levels
cd ~               # Go to your home directory (same as just typing: cd)
cd -               # Jump back to the previous directory you were in
cd logs/archive    # Go to a relative path (inside the current folder)
```

> **Tip:** Press the **Tab** key to auto-complete file and folder names. Press **Tab twice** to see all possibilities. This saves enormous time and prevents typos.

### 4. Create Folders and Files: `mkdir`, `touch`, `echo`

```bash
mkdir projects                 # Create a single folder called "projects"
mkdir -p projects/app/logs     # -p creates the whole chain of folders at once (parents too)
touch notes.txt                # Create an empty file (or update its timestamp if it exists)
touch a.txt b.txt c.txt        # Create several files in one command

echo "Hello DevOps"            # Print text to the screen
echo "Hello DevOps" > hello.txt    # Write text into a file (OVERWRITES existing content)
echo "Second line" >> hello.txt    # Add text to the end of the file (APPENDS)
```

### 5. Read Files: `cat`, `less`, `head`, `tail`, `wc`

```bash
cat hello.txt                  # Print the whole file to the screen
cat -n hello.txt               # Print with line numbers

less /var/log/syslog           # Open a large file page by page
                               #   Space = next page, b = back, /word = search,
                               #   n = next match, q = quit

head hello.txt                 # Show the first 10 lines
head -n 3 hello.txt            # Show the first 3 lines

tail hello.txt                 # Show the last 10 lines
tail -n 50 /var/log/syslog     # Show the last 50 lines of a log
tail -f /var/log/syslog        # "Follow": keep the file open and print NEW lines live
                               #   Press Ctrl + C to stop following

wc -l hello.txt                # Count lines (wc = word count; -l = lines)
wc -w hello.txt                # Count words
wc -c hello.txt                # Count bytes (characters)
```

> `tail -f` is one of the most-used commands in operations. It is how you watch a log while you test your application.

### 6. Copy, Move, Rename: `cp`, `mv`

```bash
cp hello.txt backup.txt            # Copy a file to a new name
cp hello.txt ~/practice/           # Copy a file into another folder
cp -r projects projects_backup     # -r = recursive: required to copy a FOLDER and its contents
cp -i hello.txt backup.txt         # -i = ask before overwriting an existing file

mv backup.txt archive.txt          # RENAME a file (move it to a new name)
mv archive.txt ~/practice/         # MOVE a file into another folder
mv *.txt docs/                     # Move all .txt files into the docs folder
```

### 7. Delete: `rm`, `rmdir` (Be Careful!)

```bash
rm archive.txt                 # Delete a file (no recycle bin: it is gone for good)
rm -i archive.txt              # -i = ask for confirmation before deleting
rm a.txt b.txt c.txt           # Delete several files
rm *.log                       # Delete all .log files in the current folder
rmdir emptyfolder              # Delete an EMPTY folder only (safe: refuses if not empty)
rm -r projects_backup          # -r = delete a folder and everything inside it
rm -rf projects_backup         # -f = force, no questions asked. Very dangerous!
```

> **WARNING:** Never run `rm -rf /` or `rm -rf /*` or `rm -rf` with a variable that might be empty (like `rm -rf $DIR/`). Always double-check the path before pressing Enter. On a server, there is no undo.
>
> **Analogy:** `rm` is a paper shredder, not a recycle bin.

### 8. Edit Files: `nano` and `vim`

```bash
nano notes.txt
# A beginner-friendly editor.
#   Type normally.  Ctrl + O = save (press Enter to confirm).
#   Ctrl + X = exit.  The shortcut list is shown at the bottom of the screen.

vim notes.txt
# A powerful editor available on nearly every server. Survival basics:
#   Press i        -> enter INSERT mode (now you can type)
#   Press Esc      -> go back to NORMAL mode
#   Type :wq Enter -> write (save) and quit
#   Type :q! Enter -> quit WITHOUT saving
#   Type /word Enter -> search for "word"
```

### 9. Search and Find Basics

```bash
which python3                  # Where is this command's program located?
type cd                        # Is it a program, a built-in, or an alias?
whereis nginx                  # Locate the binary and docs for a program

find ~ -name "hello.txt"       # Search your home folder for a file by name
find /var/log -name "*.log"    # Find all .log files under /var/log
grep "error" app.log           # Show lines in a file that contain the word "error"
grep -i "error" app.log        # Same but ignore upper/lower case
```

(`find` and `grep` are covered in depth in Topic 9.)

### 10. Pipes in Action

```bash
ls -l | head -n 5              # List files, but show only the first 5 lines

ls | wc -l                     # Count how many items are in the current folder

cat /etc/passwd | grep "ubuntu"    # Read a file and show only the lines containing "ubuntu"

ps aux | grep nginx            # List all running processes and filter for nginx

history | grep ssh             # Find old commands you typed that contained "ssh"
```

### 11. Redirect Errors and Output

```bash
ls /does-not-exist 2> errors.txt        # Save the error message to a file instead of the screen
ls /etc /does-not-exist > out.txt 2> err.txt   # Good output to one file, errors to another
command > /dev/null 2>&1                # Throw away ALL output (/dev/null is the "black hole")
```

### 12. System Information

```bash
date                           # Current date and time
uname -a                       # Kernel and system information
cat /etc/os-release            # Distribution name and version
uptime                         # How long the system has been running + load
df -h                          # Free and used disk space per filesystem
du -sh ~/practice              # Total size of a folder (-s summary, -h human readable)
du -sh *                       # Size of every item in the current folder
free -h                        # RAM and swap usage
```

### 13. Work Faster: history, alias, clear, shortcuts

```bash
history                        # Show a numbered list of past commands
!42                            # Re-run command number 42 from history
!!                             # Re-run the previous command
sudo !!                        # Re-run the previous command with sudo (handy after "Permission denied")

alias ll='ls -lah'             # Create a shortcut: typing "ll" now runs "ls -lah"
alias                          # List all current aliases
unalias ll                     # Remove the shortcut

clear                          # Clean the screen (or press Ctrl + L)
```

Useful keyboard shortcuts in the shell:

| Shortcut | Action |
|----------|--------|
| `Tab` | Auto-complete |
| `Ctrl + C` | Stop the running command |
| `Ctrl + R` | Search backwards through command history |
| `Ctrl + A` / `Ctrl + E` | Jump to start / end of the line |
| `Ctrl + L` | Clear the screen |
| `Ctrl + D` | Log out / end of input |
| Up / Down arrows | Browse previous commands |

### 14. Links

```bash
ln -s /opt/app/releases/v2 /opt/app/current
# Creates a SYMBOLIC link named "current" that points to the v2 release folder.
# To switch versions later, just repoint the link. Your app always uses ".../current".

ls -l /opt/app/current         # Shows: current -> /opt/app/releases/v2
```

### 15. Environment Variables (Quick Look)

```bash
echo $HOME                     # Print the value of the HOME variable
echo $PATH                     # Print the list of folders the shell searches for commands
export APP_ENV=production      # Create a variable available to programs you start
echo $APP_ENV                  # Prints: production
env | grep APP                 # Show environment variables containing "APP"
```

---

## Real-World Use Case

### Scenario: "The API is returning errors!" Investigating on a production server

A teammate messages you: *users are seeing 500 errors from the API.* You SSH into the server and use only basic commands to find the cause.

```bash
# 1. Confirm where you are and who you are
whoami && hostname
# && means "run the second command only if the first succeeded"

# 2. Check that the server is not out of disk space (a very common cause)
df -h
# Look at the "Use%" column. 100% means the disk is full and apps cannot write logs or data.

# 3. Check memory
free -h

# 4. Go to the application's log folder and find the newest files
cd /var/log/myapp
ls -lhtr
# -t sorts by time, -r reverses it, so the latest file is at the bottom

# 5. Look at the end of the current log
tail -n 100 app.log

# 6. Watch the log live while you reproduce the problem
tail -f app.log
# (Press Ctrl + C when done)

# 7. Count how many errors occurred
grep -c "ERROR" app.log
# -c prints the count of matching lines instead of the lines themselves

# 8. Show only the last 20 error lines, saved to a file for the incident report
grep "ERROR" app.log | tail -n 20 > ~/incident-errors.txt

# 9. Check which folder is eating disk space
du -sh /var/log/* | sort -h | tail -n 5
# The 5 biggest items in /var/log, smallest to largest

# 10. Check the configuration file that the app uses
cat /etc/myapp/config.yaml

# 11. Make a safe backup BEFORE editing the config
cp /etc/myapp/config.yaml /etc/myapp/config.yaml.bak
sudo nano /etc/myapp/config.yaml
```

**What happened here:** You used about ten basic commands, with no special tools, to check the machine's health (`df`, `free`), find the right logs (`cd`, `ls -lhtr`), read them (`tail`, `grep`), preserve evidence (`>`), and make a safe change (`cp` then edit). This is what most day-to-day DevOps troubleshooting looks like.

> **Analogy:** You are a doctor doing a basic check-up: temperature (`uptime`), blood pressure (`free`), weight (`df`), then reading the patient's chart (`tail` on logs) before deciding on treatment.

---

## Interview Questions & Answers

### Q1. What is the difference between an absolute path and a relative path?

**Answer:**
An **absolute path** starts from the root directory `/` and gives the full location, such as `/var/log/syslog`. It works from anywhere. A **relative path** starts from the **current working directory**, such as `logs/app.log` or `../config`. It depends on where you currently are. In scripts and automation, absolute paths are safer because they don't change meaning depending on where the script is run.

---

### Q2. What is the difference between `>` and `>>`?

**Answer:**
Both redirect command output into a file. `>` **overwrites** the file, deleting existing content first (or creating the file if missing). `>>` **appends** to the end of the file and keeps the existing content. A common mistake is using `>` on a log or config file and wiping its contents by accident.

---

### Q3. How do you watch a log file in real time, and why is that useful?

**Answer:**
Use `tail -f /path/to/logfile`. The `-f` (follow) option keeps the file open and prints new lines as they are written. It's useful when deploying or debugging because you can trigger an action (like an API request) and immediately see what the application logs in response. Press `Ctrl + C` to stop. `tail -F` is similar but also keeps following if the log file is rotated or recreated.

---

### Q4. What is the difference between `rm`, `rmdir`, and `rm -rf`? What are the risks?

**Answer:**
`rm` deletes files. `rmdir` deletes only **empty** directories and refuses otherwise, which makes it safe. `rm -r` deletes a directory and everything inside it, and adding `-f` forces deletion without any confirmation. The risk is that Linux has **no recycle bin**, so deletions are permanent. A wrong path or an empty variable in `rm -rf` can wipe critical data. Best practices: double-check the path, use `ls` first to preview what will be deleted, use `rm -i` when unsure, and take backups.

---

### Q5. What does the pipe (`|`) do? Give an example.

**Answer:**
The pipe sends the **standard output** of one command into the **standard input** of the next, allowing you to chain small tools into a powerful workflow. For example, `ps aux | grep nginx` lists all processes and then filters for lines containing "nginx". Another example: `ls | wc -l` counts the number of files in a folder. The idea is that each tool does one job, and pipes combine them.

---

## Quick Recap Cheat Sheet

| Task | Command |
|------|---------|
| Where am I? | `pwd` |
| List files (detailed, hidden, readable) | `ls -lah` |
| Change folder | `cd /path`, `cd ..`, `cd ~`, `cd -` |
| Create folder / nested folders | `mkdir name`, `mkdir -p a/b/c` |
| Create empty file | `touch file` |
| Copy file / folder | `cp a b`, `cp -r dir1 dir2` |
| Move / rename | `mv old new` |
| Delete file / folder | `rm file`, `rm -r dir` |
| Show file | `cat file`, `less file` |
| First / last lines | `head -n 5 file`, `tail -n 5 file` |
| Follow a log live | `tail -f file` |
| Count lines | `wc -l file` |
| Overwrite / append to file | `>` / `>>` |
| Chain commands | `cmd1 \| cmd2` |
| Disk / memory | `df -h`, `du -sh *`, `free -h` |
| Help | `man cmd`, `cmd --help` |
| Repeat last command with sudo | `sudo !!` |
