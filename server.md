# 🖥️ Server Setup
> Part of the [NFS File Server](README.md) project.

---

### 1. Update the system

```bash
dnf update -y
```

### 2. Install required packages

```bash
dnf install -y nfs-utils vim
```

### 3. Enable and start the NFS service

```bash
systemctl enable --now nfs-server
systemctl status nfs-server
```

### 4. Create the shared directories

```bash
mkdir -p /mnt/warehouse /mnt/accounting
```

### 5. Set permissions and ownership

```bash
chmod 755 /mnt/warehouse
chmod 755 /mnt/accounting

chown nobody:nobody /mnt/warehouse
chown nobody:nobody /mnt/accounting
```

### 6. Configure the exports file

```bash
vim /etc/exports
```

Add the following lines (replace the IP with your actual client IP):

```
/mnt/warehouse  192.168.122.Y/24(rw,sync,root_squash,no_subtree_check)
/mnt/accounting 192.168.122.Y/24(rw,sync,root_squash,no_subtree_check)
```

> **📌 Note on `root_squash`:** This option maps root requests from the client to the `nobody` user on the server, preventing a client with root access from compromising the server's files. This is the recommended setting for production environments.

### 7. Apply the exports

```bash
exportfs -arv
exportfs -s
```

### 8. Configure the firewall

```bash
firewall-cmd --permanent --add-service=nfs
firewall-cmd --permanent --add-service=mountd
firewall-cmd --permanent --add-service=rpc-bind
firewall-cmd --reload
firewall-cmd --list-services
```

### 9. Create an admin user and group

```bash
groupadd developers
useradd sysadmin
passwd sysadmin
usermod -aG developers sysadmin
usermod -aG wheel sysadmin
```

### 10. Disable root SSH login

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

> **⚠️ Important:** Before logging out, open a second terminal and confirm that the `sysadmin` user can log in via SSH. Locking yourself out is a classic mistake — don't be that person.

> From this point on, all configuration must be done using the `sysadmin` user.