# Bash Functions & Scripting Practice Notes

Five hands-on tasks covering functions, local vs global variables, strict error handling (`set -euo pipefail`), and a real system-info reporting script.

---

## Task 1 — Basic Functions

```bash
touch functions.sh      # create file
chmod +x functions.sh   # give execute permission
vim functions.sh        # edit file
```

```bash
#!/bin/bash

greet() {
    local name="$1"
    echo "Hello, $name!"
}

add() {
    local num1="$1"
    local num2="$2"
    local sum=$((num1 + num2))
    echo "Sum: $sum"
}

greet "Sarvesh"
add 10 20
```

**Run:**
```bash
bash functions.sh
```

**Concept:** Functions are defined with `name() { ... }` and called just like commands. Arguments passed to a function are read the same way as script arguments — `$1`, `$2`, etc. — but scoped to that function call.

---

## Task 2 — Disk and Memory Functions

```bash
touch disk_check.sh      # create file
chmod 744 disk_check.sh  # give permission
vim disk_check.sh        # edit file
```

```bash
#!/bin/bash

check_disk() {
    echo "===== Disk Usage ====="
    df -h /
}

check_memory() {
    echo "===== Memory Usage ====="
    free -h
}

check_service() {
    systemctl is-active --quiet nginx
    return $?
}

main() {
    check_disk
    echo
    check_memory
}

if check_service; then
    echo "Nginx is running"
else
    echo "Nginx is not running"
fi

main
```

**Concept:** `check_service` doesn't print anything — it just runs `systemctl is-active --quiet nginx` and forwards its exit code (`0` = running, non-zero = not running) with `return $?`. That exit code is what the `if check_service; then` line actually tests, since `if` checks a command's exit status, not its output.

---

## Task 3 — `set -euo pipefail`

```bash
touch strict_demo.sh      # create file
chmod +x strict_demo.sh   # give execute permission
vim strict_demo.sh        # edit file
```

```bash
#!/bin/bash
set -e
echo "Before"
ls /directory-that-does-not-exist
echo "After"
```

**Run:**
```bash
bash strict_demo.sh
```

Because of `set -e`, the script stops as soon as the `ls` command fails — `"After"` never prints.

### The three strict-mode flags

| Flag | Effect |
|------|--------|
| `-e` | Stop the script immediately if any command fails (non-zero exit code) |
| `-u` | Treat any unset/undefined variable as an error and stop |
| `-o pipefail` | Catch failures **inside** a pipeline — normally only the last command's exit code counts, this makes any failing stage fail the whole pipeline |

Combined:
```bash
set -euo pipefail
```
This is the standard "strict mode" header for production-grade Bash scripts — it stops silent failures from being ignored.

---

## Task 4 — Local vs Global Variables

```bash
touch local_demo.sh       # create file
chmod +x local_demo.sh    # give execute permission
vim local_demo.sh         # edit file
```

```bash
#!/bin/bash

show_local() {
    local message="I am local"
    echo "Inside function: $message"
}

show_global() {
    message="I am global"
    echo "Inside function: $message"
}

show_local
echo "Outside local function: ${message:-message is not defined}"

show_global
echo "Outside global function: $message"
```

**Concept:**
- `local message="..."` inside `show_local` only exists **within that function** — once the function returns, `$message` is undefined outside it. `${message:-message is not defined}` prints the fallback text since the variable isn't set.
- `show_global` sets `message` **without** `local`, which makes it a global variable — so it's still accessible after the function returns, and the last `echo` prints `"I am global"`.

**Takeaway:** always use `local` inside functions unless you deliberately want to leak a variable into the global scope.

---

## Task 5 — System Information Reporter

```bash
touch system_info.sh      # create file
chmod +x system_info.sh   # give permission
vim system_info.sh        # edit file
```

```bash
#!/bin/bash
set -euo pipefail

print_header() {
    echo
    echo "======================================"
    echo "$1"
    echo "======================================"
}

system_info() {
    print_header "HOSTNAME & OS"
    echo "Hostname: $(hostname)"
    if [ -f /etc/os-release ]; then
        . /etc/os-release
        echo "OS: $PRETTY_NAME"
    else
        echo "OS information unavailable"
    fi
}

show_uptime() {
    print_header "UPTIME"
    uptime
}

disk_usage() {
    print_header "DISK USAGE"
    echo "Top 5 directories/files:"
    du -xah / 2>/dev/null | sort -rh | head -5
}

memory_usage() {
    print_header "MEMORY USAGE"
    free -h
}

top_cpu_processes() {
    print_header "TOP 5 CPU PROCESSES"
    ps aux --sort=-%cpu | head -6
}

main() {
    system_info
    show_uptime
    disk_usage
    memory_usage
    top_cpu_processes
}

main
```

**Concept:** `print_header` is a reusable helper — every section calls it with a different title (`$1`) instead of repeating the same three `echo` lines everywhere. `main` just calls each report function in order, so the whole script reads top-to-bottom like a table of contents.

### Breaking down `du -xah / 2>/dev/null | sort -rh | head -5`

```
du -xah /
│  │ │
│  │ └── human-readable sizes (e.g. 4.0K, 2.1G)
│  └──── show all files, not just directories
└─────── disk usage
```
- `-x` — stay on one filesystem (don't cross into mounted drives)
- `2>/dev/null` — discard "Permission denied" errors so they don't clutter the output
- `sort -rh` — sort **numerically**, treating human-readable sizes correctly (`h`), largest first (`r`)
- `head -5` — keep only the top 5 results

---

## Quick Command Reference

| Command | Purpose |
|---|---|
| `touch file.sh` | Create an empty script file |
| `chmod +x file.sh` | Make a script executable |
| `bash file.sh` | Run a script |
| `local var="value"` | Declare a variable scoped to the current function |
| `set -euo pipefail` | Enable strict error handling |
| `systemctl is-active --quiet <service>` | Check if a service is running (via exit code) |

---

