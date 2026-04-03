# Unit 03: The Art of the Command Line

## Overview

Being fast in the terminal is a superpower. Most Linux developers spend hours every day at the command line — those who know it deeply get things done in seconds that take others minutes. This unit uses [The Art of the Command Line](https://github.com/jlevy/the-art-of-command-line) as its primary reference and turns its recommendations into structured practice. You will go from knowing basic commands to working the shell like an expert: navigating history instantly, editing text in-place, slicing data with one-liners, and diagnosing a live system without opening a browser.

## Prerequisites

- Unit 02 (The Linux Environment) — basic terminal navigation, man pages, file permissions

## Learning Objectives

- Navigate and edit the command line at speed using keyboard shortcuts
- Search and replay command history efficiently
- Write bash scripts with proper error handling
- Process text files and structured data from the command line
- Monitor and debug a running Linux system using CLI tools
- Build useful one-liners combining `grep`, `awk`, `sed`, `sort`, `uniq`, and `xargs`
- Use `ssh`, `tmux`, and job control for remote and long-running work

## Required Reading

**The Art of the Command Line** by Joshua Levy
https://github.com/jlevy/the-art-of-command-line

Read the whole document top to bottom before working through this unit. It is not long. Then return to each section here for deeper practice.

## Concepts

### Keyboard Shortcuts: Move at the Speed of Thought

The single highest-leverage investment in terminal productivity. These are readline shortcuts — they work in bash, gdb, python REPL, and most other interactive CLI tools.

**Navigation:**
```
Ctrl-A       move to beginning of line
Ctrl-E       move to end of line
Alt-B        move back one word
Alt-F        move forward one word
Ctrl-XX      toggle between current position and line start
```

**Editing:**
```
Ctrl-W       delete word before cursor
Alt-D        delete word after cursor
Ctrl-U       delete from cursor to beginning of line
Ctrl-K       delete from cursor to end of line
Ctrl-Y       paste (yank) last deleted text
Alt-T        swap last two words
Ctrl-T       swap last two characters
Ctrl-_       undo
```

**History:**
```
Ctrl-R       reverse incremental history search (keep pressing for older matches)
Ctrl-G       cancel history search
!!           repeat last command
!$           last argument of previous command
!*           all arguments of previous command
!ls          last command starting with 'ls'
Alt-.        insert last argument of previous command (repeatable)
```

**Process:**
```
Ctrl-C       interrupt (SIGINT)
Ctrl-Z       suspend (SIGTSTP) — resume with fg or bg
Ctrl-D       end of input (EOF)
Ctrl-L       clear screen
```

Practice these until they are muscle memory. Use `set -o vi` in `.bashrc` if you prefer vi-style editing.

### History Mastery

```bash
# Search history interactively
Ctrl-R          # type part of any previous command

# Print history with line numbers
history

# Re-run command #342
!342

# Re-run last command that started with 'git'
!git

# Print but don't run
!git:p

# Replace string in last command and run
^foo^bar        # replace first 'foo' with 'bar' in last command

# Never log a command (space prefix)
 secret_command  # leading space: not saved to history (if HISTCONTROL=ignorespace)
```

Configure history in `~/.bashrc`:
```bash
export HISTSIZE=100000
export HISTFILESIZE=100000
export HISTCONTROL=ignoredups:ignorespace
export HISTTIMEFORMAT="%Y-%m-%d %H:%M:%S "
shopt -s histappend        # append rather than overwrite history file
shopt -s cmdhist           # save multi-line commands as one history entry
```

### Shell Configuration

`~/.bashrc` is sourced for every interactive non-login shell. `~/.bash_profile` is sourced for login shells. Put most things in `.bashrc` and source it from `.bash_profile`.

Useful `.bashrc` additions:
```bash
# Better prompt showing git branch
parse_git_branch() {
    git branch 2>/dev/null | grep '^*' | sed 's/* //'
}
PS1='\u@\h:\w$(b=$(parse_git_branch); [ -n "$b" ] && echo " ($b)")\$ '

# Useful aliases
alias ll='ls -la --color=auto'
alias la='ls -A'
alias ..='cd ..'
alias ...='cd ../..'
alias grep='grep --color=auto'
alias df='df -h'
alias du='du -h'

# Safety nets
alias rm='rm -i'
alias cp='cp -i'
alias mv='mv -i'

# Quickly edit and reload config
alias reload='source ~/.bashrc'
alias bashrc='$EDITOR ~/.bashrc && reload'
```

### Job Control and Multiplexing

**Job control** lets you run multiple commands in one terminal:
```bash
# Run a command in the background
long_running_command &

# List jobs
jobs

# Bring a background job to foreground
fg %1

# Send a foreground job to background
Ctrl-Z then bg

# Disown a job (keep running after shell exits)
disown %1

# nohup: immune to hangup signal
nohup long_running_command > output.log 2>&1 &
```

**tmux** is indispensable for remote work and long sessions:
```bash
tmux                   # start new session
tmux new -s work       # named session
tmux attach -t work    # reattach to session
tmux ls                # list sessions
```

Key tmux bindings (prefix = `Ctrl-B`):
```
Ctrl-B c    new window
Ctrl-B n    next window
Ctrl-B p    previous window
Ctrl-B %    split pane vertically
Ctrl-B "    split pane horizontally
Ctrl-B z    zoom/unzoom current pane
Ctrl-B d    detach session (leaves it running)
Ctrl-B [    enter scroll/copy mode
```

### Bash Scripting: The Right Way

Scripts must be robust. Start every script with:
```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'
```

- `set -e`: exit immediately if any command fails
- `set -u`: error on undefined variables (prevents typo bugs)
- `set -o pipefail`: a pipe fails if any stage fails (not just the last)

**Variables and quoting:**
```bash
name="Alice"
echo "$name"      # always quote variables — prevents word splitting
echo "${name}s"   # disambiguate variable name with braces

# Arrays
files=("a.txt" "b.txt" "c.txt")
echo "${files[0]}"        # first element
echo "${files[@]}"        # all elements
echo "${#files[@]}"       # count

# Default values
: "${CONFIG_FILE:=/etc/myapp/config}"  # set if unset
echo "${PORT:-8080}"                    # use 8080 if PORT unset
```

**Control flow:**
```bash
# Conditionals
if [[ -f "$file" ]]; then
    echo "file exists"
elif [[ -d "$file" ]]; then
    echo "is a directory"
fi

# String tests
[[ -z "$str" ]]    # empty string
[[ -n "$str" ]]    # non-empty string
[[ "$a" == "$b" ]] # string equality
[[ "$a" =~ regex ]] # regex match

# File tests
[[ -f "$f" ]]  # regular file
[[ -d "$d" ]]  # directory
[[ -r "$f" ]]  # readable
[[ -x "$f" ]]  # executable

# For loops
for file in *.c; do
    echo "Processing $file"
done

for i in $(seq 1 10); do echo $i; done
```

**Functions:**
```bash
log() {
    echo "[$(date '+%H:%M:%S')] $*" >&2
}

die() {
    log "ERROR: $*"
    exit 1
}

require_command() {
    command -v "$1" >/dev/null 2>&1 || die "$1 is required but not installed"
}

require_command gcc
```

**Temporary files (safe):**
```bash
tmpfile=$(mktemp)
trap "rm -f $tmpfile" EXIT   # clean up on exit, even on error
```

### Text Processing: The Core Tools

These tools compose into powerful data pipelines:

**grep — search**
```bash
grep "pattern" file              # basic search
grep -i "pattern" file           # case-insensitive
grep -r "pattern" dir/           # recursive
grep -l "pattern" *.c            # print only filenames
grep -n "pattern" file           # print line numbers
grep -v "pattern" file           # invert (lines NOT matching)
grep -c "pattern" file           # count matching lines
grep -A3 -B3 "pattern" file      # 3 lines of context
grep -E "(foo|bar)" file         # extended regex
grep -o "pattern" file           # print only the matching part
```

**sed — stream editor**
```bash
sed 's/foo/bar/' file             # replace first occurrence per line
sed 's/foo/bar/g' file            # replace all occurrences
sed 's/foo/bar/gi' file           # case-insensitive replace all
sed -n '10,20p' file              # print lines 10-20
sed -i 's/foo/bar/g' file         # edit in place (modifies file)
sed -i.bak 's/foo/bar/g' file     # edit in place, keep backup
sed '/^#/d' file                  # delete comment lines
sed -n '/START/,/END/p' file      # print between patterns
```

**awk — structured text processing**
```bash
awk '{print $1}' file             # print first field (space-delimited)
awk '{print $NF}' file            # print last field
awk -F: '{print $1}' /etc/passwd  # colon-delimited, print field 1
awk '{sum += $3} END {print sum}' file  # sum column 3
awk 'NR==5' file                  # print line 5
awk 'NR>=5 && NR<=10' file        # print lines 5-10
awk '/pattern/ {print $2}' file   # print field 2 from matching lines
awk '{print FILENAME, NR, $0}' *.log  # add filename and line number
```

**sort, uniq, wc**
```bash
sort file                         # alphabetical sort
sort -n file                      # numeric sort
sort -rn file                     # reverse numeric
sort -t: -k3 -n /etc/passwd       # sort by field 3, colon-delimited
sort -u file                      # sort and remove duplicates

uniq file                         # remove consecutive duplicates (sort first)
uniq -c file                      # count occurrences
uniq -d file                      # print only duplicates

wc -l file                        # count lines
wc -w file                        # count words
wc -c file                        # count bytes
```

**cut and paste**
```bash
cut -d: -f1 /etc/passwd           # field 1, colon-delimited
cut -c1-10 file                   # characters 1-10
paste file1 file2                 # merge files side by side
```

**tr**
```bash
tr 'a-z' 'A-Z' < file            # uppercase
tr -d '\r' < dos_file             # remove carriage returns
tr -s ' ' < file                  # squeeze repeated spaces
```

**xargs — build commands from input**
```bash
find . -name "*.log" | xargs rm
find . -name "*.c"   | xargs grep "TODO"
cat urls.txt | xargs -P4 curl -O  # parallel download (-P4 = 4 at a time)
echo "a b c" | xargs -n1 echo     # one argument per line
```

### Finding Things

```bash
# Find by name
find . -name "*.c"
find . -iname "*.c"              # case-insensitive
find /etc -name "*.conf" -type f

# Find by property
find . -newer reference_file     # modified more recently than file
find . -mtime -7                 # modified in last 7 days
find . -size +10M                # larger than 10MB
find . -perm /u+x                # executable by owner

# Find and act
find . -name "*.o" -delete
find . -name "*.c" -exec wc -l {} \;
find . -name "*.c" -exec grep -l "TODO" {} \;

# locate (faster but uses a database — update with updatedb)
locate myfile.txt

# which / type
which gcc
type -a python
```

### Data Processing One-Liners

```bash
# Count occurrences of each word in a file
tr -s ' ' '\n' < file | sort | uniq -c | sort -rn | head

# Find the 10 largest files
du -sh * | sort -rh | head

# Show IP addresses making the most requests in an access log
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head

# Extract all email addresses from a file
grep -oE '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}' file

# Show set difference: lines in file1 not in file2
comm -23 <(sort file1) <(sort file2)

# Show set intersection: lines in both files
comm -12 <(sort file1) <(sort file2)

# Sum numbers in a column
awk '{sum += $1} END {print sum}' numbers.txt

# Convert CSV column to JSON array
cut -d, -f2 data.csv | tail -n +2 | jq -R . | jq -s .

# Watch a file for changes and run a command
while inotifywait -e modify file; do make; done

# Repeat a command every N seconds
watch -n 2 'ss -tnp'
```

### System Monitoring and Debugging

```bash
# CPU and memory
top              # classic process monitor
htop             # improved top (apt install htop)
vmstat 1         # virtual memory stats, updated every second
mpstat -P ALL 1  # per-CPU stats

# Disk
iostat -x 1      # disk I/O stats per second
df -h            # disk space
du -sh /var/*    # space used by each item under /var
iotop            # I/O per process (apt install iotop)
lsblk            # block devices

# Network
ss -tnp          # TCP connections with process info
ss -tlnp         # listening TCP sockets
netstat -rn      # routing table
ip addr          # network interfaces and addresses
ip route         # routing table
iftop -i eth0    # bandwidth per connection
nload            # total bandwidth graph

# Processes
pstree           # process hierarchy as tree
lsof -p PID      # files open by a process
lsof +D /path    # processes using files in a directory
fuser 8080/tcp   # what process is using port 8080

# General
dmesg | tail     # kernel messages
journalctl -xe   # systemd log (errors)
last             # login history
w                # who is logged in and what they're doing
```

### SSH Power Usage

```bash
# Passwordless login
ssh-keygen -t ed25519
ssh-copy-id user@host

# Port forwarding
ssh -L 8080:localhost:80 user@host   # local:8080 → remote:80
ssh -R 9090:localhost:9090 user@host # remote:9090 → local:9090

# Execute a command remotely
ssh user@host 'ls -la /var/log'

# Copy files
scp file user@host:/path/
scp -r dir user@host:/path/
rsync -avz ./src/ user@host:/dest/   # incremental, compressed

# Keep connections alive in ~/.ssh/config
# Host *
#   ServerAliveInterval 60
#   ControlMaster auto
#   ControlPath ~/.ssh/sockets/%r@%h:%p
#   ControlPersist 600
```

### Redirection and Process Substitution

```bash
# Standard redirections
cmd > out.txt           # stdout to file
cmd >> out.txt          # append stdout
cmd 2> err.txt          # stderr to file
cmd > out.txt 2>&1      # stdout and stderr to file
cmd 2>/dev/null         # suppress stderr
cmd > /dev/null 2>&1    # suppress all output

# Process substitution (creates a virtual file from a command's output)
diff <(sort file1) <(sort file2)     # diff sorted versions
grep pattern <(cat *.log)            # grep across multiple files as one
command > >(tee output.log)          # tee stdout to file AND continue pipeline

# Here documents
cat <<'EOF' > script.sh
#!/bin/bash
echo "generated script"
EOF

# Here strings
wc -w <<< "count these words"
grep "foo" <<< "$variable"
```

## Exercises

### Exercise 1: Keyboard Shortcut Drill

Open a terminal and practice each of the following — do not use arrow keys or the mouse for any of it:

1. Type a long command (e.g. `find /usr -name "*.h" -type f -mtime -30 | head -20`), then use `Ctrl-A` to go to the beginning, `Alt-F` to move forward three words, `Ctrl-K` to delete to the end, then `Ctrl-Y` to paste it back.
2. Run any command, then use `!!` to repeat it and `!$` to use its last argument in a new command.
3. Search history with `Ctrl-R` to find a previous command. Cancel with `Ctrl-G`.
4. Use `Alt-.` three times in a row to cycle through the last arguments of previous commands.

Do each operation 10 times until it requires no thought.

### Exercise 2: Shell Configuration

Build a `.bashrc` that includes:
1. History settings: 100k entries, timestamps, no duplicates, append mode
2. At least 5 useful aliases for your workflow
3. A prompt that shows: `user@host:directory (git-branch)$`
4. A `mkcd` function: creates a directory and immediately `cd`s into it
5. A `up N` function: goes up N levels (`up 3` = `cd ../../..`)

Source it and verify each piece works.

### Exercise 3: Text Processing Pipeline

Given this input (create a file `servers.txt`):
```
web01  192.168.1.10  running  8 cores  16GB
web02  192.168.1.11  stopped  4 cores  8GB
db01   192.168.1.20  running  16 cores 64GB
db02   192.168.1.21  running  16 cores 64GB
cache  192.168.1.30  running  2 cores  4GB
```

Write single-pipeline commands that:
1. Print only running servers
2. Count how many servers are running vs stopped
3. Print the hostname and IP of all servers with more than 4 cores
4. Sum the total RAM across all servers
5. Sort servers by number of cores, descending

Each answer must be a single pipeline (no loops, no temp files).

### Exercise 4: Log Analysis

Download or create a fake web server access log (common log format):
```bash
# Generate a sample log
for i in $(seq 1 1000); do
    ip="192.168.$(( RANDOM % 5 )).$(( RANDOM % 254 + 1 ))"
    code=$(( RANDOM % 3 == 0 ? 404 : 200 ))
    size=$(( RANDOM % 10000 + 100 ))
    echo "$ip - - [01/Jan/2025:12:00:00 +0000] \"GET /path/$i HTTP/1.1\" $code $size"
done > access.log
```

Write single-pipeline one-liners to answer:
1. How many unique IP addresses are in the log?
2. Which IP made the most requests?
3. What percentage of requests returned 404?
4. What is the total bytes served (sum of the last column)?
5. List the top 5 most-requested paths.

### Exercise 5: System Snapshot Script

Write a bash script `snapshot.sh` that:
1. Uses `set -euo pipefail`
2. Accepts an optional output directory (default: `/tmp/snapshot-$(date +%Y%m%d)`)
3. Creates the directory if it doesn't exist
4. Saves the following to separate files in that directory:
   - `processes.txt`: full process list (`ps aux`)
   - `network.txt`: listening ports and connections (`ss -tnp`, `ss -tlnp`)
   - `disk.txt`: disk usage (`df -h`, `du -sh /var/* /home/* /tmp/*`)
   - `memory.txt`: memory info (`free -h`, `vmstat 1 3`)
   - `open_files.txt`: top 20 processes by open file count
5. Prints a summary: how many files were created and total size
6. Exits cleanly with status 0 on success, non-zero on any failure

### Exercise 6: Find and Transform

Without using any loop (`for`, `while`, `until`), write commands using `find` + `xargs` that:

1. Find all `.c` files in `/usr/src` (or any C source tree you have) and count the total lines of code
2. Find all files larger than 100MB and print their paths with human-readable sizes
3. Find all broken symbolic links under `/etc` and print them
4. Find all files modified in the last 24 hours and archive them into `recent.tar.gz`

### Exercise 7: tmux Session Setup

Create a tmux session called `dev` with:
- Window 1 named `editor`: split horizontally, top pane running `vim`, bottom pane in project directory
- Window 2 named `build`: a single pane ready to run make
- Window 3 named `monitor`: split into 4 panes running `htop`, `watch -n1 free -h`, `watch -n1 df -h`, and `journalctl -f`

Write a shell script `start_dev.sh` that creates this entire layout with one command.

## What Comes Next

Unit 05 begins C programming. Everything you have learned about the shell makes the development workflow faster: compiling, running, piping output through filters, and grepping source code. The text processing tools you practiced here are how you will analyze log files, inspect binary formats, and process build output for the rest of this curriculum.
