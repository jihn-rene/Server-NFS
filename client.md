# 💻 Client Setup
> Part of the [NFS File Server](README.md) project.

---

### 1. Update the system

```bash
dnf update -y
```

### 2. Install required packages

```bash
dnf install -y nfs-utils rsync vim
```

### 3. Create mount points

The `/srv/nfs/` directory is the standard location for data served over the network.
Regular users do not need to know about or access `/srv/nfs/` directly.

```bash
mkdir -p /srv/nfs/stock /srv/nfs/finance
```

### 4. Test manual mount

Before editing fstab, confirm the server is reachable and exporting correctly:

```bash
showmount -e 192.168.122.X
```

Expected output:
```
Export list for 192.168.122.X:
/mnt/accounting 192.168.122.Y/24
/mnt/warehouse  192.168.122.Y/24
```

Then mount manually to verify:

```bash
mount -t nfs 192.168.122.X:/mnt/warehouse  /srv/nfs/stock
mount -t nfs 192.168.122.X:/mnt/accounting /srv/nfs/finance

df -h | grep srv
```

### 5. Configure automatic mount on boot

```bash
vim /etc/fstab
```

Add the following lines at the end of the file:

```
192.168.122.X:/mnt/warehouse  /srv/nfs/stock   nfs  vers=3,defaults,_netdev  0  0
192.168.122.X:/mnt/accounting /srv/nfs/finance  nfs  vers=3,defaults,_netdev  0  0
```

> **📌 Note on `vers=3`:** NFSv4 does not support the `acl` export option in the same way as NFSv3. Forcing NFSv3 here ensures that `setfacl` works correctly on the client side over the NFS mount.

> **📌 Note on `_netdev`:** This option tells the system to wait for the network to be available before attempting to mount NFS volumes. Without it, the system may fail to boot if the NFS server is unreachable.

```bash
systemctl daemon-reload
mount -a
df -h
```

### 6. Test write access

```bash
echo "Testing stock directory"   > /srv/nfs/stock/test.txt
echo "Testing finance directory" > /srv/nfs/finance/test.txt

# Verify on the server that the files were created:
ls -la /mnt/warehouse/
ls -la /mnt/accounting/
```

### 7. Create users and groups

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

### 8. Disable root SSH login

Rocky Linux may have a drop-in configuration file that overrides `sshd_config`. Check and edit it directly:

```bash
vim /etc/ssh/sshd.d/01-permitrootlogin.conf
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

### 9. Set directory permissions and ownership on mount points

Since `no_root_squash` is enabled on the server, `chown` operations from the client are allowed over the NFS mount:

```bash
chmod 770 /srv/nfs/finance/
chmod 770 /srv/nfs/stock/

chown root:accounting /srv/nfs/finance/
chown root:warehouse  /srv/nfs/stock/
```

> **📌 Note:** After applying these permissions, `ana` (accounting group) cannot access `/srv/nfs/stock/`, and `eli` (warehouse group) cannot access `/srv/nfs/finance/`. This was tested and confirmed in the lab.

### 10. Configure ACL for fine-grained access control

ACL allows granting specific permissions to individual users beyond what standard group permissions allow.
In this case, the `administrator` user is granted full access to both directories:

```bash
setfacl -m u:administrator:rwx /srv/nfs/finance/
setfacl -m u:administrator:rwx /srv/nfs/stock/
```

Apply default ACL so that new files and directories created inside inherit the same rules:

```bash
setfacl -d -m u:administrator:rwx /srv/nfs/finance/
setfacl -d -m u:administrator:rwx /srv/nfs/stock/
```

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

Schedule the backup with cron to run every day at 7:00 and 12:00:

```bash
vim /etc/crontab
```

Add the following lines:

```
0 7,12 * * * root /usr/local/bin/nfs_backup.sh
```

> **📌 Note on the cron syntax:** `0 7,12 * * *` means "at minute 0 of hours 7 and 12, every day".

> **📌 Why backup on the client?** The NFS server holds the primary copy of the data. The client backup in `/backup/nfs/` is an independent local copy. If the server fails or a file is accidentally deleted, the client still has a recent copy available — which is the entire purpose of this setup.