# Day 19 — Log Rotation, Backup & Crontab

Hands-on lab covering log rotation, tar/gzip backups, and cron scheduling — building up to a combined maintenance script.

---

## Part 1: Commands You Need

### `find` — locate files by condition
```bash
find /path -name "*.log"                  # find .log files
find /path -name "*.log" -mtime +7        # .log files older than 7 days
find /path -name "*.gz" -mtime +30        # .gz files older than 30 days
```

### `gzip` — compress a file
```bash
gzip application.log       # becomes application.log.gz
ls -lh                     # verify
```

### `tar` — create/extract archives
```bash
tar -czf backup.tar.gz /source-directory   # create compressed archive
tar -xzf backup.tar.gz                     # extract
tar -tzf backup.tar.gz                     # list contents without extracting
```
| Flag | Meaning |
|---|---|
| `-c` | create archive |
| `-z` | gzip compression |
| `-f` | specify filename |
| `-x` | extract |
| `-t` | list contents |

### `date` — timestamps
```bash
date                         # current date
date +%Y-%m-%d               # e.g. 2026-08-12
date +%Y-%m-%d_%H-%M-%S       # e.g. 2026-08-12_20-45-10
```

### `du` — check size
```bash
du -sh /var/log      # total size of a directory
du -sh /var/log/*    # size of each item inside
```

### `stat` — file metadata
```bash
stat backup.tar.gz
```

### `mkdir`
```bash
mkdir -p /tmp/day19-lab
```

### `crontab`
```bash
crontab -l   # view current cron jobs
crontab -e   # edit cron jobs
crontab -r   # remove ALL of your user's cron jobs
```
> ⚠️ Don't run `crontab -r` during practice unless you intentionally want to wipe all your cron entries.

---

## Part 2: Cron Syntax

```
* * * * * command
│ │ │ │ │
│ │ │ │ └── Day of week (0–6, Sun=0)
│ │ │ └──── Month
│ │ └────── Day of month
│ └──────── Hour
└────────── Minute
```

| Schedule | Cron expression |
|---|---|
| Every minute | `* * * * *` |
| Every 5 minutes | `*/5 * * * *` |
| Every day at 2 AM | `0 2 * * *` |
| Every day at 1 AM | `0 1 * * *` |
| Every Sunday at 3 AM | `0 3 * * 0` |

---

## Part 3: Build a Safe Lab Environment

```bash
mkdir -p ~/day-19
cd ~/day-19

mkdir -p /tmp/day19-lab/logs
mkdir -p /tmp/day19-lab/backups
mkdir -p /tmp/day19-lab/source

touch /tmp/day19-lab/logs/app.log
touch /tmp/day19-lab/logs/error.log
touch /tmp/day19-lab/logs/access.log

echo "Application started" > /tmp/day19-lab/logs/app.log
echo "ERROR database connection failed" > /tmp/day19-lab/logs/error.log
echo "GET / HTTP/1.1 200" > /tmp/day19-lab/logs/access.log

echo "Application configuration" > /tmp/day19-lab/source/app.conf
echo "Database configuration" > /tmp/day19-lab/source/database.conf
echo "DevOps backup test" > /tmp/day19-lab/source/readme.txt

# verify
find /tmp/day19-lab -type f -ls
```

---

## Part 4: Task 1 — Log Rotation Script

```bash
nano log_rotate.sh
```

```bash
#!/bin/bash
set -euo pipefail

LOG_DIR="${1:-}"

if [ -z "$LOG_DIR" ]; then
    echo "Usage: $0 <log-directory>"
    exit 1
fi

if [ ! -d "$LOG_DIR" ]; then
    echo "ERROR: Directory does not exist: $LOG_DIR"
    exit 1
fi

compressed=0
deleted=0

echo "Starting log rotation..."
echo "Log directory: $LOG_DIR"

while IFS= read -r file; do
    gzip "$file"
    ((compressed+=1))
done < <(find "$LOG_DIR" -type f -name "*.log" -mtime +7 -print)

while IFS= read -r file; do
    rm -f "$file"
    ((deleted+=1))
done < <(find "$LOG_DIR" -type f -name "*.gz" -mtime +30 -print)

echo "Files compressed: $compressed"
echo "Files deleted:   $deleted"
echo "Log rotation completed."
```

```bash
chmod +x log_rotate.sh
./log_rotate.sh /tmp/day19-lab/logs
```

Since the test files are new, expect:
```
Files compressed: 0
Files deleted:   0
```

**Why `-mtime +7`?** It matches `.log` files whose last-modified time is more than ~7 days old — so the script never touches today's active logs.

**Why `LOG_DIR="${1:-}"`?** This prevents an "unbound variable" crash under `set -u` if no argument is supplied.

```bash
./log_rotate.sh
# → Usage: ./log_rotate.sh <log-directory>

./log_rotate.sh /tmp/does-not-exist
# → ERROR: Directory does not exist: /tmp/does-not-exist
```

### Test the rotation logic (make a file "old")
```bash
touch -d "10 days ago" /tmp/day19-lab/logs/app.log
ls -l /tmp/day19-lab/logs

./log_rotate.sh /tmp/day19-lab/logs
ls -lh /tmp/day19-lab/logs   # → app.log.gz
```

### Test the delete logic
```bash
echo "old log" > /tmp/day19-lab/logs/old.log
gzip /tmp/day19-lab/logs/old.log
touch -d "40 days ago" /tmp/day19-lab/logs/old.log.gz

ls -lh /tmp/day19-lab/logs
./log_rotate.sh /tmp/day19-lab/logs
ls -lh /tmp/day19-lab/logs   # old.log.gz should be gone
```

---

## Part 5: Task 2 — Server Backup Script

```bash
nano backup.sh
```

```bash
#!/bin/bash
set -euo pipefail

SOURCE="${1:-}"
DESTINATION="${2:-}"

if [ -z "$SOURCE" ] || [ -z "$DESTINATION" ]; then
    echo "Usage: $0 <source-directory> <backup-directory>"
    exit 1
fi

if [ ! -d "$SOURCE" ]; then
    echo "ERROR: Source directory does not exist: $SOURCE"
    exit 1
fi

mkdir -p "$DESTINATION"

DATE=$(date +%Y-%m-%d_%H-%M-%S)
ARCHIVE="$DESTINATION/backup-$DATE.tar.gz"

echo "Starting backup..."
echo "Source: $SOURCE"
echo "Destination: $ARCHIVE"

tar -czf "$ARCHIVE" -C "$(dirname "$SOURCE")" "$(basename "$SOURCE")"

if [ ! -f "$ARCHIVE" ]; then
    echo "ERROR: Backup failed"
    exit 1
fi

SIZE=$(du -h "$ARCHIVE" | cut -f1)

echo "Backup completed successfully."
echo "Archive: $ARCHIVE"
echo "Size: $SIZE"

echo "Removing backups older than 14 days..."
find "$DESTINATION" -type f -name "backup-*.tar.gz" -mtime +14 -delete
echo "Old backup cleanup completed."
```

```bash
chmod +x backup.sh
./backup.sh /tmp/day19-lab/source /tmp/day19-lab/backups
```

### Verify
```bash
ls -lh /tmp/day19-lab/backups
# backup-2026-08-12_20-50-10.tar.gz

tar -tzf /tmp/day19-lab/backups/backup-*.tar.gz

mkdir -p /tmp/day19-lab/restore-test
tar -xzf /tmp/day19-lab/backups/backup-*.tar.gz -C /tmp/day19-lab/restore-test
find /tmp/day19-lab/restore-test -type f
```

### Error handling test
```bash
./backup.sh /tmp/not-exist /tmp/day19-lab/backups
# → ERROR: Source directory does not exist: /tmp/not-exist
```
A production backup script should never silently "succeed" when the source is missing.

---

## Part 6: Task 3 — Crontab

```bash
crontab -l    # "no crontab for ubuntu" is fine — means none set yet
crontab -e
```

> Don't apply these until the paths make sense for your setup.

```bash
# Log rotation every day at 2 AM
0 2 * * * /home/ubuntu/day-19/log_rotate.sh /var/log/myapp >> /home/ubuntu/log_rotate-cron.log 2>&1

# Backup every Sunday at 3 AM
0 3 * * 0 /home/ubuntu/day-19/backup.sh /opt/myapp /backup/myapp >> /home/ubuntu/backup-cron.log 2>&1

# Health check every 5 minutes
*/5 * * * * /home/ubuntu/day-19/health_check.sh >> /home/ubuntu/health-check.log 2>&1
```

**Why `>> logfile 2>&1`?**
- `>> logfile` appends normal (stdout) output to the file.
- `2>&1` redirects stderr into the same place as stdout.
- Together: both normal output **and** errors get saved to the log — critical for debugging cron jobs later, since cron won't show you anything on screen.

---

## Part 7: Task 4 — Combined Maintenance Script

```bash
nano maintenance.sh
```

```bash
#!/bin/bash
set -euo pipefail

LOG_DIR="/tmp/day19-lab/logs"
SOURCE_DIR="/tmp/day19-lab/source"
BACKUP_DIR="/tmp/day19-lab/backups"
LOG_FILE="/tmp/day19-lab/maintenance.log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

rotate_logs() {
    log "Starting log rotation"
    find "$LOG_DIR" -type f -name "*.log" -mtime +7 -exec gzip {} \;
    find "$LOG_DIR" -type f -name "*.gz" -mtime +30 -delete
    log "Log rotation completed"
}

create_backup() {
    local timestamp
    local archive

    timestamp=$(date +%Y-%m-%d_%H-%M-%S)
    archive="$BACKUP_DIR/maintenance-backup-$timestamp.tar.gz"

    log "Starting backup"

    tar -czf "$archive" \
        -C "$(dirname "$SOURCE_DIR")" \
        "$(basename "$SOURCE_DIR")"

    if [ -f "$archive" ]; then
        log "Backup created: $archive"
        log "Backup size: $(du -h "$archive" | cut -f1)"
    else
        log "ERROR: Backup creation failed"
        return 1
    fi

    find "$BACKUP_DIR" \
        -type f \
        -name "maintenance-backup-*.tar.gz" \
        -mtime +14 \
        -delete

    log "Old backup cleanup completed"
}

main() {
    log "========== Maintenance Started =========="
    rotate_logs
    create_backup
    log "========== Maintenance Completed =========="
}

main
```

```bash
chmod +x maintenance.sh
./maintenance.sh
cat /tmp/day19-lab/maintenance.log
```

Expected output:
```
[2026-08-12 20:55:00] ========== Maintenance Started ==========
[2026-08-12 20:55:00] Starting log rotation
[2026-08-12 20:55:00] Log rotation completed
[2026-08-12 20:55:00] Starting backup
[2026-08-12 20:55:00] Backup created: /tmp/day19-lab/backups/maintenance-backup-2026-08-12_20-55-00.tar.gz
[2026-08-12 20:55:00] Backup size: 4.0K
[2026-08-12 20:55:00] Old backup cleanup completed
[2026-08-12 20:55:00] ========== Maintenance Completed ==========
```

---

## Part 8: Cron for the Maintenance Script

```bash
0 1 * * * /home/ubuntu/day-19/maintenance.sh >> /home/ubuntu/day-19/maintenance-cron.log 2>&1
```
Runs the full maintenance routine every day at 1:00 AM.

---

## 🧪 Complete Practice Sequence

```bash
cd ~/day-19
ls -l
find /tmp/day19-lab -type f -ls
./log_rotate.sh /tmp/day19-lab/logs
./backup.sh /tmp/day19-lab/source /tmp/day19-lab/backups
ls -lh /tmp/day19-lab/backups
tar -tzf /tmp/day19-lab/backups/backup-*.tar.gz
./maintenance.sh
cat /tmp/day19-lab/maintenance.log
crontab -l
```

---

## 🔥 DevOps Challenge — Build It Yourself

Create `health_check.sh` without looking at the earlier scripts. Requirements:

1. Use `set -euo pipefail`
2. Check disk usage
3. Check memory
4. Check nginx status
5. Print a timestamp
6. Return a failure if nginx is not running

**Useful commands:**
```bash
df -h /
free -h
systemctl is-active nginx
date
```

```bash
chmod +x health_check.sh
./health_check.sh
```

**Schedule it:**
```bash
*/5 * * * * /home/ubuntu/day-19/health_check.sh >> /home/ubuntu/day-19/health-check.log 2>&1
```

---

## ⚠️ Production Considerations

These lab scripts are intentionally simple. A real production setup would go further:

**Log rotation** — use the built-in `logrotate` tool instead of a custom script:
```bash
logrotate
ls -l /etc/logrotate.d/
```

**Backups** should typically also include:
- Remote storage
- Encryption
- Retention policy
- Backup verification
- Monitoring & alerts
- Restore testing

A production AWS design might look like:
```
EC2
 │
 ├── Application logs
 │
 ├── Backup script
 │
 └── S3
       │
       └── Lifecycle / retention
```

**Cron** works fine for labs, but production environments often prefer:
- systemd timers
- AWS EventBridge
- AWS Systems Manager
- CI/CD schedulers
- Kubernetes CronJobs

---

## Author
**Sarvesh** — Day 19: Log Rotation, Backup & Crontab
