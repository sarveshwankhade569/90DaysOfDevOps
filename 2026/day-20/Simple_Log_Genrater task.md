# Day 20 – Log Analyzer & Report Generator

A Bash script that reads a log file and tells you what's going on in it: how many errors, what critical events happened, and which errors show up the most. It also saves everything into a daily report file.

## What we're working with

A log file that looks like this:

```
2026-08-13 09:10:11 INFO Application started
2026-08-13 09:11:22 ERROR Connection timed out
2026-08-13 09:12:10 INFO User logged in
2026-08-13 09:13:45 ERROR Connection timed out
2026-08-13 09:14:20 CRITICAL Disk space below threshold
```

The script answers:
- How many errors happened?
- What were the critical events?
- What are the most common errors?
- How many lines did we process?

And it saves the answer to `log_report_2026-08-13.txt`.

## The commands behind it

A few Linux tools do all the heavy lifting here:

- **`grep`** – searches for text. `grep "ERROR" file.log` finds every line with "ERROR" in it.
- **`grep -c`** – same, but just gives you the count instead of the lines.
- **`grep -n`** – shows line numbers along with the matches, handy for finding *where* something happened.
- **`grep -E "ERROR|Failed"`** – matches multiple patterns at once (either word).
- **`wc -l`** – counts lines. Use `wc -l < file.log` (with the `<`) if you want just the number, no filename attached — useful when you're saving it into a variable.
- **`sed`** – edits text on the fly. We use it to strip the timestamp off each line so we're left with just the error message.
- **`sort`** and **`uniq -c`** – sort groups identical lines together, `uniq -c` then counts how many times each one repeats.
- **`head -5`** – keeps just the top 5 results.

## Finding the most common errors

This is the fun part — chaining commands together to turn raw log lines into a ranked list.

Start with the error lines:
```bash
grep "ERROR" sample_log.log
```

Strip off the timestamp and the word ERROR, keeping just the message:
```bash
grep "ERROR" sample_log.log | sed 's/^.*ERROR //'
```

Now group identical messages together and count them:
```bash
grep "ERROR" sample_log.log | sed 's/^.*ERROR //' | sort | uniq -c
```

And sort so the most frequent error is on top:
```bash
grep "ERROR" sample_log.log | sed 's/^.*ERROR //' | sort | uniq -c | sort -rn | head -5
```

Step by step, that pipeline is: **find the errors → strip the noise → group duplicates → count them → sort by count → keep the top 5.**

## The script

```bash
#!/bin/bash
set -euo pipefail

LOG_FILE="${1:-}"

if [ -z "$LOG_FILE" ]; then
    echo "Usage: $0 <log-file>"
    exit 1
fi

if [ ! -f "$LOG_FILE" ]; then
    echo "ERROR: Log file does not exist: $LOG_FILE"
    exit 1
fi

DATE=$(date +%Y-%m-%d)
REPORT="log_report_${DATE}.txt"

TOTAL_LINES=$(wc -l < "$LOG_FILE")
ERROR_COUNT=$(grep -Ec "ERROR|Failed" "$LOG_FILE" || true)
CRITICAL_EVENTS=$(grep -n "CRITICAL" "$LOG_FILE" || true)
TOP_ERRORS=$(grep "ERROR" "$LOG_FILE" | sed 's/^.*ERROR //' | sort | uniq -c | sort -rn | head -5 || true)

echo "Analyzing log file..."
echo "Log file: $LOG_FILE"
echo "Total lines: $TOTAL_LINES"
echo "Total errors: $ERROR_COUNT"

echo
echo "--- Critical Events ---"
[ -n "$CRITICAL_EVENTS" ] && echo "$CRITICAL_EVENTS" || echo "No critical events found."

echo
echo "--- Top 5 Error Messages ---"
[ -n "$TOP_ERRORS" ] && echo "$TOP_ERRORS" || echo "No ERROR messages found."

cat > "$REPORT" <<EOF
========================================
       DAILY LOG ANALYSIS REPORT
========================================

Date of Analysis: $DATE
Log File: $LOG_FILE

Total Lines Processed: $TOTAL_LINES
Total Error Count: $ERROR_COUNT

--- Top 5 Error Messages ---

$TOP_ERRORS

--- Critical Events ---

$CRITICAL_EVENTS

========================================
Report generated successfully.
========================================
EOF

echo
echo "Report created: $REPORT"
```

**One thing worth noticing:** several commands end with `|| true`. Since the script uses `set -e`, any command that fails would normally kill the script — but `grep` "fails" (exit code 1) whenever it finds nothing, which isn't actually an error here. `|| true` tells Bash "if this finds nothing, that's fine, keep going."

## Running it

```bash
chmod +x log_analyzer.sh
./log_analyzer.sh sample_log.log
```

Expected output looks something like:
```
Analyzing log file...
Log file: sample_log.log
Total lines: 20
Total errors: 9

--- Critical Events ---
9:2026-08-13 09:08:12 CRITICAL Database connection lost
16:2026-08-13 09:15:30 CRITICAL Disk space below threshold

--- Top 5 Error Messages ---
4 Connection timed out
3 Permission denied

Report created: log_report_2026-08-13.txt
```

Check the report:
```bash
cat log_report_$(date +%Y-%m-%d).txt
```

It also handles the obvious mistakes:
```bash
./log_analyzer.sh              # → Usage: ./log_analyzer.sh <log-file>
./log_analyzer.sh missing.log  # → ERROR: Log file does not exist: missing.log
```

## Optional: archiving old logs

Once a log file's been analyzed, you might want to move it out of the way:
```bash
mkdir -p archive
mv "$LOG_FILE" archive/
```
Just don't add this on your first test run — it'll move your sample log away, and your next test will fail because the file's gone.

## What this teaches (the real takeaway)

The point isn't this one script — it's a pattern you'll reuse constantly when working with Linux servers:

```
raw log data → grep (extract) → sed/awk (clean) → sort/uniq (aggregate) → report → automate with cron
```

Once that pattern clicks, you can point it at almost any log file and pull out something useful.
