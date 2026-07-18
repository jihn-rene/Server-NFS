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

### 4. Create department groups and shared directories

Create a dedicated group for each department and a corresponding shared directory:

```bash
groupadd warehouse
groupadd accounting
```

```bash
mkdir -p /mnt/warehouse /mnt/accounting
```

### 5. Set permissions and ownership

Assign each directory to its corresponding group and restrict access to group members only:

```bash
chmod 770 /mnt/warehouse
chmod 770 /mnt/accounting

chown root:warehouse /mnt/warehouse
chown root:accounting /mnt/accounting
```

> **📌 Note:** Using `root` as the owner and the department group as the group owner is the recommended approach. This prevents any single user from having full ownership of a shared directory.

### 6. Configure the exports file

```bash
vim /etc/exports
```

Add the following lines (replace the IP with your actual client IP):

```
/mnt/warehouse  192.168.122.Y/24(rw,sync,no_root_squash,no_subtree_check,acl)
/mnt/accounting 192.168.122.Y/24(rw,sync,no_root_squash,no_subtree_check,acl)
```

> **📌 Note on `no_root_squash`:** This option allows the client's root user to operate as root on the server. It is used here intentionally to enable `chown` and `setfacl` commands to work correctly over the NFS mount. In a production environment, this should be replaced with Kerberos-based authentication (NFSv4 + Kerberos), which handles user identity mapping securely without requiring `no_root_squash`.

> **📌 Note on `acl`:** The `acl` export option enables ACL support over the NFS share. This is required for `setfacl` to work on the client side when using NFSv3.

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

### 9. Create an admin user

```bash
useradd sysadmin
passwd sysadmin
usermod -aG wheel sysadmin
```

### 10. Disable root SSH login

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

> **⚠️ Important:** Before logging out, open a second terminal and confirm that the `sysadmin` user can connect via SSH. Locking yourself out of the server is a classic mistake — always verify access before closing the current session.

> From this point on, all administration must be performed using the `sysadmin` user with sudo.
