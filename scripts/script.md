# 📜 Scripts

This directory contains utility scripts for the NFS File Server project.
Copy the desired script to the target machine and follow the instructions below.

Create the backup script:

```bash
vim /usr/local/bin/nfs_backup.sh
```

```bash
#!/bin/bash
# NFS local backup script
# Runs on the client — copies NFS mount points to local storage

DATE=$(date +%Y-%m-%d)
LOG="/var/log/nfs_backup.log"

# Verify that NFS volumes are mounted before proceeding
if ! mountpoint -q /srv/nfs/stock; then
    echo "[$DATE] ERROR: /srv/nfs/stock is not mounted. Backup aborted." >> "$LOG"
    exit 1
fi

if ! mountpoint -q /srv/nfs/finance; then
    echo "[$DATE] ERROR: /srv/nfs/finance is not mounted. Backup aborted." >> "$LOG"
    exit 1
fi

rsync -av --delete /srv/nfs/stock/   /backup/nfs/stock/   >> "$LOG" 2>&1
rsync -av --delete /srv/nfs/finance/ /backup/nfs/finance/  >> "$LOG" 2>&1

echo "[$DATE] Backup completed successfully." >> "$LOG"
```

Make the script executable:

```bash
chmod +x /usr/local/bin/nfs_backup.sh
```