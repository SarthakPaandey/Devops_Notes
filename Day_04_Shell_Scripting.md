# Day 4: Deep Dive into Shell Scripting & Automation
**Date:** 3rd November

## 1. Introduction to Shell Scripting
A shell script is a program written for the shell to execute. It's the "glue" of DevOps.
*   **Use Cases:** System Admin, Backups, Monitoring, Deployments (CI/CD steps).
*   **Interpreters:** Bash (`/bin/bash`), Sh (`/bin/sh`), Python (`/usr/bin/python3`).

## 2. The Shebang (`#!`)
Always the first line. Tells the kernel what interpreter to use.
*   `#!/bin/bash`: Standard Bash.
*   `#!/bin/sh`: POSIX compliant (Strict/Minimal).
*   `#!/usr/bin/env python3`: Python script.
*   `set -e`: Exit immediately if a command exits with a non-zero status.
*   `set -u`: Treat unset variables as an error.
*   `set -o pipefail`: Return value of a pipeline is the value of the last (rightmost) command to exit with a non-zero status.

---

## 3. Core Concepts

### Variables
*   **String:** `NAME="DevOps"` (No spaces around `=`).
*   **Interpolation:** `echo "Hello ${NAME}"`.
*   **Command Substitution:** `DATE=$(date +%F)`.
*   **Arithmetic:** `COUNT=$((COUNT + 1))`.

### Input/Output
*   `read -p "Enter name: " USERNAME`: Prompt user.
*   `$1`, `$2`: Positional arguments (passed to script).
*   `$@`: All arguments as a list.
*   `$#`: Number of arguments.

### Conditionals (`if-else`)
```bash
if [ -d "/tmp" ]; then
  echo "Folder exists"
elif [ "$USER" == "root" ]; then
  echo "Hello Admin"
else
  echo "Who are you?"
fi
```
*   `-f`: File exists.
*   `-z`: String is empty.
*   `-n`: String is not empty.

### Loops (`for`, `while`)
```bash
# C-Style For Loop
for ((i=1; i<=5; i++)); do
   echo "Number $i"
done

# File Iteration
for file in *.log; do
   gzip "$file"
done
```

---

## 4. Advanced: Power Parsing (Regex)

### 1. Grep (Global Regular Expression Print)
Search text.
*   `grep "Error" file.log`: Find "Error".
*   `grep -i`: Case insensitive.
*   `grep -v`: Invert match (Show lines NOT matching).
*   `grep -E "Error|Warning"`: Extended Regex (OR).
*   `grep -r "TODO" .`: Recursive search.

### 2. Sed (Stream Editor)
Edit text streams.
*   `sed 's/foo/bar/g' file`: Replace 'foo' with 'bar' globally.
*   `sed -i`: In-place edit (Modifies file).
*   `sed '/^#/d' config.conf`: Delete lines starting with # (Comments).

### 3. Awk (Pattern Scanning & Processing)
Column processing.
*   `awk '{print $1}'`: Print first column.
*   `awk -F: '{print $1}' /etc/passwd`: Print usernames (field separator is `:`).
*   `awk '/Error/ {print $0}'`: Print lines matching "Error".

---

## 5. Automation Example: Production Backup Script
A robust script with logging, error handling, and rotation.

```bash
#!/bin/bash
set -euo pipefail

# Configuration
SOURCE="/var/www/html"
DEST="/backup"
DATE=$(date +%Y-%m-%d_%H-%M-%S)
FILENAME="site_backup_$DATE.tar.gz"
LOGFILE="/var/log/backup.log"
RETENTION_DAYS=7

# Logging Function
log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOGFILE"
}

# Prerequisites Check
if [ ! -d "$SOURCE" ]; then
    log "ERROR: Source directory $SOURCE does not exist!"
    exit 1
fi

mkdir -p "$DEST"

log "INFO: Starting backup of $SOURCE..."

# Create compressed archive
tar -czf "$DEST/$FILENAME" "$SOURCE" 2>> "$LOGFILE"

if [ $? -eq 0 ]; then
    log "SUCCESS: Backup created at $DEST/$FILENAME"
else
    log "ERROR: Tar command failed!"
    exit 1
fi

# Cleanup old backups (Retention)
log "INFO: Cleaning up backups older than $RETENTION_DAYS days..."
find "$DEST" -name "site_backup_*.tar.gz" -mtime +$RETENTION_DAYS -exec rm {} \;

log "INFO: Backup process completed."
```

---

## 6. Interview Questions
1.  **Q: What is the difference between `$@` and `$*`?**
    *   *A:* `$@` treats arguments as separate strings ("arg1" "arg2"). `$*` treats them as one single string ("arg1 arg2"). Always use `"$@"`.
2.  **Q: How do you check if the previous command failed?**
    *   *A:* Check the exit code `$?`. `if [ $? -ne 0 ]; then ...`.
3.  **Q: Explain `2>&1`.**
    *   *A:* Redirects Stderr (2) to Stdout (1). Used to capture error messages into the same log file as normal output. `command > log.txt 2>&1`.
4.  **Q: What is a Cron Job?**
    *   *A:* A scheduled task. Edited via `crontab -e`.
    *   Format: `* * * * * command` (Min, Hour, DayOfMonth, Month, DayOfWeek).
    *   Example: `0 3 * * * /backup.sh` (Run at 3 AM daily).
