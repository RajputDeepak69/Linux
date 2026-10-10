# Linux Processes, Jobs and Threads

A **process** is a running program. A **thread** is an execution unit within a process. A **job** is a command or pipeline managed by the current shell.

---

## 1. Process vs Thread vs Job

| Term | Meaning |
|---|---|
| **Process** | Running program with its own virtual address space |
| **Thread** | Execution unit that shares resources within a process |
| **Job** | Command or pipeline tracked by the current shell |
| **PID** | Process ID |
| **TID** | Thread ID |
| **PPID** | Parent Process ID |

---

## 2. Process Discovery

```bash
ps aux              # List running processes
ps -ef              # Detailed process list
pgrep nginx         # Find process by name
pstree -p           # Display process hierarchy
top                 # Monitor processes in real time
```

---

## 3. Jobs vs Processes

| Command | Purpose |
|---|---|
| `ps` | Shows system processes |
| `jobs` | Shows jobs managed by the current shell |
| `jobs -l` | Shows jobs with their PIDs |
| `command &` | Runs a command in the background |

---

## 4. Job Control: bg, fg and Ctrl + Z

```bash
sleep 300 &     # Start a background job
jobs            # List shell jobs
fg %1           # Bring job 1 to foreground
Ctrl + Z        # Suspend the foreground job
bg %1           # Resume job 1 in background
wait %1         # Wait for job 1 to finish
```

### Key Shortcuts and Behaviors

- **`Ctrl + C`** sends an interrupt, usually `SIGINT`.
- **`Ctrl + Z`** suspends the foreground job; it does not terminate it.
- **`bg`** resumes a stopped job in the background.
- **`fg`** brings a job to the foreground.

---

## 5. Stop vs Kill

```bash
kill -STOP <PID>    # Stop process immediately
kill -CONT <PID>    # Resume stopped process
kill <PID>          # Request graceful termination (SIGTERM)
kill -9 <PID>       # Force termination (SIGKILL)
```

### Common Signals

| Signal | Number | Purpose |
|---|---|---|
| `SIGHUP` | 1 | Hangup; sometimes used to reload services |
| `SIGINT` | 2 | Interrupt (terminal break, `Ctrl + C`) |
| `SIGTERM` | 15 | Request graceful termination (allows cleanup) |
| `SIGCONT` | 18* | Resume a stopped process |
| `SIGSTOP` | 19* | Stop a process unconditionally |
| `SIGKILL` | 9 | Force termination (cannot be caught or ignored) |

> **Note:** Signal numbers marked with `*` (18 and 19) are architecture-dependent. Prefer `kill -CONT` and `kill -STOP` names for portability.
> 
> **Remember:** `SIGTERM` allows cleanup; `SIGKILL` cannot be caught or ignored.

---

## 6. Thread Discovery and Monitoring

```bash
ps -T -p <PID>       # List threads of a process
top -H -p <PID>      # Monitor individual threads
```

---

## 7. Useful Monitoring Tools

- **`ps`** — Process snapshot and details
- **`top` / `htop`** — Live interactive process monitoring
- **`pidstat`** — Process and thread resource statistics
- **`strace`** — Trace system calls and signals

---

## Key Takeaway

- **`ps`** discovers processes across the system.
- **`jobs`** manages jobs tracked by the current shell session.
- **`bg`** and **`fg`** control background and foreground execution.
- **`kill`** sends signals to processes.
- **`top -H`** monitors individual threads per process.