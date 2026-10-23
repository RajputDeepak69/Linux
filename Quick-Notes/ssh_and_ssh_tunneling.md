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

## 2. `~/.ssh/config` Configuration & Tuning

Simplify complex connections into single-word aliases. Edit or create `~/.ssh/config`:

```ssh-config
# Global Defaults applied to all hosts
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
    IdentitiesOnly yes
    AddKeysToAgent yes
    Compression yes

# Direct host configuration
Host prod-db
    HostName 10.0.4.15
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519
    Port 2222

# Bastion / Jump Host definition
Host bastion
    HostName bastion.company.com
    User ops
    IdentityFile ~/.ssh/id_bastion_ed25519

# Target host accessed through the Bastion
Host internal-app
    HostName 172.16.10.50
    User dev
    IdentityFile ~/.ssh/id_ed25519
    ProxyJump bastion
```

With this configuration:
* Running `ssh internal-app` automatically routes your traffic through `bastion` to reach `172.16.10.50`.
* Connection drops are prevented by sending keep-alive probes every 60 seconds (`ServerAliveInterval 60`).

---

## 3. SSH Tunneling & Port Forwarding

SSH tunneling encapsulates arbitrary TCP traffic inside an encrypted SSH session.

---

### Local Port Forwarding (`-L`)

**Use case:** Connect from your local machine to a remote service that is only reachable from the remote SSH server (e.g., reaching a private database behind a firewall).

```text
Local Machine                          Remote SSH Server                 Private Resource
127.0.0.1:5433  ════[Encrypted SSH]═══▶  jump.company.com  ────[Plain TCP]────▶  db-private:5432
```

```bash
# Syntax: ssh -L [local_port]:[destination_host]:[destination_port] [ssh_user]@[ssh_server]
ssh -L 5433:db-private.internal:5432 user@jump.company.com -N
```

* `-N`: Do not execute a remote shell (useful for forwarding ports only).
* You can now connect to `127.0.0.1:5433` on your workstation, and traffic is forwarded to `db-private.internal:5432`.

---

### Remote Port Forwarding (`-R`)

**Use case:** Expose a local service running on your machine to a remote server or teammates (e.g., sharing a local dev web server running on port 3000).

```text
Local Machine                          Remote Public Server              External Client
127.0.0.1:3000  ◀═══[Encrypted SSH]════  public-server.com:8080  ◀────────────  Web Browser
```

```bash
# Syntax: ssh -R [remote_port]:[local_host]:[local_port] [ssh_user]@[ssh_server]
ssh -R 8080:127.0.0.1:3000 user@public-server.com -N
```

> **Note:** To allow outside clients to connect to `public-server.com:8080`, the remote SSH server must have `GatewayPorts yes` configured in `/etc/ssh/sshd_config`.

---

### Dynamic Port Forwarding (`-D` SOCKS5 Proxy)

**Use case:** Route all browser or application traffic dynamically through an encrypted SSH server without configuring ports individually.

```text
Local Browser                        Local SOCKS Proxy                 Target Websites
Any Destination  ────[SOCKS5]──────▶  127.0.0.1:1080  ═══[SSH]═══▶  remote-server  ───▶  Web
```

```bash
ssh -D 1080 user@remote-proxy-server.com -N
```

Configure your browser or CLI tool to use SOCKS5 proxy at `127.0.0.1:1080`:

```bash
curl --socks5 127.0.0.1:1080 https://ifconfig.me
```

---

## 4. Bastion Hosts & ProxyJump

### Single Jump Host

```bash
# CLI flag syntax
ssh -J user@bastion.example.com target-user@10.0.1.20
```

### Multi-Hop Jump (Chained Bastions)

```bash
ssh -J user1@bastion1.com,user2@bastion2.com target-user@10.0.2.100
```

### In `~/.ssh/config`:

```ssh-config
Host bastion1
    HostName bastion1.example.com
    User ops

Host bastion2
    HostName 10.0.1.5
    User ops
    ProxyJump bastion1

Host target-node
    HostName 192.168.10.25
    User app
    ProxyJump bastion2
```

Connecting to `ssh target-node` will automatically traverse `bastion1` → `bastion2` → `target-node`.

---

## 5. SSH Agent & Key Management

Avoid typing your private key passphrase repeatedly during active sessions.

```bash
# Start the SSH agent in the current shell
eval "$(ssh-agent -s)"

# Add default private key
ssh-add ~/.ssh/id_ed25519

# List keys currently held by the agent
ssh-add -l

# Remove all keys from the agent
ssh-add -D
```

### Agent Forwarding (`-A`) vs ProxyJump

* **Avoid Agent Forwarding (`ssh -A`)** unless strictly necessary. If the bastion server is compromised, root users on that machine can impersonate your active SSH agent to pivot.
* **Prefer `ProxyJump` (`-J` or `ProxyJump` in config)**: It tunnels raw encrypted bytes through the bastion directly to the target; the bastion never sees or interacts with your keys or credentials.

---

## 6. Production Hardening Checklist (`/etc/ssh/sshd_config`)

To harden target servers against automated attacks and brute force:

```ini
# /etc/ssh/sshd_config

# Disable password-based logins completely
PasswordAuthentication no
PermitEmptyPasswords no

# Disable root login over SSH
PermitRootLogin no

# Enforce public-key authentication
PubkeyAuthentication yes

# Limit authentication attempts
MaxAuthTries 3

# Close stale / abandoned connections
ClientAliveInterval 300
ClientAliveCountMax 2

# Restrict allowed users/groups (optional)
# AllowUsers ops-admin deployer
```

Apply changes after testing configuration syntax:

```bash
sudo sshd -t && sudo systemctl reload sshd
```

---

## 7. Quick Reference / Cheat Sheet

| Task | Command |
|---|---|
| Generate modern Ed25519 key | `ssh-keygen -t ed25519 -C "name"` |
| Copy key to remote server | `ssh-copy-id user@host` |
| Local port forward | `ssh -L 8080:localhost:80 user@remote -N` |
| Remote port forward | `ssh -R 9000:localhost:3000 user@remote -N` |
| Dynamic SOCKS5 proxy | `ssh -D 1080 user@remote -N` |
| Jump host connection | `ssh -J user@bastion user@target` |
| Run background tunnel | `ssh -f -N -L 5432:db:5432 user@remote` |
| Test sshd configuration syntax | `sudo sshd -t` |