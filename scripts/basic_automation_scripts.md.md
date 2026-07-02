## Linux Automation Scripts

A curated set of Bash utilities for workspace setup, file management, backups, system monitoring, user provisioning, security auditing, Docker/Kubernetes operations, AWS, and CI/CD.

### Workspace Initializer

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

### Smart File Mover

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

### Remote Backup and Transfer

Archives a directory into a ZIP file and transfers it to a remote server over SCP. This requires passwordless SSH key authentication to be configured for the remote host.

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

### Local Backup and Restore

Interactive menu to back up a target directory and restore from a chosen archive.

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

### System Health Monitor

Checks CPU, memory, and disk usage against thresholds, prints alerts, and appends a row to a CSV log for reporting and trend analysis.

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

### Quick Disk Space Alert

A lightweight, single-purpose disk usage check, useful as a cron job.

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

### System Snapshot

Reports uptime, CPU usage, memory usage, and disk space in one pass.

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

### Finding the Largest Files

```bash
#!/bin/bash

echo "Top 5 largest files in the system:"
find / -type f -exec du -h {} + 2>/dev/null | sort -rh | head -n 5
```

### Active SSH Sessions

```bash
#!/bin/bash

echo "Active SSH Sessions:"
who | grep "pts"
```

### Bulk User Creation

Reads usernames from `users.txt`, creates accounts, sets a temporary password, and forces a password reset on first login. Logs actions to `/var/log/user_provision.log`. This script requires root and the temporary password should be changed before using it in production.

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

### Bulk User Deletion

Reads usernames from `users.txt` and removes each account along with its home directory. This script requires root and is destructive, so verify `users.txt` before running it.

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

### Quick Single User Creation

Interactive one-off user creation for quick testing. For bulk or production provisioning, use the bulk creation script above instead.

```bash
#!/bin/bash

read -p "Enter username: " USERNAME
PASSWORD="Password@123"

sudo useradd -m -s /bin/bash "$USERNAME"
echo "$USERNAME:$PASSWORD" | sudo chpasswd

echo "✅ User $USERNAME created with a temporary password."
```

### Log Auditor and Rotator

Scans `auth.log` for repeated failed login attempts (more than 3), writes offending sources to `blacklist.txt`, then archives and removes logs older than 7 days.

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

### Automated Log Cleanup

A simpler cleanup-only variant that deletes logs older than a configurable number of days without archiving. Use the log auditor above instead if you need an audit trail.

```bash
#!/bin/bash

LOG_DIR="/var/log"
DAYS=30

echo "Cleaning logs older than $DAYS days in $LOG_DIR..."
find "$LOG_DIR" -type f -name "*.log" -mtime +$DAYS -exec rm -f {} \;
echo "Log cleanup completed."
```

### Service Health Check and Auto-Restart

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

### Listing Running Docker Containers

```bash
#!/bin/bash

echo "Running Docker containers:"
docker ps --format "table {{.ID}}\t{{.Image}}\t{{.Status}}"
```

### Pruning Unused Docker Images

```bash
#!/bin/bash

echo "Deleting unused Docker images..."
docker image prune -a -f
echo "✅ Cleanup done!"
```

### Backing Up Docker Containers and Images

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

### Kubernetes Pod Health Check

Reports any pod not in the `Running` state for a given namespace.

```bash
#!/bin/bash

NAMESPACE="default"

echo "Checking Kubernetes Pod Health..."
kubectl get pods -n "$NAMESPACE" --no-headers | awk '$3 != "Running" {print "⚠️ Pod "$1" is in state "$3}'
echo "Health Check Completed."
```

### Kubernetes Node Status Check

```bash
#!/bin/bash

echo "Checking Kubernetes Nodes..."
kubectl get nodes | grep -v "Ready"
```

### Restarting All Pods in a Namespace

This force-deletes all pods in the target namespace so they are recreated by their controllers. Treat it as a destructive operation.

```bash
#!/bin/bash

NAMESPACE="default"

echo "Restarting all pods in $NAMESPACE..."
kubectl delete pods --all -n "$NAMESPACE" --grace-period=0 --force
echo "✅ All pods restarted!"
```

### Live Pod Monitor

```bash
#!/bin/bash

NAMESPACE="default"

echo "Monitoring pods in namespace: $NAMESPACE..."
kubectl get pods -n "$NAMESPACE" --watch
```

### Listing EC2 Instances

```bash
#!/bin/bash

echo "🔎 Fetching EC2 Instances..."
aws ec2 describe-instances --query "Reservations[*].Instances[*].[InstanceId, PublicIpAddress]" --output table
```

### S3 Bucket Sync

```bash
#!/bin/bash

BUCKET_NAME="my-backup-bucket"
SOURCE_DIR="/backup"

echo "Syncing $SOURCE_DIR to S3 bucket $BUCKET_NAME..."
aws s3 sync "$SOURCE_DIR" "s3://$BUCKET_NAME" --delete
echo "Sync Completed!"
```

### Triggering a Jenkins Job

Do not commit real API tokens. Load `USER` and `API_TOKEN` from environment variables or a secrets manager instead of hardcoding them.

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

### Checking the Last Jenkins Build Status

```bash
#!/bin/bash

JENKINS_URL="http://localhost:8080"
JOB_NAME="MyJob"
USER="${JENKINS_USER:?Set JENKINS_USER env var}"
API_TOKEN="${JENKINS_API_TOKEN:?Set JENKINS_API_TOKEN env var}"

echo "Fetching status of last build..."
curl -s "$JENKINS_URL/job/$JOB_NAME/lastBuild/api/json" --user "$USER:$API_TOKEN" | jq -r '.result'
```

### Challenges

1. Modify the workspace initializer so the number of dummy files and the workspace name are passed in as arguments instead of hardcoded.
2. Extend the smart file mover to accept a file extension as an argument instead of only moving `.txt` files.
3. Change the remote backup script to also verify the archive's checksum after transfer, so you can confirm it wasn't corrupted in transit.
4. Modify the system health monitor to send an email or Slack notification when a threshold is breached, instead of only printing to the terminal.
5. Rewrite the bulk user creation script to generate a unique random password per user instead of reusing the same temporary password.
6. Add a dry-run mode to the bulk user deletion script that lists which accounts would be deleted without actually removing them.
7. Extend the log auditor to also block the offending IP addresses using `iptables` or `ufw` once they're written to `blacklist.txt`.
8. Modify the service health check script to monitor a list of services instead of a single hardcoded one.
9. Update the Kubernetes pod restart script to restart pods one at a time with a delay, instead of force-deleting all of them at once.
10. Rewrite the Jenkins trigger script to poll the build status after triggering it and print the final result (success or failure) instead of exiting immediately.

## 📝 Document Info

| Field | Value |
|---|---|
| **Author** | Kartik Hegadi |
| **Organization** | InfraCorps |
| **Repository Topic** | Basic Scripting |
| **Audience** | Learners |
