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

```bash
mkdir -p /mnt/stock /mnt/finance
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
mount -t nfs 192.168.122.X:/mnt/warehouse  /mnt/stock
mount -t nfs 192.168.122.X:/mnt/accounting /mnt/finance

df -h | grep mnt
```

### 5. Configure automatic mount on boot

```bash
vim /etc/fstab
```

Add the following lines at the end of the file:

```
192.168.122.X:/mnt/warehouse  /mnt/stock   nfs  defaults,_netdev  0  0
192.168.122.X:/mnt/accounting /mnt/finance  nfs  defaults,_netdev  0  0
```

> **📌 Note on `_netdev`:** This option tells the system to wait for the network to be available before attempting to mount NFS volumes. Without it, the system may fail to boot if the NFS server is unreachable.

```bash
systemctl daemon-reload
mount -a
df -h
```

### 6. Test write access

```bash
echo "Testing stock directory"   > /mnt/stock/test.txt
echo "Testing finance directory" > /mnt/finance/test.txt
```

### 7. Create users and groups

```bash
# Admin user
useradd administrator
passwd administrator

# Department users
useradd ana
passwd ana

useradd eli
passwd eli
```

```bash
# Create department groups
groupadd finance
groupadd warehouse
```

```bash
# Assign users to their groups
usermod -aG wheel       administrator
usermod -aG finance     ana
usermod -aG warehouse   eli
```

### 8. Disable root SSH login

```bash
vim /etc/ssh/sshd_config
```

Find and set the following line:

```
PermitRootLogin no
```

Then restart the SSH service:

```bash
systemctl restart sshd
```

> **⚠️ Important:** Before logging out, open a second terminal and confirm that the `administrator` user can log in via SSH. Always verify access before closing the current session.

> From this point on, all configuration must be done using the `administrator` user.