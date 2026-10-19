# Pro Linux Commands: AWK, GREP, FIND, SED

> **Who is this for?** A backend developer who can navigate Linux, read logs with `tail` and `cat`, and now faces files with **millions of lines** and **thousands of folders**. These four commands turn "I can't find anything in all this data" into "I have the answer in one line". They are the tools that separate a beginner from someone who can actually *investigate* a production system.

---

## What is it and Why do we need it?

### The four tools in one sentence each

| Tool | Stands for / Meaning | Its one job |
|------|----------------------|-------------|
| **`grep`** | **G**lobal **R**egular **E**xpression **P**rint | **Search** inside files for lines that match a pattern |
| **`find`** | find | **Locate** files and directories by name, size, age, owner, permissions, and more |
| **`sed`** | **S**tream **ED**itor | **Edit** text automatically as it flows past: replace, delete, insert |
| **`awk`** | named after its creators: **A**ho, **W**einberger, **K**ernighan | **Extract and calculate** from structured text: columns, sums, counts, reports |

> **Analogy: A Library Research Team**
> - **`find`** is the **librarian who locates the right book** on the right shelf ("all books bigger than 500 pages, published before 2020").
> - **`grep`** is the **researcher who flips through the book** and marks every page that mentions a keyword.
> - **`awk`** is the **analyst who reads a table** in the book and tells you "the third column adds up to 4,480".
> - **`sed`** is the **editor with a correction pen** who fixes every occurrence of a wrong word, automatically, across the whole manuscript.

### Why do we need them?

Servers produce a flood of text: logs, configuration files, command output. Reading it by eye is impossible, and opening a graphical editor is often not an option. These tools:

- **Find the needle**: which of 50,000 files contains the error? Which IP address hit the server 10,000 times?
- **Work in pipelines**: the output of one becomes the input of the next (Topic 3), so small commands combine into powerful ones.
- **Are scriptable**: the same one-liner runs identically on 1 server or 1,000, which is the heart of DevOps automation.
- **Are available everywhere**: on every Linux server, container, and CI runner, with nothing to install.

---

## Core Concepts

### 1. Sample Data Used Throughout This Guide

Create these two practice files so that every example below can be run and its output verified.

```bash
mkdir -p ~/lab9 && cd ~/lab9

cat > access.log <<'EOF'
203.0.113.5 - - [10/Jan/2025:09:15:01 +0000] "GET /index.html HTTP/1.1" 200 1024 "-" "Mozilla/5.0"
203.0.113.5 - - [10/Jan/2025:09:15:03 +0000] "GET /api/users HTTP/1.1" 200 512 "-" "Mozilla/5.0"
198.51.100.7 - - [10/Jan/2025:09:16:10 +0000] "POST /api/login HTTP/1.1" 401 128 "-" "curl/8.0"
198.51.100.7 - - [10/Jan/2025:09:16:12 +0000] "POST /api/login HTTP/1.1" 401 128 "-" "curl/8.0"
198.51.100.7 - - [10/Jan/2025:09:16:14 +0000] "POST /api/login HTTP/1.1" 401 128 "-" "curl/8.0"
192.0.2.44 - - [10/Jan/2025:10:02:45 +0000] "GET /missing.png HTTP/1.1" 404 0 "-" "Mozilla/5.0"
203.0.113.5 - - [10/Jan/2025:10:05:00 +0000] "GET /api/orders HTTP/1.1" 500 256 "-" "Mozilla/5.0"
192.0.2.44 - - [10/Jan/2025:10:07:30 +0000] "GET /index.html HTTP/1.1" 200 1024 "-" "Mozilla/5.0"
203.0.113.5 - - [10/Jan/2025:10:09:12 +0000] "GET /api/orders HTTP/1.1" 500 256 "-" "Mozilla/5.0"
203.0.113.5 - - [10/Jan/2025:10:10:00 +0000] "GET /index.html HTTP/1.1" 200 1024 "-" "Mozilla/5.0"
EOF

cat > app.conf <<'EOF'
# Application settings
server_name = myapp
port = 8080

# Database
db_host = localhost
db_port = 5432
debug = true
EOF
```

How to read one web server log line (the common "combined" format):

```text
203.0.113.5 - - [10/Jan/2025:09:15:01 +0000] "GET /index.html HTTP/1.1" 200 1024 "-" "Mozilla/5.0"
    $1                $4                      $6   $7          $8        $9   $10
 client IP        timestamp              method   URL       protocol  status bytes
```

The `$1`, `$4`, and so on are **field numbers** that `awk` uses, which are split on spaces by default.

---

### 2. Regular Expressions (Regex): The Pattern Language

`grep`, `sed`, and `awk` all understand **regular expressions**: a mini-language for describing text patterns. Learn it once, use it everywhere.

> **Analogy:** A regex is a **wildcard on steroids**. In Topic 3, `*.log` matched file names. A regex can say "a word starting with `err`, followed by one or more digits, at the end of a line".

| Symbol | Meaning | Example | Matches |
|--------|---------|---------|---------|
| `.` | Any single character | `h.t` | hat, hot, hit |
| `*` | Zero or more of the previous item | `ab*` | a, ab, abbb |
| `+` | One or more of the previous item (extended regex) | `ab+` | ab, abbb (not a) |
| `?` | Zero or one of the previous item (extended regex) | `colou?r` | color, colour |
| `^` | Start of the line | `^ERROR` | lines **beginning** with ERROR |
| `$` | End of the line | `failed$` | lines **ending** with failed |
| `[abc]` | Any one character in the set | `[aeiou]` | any vowel |
| `[^abc]` | Any one character **not** in the set | `[^0-9]` | any non-digit |
| `[a-z]`, `[0-9]` | A range | `[0-9]` | any digit |
| `{n}`, `{n,m}` | Repeat exactly n, or n to m times (extended regex) | `[0-9]{3}` | exactly 3 digits |
| `(a)(b)` | Group items | `(ab)+` | ab, abab |
| `\|` | Alternation: "or" (extended regex) | `cat\|dog` | cat or dog |
| `\.` | A literal dot (the backslash "escapes" the special meaning) | `1\.5` | 1.5 only |
| `\b` | Word boundary (GNU) | `\bcat\b` | the word cat, not "concatenate" |
| `[[:digit:]]`, `[[:space:]]`, `[[:alpha:]]` | Named character classes | `[[:digit:]]+` | one or more digits |

**Basic vs. extended regex:** By default, `grep` and `sed` use **basic** regex, where `+ ? | ( ) { }` must be written with a backslash to have special meaning. Adding **`-E`** (extended regex) lets you write them naturally. **Always prefer `-E`**: it is easier to read.

---

### 3. How the Four Tools Fit Together

```
 find  ---->  produces a LIST of files
   |
   v
 grep  ---->  filters LINES matching a pattern
   |
   v
 sed   ---->  TRANSFORMS the text of those lines
   |
   v
 awk   ---->  pulls out COLUMNS and does MATH on them
```

Typical pipeline thinking: *"find the right files, grep out the interesting lines, awk out the interesting columns, sort and count them."*

---

### 4. How `awk` Thinks

`awk` reads input **one line (record) at a time**, splits each line into **fields**, and runs your instructions. A program has the form:

```
awk 'BEGIN { before any input } pattern { action } END { after all input }' file
```

| Part | Meaning |
|------|---------|
| `BEGIN { ... }` | Runs **once, before** reading input (set up variables, print headers) |
| `pattern { action }` | For every line that matches the **pattern**, run the **action**. No pattern means every line |
| `END { ... }` | Runs **once, after** the last line (print totals, summaries) |

**Built-in variables:**

| Variable | Meaning |
|----------|---------|
| `$0` | The entire current line |
| `$1`, `$2`, ... | The first, second, ... field |
| `$NF` | The **last** field (`NF` = Number of Fields); `$(NF-1)` is the second to last |
| `NR` | Number of the current **R**ecord (line number, across all files) |
| `FS` | **F**ield **S**eparator for input (default: whitespace); set with `-F` |
| `OFS` | **O**utput **F**ield **S**eparator, the character printed between fields in `print a, b` (default: a space) |

> **Analogy:** `awk` is like a **spreadsheet that works on text files**. Every line is a row, every word is a cell, `$3` is "column C", and `END` is the "total" row at the bottom.

---

### 5. How `sed` Thinks

`sed` reads a file **line by line**, applies your editing commands to each line, and prints the result. By default it **does not change the original file**; it just prints the edited version. To change the file itself, you must ask explicitly with `-i` ("in-place").

The command you will use 90% of the time is **substitute**:

```
sed 's/OLD/NEW/flags'
     | |   |   |
     | |   |   +-- optional flags: g = every match on the line, i = ignore case
     | |   +-- replacement text
     | +-- pattern to find (a regex)
     +-- s = substitute
```

> **Analogy:** Word's "Find and Replace", but scriptable, repeatable, and usable on a thousand files at once.

---

### 6. How `find` Thinks

`find` walks a directory tree and tests **every file** against the conditions you give it, then performs an action on those that pass all tests.

```
find  WHERE   TESTS...              ACTION
find  /var/log -type f -name "*.log" -mtime +7  -delete
        |        |          |            |         |
     start     only      name        older       what to do
     folder    files     pattern     than 7 days  with matches
```

If no action is given, the default is to **print** the matching paths.

| Time test | Meaning |
|-----------|---------|
| `-mtime +7` | modified **more than** 7 days ago |
| `-mtime -7` | modified **within the last** 7 days |
| `-mtime 7` | modified exactly 7 days ago (rarely useful) |
| `-mmin -30` | modified within the last 30 **minutes** |

---

## Practical Examples & Commands

All examples assume you are inside `~/lab9` with the two sample files above.

### PART 1: `grep`: Search Text

#### 1.1 The basics

```bash
grep "ERROR" /var/log/syslog
# Print every line in the file containing the text ERROR.

grep "POST" access.log
# Show the three POST requests.

grep -i "error" /var/log/syslog
# -i = ignore case: matches ERROR, Error, error.

grep -n "port" app.conf
# -n = show line numbers.
# Output:
#   3:port = 8080
#   7:db_port = 5432

grep -c " 500 " access.log
# -c = COUNT matching lines (not print them). Output: 2

grep -v "^#" app.conf
# -v = INVERT: show lines that do NOT match. Here: hide comment lines.

grep -v -e "^#" -e "^$" app.conf
# Hide comments AND blank lines (multiple patterns with -e). A great way to read config files.

grep -w "port" app.conf
# -w = match WHOLE WORDS only. Matches "port = 8080" but not "db_port" (underscore counts as part of the word).

grep -x "debug = true" app.conf
# -x = the WHOLE line must match exactly.

grep -o "GET\|POST" access.log
# -o = print ONLY the matching part, one per line (no surrounding text).
```

#### 1.2 Searching many files

```bash
grep -r "db_host" /etc/myapp/
# -r = recursive: search every file under the directory.

grep -rn "TODO" ~/projects --include="*.py"
# Search only Python files, with line numbers.

grep -rn "password" /etc --exclude-dir=ssl -s
# --exclude-dir skips folders; -s hides "permission denied" noise.

grep -rl "db_host" /etc/myapp/
# -l = list only the NAMES of files that contain a match.

grep -L "debug" *.conf
# -L = list files that do NOT contain the pattern (opposite of -l).

grep -h "ERROR" *.log
# -h = hide the file names in the output (when searching multiple files).
```

#### 1.3 Context around the match

```bash
grep -A 3 "ERROR" app.log
# -A 3 = show the matching line plus 3 lines AFTER it.

grep -B 2 "ERROR" app.log
# -B 2 = show 2 lines BEFORE the match.

grep -C 2 "Exception" app.log
# -C 2 = show 2 lines of CONTEXT on both sides. Perfect for stack traces.
```

#### 1.4 Extended regex with `-E`

```bash
grep -E "ERROR|WARN|FATAL" app.log
# Lines containing ANY of the three words (| means "or").

grep -E "^(GET|POST)" requests.txt
# Lines that START with GET or POST.

grep -E "[0-9]{3}-[0-9]{4}" contacts.txt
# Pattern: three digits, a dash, four digits.

grep -Eo "([0-9]{1,3}\.){3}[0-9]{1,3}" access.log
# Extract every IPv4-looking address from the file.
#   [0-9]{1,3}\.   = 1 to 3 digits followed by a literal dot
#   (...){3}       = repeated three times
#   [0-9]{1,3}     = the last group of digits

grep -E "\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[a-z]{2,}\b" notes.txt
# A simple email address finder.

grep -F "1.5" versions.txt
# -F = treat the pattern as a FIXED string, not a regex (so "." is just a dot). Faster and safer for literal text.
```

#### 1.5 Using `grep` in pipelines

```bash
ps aux | grep nginx
# Filter another command's output. (This also matches the grep process itself.)

ps aux | grep "[n]ginx"
# Trick: the [n] stops grep from matching its own command line.

history | grep ssh
# Find an old command you typed.

grep -q "debug = true" app.conf && echo "Debug is ON"
# -q = quiet: print nothing, just set the exit status (0 if found). Ideal in scripts and conditions.

dpkg -l | grep -i nginx
# Is a package installed?

journalctl -u myapp --since today | grep -ci error
# Count the error lines for a service today.
```

---

### PART 2: `find`: Locate Files

#### 2.1 Find by name and type

```bash
find . -name "app.conf"
# Search from the current folder (.) downward for a file with this exact name.

find /etc -name "*.conf"
# All files ending in .conf under /etc. (Quote the pattern so the SHELL does not expand it.)

find /home -iname "readme*"
# -iname = case-insensitive name match: README, readme.md, Readme.txt.

find /var/www -type d -name "uploads"
# -type d = directories only.

find /var/log -type f
# -type f = regular files only. (Other types: l = symbolic link.)

find . -maxdepth 1 -type f
# -maxdepth 1 = do not go into subfolders. (Put maxdepth BEFORE the other tests.)

find . -mindepth 2 -type f
# Skip the top level: only files at least 2 levels deep.
```

#### 2.2 Find by size, age, owner, permissions

```bash
find /var -type f -size +100M
# Files LARGER than 100 megabytes. (Units: k, M, G. "+" = larger, "-" = smaller.)

find /var -type f -size +1G -exec ls -lh {} +
# Large files with their sizes displayed.

find /var/log -type f -name "*.log" -mtime +7
# Log files last modified MORE than 7 days ago.

find /tmp -type f -mtime -1
# Files modified in the LAST 24 hours.

find /var/log -type f -mmin -30
# Files modified in the last 30 MINUTES. Great during an incident: "what just changed?"

find /etc -type f -newer /etc/hostname
# Files modified more recently than another file.

find /home -user alice
# Files owned by alice.

find /srv -group devs -type f
# Files owned by the group devs.

find /var/www -type f -perm 777
# Files with EXACTLY these permissions (a security smell!).

find / -xdev -type f -perm -4000 2>/dev/null
# Setuid files (see Topic 6). -xdev stays on one filesystem; 2>/dev/null hides permission errors.

find /var/www -type f -perm /o+w
# Files that are WRITABLE by others (the "/" means "any of these bits").

find /data -empty -type d
# Empty directories.  (Also works with -type f for empty files.)
```

#### 2.3 Combine conditions

```bash
find /var/log -type f \( -name "*.log" -o -name "*.gz" \)
# -o means OR. Parentheses must be escaped with backslashes. Conditions are ANDed by default.

find . -type f -not -name "*.tmp"
# NOT: everything except .tmp files. (Also written as: ! -name "*.tmp")

find . -path ./node_modules -prune -o -type f -name "*.js" -print
# Skip a directory entirely (-prune), then search the rest.
# Standard "ignore node_modules" pattern.

find /var/log -type f -name "*.log" -size +50M -mtime +3
# Several tests together: ALL must be true.
```

#### 2.4 Do something with the results

```bash
find . -name "*.bak" -ls
# Print a detailed listing (like ls -l) for each match.

find /tmp/uploads -type f -mtime +7 -delete
# DELETE matching files. DANGEROUS: run the same command WITHOUT -delete first to preview!

find /var/www -type d -exec chmod 755 {} \;
# -exec runs a command for each match. {} is replaced by the file name; \; ends the command.
# This runs chmod once PER directory.

find /var/www -type f -exec chmod 644 {} +
# Ending with "+" instead of "\;" passes MANY files to one chmod call: much faster.

find . -name "*.log" -exec grep -l "ERROR" {} +
# Of all .log files, list those that contain the word ERROR.

find . -type f -name "*.tmp" -print0 | xargs -0 rm -f
# Safe pipeline for file names with spaces: -print0 and -0 use a NUL separator.

find /var -type f -printf "%s %p\n" 2>/dev/null | sort -nr | head -n 10
# The 10 largest files: prints "size path", sorts numerically descending, shows the top 10.
# (-printf is a GNU find feature.)

find /var/log -type f -name "*.log" -exec gzip -9 {} \;
# Compress each matching log individually.

find ~/projects -name "*.py" -exec grep -Hn "TODO" {} +
# Search for TODO across all Python files. -H forces file names to be shown.
```

> **Safety habit:** *Always run a destructive `find` (`-delete`, `-exec rm`) first with `-print` or `-ls`, check the list, then run it for real.*

---

### PART 3: `sed`: Edit Text Streams

#### 3.1 Substitute

```bash
sed 's/localhost/db.internal/' app.conf
# Replace the FIRST "localhost" on each line. Prints the result; the file is NOT changed.

sed 's/o/0/g' app.conf
# g = replace EVERY occurrence on each line, not just the first.

sed 's/DEBUG/info/gI' app.conf
# I (capital) = ignore case.  (GNU sed)

sed 's|/usr/local/bin|/opt/bin|g' script.sh
# Use a different delimiter (| instead of /) when the text itself contains slashes.
# Much more readable than: s/\/usr\/local\/bin/\/opt\/bin/g

sed 's/port = 8080/port = 9090/' app.conf
# A literal replacement of a configuration value.

sed -E 's/([a-z_]+) = (.*)/\2 = \1/' app.conf
# Capture groups: \1 and \2 refer to the parts matched in parentheses. This SWAPS key and value.

sed 's/^/    /' app.conf
# Add four spaces at the start of every line (indent).  ^ = start of line.

sed 's/$/;/' app.conf
# Append a semicolon to the end of every line.  $ = end of line.

sed 's/.*/"&"/' app.conf
# & stands for "the entire matched text": wrap each line in double quotes.
```

#### 3.2 Delete lines

```bash
sed '3d' app.conf              # Delete line 3
sed '2,4d' app.conf            # Delete lines 2 to 4
sed '$d' app.conf              # Delete the LAST line
sed '/^#/d' app.conf           # Delete lines that START with # (comments)
sed '/^$/d' app.conf           # Delete EMPTY lines
sed '/^#/d;/^$/d' app.conf     # Delete comments AND empty lines (separate commands with ;)
sed '/debug/d' app.conf        # Delete every line containing "debug"
sed '/^\s*#/d;/^\s*$/d' app.conf
# Same, but also catches comments and "blank" lines that start with spaces/tabs. (\s is a GNU extension.)
```

#### 3.3 Print only certain lines (`-n` and `p`)

```bash
sed -n '5,10p' file.txt        # Print ONLY lines 5 to 10
sed -n '$p' file.txt           # Print only the last line
sed -n '/ERROR/p' app.log      # Print only lines matching ERROR (like grep)
sed -n '/START/,/END/p' app.log
# Print everything BETWEEN a line matching START and a line matching END (inclusive).
```

`-n` turns off the automatic printing of every line, so only lines affected by the `p` command appear.

#### 3.4 Insert, append, and change lines (GNU sed)

```bash
sed '1i\# Managed by DevOps team' app.conf
# INSERT a line BEFORE line 1.

sed '$a\# end of file' app.conf
# APPEND a line AFTER the last line.

sed '/^port/a\max_connections = 100' app.conf
# Add a new line AFTER any line that starts with "port".

sed '/^debug/c\debug = false' app.conf
# CHANGE (replace) the entire matched line with new text.
```

#### 3.5 Edit files in place (the real power, and the real risk)

```bash
sed -i 's/debug = true/debug = false/' app.conf
# -i = modify the FILE ITSELF (GNU sed on Linux). No undo!

sed -i.bak 's/debug = true/debug = false/' app.conf
# -i.bak = edit in place BUT keep a backup copy named app.conf.bak. USE THIS HABIT.

diff app.conf.bak app.conf
# Compare the backup and the new file to verify exactly what changed.

sed -i.bak 's/old-server/new-server/g' /etc/myapp/*.conf
# Replace in MANY files at once.

grep -rl "old-server" /etc/myapp | xargs sed -i.bak 's/old-server/new-server/g'
# Only touch files that actually contain the text.

sudo sed -i.bak 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
# Realistic example: set PasswordAuthentication to "no" whether the line was commented (#) or not.
#   ^#\?   = optionally a leading #  (\? means "zero or one", a GNU basic-regex form)
#   .*     = the rest of the line
sudo sshd -t && echo "sshd config OK"
# ALWAYS validate the SSH configuration before restarting the service, or you may lock yourself out.
```

> **macOS note:** BSD sed (macOS) requires an argument after `-i`: use `sed -i '' 's/a/b/' file`, or `sed -i.bak ...`. The examples here use GNU sed (Linux).
>
> **Safety habit:** Run the command **without `-i`** first to preview the output. Then add `-i.bak`.

#### 3.6 Multiple commands

```bash
sed -e 's/localhost/db.internal/' -e 's/5432/5433/' app.conf
# Apply several edits in one pass with multiple -e options.

sed -n '2p;4p' app.conf
# Print only lines 2 and 4.

sed 'y/abc/ABC/' file.txt
# Transliterate characters: every a becomes A, b becomes B, c becomes C.
```

---

### PART 4: `awk`: Columns, Math, and Reports

#### 4.1 Print columns

```bash
awk '{print $1}' access.log
# Print the FIRST field of each line (the client IP).

awk '{print $1, $9}' access.log
# Print the IP and the status code, separated by a space.

awk '{print $1 " -> " $7}' access.log
# Add your own text between fields: "203.0.113.5 -> /index.html"

awk '{print $NF}' access.log
# The LAST field of each line.

awk '{print NR": "$0}' app.conf
# Prefix each line with its line number (like cat -n).

awk 'NR==3' app.conf                  # Print only line 3
awk 'NR>=2 && NR<=4' app.conf         # Print lines 2 to 4
awk 'NR>1' data.csv                   # Skip the first (header) line
awk 'END{print NR}' access.log        # Count the lines (prints: 10)
```

#### 4.2 Choose the field separator: `-F`

```bash
awk -F: '{print $1}' /etc/passwd
# -F: = split on a colon. Prints every username.

awk -F: '{print $1, $3, $7}' /etc/passwd
# Username, UID, and login shell.

awk -F: '$3 >= 1000 {print $1}' /etc/passwd
# Only users whose UID (field 3) is 1000 or more: the human accounts (see Topic 5).

awk -F, '{print $2}' data.csv
# Comma-separated values. (Works for simple CSV; fields containing quoted commas need a proper CSV tool.)

awk 'BEGIN{FS=","; OFS=" | "} {print $1, $2}' data.csv
# Set the input separator to comma and the output separator to " | " inside a BEGIN block.
```

#### 4.3 Filter lines with patterns and conditions

```bash
awk '/POST/' access.log
# Lines that match a regex (like grep).

awk '/POST/ {print $1, $7}' access.log
# For POST lines only, print the IP and URL.

awk '$9 == 500' access.log
# Lines whose 9th field (status) equals 500. Prints 2 lines.

awk '$9 >= 400 {print $1, $9, $7}' access.log
# All error responses (4xx and 5xx), with IP, status, and URL.

awk '$9 == 200 && $10 > 500' access.log
# Combine conditions with && (and) and || (or).

awk '$1 == "203.0.113.5" {print $7}' access.log
# Which URLs did one particular IP request?

awk '$7 ~ /^\/api\//' access.log
# ~ means "matches the regex". Lines where the URL starts with /api/.

awk '$7 !~ /\.(png|css|js)$/' access.log
# !~ means "does NOT match": exclude static files.

awk 'length($0) > 100' access.log
# Lines longer than 100 characters.
```

#### 4.4 Calculate: sums, averages, counts

```bash
awk '{sum += $10} END {print "Total bytes:", sum}' access.log
# Add up the bytes column. Output: Total bytes: 4480

awk '{sum += $10} END {print "Average:", sum/NR}' access.log
# Average bytes per request. Output: Average: 448

awk 'BEGIN{max=0} $10 > max {max=$10; line=$0} END {print max; print line}' access.log
# Find the line with the largest value in a column.

awk '$9 == 500 {n++} END {print n " server errors"}' access.log
# Count matching lines. Output: 2 server errors

awk -v limit=500 '$10 > limit {print $7, $10}' access.log
# -v passes a shell value INTO awk as a variable.
```

#### 4.5 Group and count with arrays (the awk superpower)

```bash
awk '{count[$9]++} END {for (code in count) print code, count[code]}' access.log
# Count how many times each STATUS CODE appears.
# count[$9]++ means: "add 1 to the counter named after this line's status code".
# Typical output (the order of "for ... in" is not guaranteed):
#   200 4
#   401 3
#   404 1
#   500 2

awk '{bytes[$1] += $10} END {for (ip in bytes) print ip, bytes[ip]}' access.log
# Total bytes transferred PER IP address.

awk '{count[$1]++} END {for (ip in count) print count[ip], ip}' access.log | sort -nr
# Count per IP, then sort numerically (n) in reverse (r), highest first. Output:
#   5 203.0.113.5
#   3 198.51.100.7
#   2 192.0.2.44
```

#### 4.6 Formatting, built-in functions, and conditions

```bash
awk '{printf "%-15s %5s %s\n", $1, $9, $7}' access.log
# printf gives neat aligned columns:
#   %-15s = a string, left-aligned in 15 characters
#   %5s   = a string, right-aligned in 5 characters

awk 'BEGIN{printf "%-15s %s\n", "IP", "STATUS"} {printf "%-15s %s\n", $1, $9}' access.log
# BEGIN prints a header row before the data.

awk '{ if ($9 >= 500) print "SERVER ERROR:", $7; else if ($9 >= 400) print "CLIENT ERROR:", $7 }' access.log
# if / else if logic.

awk '{print toupper($7)}' access.log          # Convert to upper case
awk '{print length($7)}' access.log           # Length of a field
awk '{print substr($4, 2, 11)}' access.log    # Part of a string: the date, e.g. 10/Jan/2025
awk '{gsub(/\//, "-", $4); print $4}' access.log   # Replace characters within a field
```

#### 4.7 awk with other system commands

```bash
df -h | awk 'NR>1 && $5+0 > 80 {print $6, $5}'
# Disk partitions above 80% full.
#   NR>1  skip the header
#   $5+0  converts a value like "85%" into the number 85 so it can be compared
#   print the mount point ($6) and its usage ($5)

ps aux | awk '$3 > 50 {print $2, $3, $11}'
# Processes using more than 50% CPU: PID, %CPU, command.

ps aux | awk '{mem += $4} END {print "Total memory used by all processes: " mem "%"}'
# Sum a column from process output.

free -m | awk '/^Mem:/ {printf "Memory used: %.1f%%\n", $3/$2*100}'
# Compute a percentage from the "free" command output.

ls -l | awk '{sum += $5} END {print sum " bytes"}'
# Total size of the files in a listing.

awk -F: '{print $7}' /etc/passwd | sort | uniq -c | sort -nr
# How many users use each login shell?
```

---

### PART 5: Putting It All Together (Pipelines)

Using `access.log`:

```bash
# 1. Top client IPs by number of requests
awk '{print $1}' access.log | sort | uniq -c | sort -nr | head -n 5
# Output:
#       5 203.0.113.5
#       3 198.51.100.7
#       2 192.0.2.44

# 2. Distribution of HTTP status codes
awk '{print $9}' access.log | sort | uniq -c | sort -nr
# Output:
#       4 200
#       3 401
#       2 500
#       1 404

# 3. Which URLs produce server errors (5xx)?
awk '$9 >= 500 {print $7}' access.log | sort | uniq -c
# Output:
#       2 /api/orders

# 4. Which IP failed to log in (401), and how many times?
grep " 401 " access.log | awk '{print $1}' | sort | uniq -c | sort -nr
# Output:
#       3 198.51.100.7

# 5. Requests per hour
awk '{print substr($4, 2, 14)}' access.log | sort | uniq -c
# Output:
#       5 10/Jan/2025:09
#       5 10/Jan/2025:10

# 6. Failed SSH logins by source IP (Debian/Ubuntu auth log)
grep "Failed password" /var/log/auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -nr | head
# In "... Failed password for invalid user admin from 198.51.100.7 port 22 ssh2",
# the IP is the 4th field from the end, so $(NF-3).

# 7. Find the config files that mention an old hostname and replace it safely
grep -rl "old-server" /etc/myapp                  # 1) see which files are affected
grep -rl "old-server" /etc/myapp | xargs sed -i.bak 's/old-server/new-server/g'   # 2) replace, with backups

# 8. Delete log files older than 14 days, preview first
find /var/log/myapp -type f -name "*.log" -mtime +14 -print        # 1) preview
find /var/log/myapp -type f -name "*.log" -mtime +14 -delete       # 2) run for real

# 9. Total size of all .log files older than 7 days
find /var/log -type f -name "*.log" -mtime +7 -printf "%s\n" 2>/dev/null | awk '{s+=$1} END {printf "%.1f MB\n", s/1024/1024}'

# 10. Show only active (non-comment, non-blank) settings in a config file
grep -Ev '^\s*(#|$)' /etc/ssh/sshd_config
```

---

## Real-World Use Case

### Scenario: "The API had an outage at 10 AM. What happened?" (Log forensics with the four tools)

At 10 AM, monitoring alerts fire: **error rate up, a few customers can't log in, disk almost full on the web server**. You SSH in and use these commands to find answers fast.

**Step 1: `find`, which log files were touched recently, and what is eating the disk?**

```bash
df -h /var                                              # Is the disk really nearly full?

find /var/log -type f -mmin -120 -ls                    # Which logs changed in the last 2 hours?

find /var -xdev -type f -size +200M -printf "%s %p\n" 2>/dev/null | sort -nr | head -n 5
# The 5 largest files on this filesystem. (Finds, say, a 6 GB app.log that grew out of control.)
```

**Step 2: `grep`, narrow to the incident window and the errors**

```bash
grep "10/Jan/2025:10:" access.log | grep -c " 500 "     # How many 500 errors during the 10 AM hour?

grep -E "ERROR|FATAL" /var/log/myapp/app.log | tail -n 20   # Latest application errors

grep -C 3 "OutOfMemoryError" /var/log/myapp/app.log         # Context around a specific failure
```

**Step 3: `awk`, quantify who and what**

```bash
# Which endpoints failed?
awk '$9 >= 500 {print $7}' access.log | sort | uniq -c | sort -nr
#       2 /api/orders          <-- the orders endpoint is the problem

# Who is hammering the login endpoint and getting 401?
awk '$7 == "/api/login" && $9 == 401 {print $1}' access.log | sort | uniq -c | sort -nr
#       3 198.51.100.7         <-- repeated failures from one IP: a possible brute-force attempt

# Error rate for the whole log
awk '{total++} $9 >= 500 {err++} END {printf "5xx rate: %.1f%%\n", err/total*100}' access.log
# 5xx rate: 20.0%
```

**Step 4: `sed`, apply the fix and clean up safely**

```bash
# The developers say the DB connection pool setting was wrong. Fix it across all environments' configs:
grep -rl "pool_size = 5" /etc/myapp                                      # Which files?
sudo sed -i.bak 's/pool_size = 5/pool_size = 20/' /etc/myapp/*.conf      # Update with backups
diff /etc/myapp/app.conf.bak /etc/myapp/app.conf                         # Verify exactly what changed
sudo systemctl restart myapp

# Reclaim disk space: delete old rotated logs (preview first!)
find /var/log/myapp -type f -name "*.gz" -mtime +14 -print
find /var/log/myapp -type f -name "*.gz" -mtime +14 -delete

# Block the abusive IP (with the firewall from Topic 1) based on what awk revealed
sudo ufw deny from 198.51.100.7
```

**Result:** In about ten minutes, with no special tools, you determined that (1) the `/api/orders` endpoint was returning 500 errors due to a misconfigured connection pool, (2) one IP was attacking the login endpoint, and (3) an oversized log file was filling the disk. Each conclusion came from a short, repeatable command that you can paste into the incident report.

**Why this matters in production:**

- **Speed:** incident time is measured in lost revenue. One-liners answer questions in seconds that would take hours by eye.
- **Evidence:** every conclusion is backed by a command and its output that colleagues can reproduce.
- **Automation:** the same pipelines drop straight into cron jobs, monitoring scripts, and CI pipelines (for example, "fail the deployment if the 5xx rate exceeds 1%").
- **Safety at scale:** `grep -rl ... | xargs sed -i.bak ...` updates hundreds of config files consistently, with a rollback copy of every one.

> **Analogy:** The outage is a crime scene. `find` locates the evidence boxes. `grep` pulls out the pages that mention the suspect. `awk` tallies the statistics (who, how often, how much). `sed` repairs the damage and tidies up, and you leave a backup copy of everything you changed.

---

## Interview Questions & Answers

### Q1. What is the difference between `grep`, `egrep` (`grep -E`), and `fgrep` (`grep -F`)? How do you search recursively and show surrounding context?

**Answer:**
Plain `grep` uses **basic regular expressions**, where characters like `+`, `?`, `|`, `(`, `)` must be escaped to be special. `grep -E` (formerly `egrep`) uses **extended regular expressions**, so `ERROR|WARN`, `[0-9]+`, and groups work naturally. `grep -F` (formerly `fgrep`) treats the pattern as a **fixed literal string** with no regex, which is faster and safer when searching for text containing special characters such as dots. To search recursively: `grep -rn "pattern" /path` (add `--include="*.log"` to limit file types). For context: `-A N` (after), `-B N` (before), `-C N` (both sides), for example `grep -C 3 "Exception" app.log`. Other useful flags are `-i` (ignore case), `-v` (invert), `-c` (count), `-l` (list file names), and `-o` (print only the matched part).

---

### Q2. How do you find files modified in the last 24 hours that are larger than 100 MB, and what is the difference between `-exec ... \;`, `-exec ... +`, and piping to `xargs`?

**Answer:**
`find /var -type f -mtime -1 -size +100M`. (`-mtime -1` means within the last day; `-size +100M` means larger than 100 megabytes.) To take action on the results: `-exec cmd {} \;` runs the command **once per file**, which is simple but slow for many files. `-exec cmd {} +` **batches** many files into each command invocation, which is much faster. Piping to `xargs` does a similar batching; for file names with spaces or special characters the safe form is `find ... -print0 | xargs -0 cmd`. For destructive actions I first run the `find` with `-print` or `-ls` to preview the matches, and then add `-delete` or `-exec rm`.

---

### Q3. How do you replace a string in a file using `sed`, and how do you do it safely across multiple files?

**Answer:**
`sed 's/old/new/g' file` prints the file with every `old` replaced by `new` on each line (without `g`, only the first match per line is replaced). The original is unchanged until you add `-i` (in-place). The safe practice is: (1) preview the result without `-i`; (2) use `-i.bak` so a backup copy is created automatically, for example `sed -i.bak 's/old/new/g' file`; (3) review the changes with `diff file.bak file`; and for multiple files, (4) limit the edit to files that actually contain the text with `grep -rl "old" /dir | xargs sed -i.bak 's/old/new/g'`. If the strings contain slashes, I switch delimiters, for example `s|/usr/local|/opt|g`. On macOS (BSD sed), the syntax is `sed -i '' ...`.

---

### Q4. How would you use `awk` to print the second column of a file, sum a numeric column, and count how many times each value appears?

**Answer:**
Print the second column: `awk '{print $2}' file`. If the separator is not whitespace, use `-F`, for example `awk -F, '{print $2}' file.csv`. Sum a column: `awk '{sum += $3} END {print sum}' file`, where the `END` block runs after all lines are read. Count occurrences of each value in column 1: `awk '{count[$1]++} END {for (k in count) print count[k], k}' file | sort -nr`, using an associative array indexed by the value. Useful built-in variables are `NR` (line number), `NF` (number of fields), `$NF` (last field), and `$0` (whole line), and patterns such as `awk '$9 >= 500'` filter lines by condition.

---

### Q5. Given a web server access log, how would you list the top 5 IP addresses by request count? Explain each stage of the pipeline.

**Answer:**
`awk '{print $1}' access.log | sort | uniq -c | sort -nr | head -n 5`

1. `awk '{print $1}'` extracts the first field (the client IP) from every line.
2. `sort` groups identical IPs together, which is required because `uniq` only collapses **adjacent** duplicates.
3. `uniq -c` collapses duplicates and prefixes each unique IP with its **count**.
4. `sort -nr` sorts those counts **numerically** (`-n`) in **reverse** order (`-r`), so the highest comes first.
5. `head -n 5` keeps only the top five lines.

The same pattern works for status codes (`$9`), URLs (`$7`), or any column, and can be filtered first with `grep` or an `awk` condition (for example, only 5xx responses).

---

## Quick Recap Cheat Sheet

| Task | Command |
|------|---------|
| Search text in a file | `grep "text" file` |
| Ignore case / count / line numbers | `grep -i`, `grep -c`, `grep -n` |
| Invert (exclude) matches | `grep -v "text" file` |
| Hide comments and blank lines | `grep -Ev '^\s*(#\|$)' file` |
| Several patterns (or) | `grep -E "ERROR\|WARN" file` |
| Search recursively | `grep -rn "text" dir --include="*.py"` |
| Context around match | `grep -C 3 "text" file` |
| Only the matching part | `grep -o` or `grep -Eo 'regex' file` |
| Find by name | `find /path -name "*.log"` |
| Find by type / size / age | `find . -type f -size +100M -mtime +7` |
| Find recently changed (minutes) | `find . -mmin -30` |
| Run a command on matches | `find . -name "*.tmp" -exec rm {} +` |
| Safe delete | preview with `-print`, then `-delete` |
| Replace text (preview) | `sed 's/old/new/g' file` |
| Replace in place with backup | `sed -i.bak 's/old/new/g' file` |
| Delete matching lines | `sed '/pattern/d' file` |
| Print line range | `sed -n '5,10p' file` |
| Print a column | `awk '{print $3}' file` |
| Custom delimiter | `awk -F: '{print $1}' /etc/passwd` |
| Filter by condition | `awk '$9 >= 500' access.log` |
| Sum a column | `awk '{s+=$10} END {print s}' file` |
| Count by key | `awk '{c[$1]++} END {for(k in c) print c[k],k}' file` |
| Top-N pattern | `... \| sort \| uniq -c \| sort -nr \| head` |

**Remember:**
- `grep` = **search lines**, `find` = **locate files**, `sed` = **edit text**, `awk` = **columns and math**.
- Use `-E` with `grep` and `sed` for readable regular expressions.
- **Preview first** (`-print`, no `-i`), then act. Keep backups with `sed -i.bak`.
- `sort` before `uniq`, always.
