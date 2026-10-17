# `journalctl` — Reference Guide & Cheat Sheet

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

## Log Analysis with `journalctl`

`journald` stores logs in binary format, allowing rapid querying by metadata (unit, PID, timestamp, priority level) without manual text parsing.

### 1. Real-Time Streaming & Tail

```bash
# Follow live logs across the entire system (equivalent to tail -f /var/log/syslog)
journalctl -f

# Follow logs for a specific service
journalctl -u nginx -f

# Show last 50 lines and follow live
journalctl -u nginx -n 50 -f

# Do not pipe to pager (prints raw text directly to terminal)
journalctl -u nginx -n 20 --no-pager
```

---

### 2. Time-Based Filtering

`journalctl` supports human-readable relative intervals and absolute ISO timestamps.

```bash
# Logs since yesterday
journalctl --since "yesterday"

# Logs from the last 15 minutes
journalctl --since "15m ago"

# Logs for a specific service within an exact timeframe
journalctl -u nginx --since "2026-10-15 08:00:00" --until "2026-10-15 12:30:00"

# Logs starting from a specific time up to current moment
journalctl -u myapp --since "1 hour ago"
```

---

### 3. Boot-Specific Logs

```bash
# List all recorded system boots
journalctl --list-boots

# View logs from the current boot (boot 0)
journalctl -b

# View logs from the previous boot (boot -1) — invaluable after sudden crashes/reboots
journalctl -b -1

# View logs for a specific service during the previous boot
journalctl -u myapp -b -1
```

---

### 4. Priority / Log-Level Filtering (`-p`)

Filter by standard Syslog severity levels:

| Priority | Level | Description |
|---|---|---|
| `emerg` | `0` | System is unusable |
| `alert` | `1` | Action must be taken immediately |
| `crit` | `2` | Critical conditions |
| `err` | `3` | Error conditions |
| `warning` | `4` | Warning conditions |
| `notice` | `5` | Normal but significant conditions |
| `info` | `6` | Informational messages |
| `debug` | `7` | Debug-level messages |

```bash
# View errors and higher (emergencies, alerts, criticals, errors)
journalctl -p err

# View errors for a specific service since today
journalctl -u nginx -p err --since today

# View warnings and errors continuously
journalctl -u myapp -p warning..err -f
```

---

### 5. Advanced Metadata Filtering

`systemd` tags incoming log entries with internal system fields:

```bash
# Filter by Process ID (PID)
journalctl _PID=14231

# Filter by user ID (e.g., UID 1000)
journalctl _UID=1000

# Filter by binary path
journalctl /usr/sbin/nginx

# View kernel ring buffer logs (similar to dmesg)
journalctl -k
# or
journalctl _TRANSPORT=kernel

# View logs in JSON format for parsing with jq
journalctl -u nginx -n 5 -o json-pretty
```

---

### 6. Disk Usage & Journal Maintenance

```bash
# Check current disk space consumed by journal logs
journalctl --disk-usage

# Verify journal file integrity
journalctl --verify

# Delete journal logs older than 7 days
sudo journalctl --vacuum-time=7d

# Reduce journal disk footprint to 500 Megabytes
sudo journalctl --vacuum-size=500M
```

---

## Debugging Workflow: Recovering a Failed Service

```text
Step 1: Check status
$ systemctl status <service>
        │
        ▼
Step 2: Inspect failure state
$ systemctl is-failed <service>
        │
        ▼
Step 3: Extract recent error logs
$ journalctl -u <service> -e -n 50 --no-pager
        │
        ▼
Step 4: Check configuration syntax / edit unit
$ sudo nano /etc/systemd/system/<service>.service
        │
        ▼
Step 5: Reload daemon & restart
$ sudo systemctl daemon-reload
$ sudo systemctl restart <service>
        │
        ▼
Step 6: Follow live logs to verify stabilization
$ journalctl -u <service> -f
```

---

## Quick Reference / Cheat Sheet

| Task | Command |
|---|---|
| **Start / Stop** | `systemctl start\|stop <unit>` |
| **Restart / Reload** | `systemctl restart\|reload <unit>` |
| **Enable at boot + start now** | `systemctl enable --now <unit>` |
| **Disable from boot** | `systemctl disable <unit>` |
| **Lock service against starts** | `systemctl mask <unit>` |
| **List failed services** | `systemctl --failed` |
| **Reload systemd after edit** | `systemctl daemon-reload` |
| **Tail service logs live** | `journalctl -u <unit> -f` |
| **Show last 100 service logs** | `journalctl -u <unit> -n 100 --no-pager` |
| **Logs from last boot** | `journalctl -b -1` |
| **Errors only since 1 hour ago** | `journalctl -p err --since "1h ago"` |
| **Check log disk usage** | `journalctl --disk-usage` |
| **Vacuum logs older than 14 days** | `sudo journalctl --vacuum-time=14d` |