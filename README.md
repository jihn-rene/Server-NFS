# 🗄️ NFS File Server — Centralized Storage for Small Organizations

[![Rocky Linux](https://img.shields.io/badge/-Rocky%20Linux-%2310B981?style=for-the-badge&logo=rockylinux&logoColor=white)](https://rockylinux.org/)
[![NFS](https://img.shields.io/badge/NFS-File%20System-blue?style=for-the-badge&logo=linux&logoColor=white)](https://linux.die.net/man/5/exports)
[![rsync](https://img.shields.io/badge/rsync-Backup-orange?style=for-the-badge&logo=linux&logoColor=white)](https://rsync.samba.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=for-the-badge)](LICENSE)

> A practical Linux infrastructure project: NFS file server with automatic synchronization via rsync and cron, simulating a real small business environment.

---

## 📋 Table of Contents

- [About the Project](#-about-the-project)
- [Architecture](#-architecture)
- [Technologies](#-technologies)
- [Prerequisites](#-prerequisites)
- [📄 Server Setup](server.md)
- [📄 Client Setup](client.md)
- [📜 Scripts](scripts/script.md)
- [Security](#-security)
- [Troubleshooting](#-troubleshooting)
- [Roadmap](#-roadmap)
- [License](#-license)
- [Author](#-author)

---

## 📌 About the Project

This project implements an NFS file server to centralize directory storage for a small organization, with two isolated departments: **Warehouse** and **Accounting**.

The client automatically mounts the remote directories on demand via **autofs** and creates independent local backups via **rsync + cron**.

> **Important:** NFS is not a backup solution. It provides real-time shared access to files stored on the server. If a file is deleted on the client, it is immediately deleted on the server as well. The rsync + cron setup exists precisely to address this — it creates independent local copies on the client that survive server failures or accidental deletions.

**Use case:**
- Small organization with multiple Linux clients
- Centralized file sharing per department
- On-demand mounting via autofs (mount points are only active when accessed)
- Automatic local backup after client-side modifications

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────┐
│              NFS SERVER                 │
│           Rocky Linux 10.2             │
│           IP: 192.168.122.X            │
│                                         │
│  /mnt/warehouse    /mnt/accounting      │
│         │                   │           │
│    nfs-server          firewalld        │
│    (NFS, mountd,       (nfs, mountd,   │
│     rpc-bind)           rpc-bind)       │
└──────────────┬──────────────────────────┘
               │ NFSv3 over TCP/UDP
               │ Port 2049
┌──────────────▼──────────────────────────┐
│              NFS CLIENT                 │
│           Rocky Linux 10.2             │
│           IP: 192.168.122.Y              │
│                                         │
│  /srv/nfs/stock    /srv/nfs/finance     │
│  (← warehouse)     (← accounting)      │
│                                         │
│  autofs        → on-demand mount       │
│  rsync + cron  → local backup          │
│  /backup/nfs/  → backup destination    │
└─────────────────────────────────────────┘
```

---

## 🛠️ Technologies

| Technology | Version | Role |
|---|---|---|
| Rocky Linux | 10.2 | Operating system (server and client) |
| nfs-utils | 2.8.3 | NFS server and client |
| autofs | 5.1.9 | On-demand NFS mount management |
| rsync | 3.3.4 | Incremental file synchronization |
| crond | 1.7.0 | Task scheduling for backup |
| firewalld | 2.4.2 | Network access control |

---

## ✅ Prerequisites

- Two machines (physical or VMs) running Rocky Linux 10.2
- Network connectivity between server and client
- Root access or a user with sudo privileges on both machines

> **⚠️ Important:** Replace `192.168.122.X` and `192.168.122.Y` with the actual IP of your server, and update the client IP accordingly before running any command.

---

## 🔐 Security

| Item | Status | Notes |
|---|---|---|
| `no_root_squash` | ✅ Enabled | Required to allow chown and ACL operations over NFSv3 in this lab setup |
| Firewall | ✅ Configured | Only NFS-required services are allowed |
| Directory permissions | ✅ `770` | Only owner and group can read and write |
| IP-restricted access | ✅ | Exports restricted to client IP range |
| Root SSH login disabled | ✅ Done | `PermitRootLogin no` enforced via `/etc/ssh/sshd.d/01-permitrootlogin.conf` |
| Dedicated users per department | ✅ Done | Users and groups created and assigned |
| ACL per user on mount points | ✅ Done | setfacl applied on client for fine-grained access control |
| On-demand mounting via autofs | ✅ Done | Directories are only mounted when accessed |
| Autofs logging | ✅ Done | Verbose logging enabled via rsyslog |
| Local backup via rsync + cron | ✅ Done | Independent copy stored in `/backup/nfs/` on the client |
| Kerberos authentication (NFSv4) | 📋 Planned | For enterprise-grade environments |

> **📌 Note on `no_root_squash`:** In this project, `no_root_squash` is used intentionally to allow `chown` and `setfacl` commands to work correctly over the NFS mount. In a production environment, Kerberos-based authentication with NFSv4 is the recommended approach for proper user identity mapping without relying on `no_root_squash`.

---

## 🔧 Troubleshooting

**Issue: `mount.nfs: access denied by server`**
```bash
# Check active exports on the server
exportfs -s

# Make sure the client IP matches the one configured in /etc/exports
# Re-apply after any change:
exportfs -arv
```

**Issue: `showmount: clnt_create: RPC: Port mapper failure`**
```bash
# Check if the firewall is allowing rpc-bind
firewall-cmd --list-services

# Check if the NFS service is running
systemctl status nfs-server rpcbind
```

**Issue: Directories not mounting on boot**
```bash
# Confirm autofs is running
systemctl status autofs

# Check auto.master and auto.share for typos
cat /etc/auto.master
cat /etc/auto.share
```

**Issue: Files created on the client show up as `nobody` on the server**
```bash
# This happens when root_squash is active and idmapd is not configured.
# Enable the idmapd service to fix user identity mapping:
systemctl enable --now nfs-idmapd
```

**Issue: `setfacl` returns "Operation not supported"**
```bash
# This occurs when using NFSv4 or when vers=3 is missing in auto.share.
# Confirm that vers=3 is present in /etc/auto.share:
cat /etc/auto.share
# Expected: stock -fstype=nfs,rw,noatime,vers=3 192.168.10.50:/mnt/warehouse
# Then restart autofs:
systemctl restart autofs
```

**Issue: Root SSH login still works after setting `PermitRootLogin no`**
```bash
# Edit the correct file:
vim /etc/ssh/sshd_config.d/01-permitrootlogin.conf
# Set: PermitRootLogin no
systemctl restart sshd
```

---

## 🗺️ Roadmap

- [x] Base NFS server configuration
- [x] Firewall configured
- [x] Dedicated users and groups created per department
- [x] Root SSH login disabled on server and client
- [x] Directory permissions with chmod and chown
- [x] ACL configuration on client mount points
- [x] On-demand mounting via autofs (NFSv3)
- [x] Autofs logging via rsyslog
- [x] Local backup with rsync + cron
- [ ] NFSv4 with Kerberos authentication

---

## 📄 License

Distributed under the MIT License. See the [LICENSE](LICENSE) file for details.

---

## 👤 Author

**Matheus Ferreira da Silva**
- GitHub: [@jihn-rene](https://github.com/jihn-rene)
- LinkedIn: [linkedin.com/in/matheus-ferreira-954253295](https://www.linkedin.com/in/matheus-ferreira-954253295/)

---

> 💡 **About this project:** Developed as part of a practical Linux infrastructure portfolio, focused on sysadmin and DevOps. Every configuration was tested in a lab environment running Rocky Linux 10.2.
