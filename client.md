# 💻 Client Setup
> Part of the [NFS File Server](README.md) project.

---

### 1. Update the system

```bash
dnf update -y
```

### 2. Install required packages

```bash
dnf install -y nfs-utils rsync vim acl autofs
```

### 3. Create mount points

The `/srv/nfs/` directory is the standard location for data served over the network.
Regular users do not need to know about or access `/srv/nfs/` directly — autofs handles mounting transparently.

```bash
mkdir -p /srv/nfs/stock /srv/nfs/finance
```

### 4. Test manual mount

Before configuring autofs, confirm the server is reachable and exporting correctly:

```bash
showmount -e 192.168.122.X
```

Expected output:
```
Export list for 192.168.122.X:
/mnt/accounting 192.168.122.Y/24
/mnt/warehouse  192.168.122.Y/24
```

### 5. Create users and groups

```bash
# Admin user
useradd administrator
passwd administrator
usermod -aG wheel administrator

# Department users
useradd ana
passwd ana

useradd eli
passwd eli
```

```bash
# Create department groups
groupadd accounting
groupadd warehouse
```

```bash
# Assign users to their respective groups
usermod -aG accounting ana
usermod -aG warehouse  eli
```

### 6. Disable root SSH login

```bash
vim /etc/ssh/sshd_config.d/01-permitrootlogin.conf
```

Set the following line:

```
PermitRootLogin no
```

Then restart the SSH service:

```bash
systemctl restart sshd
```

> **⚠️ Important:** Before logging out, open a second terminal and confirm that the `administrator` user can connect via SSH. Always verify access before closing the current session.

> From this point on, all administration must be performed using the `administrator` user with sudo.

### 7. Configure autofs for on-demand mounting

Autofs mounts the NFS directories automatically when they are accessed and unmounts them after a period of inactivity. This is more efficient than static `/etc/fstab` entries.

Enable and start the autofs service:

```bash
systemctl enable --now autofs
systemctl status autofs
```

Edit the master autofs configuration file:

```bash
vim /etc/auto.master
```

Add the following line at the end of the file:

```
/srv/nfs   /etc/auto.share   --timeout=60
```

> **📌 Note:** `/srv/nfs` is the base directory. The `--timeout=60` option unmounts the share automatically after 60 seconds of inactivity.

Create the map file that defines what to mount inside `/srv/nfs`:

```bash
vim /etc/auto.share
```

Add the following lines:

```
stock    -fstype=nfs,rw,noatime,vers=3   192.168.122.X:/mnt/warehouse
finance  -fstype=nfs,rw,noatime,vers=3   192.168.122.X:/mnt/accounting
```

> **📌 Note on `vers=3`:** NFSv4 does not support the `acl` export option in the same way as NFSv3. Forcing NFSv3 here ensures that `setfacl` works correctly on the client side over the NFS mount. Without this option, `setfacl` returns "Operation not supported".

Restart autofs to apply the changes:

```bash
systemctl restart autofs
```

Test the on-demand mount by accessing the directories:

```bash
ls /srv/nfs/stock
ls /srv/nfs/finance

# Confirm the mounts are active
mount | grep nfs
```

### 8. Enable autofs logging

Enable verbose logging in the autofs configuration:

```bash
vim /etc/sysconfig/autofs
```

Find and set the following line, if it does not exist, you creator:

```
LOGGING="verbose"
```

> **📌 Available log levels:** `none` (default, no logging), `verbose` (mount and unmount events), `debug` (full detail — use only for troubleshooting).

Create a dedicated rsyslog rule to redirect autofs logs to a separate file:

```bash
vim /etc/rsyslog.d/autofs.conf
```

Add the following lines:

```
:programname, isequal, "automount" /var/log/autofs.log
& stop
```

Restart both services to apply the changes:

```bash
systemctl restart autofs
systemctl restart rsyslog
```

Monitor the log in real time:

```bash
tail -f /var/log/autofs.log
```

Expected log output when accessing a mount point:
```
automount[PID]: attempting to mount entry /srv/nfs/stock
automount[PID]: mounted /srv/nfs/stock
```

After 60 seconds of inactivity:
```
automount[PID]: expiring path /srv/nfs/stock
automount[PID]: umounting /srv/nfs/stock
```

### 9. Set directory permissions and ownership on mount points

Since `no_root_squash` is enabled on the server, `chown` operations from the client are allowed over the NFS mount.

Access the directories first to trigger autofs mounting:

```bash
ls /srv/nfs/stock
ls /srv/nfs/finance
```

Then apply permissions:

```bash
chmod 770 /srv/nfs/finance/
chmod 770 /srv/nfs/stock/

chown root:accounting /srv/nfs/finance/
chown root:warehouse  /srv/nfs/stock/
```

> **📌 Note:** After applying these permissions, `ana` (accounting group) can only access `/srv/nfs/finance/`, and `eli` (warehouse group) can only access `/srv/nfs/stock/`. This was tested and confirmed in the lab.

### 10. Configure ACL for fine-grained access control

ACL allows granting specific permissions to individual users beyond what standard group permissions provide.

Grant the `administrator` user full access to both directories:

```bash
setfacl -m u:administrator:rwx /srv/nfs/finance/
setfacl -m u:administrator:rwx /srv/nfs/stock/
```

Grant department users access to their respective directories:

```bash
setfacl -m u:ana:rwx /srv/nfs/finance/
setfacl -m u:eli:rwx /srv/nfs/stock/
```

Apply default ACL so that new files and subdirectories inherit the same rules:

```bash
setfacl -d -m u:administrator:rwx /srv/nfs/finance/
setfacl -d -m u:administrator:rwx /srv/nfs/stock/

setfacl -d -m u:ana:rwx /srv/nfs/finance/
setfacl -d -m u:eli:rwx /srv/nfs/stock/
```

> **📌 Note on the `-d` flag:** Without `-d`, new files and subdirectories created inside the directory do not inherit the ACL rules. The `-d` flag sets a default ACL that is automatically applied to all new content.

Verify the applied ACL:

```bash
getfacl /srv/nfs/finance/
getfacl /srv/nfs/stock/
```

### 11. Configure local backup with rsync and cron

The backup runs on the client and copies data from the NFS mount points to a local directory.
This ensures that files remain available locally even if the NFS server goes offline.

Create the backup directory structure:

```bash
mkdir -p /backup/nfs/stock
mkdir -p /backup/nfs/finance
chmod 700 /backup/nfs
chown root:root /backup/nfs
```

Create and configure the backup script (see [Scripts](scripts/script.md) for the full script):

```bash
vim /usr/local/bin/nfs_backup.sh
chmod +x /usr/local/bin/nfs_backup.sh
```

Schedule the backup with cron to run every day at 07:00 and 12:00:

```bash
vim /etc/crontab
```

Add the following line:

```
0 7,12 * * * root /usr/local/bin/nfs_backup.sh
```

> **📌 Note on the cron syntax:** `0 7,12 * * *` means "at minute 0 of hours 7 and 12, every day".

> **📌 Why backup on the client?** The NFS server holds the primary copy of the data. The client backup in `/backup/nfs/` is a completely independent local copy. If the server fails or a file is accidentally deleted, the client still has a recent copy available — which is the entire purpose of this setup.
