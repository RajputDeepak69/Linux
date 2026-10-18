# SSH & SSH Tunneling — Comprehensive Reference Guide

A complete guide to modern SSH key management, configuration tuning, port forwarding, and bastion jump host architectures.

---

## 1. Modern Key Generation

RSA keys are legacy; modern deployments prioritize `Ed25519` for superior security, performance, and compact key lengths.

### Generate an Ed25519 Key (Recommended)

```bash
ssh-keygen -t ed25519 -C "admin@company.internal"
```

* `-t ed25519`: Specifies the Ed25519 algorithm.
* `-C "..."`: Adds a human-readable comment (email, device name, or purpose).

### Generate a Strong RSA Key (Fallback Only)

If legacy devices do not support Ed25519, generate a 4096-bit RSA key:

```bash
ssh-keygen -t rsa -b 4096 -C "admin@company.internal"
```

### Install Public Key to Remote Server

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@remote-server.com
```

Or manually via pipe if `ssh-copy-id` is unavailable:

```bash
cat ~/.ssh/id_ed25519.pub | ssh user@remote-server.com "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

### File Permissions Checklist

SSH will refuse connections if permissions on key files are too permissive:

| File / Directory | Recommended Permission | Command |
|---|---|---|
| `~/.ssh` directory | `700` (`drwx------`) | `chmod 700 ~/.ssh` |
| Private keys (`id_ed25519`) | `600` (`-rw-------`) | `chmod 600 ~/.ssh/id_ed25519` |
| Public keys (`id_ed25519.pub`) | `644` (`-rw-r--r--`) | `chmod 644 ~/.ssh/id_ed25519.pub` |
| `authorized_keys` | `600` (`-rw-------`) | `chmod 600 ~/.ssh/authorized_keys` |
| `config` | `600` (`-rw-------`) | `chmod 600 ~/.ssh/config` |

---

