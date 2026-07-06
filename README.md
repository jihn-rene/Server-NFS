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
- [Security](#-security)
- [Troubleshooting](#-troubleshooting)
- [Roadmap](#-roadmap)
- [License](#-license)
- [Author](#-author)

---

## 📌 About the Project

This project implements an NFS file server to centralize directory storage for a small organization, with two isolated departments: **Warehouse** and **Accounting**.

The client automatically mounts the remote directories at system boot and synchronizes changes back to the server via **rsync + cron**, acting as an incremental backup layer.

**Use case:**
- Small organization with multiple Linux clients
- Centralized file sharing per department
- Automatic backup after client-side modifications

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
               │ NFS over TCP/UDP
               │ Port 2049
┌──────────────▼──────────────────────────┐
│              NFS CLIENT                 │
│           Rocky Linux 10.2             │
│           IP: 192.168.122.Y            │
│                                         │
│  /mnt/stock      /mnt/finance           │
│  (← warehouse)   (← accounting)        │
│                                         │
│  /etc/fstab  → automatic mount         │
│  rsync + cron → synchronization        │
└─────────────────────────────────────────┘
```

---

## 🛠️ Technologies

| Technology | Version | Role |
|---|---|---|
| Rocky Linux | 10.2 | Operating system (server and client) |
| nfs-utils | 2.8.3 | NFS server and client |
| rsync | — | Incremental file synchronization |
| crond | — | Task scheduling for synchronization |
| firewalld | 2.4.2 | Network access control |

---

## ✅ Prerequisites

- Two machines (physical or VMs) running Rocky Linux 10.2
- Network connectivity between server and client
- Root access or a user with sudo privileges on both machines

> **⚠️ Important:** Replace `192.168.122.X` and `192.168.122.Y` with the actual IPs of your environment before running any command.

---

## 🔐 Security

| Item | Status | Notes |
|---|---|---|
| `root_squash` | ✅ Enabled | Client root mapped to `nobody` on the server |
| Firewall | ✅ Configured | Only NFS-required services are allowed |
| Directory permissions | ✅ `755` | No write access for others |
| IP-restricted access | ✅ | Exports restricted to client IP |
| Disable root login (SSH) | ✅ Done | `PermitRootLogin no` set in sshd_config |
| Dedicated users per department | ✅ Done | Users and groups created and assigned |
| chmod, chown and ACL on client | 🔄 In progress | Planned in roadmap |
| Kerberos authentication (NFSv4) | 📋 Planned | For enterprise environments |

---

## 🔧 Troubleshooting

**Issue: `mount.nfs: access denied by server`**
```bash
# Check active exports on the server
exportfs -s

# Make sure the client IP matches the one set in /etc/exports
# Re-apply after any change:
exportfs -arv
```

**Issue: `showmount: clnt_create: RPC: Port mapper failure`**
```bash
# Check if firewall is allowing rpc-bind
firewall-cmd --list-services

# Check if NFS service is running
systemctl status nfs-server rpcbind
```

**Issue: Directories not mounting on boot**
```bash
# Confirm _netdev is present in fstab
grep nfs /etc/fstab

# Test manually
mount -a -v
```

**Issue: Files created on the client show up as `nobody` on the server**
```bash
# This is expected behavior when root_squash is active.
# To properly map users, configure idmapd:
systemctl enable --now nfs-idmapd
```

---

## 🗺️ Roadmap

- [x] Base NFS server configuration
- [x] Automatic mounting via fstab on the client
- [x] Firewall configured
- [x] Dedicated users and groups created per department
- [x] Root login disabled on server and client
- [ ] chmod, chown and ACL configuration on client
- [ ] Automatic synchronization with rsync + cron
- [ ] autofs setup on client (on-demand mounting)
- [ ] NFS server availability monitoring
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