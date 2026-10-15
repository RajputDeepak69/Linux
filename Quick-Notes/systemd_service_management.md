# systemd: `systemctl` — Reference Guide & Cheat Sheet

A comprehensive, production-oriented reference guide for managing system services using `systemctl` and querying/troubleshooting system and application logs using `journalctl`.

---

## Overview

`systemd` is the standard init system and service manager across modern Linux distributions (Debian, Ubuntu, RHEL, CentOS, Rocky Linux, Fedora, Arch Linux).

```text
               Kernel Boot
                    │
                    ▼
           systemd (PID 1)
                    │
       ┌────────────┴────────────┐
       ▼                         ▼
   systemctl                 journald
(Service & Target        (Centralized Binary
   Management)              Log Storage)
                                 │
                                 ▼
                             journalctl
                        (Query & Analysis)
```

- **`systemctl`**: Command-line control plane to inspect, start, stop, enable, mask, and debug units (services, timers, mounts, sockets, targets).
- **`systemd-journald` / `journalctl`**: Centralized, structured, indexed logging daemon and query tool capturing kernel, service, boot, and syslog messages.

---

## Service Management with `systemctl`

### 1. Basic Service Lifecycle

| Command | Action |
|---|---|
| `systemctl start <service>` | Start an inactive service immediately |
| `systemctl stop <service>` | Stop a running service immediately |
| `systemctl restart <service>` | Stop and immediately start a service |
| `systemctl reload <service>` | Reload service configuration without dropping connections |
| `systemctl reload-or-restart <service>` | Reload config if supported; restart if reload is unsupported |

```bash
# Example: Managing Nginx
sudo systemctl start nginx
sudo systemctl reload nginx
sudo systemctl stop nginx
```

---

### 2. Boot Behavior: Enable, Disable, and Mask

- **Enabled vs Started:** `enable` sets the service to start automatically at boot (creates symlinks in `/etc/systemd/system/`). It does **not** start the service immediately unless `--now` is passed.
- **Masking:** Creates a symlink pointing to `/dev/null`, preventing the service from being started manually or by other dependency chains.

```bash
# Enable at boot
sudo systemctl enable nginx

# Enable and start immediately
sudo systemctl enable --now nginx

# Disable from boot (removes symlink)
sudo systemctl disable nginx

# Disable and stop immediately
sudo systemctl disable --now nginx

# Mask a service (cannot be started by any user or dependency)
sudo systemctl mask nginx

# Unmask a service
sudo systemctl unmask nginx
```

---

### 3. Inspecting Service State & Configuration

```bash
# Detailed runtime status, exit code, recent journal logs, and PID
systemctl status nginx

# Check active status only (useful in shell scripts; exit status 0 if active)
systemctl is-active nginx

# Check boot-enablement status (exit status 0 if enabled)
systemctl is-enabled nginx

# Check if a service has entered a failed state
systemctl is-failed nginx

# View the full unit configuration file
systemctl cat nginx

# View all low-level runtime properties (CGroup, MemoryCurrent, ExecMainPID, etc.)
systemctl show nginx

# View specific property
systemctl show -p ActiveState,SubState,MainPID nginx
```

---

### 4. Listing Units & Targets

```bash
# List all active units
systemctl list-units

# List all running service units
systemctl list-units --type=service --state=running

# List all services that failed during runtime or boot
systemctl list-units --type=service --state=failed

# List all unit files and their enablement status (enabled, disabled, static, masked)
systemctl list-unit-files --type=service

# List active timer units (systemd crons)
systemctl list-timers
```

---

### 5. Writing a Custom `systemd` Service

Custom service files belong in `/etc/systemd/system/<service-name>.service`.

#### Complete Production Template: `/etc/systemd/system/myapp.service`

```ini
[Unit]
Description=My Go/Python Backend Application
Documentation=https://docs.example.com
After=network.target remote-fs.target
Wants=network-online.target

[Service]
Type=simple
User=appuser
Group=appgroup
WorkingDirectory=/opt/myapp

ExecStart=/opt/myapp/bin/server --config /opt/myapp/config.yaml
ExecReload=/bin/kill -HUP $MAINPID

# Restart policy: automatically recover on failure
Restart=on-failure
RestartSec=5s

# Resource limits & safety
LimitNOFILE=65535
TimeoutStopSec=30s

# Sandboxing & security hardening
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
NoNewPrivileges=true
ReadWritePaths=/opt/myapp/data /var/log/myapp

# Environment variables
Environment="APP_ENV=production"
EnvironmentFile=-/etc/default/myapp

[Install]
WantedBy=multi-user.target
```

#### Service Lifecycle Workflow

Whenever you create or modify a `.service` file:

```bash
# 1. Reload systemd manager configuration (reads new/updated unit files)
sudo systemctl daemon-reload

# 2. Enable and start the service
sudo systemctl enable --now myapp.service

# 3. Verify health
systemctl status myapp.service
```

---