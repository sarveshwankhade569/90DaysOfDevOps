# Shell Scripting Cheat Sheet

A practical reference for Bash — the stuff you actually reach for while writing DevOps scripts.

## Quick Reference

| Topic | Syntax | Example |
|---|---|---|
| Shebang | `#!/bin/bash` | `#!/bin/bash` |
| Variable | `VAR="value"` | `NAME="DevOps"` |
| Argument | `$1`, `$2` | `./script.sh nginx` |
| All arguments | `$@` | `for arg in "$@"; do` |
| Argument count | `$#` | `if [ $# -eq 0 ]; then` |
| Exit status | `$?` | `echo $?` |
| If | `if [ condition ]; then` | `if [ -f file ]; then` |
| For loop | `for i in list; do` | `for i in 1 2 3; do` |
| While | `while [ condition ]; do` | `while [ $count -lt 5 ]; do` |
| Function | `name() { ... }` | `check_disk() { df -h; }` |
| Grep | `grep pattern file` | `grep -i "error" app.log` |
| Awk | `awk '{print $1}' file` | `awk -F: '{print $1}' /etc/passwd` |
| Sed | `sed 's/old/new/g' file` | `sed -i 's/http/https/g' config` |
| Find | `find path condition` | `find /var/log -name "*.log"` |

## Bash basics

Every script starts with a shebang line:
```bash
#!/bin/bash
```
This tells Linux to run the script using Bash — it should always be the first line.

To create and run a script:
```bash
touch script.sh
chmod +x script.sh
./script.sh
```
Or skip the permission step entirely and run it with `bash script.sh` — the difference is `./script.sh` needs execute permission, `bash script.sh` doesn't.

Comments start with `#` and are ignored by Bash — use them to explain what your script is doing.

## Variables

```bash
NAME="Sarvesh"
echo "$NAME"     # → Sarvesh
```

No spaces around the `=` — `NAME = "Sarvesh"` will actually break.

Double quotes let variables expand, single quotes don't:
```bash
echo "$NAME"   # → Sarvesh
echo '$NAME'   # → $NAME (literally)
```
Rule of thumb: always quote your variables (`"$NAME"`), it avoids a lot of subtle bugs.

## User input

```bash
read -p "Enter your name: " NAME
echo "Hello $NAME"
```

## Command-line arguments

For `./script.sh nginx production`:

| Variable | Value |
|---|---|
| `$0` | `./script.sh` (script name) |
| `$1` | `nginx` (first argument) |
| `$2` | `production` (second argument) |
| `$#` | `2` (number of arguments) |
| `$@` | `nginx production` (all arguments) |
| `$?` | exit status of the last command (`0` = success) |

## Conditions

**Strings:**
```bash
if [ "$NAME" = "Sarvesh" ]; then echo "matched"; fi
if [ "$NAME" != "admin" ]; then echo "not admin"; fi
if [ -z "$NAME" ]; then echo "empty"; fi     # empty
if [ -n "$NAME" ]; then echo "not empty"; fi # not empty
```

**Numbers:** `-eq` equal · `-ne` not equal · `-lt` less than · `-gt` greater than · `-le` less/equal · `-ge` greater/equal
```bash
if [ "$CPU" -gt 80 ]; then echo "High CPU"; fi
```

**Files** (very common in DevOps scripts):

| Test | Checks |
|---|---|
| `-f file` | regular file exists |
| `-d dir` | directory exists |
| `-e path` | path exists (any type) |
| `-r file` | readable |
| `-w file` | writable |
| `-x file` | executable |
| `-s file` | exists and isn't empty |

**If / elif / else:**
```bash
if [ "$CPU" -gt 80 ]; then
    echo "High CPU"
elif [ "$CPU" -gt 50 ]; then
    echo "Normal CPU"
else
    echo "Low CPU"
fi
```

**Logic:**
```bash
if [ "$CPU" -gt 80 ] && [ "$MEM" -gt 80 ]; then echo "both high"; fi
if [ "$CPU" -gt 80 ] || [ "$MEM" -gt 80 ]; then echo "needs attention"; fi
if ! systemctl is-active --quiet nginx; then echo "nginx is down"; fi
```

**Case statement** — cleaner than a long if/elif chain when you're matching one value against several options:
```bash
case "$OPTION" in
    start)   echo "Starting service" ;;
    stop)    echo "Stopping service" ;;
    restart) echo "Restarting service" ;;
    *)       echo "Invalid option" ;;
esac
```

## Loops

```bash
# over a list
for server in web1 web2 web3; do echo "$server"; done

# C-style
for ((i=1; i<=5; i++)); do echo "$i"; done

# while
COUNT=1
while [ "$COUNT" -le 5 ]; do
    echo "$COUNT"
    COUNT=$((COUNT + 1))
done

# until — opposite of while, runs until the condition becomes true
until [ "$COUNT" -gt 5 ]; do
    COUNT=$((COUNT + 1))
done
```

`break` stops the loop entirely, `continue` skips to the next iteration.

Looping over files or command output:
```bash
for file in /var/log/*.log; do echo "Checking $file"; done

while read -r line; do
    echo "Server: $line"
done < servers.txt
```

## Functions

```bash
greet() {
    echo "Hello $1"
}
greet "Sarvesh"   # → Hello Sarvesh
```

`echo` inside a function returns data you can capture:
```bash
get_status() { echo "Running"; }
STATUS=$(get_status)
```

`return` sends back an exit status (0–255) instead, useful for pass/fail checks:
```bash
check_file() {
    if [ -f "$1" ]; then return 0; else return 1; fi
}
if check_file "/tmp/test.txt"; then echo "exists"; fi
```

Use `local` for variables inside functions so they don't leak into the global scope:
```bash
greet() {
    local NAME="DevOps"
    echo "$NAME"
}
```

## Text processing

**grep** — search text
```bash
grep -i "error" app.log       # case-insensitive
grep -n "ERROR" app.log       # with line numbers
grep -c "ERROR" app.log       # just the count
grep -r "ERROR" /var/log/     # recursive
grep -v "INFO" app.log        # exclude matching lines
grep -E "ERROR|CRITICAL" app.log  # multiple patterns
```

**awk** — extract columns
```bash
awk '{print $1}' file.txt              # first column
awk -F: '{print $1}' /etc/passwd       # custom delimiter
awk -F: '$3 == 0 {print $1}' /etc/passwd  # filter by column value
```

**sed** — find & replace
```bash
sed 's/old/new/' file.txt        # first match per line
sed 's/old/new/g' file.txt       # all matches
sed -i 's/http/https/g' config.txt  # edit the file in place
sed '2d' file.txt                 # delete line 2
```

**cut** — extract a field
```bash
cut -d: -f1 /etc/passwd   # -d: delimiter, -f1: first field
```

**sort / uniq** — order and dedupe
```bash
sort -n numbers.txt       # numeric sort
sort -r names.txt         # reverse
uniq -c file.txt          # count duplicate lines
sort file.txt | uniq -c | sort -rn   # most common lines, ranked
```

**tr** — translate characters
```bash
echo "devops" | tr 'a-z' 'A-Z'   # → DEVOPS
```

**wc** — count things
```bash
wc -l file.txt   # lines
wc -w file.txt   # words
grep "ERROR" app.log | wc -l   # count matching lines
```

**head / tail**
```bash
head -n 5 file.txt
tail -n 50 app.log
tail -f /var/log/nginx/access.log   # follow a live log — very common while debugging
```

## Handy one-liners

```bash
find /var/log -name "*.log" -mtime +7            # logs older than 7 days
find /tmp -name "*.log" -mtime +7 -delete         # ...and delete them (be careful!)
grep -i "error" /var/log/app.log | wc -l          # count errors
systemctl is-active --quiet nginx && echo "Running" || echo "Not running"
tail -f /var/log/app.log | grep --line-buffered "ERROR"   # watch for errors live

# disk usage check with a threshold
USAGE=$(df / | awk 'NR==2 {gsub("%","",$5); print $5}')
if [ "$USAGE" -gt 80 ]; then echo "WARNING: Disk usage is ${USAGE}%"; fi

# top IPs hitting an nginx access log
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head
```

## Exit codes & error handling

Every command returns an exit code — `0` means success, anything else means failure. Check it with `echo $?`, or exit explicitly with `exit 0` / `exit 1`.

**Strict mode** — put this at the top of production scripts:
```bash
set -euo pipefail
```
- `-e` — stop immediately if any command fails
- `-u` — error out on unset variables
- `-o pipefail` — catch failures inside a pipeline, not just the last command

**Debugging** — `set -x` prints every command as it runs (turn off with `set +x`), useful when a script is misbehaving and you need to see exactly what's happening.

**trap** — run cleanup code automatically when a script exits:
```bash
cleanup() {
    rm -f /tmp/tempfile
    echo "Cleanup complete"
}
trap cleanup EXIT
```

## A typical production-style script

```bash
#!/bin/bash
set -euo pipefail

LOG_FILE="/var/log/app.log"

if [ ! -f "$LOG_FILE" ]; then
    echo "ERROR: Log file does not exist"
    exit 1
fi

ERROR_COUNT=$(grep -ci "error" "$LOG_FILE")
echo "Log file: $LOG_FILE"
echo "Error count: $ERROR_COUNT"

if [ "$ERROR_COUNT" -gt 100 ]; then
    echo "WARNING: High number of errors"
fi

exit 0
```
Nothing fancy — just variables, a file check, `grep`, and strict mode working together.

## Commands worth knowing cold

If you're troubleshooting a Linux/AWS server, these come up constantly:

```
ps aux
top
systemctl status nginx
journalctl -u nginx -n 50
df -h
free -h
find /var/log -name "*.log"
grep -i "error" app.log
tail -f app.log
ss -tulpn
curl -I http://localhost
ip addr
dig google.com
```

A natural troubleshooting flow: **service status → journalctl → ps/top → disk/memory → open ports (ss) → curl the endpoint → grep/tail the logs.** Worth internalizing as a habit rather than memorizing commands in isolation.
