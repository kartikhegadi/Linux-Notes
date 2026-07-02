# Linux & DevOps Automation Scripts Collection

A curated set of Bash utilities for workspace setup, file management, backups, system monitoring, user provisioning, security auditing, Docker/Kubernetes operations, AWS, and CI/CD.

## Table of Contents

- [1. Workspace & File Management](#1-workspace--file-management)
  - [1.1 Workspace Initializer](#11-workspace-initializer)
  - [1.2 Smart File Mover](#12-smart-file-mover)
- [2. Backup & Restore](#2-backup--restore)
  - [2.1 Remote Backup & Transfer (SCP)](#21-remote-backup--transfer-scp)
  - [2.2 Local Backup & Restore Utility](#22-local-backup--restore-utility)
- [3. System Monitoring](#3-system-monitoring)
  - [3.1 System Health Monitor (CSV Logging)](#31-system-health-monitor-csv-logging)
  - [3.2 Quick Disk Space Alert](#32-quick-disk-space-alert)
  - [3.3 System Uptime, CPU & Memory Snapshot](#33-system-uptime-cpu--memory-snapshot)
  - [3.4 Find Top 5 Largest Files](#34-find-top-5-largest-files)
  - [3.5 Active SSH Sessions](#35-active-ssh-sessions)
- [4. User Provisioning](#4-user-provisioning)
  - [4.1 Bulk User Creation](#41-bulk-user-creation)
  - [4.2 Bulk User Deletion](#42-bulk-user-deletion)
  - [4.3 Single Quick User Creation](#43-single-quick-user-creation)
- [5. Security & Log Auditing](#5-security--log-auditing)
  - [5.1 Log Auditor & Rotator](#51-log-auditor--rotator)
  - [5.2 Automated Log Cleanup](#52-automated-log-cleanup)
- [6. Service Management](#6-service-management)
  - [6.1 Service Health Check & Auto-Restart](#61-service-health-check--auto-restart)
- [7. Docker](#7-docker)
  - [7.1 List Running Containers](#71-list-running-containers)
  - [7.2 Prune Unused Images](#72-prune-unused-images)
  - [7.3 Backup Running Containers & Images](#73-backup-running-containers--images)
- [8. Kubernetes](#8-kubernetes)
  - [8.1 Pod Health Check](#81-pod-health-check)
  - [8.2 Node Status Check](#82-node-status-check)
  - [8.3 Restart All Pods in a Namespace](#83-restart-all-pods-in-a-namespace)
  - [8.4 Live Pod Monitor](#84-live-pod-monitor)
- [9. AWS](#9-aws)
  - [9.1 List EC2 Instances](#91-list-ec2-instances)
  - [9.2 S3 Bucket Sync](#92-s3-bucket-sync)
- [10. CI/CD (Jenkins)](#10-cicd-jenkins)
  - [10.1 Trigger a Jenkins Job](#101-trigger-a-jenkins-job)
  - [10.2 Check Last Build Status](#102-check-last-build-status)

---

## 1. Workspace & File Management

### 1.1 Workspace Initializer

Sets up a development directory and populates it with dummy files for testing.

```bash
#!/bin/bash
mkdir -p workspace
cd workspace
for i in {1..10}
do
    touch "$i.file.txt"
done
echo "Workspace created with 10 files."
```

### 1.2 Smart File Mover

Moves `.txt` files between directories, with validation and error handling.

```bash
#!/bin/bash
# 1. Ask for the Source Directory
read -p "Enter the absolute path of the SOURCE directory: " src_path

if [ ! -d "$src_path" ]; then
    echo "Error: Source directory '$src_path' does not exist."
    exit 1
fi

# 2. Ask for the Destination Directory
read -p "Enter the absolute path of the DESTINATION directory: " dest_path

# 3. Create destination if it doesn't exist
if [ ! -d "$dest_path" ]; then
    echo "Destination does not exist. Creating it now..."
    mkdir -p "$dest_path" || { echo "Failed to create directory"; exit 1; }
fi

# 4. Move the files
shopt -s nullglob
files=("$src_path"/*.txt)

if [ ${#files[@]} -gt 0 ]; then
    mv "$src_path"/*.txt "$dest_path"
    echo "Success: Moved ${#files[@]} .txt file(s) from $src_path to $dest_path"
else
    echo "No .txt files found in '$src_path'."
fi
shopt -u nullglob
```

---

## 2. Backup & Restore

### 2.1 Remote Backup & Transfer (SCP)

Archives a directory into a ZIP file and transfers it to a remote server over SCP.

> **Prerequisite:** passwordless SSH key auth configured to the remote host.

```bash
#!/bin/bash

# --- 1. ZIP THE SOURCE ---
read -p "Enter the folder/file path you want to zip: " SRC_PATH
ZIP_NAME="backup_$(date +%Y%m%d_%H%M%S).zip"

if [ -e "$SRC_PATH" ]; then
    echo "Creating archive: $ZIP_NAME..."
    zip -rq "$ZIP_NAME" "$SRC_PATH"

    if [ $? -ne 0 ]; then
        echo "Error: Failed to create zip."
        exit 1
    fi
    echo "Zip created successfully."
else
    echo "Error: Path '$SRC_PATH' does not exist."
    exit 1
fi

# --- 2. TRANSFER TO REMOTE ---
echo "--- Remote Destination Details ---"
read -p "Enter Remote Username: " R_USER
read -p "Enter Remote IP: " R_IP
read -p "Enter Remote Destination Path: " R_PATH

scp "$ZIP_NAME" "$R_USER@$R_IP:$R_PATH"

# --- 3. FINAL CHECK & CLEANUP ---
if [ $? -eq 0 ]; then
    echo "SUCCESS: $ZIP_NAME transferred to $R_IP"
    read -p "Remove local zip file? (y/n): " CLEANUP
    if [ "$CLEANUP" == "y" ]; then
        rm "$ZIP_NAME"
        echo "Local zip removed."
    fi
else
    echo "Error: Transfer failed."
fi
```

### 2.2 Local Backup & Restore Utility

Interactive menu to back up `/var/www/html` (or any configured directory) and restore from a chosen archive.

```bash
#!/bin/bash

BACKUP_DIR="/backup"
SOURCE_DIR="/var/www/html"
TIMESTAMP=$(date +"%F-%H-%M-%S")
BACKUP_FILE="$BACKUP_DIR/backup-$TIMESTAMP.tar.gz"

mkdir -p "$BACKUP_DIR"

backup() {
    echo "Starting backup..."
    tar -czf "$BACKUP_FILE" "$SOURCE_DIR"
    echo "Backup completed: $BACKUP_FILE"
}

restore() {
    echo "Available backups:"
    ls -lh "$BACKUP_DIR"
    read -p "Enter backup file name to restore: " FILE
    tar -xzf "$BACKUP_DIR/$FILE" -C /
    echo "Restore completed."
}

echo "1. Backup"
echo "2. Restore"
read -p "Choose an option: " CHOICE

case $CHOICE in
    1) backup ;;
    2) restore ;;
    *) echo "Invalid option" ;;
esac
```

---

## 3. System Monitoring

### 3.1 System Health Monitor (CSV Logging)

Checks CPU, memory, and disk usage against thresholds, prints alerts, and appends a row to a CSV log for reporting/trend analysis.

```bash
#!/bin/bash

CPU_THRESHOLD=80
MEM_THRESHOLD=80
DISK_THRESHOLD=80
LOG_FILE="system_report.csv"

echo "Checking System Health..."

CPU_USAGE=$(top -bn1 | grep "Cpu(s)" | awk '{print $2 + $4}')
CPU_INT=${CPU_USAGE%.*}

MEM_USAGE=$(free | grep Mem | awk '{print $3/$2 * 100.0}')
MEM_INT=${MEM_USAGE%.*}

DISK_USAGE=$(df -h / | awk 'NR==2 {print $5}' | sed 's/%//')

# Log to CSV
echo "$(date), CPU: ${CPU_INT}%, Mem: ${MEM_INT}%, Disk: ${DISK_USAGE}%" >> "$LOG_FILE"

if [ "$CPU_INT" -ge "$CPU_THRESHOLD" ]; then
    echo -e "\e[31m⚠️  CPU Usage High: ${CPU_INT}%\e[0m"
fi

if [ "$MEM_INT" -ge "$MEM_THRESHOLD" ]; then
    echo -e "\e[31m⚠️  Memory Usage High: ${MEM_INT}%\e[0m"
fi

if [ "$DISK_USAGE" -ge "$DISK_THRESHOLD" ]; then
    echo -e "\e[31m⚠️  Disk Usage High: ${DISK_USAGE}%\e[0m"
fi

echo "System Health Check Completed."
```

### 3.2 Quick Disk Space Alert

Lightweight, single-purpose disk usage check — useful as a cron job.

```bash
#!/bin/bash

THRESHOLD=80
DISK_USAGE=$(df -h / | awk 'NR==2 {print $5}' | sed 's/%//')

if [ "$DISK_USAGE" -ge "$THRESHOLD" ]; then
    echo "⚠️ Disk usage is high: $DISK_USAGE%"
else
    echo "✅ Disk usage is normal: $DISK_USAGE%"
fi
```

### 3.3 System Uptime, CPU & Memory Snapshot

```bash
#!/bin/bash

echo "System Uptime:"
uptime

echo -e "\nCPU Usage:"
top -bn1 | grep "Cpu(s)"

echo -e "\nMemory Usage:"
free -m

echo -e "\nDisk Space Usage:"
df -h
```

### 3.4 Find Top 5 Largest Files

```bash
#!/bin/bash

echo "Top 5 largest files in the system:"
find / -type f -exec du -h {} + 2>/dev/null | sort -rh | head -n 5
```

### 3.5 Active SSH Sessions

```bash
#!/bin/bash

echo "Active SSH Sessions:"
who | grep "pts"
```

---

## 4. User Provisioning

### 4.1 Bulk User Creation

Reads usernames from `users.txt`, creates accounts, sets a temporary password, and forces a password reset on first login. Logs actions to `/var/log/user_provision.log`.

> **Requires root.** Change the temporary password before using in production, or generate one per user.

```bash
#!/bin/bash
# Note: This script must be run with sudo
USER_FILE="users.txt"

if [[ $EUID -ne 0 ]]; then
   echo "CRITICAL: This script must be run as root."
   exit 1
fi

if [ ! -f "$USER_FILE" ]; then
    echo "Error: $USER_FILE not found!"
    exit 1
fi

while IFS= read -r user || [ -n "$user" ]; do
    [ -z "$user" ] && continue

    if id "$user" &>/dev/null; then
        echo "User '$user' already exists. Skipping..."
    else
        useradd -m -s /bin/bash "$user"
        echo "$user:Welcome@2026" | chpasswd
        chage -d 0 "$user"
        echo "[$(date)] Created user: $user" >> /var/log/user_provision.log
        echo "Successfully onboarded: $user"
    fi
done < "$USER_FILE"
```

### 4.2 Bulk User Deletion

Reads usernames from `users.txt` and removes each account along with its home directory.

> **Requires root. Destructive — verify `users.txt` before running.**

```bash
#!/bin/bash
# Note: This script must be run with sudo
USER_FILE="users.txt"

if [[ $EUID -ne 0 ]]; then
   echo "CRITICAL: This script must be run as root."
   exit 1
fi

if [ ! -f "$USER_FILE" ]; then
    echo "Error: $USER_FILE not found!"
    exit 1
fi

while IFS= read -r user || [ -n "$user" ]; do
    [ -z "$user" ] && continue

    if id "$user" &>/dev/null; then
        userdel -r "$user"
        echo "[$(date)] Deleted user and home directory: $user" >> /var/log/user_provision.log
        echo "Deleted user and home directory: $user"
    else
        echo "User '$user' does not exist."
    fi
done < "$USER_FILE"
```

### 4.3 Single Quick User Creation

Interactive one-off user creation for quick testing (not intended for bulk/production provisioning — see 4.1).

```bash
#!/bin/bash

read -p "Enter username: " USERNAME
PASSWORD="Password@123"

sudo useradd -m -s /bin/bash "$USERNAME"
echo "$USERNAME:$PASSWORD" | sudo chpasswd

echo "✅ User $USERNAME created with a temporary password."
```

---

## 5. Security & Log Auditing

### 5.1 Log Auditor & Rotator

Scans `auth.log` for repeated failed login attempts (>3), writes offending sources to `blacklist.txt`, then archives and removes logs older than 7 days.

```bash
#!/bin/bash
# Industrial Log Auditor

LOG_DIR="/var/log"
ARCHIVE_DIR="./log_archive"
mkdir -p "$ARCHIVE_DIR"

echo "--- Starting Security Audit ---"
# 1. Find failed login attempts (> 3) and save to blacklist.txt
grep "Failed password" /var/log/auth.log | awk '{print $(NF-3)}' | sort | uniq -c | awk '$1 > 3 {print $2}' > blacklist.txt
echo "Threats identified and saved to blacklist.txt"

# 2. Rotate logs older than 7 days
echo "--- Rotating Old Logs ---"
find "$LOG_DIR" -name "*.log" -mtime +7 -exec tar -rvf "$ARCHIVE_DIR/old_logs_$(date +%F).tar" {} +

# 3. Cleanup: Delete the original old files
find "$LOG_DIR" -name "*.log" -mtime +7 -delete

echo "Old logs archived and storage cleared."
```

### 5.2 Automated Log Cleanup

Simpler cleanup-only variant: deletes logs older than a configurable number of days without archiving. Use 5.1 instead if you need an audit trail.

```bash
#!/bin/bash

LOG_DIR="/var/log"
DAYS=30

echo "Cleaning logs older than $DAYS days in $LOG_DIR..."
find "$LOG_DIR" -type f -name "*.log" -mtime +$DAYS -exec rm -f {} \;
echo "Log cleanup completed."
```

---

## 6. Service Management

### 6.1 Service Health Check & Auto-Restart

```bash
#!/bin/bash

SERVICE="nginx"

if systemctl is-active --quiet "$SERVICE"; then
    echo "✅ $SERVICE is running"
else
    echo "⚠️ $SERVICE is not running, restarting..."
    systemctl restart "$SERVICE"
fi
```

---

## 7. Docker

### 7.1 List Running Containers

```bash
#!/bin/bash

echo "Running Docker containers:"
docker ps --format "table {{.ID}}\t{{.Image}}\t{{.Status}}"
```

### 7.2 Prune Unused Images

```bash
#!/bin/bash

echo "Deleting unused Docker images..."
docker image prune -a -f
echo "✅ Cleanup done!"
```

### 7.3 Backup Running Containers & Images

Commits each running container to an image and saves both containers and all images as `.tar` archives.

```bash
#!/bin/bash

BACKUP_DIR="/backup/docker"
mkdir -p "$BACKUP_DIR"

echo "🔄 Backing up running containers..."
for container in $(docker ps -q); do
    docker commit "$container" "$container-backup"
    docker save -o "$BACKUP_DIR/$container.tar" "$container-backup"
done

echo "🔄 Saving all Docker images..."
docker images -q | xargs -I {} docker save -o "$BACKUP_DIR/{}.tar" {}

echo "✅ Docker backup completed."
```

---

## 8. Kubernetes

### 8.1 Pod Health Check

Reports any pod not in the `Running` state for a given namespace.

```bash
#!/bin/bash

NAMESPACE="default"

echo "Checking Kubernetes Pod Health..."
kubectl get pods -n "$NAMESPACE" --no-headers | awk '$3 != "Running" {print "⚠️ Pod "$1" is in state "$3}'
echo "Health Check Completed."
```

### 8.2 Node Status Check

```bash
#!/bin/bash

echo "Checking Kubernetes Nodes..."
kubectl get nodes | grep -v "Ready"
```

### 8.3 Restart All Pods in a Namespace

> **Destructive** — force-deletes all pods in the target namespace so they are recreated by their controllers.

```bash
#!/bin/bash

NAMESPACE="default"

echo "Restarting all pods in $NAMESPACE..."
kubectl delete pods --all -n "$NAMESPACE" --grace-period=0 --force
echo "✅ All pods restarted!"
```

### 8.4 Live Pod Monitor

```bash
#!/bin/bash

NAMESPACE="default"

echo "Monitoring pods in namespace: $NAMESPACE..."
kubectl get pods -n "$NAMESPACE" --watch
```

---

## 9. AWS

### 9.1 List EC2 Instances

```bash
#!/bin/bash

echo "🔎 Fetching EC2 Instances..."
aws ec2 describe-instances --query "Reservations[*].Instances[*].[InstanceId, PublicIpAddress]" --output table
```

### 9.2 S3 Bucket Sync

```bash
#!/bin/bash

BUCKET_NAME="my-backup-bucket"
SOURCE_DIR="/backup"

echo "Syncing $SOURCE_DIR to S3 bucket $BUCKET_NAME..."
aws s3 sync "$SOURCE_DIR" "s3://$BUCKET_NAME" --delete
echo "Sync Completed!"
```

---

## 10. CI/CD (Jenkins)

> **Note:** Do not commit real API tokens. Load `USER` / `API_TOKEN` from environment variables or a secrets manager instead of hardcoding them.

### 10.1 Trigger a Jenkins Job

```bash
#!/bin/bash

JENKINS_URL="http://localhost:8080"
JOB_NAME="MyJob"
USER="${JENKINS_USER:?Set JENKINS_USER env var}"
API_TOKEN="${JENKINS_API_TOKEN:?Set JENKINS_API_TOKEN env var}"

echo "Triggering Jenkins job: $JOB_NAME..."
curl -X POST "$JENKINS_URL/job/$JOB_NAME/build" --user "$USER:$API_TOKEN"
echo "✅ Job triggered successfully!"
```

### 10.2 Check Last Build Status

```bash
#!/bin/bash

JENKINS_URL="http://localhost:8080"
JOB_NAME="MyJob"
USER="${JENKINS_USER:?Set JENKINS_USER env var}"
API_TOKEN="${JENKINS_API_TOKEN:?Set JENKINS_API_TOKEN env var}"

echo "Fetching status of last build..."
curl -s "$JENKINS_URL/job/$JOB_NAME/lastBuild/api/json" --user "$USER:$API_TOKEN" | jq -r '.result'
```

---

## Notes for Contributors

- Scripts assume a Debian/Ubuntu-style environment (`apt`, `/var/log/auth.log`) unless noted otherwise.
- Scripts requiring root (`useradd`, `userdel`, log rotation in `/var/log`) check `$EUID` before proceeding — keep this pattern when adding new scripts.
- Avoid hardcoding credentials or tokens (see section 10) — use environment variables or a secrets manager.
- Destructive operations (pod force-delete, user deletion, log deletion) are flagged with a warning callout above the code block.

## 📝 Document Info

| Field | Value |
|---|---|
| **Author** | Kartik Hegadi |
| **Organization** | InfraCorps |
| **Repository Topic** | Basic Scripting |
| **Audience** | Learners |