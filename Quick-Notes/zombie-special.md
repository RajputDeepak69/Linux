# Linux Zombie Processes

A reference guide to understanding, identifying, and resolving zombie (defunct) processes in Linux systems.

---

## Overview

A **zombie process** is a process that has completed execution but still retains an entry in the system process table because its parent process has not yet collected its exit status.

* Zombie processes do **not** run instructions or consume CPU time or active memory.
* Their process-table entries remain until they are reaped.
* If left unaddressed, large numbers of zombie processes can exhaust available Process IDs (`PID`), preventing new processes from starting.

---

## Identifying Zombie Processes

Use `ps` to inspect Process IDs (`PID`), Parent Process IDs (`PPID`), process execution states, and commands:

```bash
ps -eo pid,ppid,state,cmd
```

To filter specifically for processes in the zombie state:

```bash
ps -eo pid,ppid,state,cmd | grep ' Z '
```

> **Note:** If the command returns no output, there are no active zombie processes

### Important Fields

| Field | Description |
|---|---|
| `PID` | Process ID |
| `PPID` | Parent Process ID |
| `STATE` | Current process state |
| `CMD` | Command or binary name associated with the process |
| `Z` | Indicates the process is in the **Zombie** (defunct) state |

---

## Why Do Zombie Processes Occur?

When a child process terminates, the Linux kernel retains a record of its exit code in the process table.

```text
Parent Process
     │
     │ forks child
     ▼
Child Process (executes & terminates)
     │
     ▼
State: Z (Zombie / Defunct)
     │
     │ Parent calls wait() / waitpid()
     ▼
Process Table Entry Reaped & Removed
```

* The parent process is responsible for reading the exit code via system calls such as `wait()` or `waitpid()`.
* Until this status collection occurs, the child process remains in the `Z` state.
* **A zombie cannot be terminated with `kill` (including `kill -9`)** because it has already finished executing and is no longer an active program.

---

## Handling Zombie Processes

```text
Identify Zombie (PID & PPID)
             │
             ▼
Investigate Parent Process
             │
   ┌─────────┴─────────┐
   ▼                   ▼
Parent is active     Parent is stuck / deadlocked
& can be repaired              │
                               ▼
                    Restart / Terminate Parent
                               │
                               ▼
                    Init/systemd (PID 1) inherits child
                               │
                               ▼
                    Child reaped automatically
```

1. **Identify the zombie and its parent PID (`PPID`):**
   ```bash
   ps -eo pid,ppid,state,cmd | grep ' Z '
   ```
2. **Investigate the parent process:** Determine why the parent is failing to collect the exit status or missing `SIGCHLD` signals.
3. **Fix or patch the parent service:** Ensure the parent code correctly implements child process harvesting.
4. **Restart or terminate the parent process (if necessary):**
   * If the parent process is terminated, the orphaned zombie is adopted by `init` or `systemd` (PID 1).
   * PID 1 automatically issues `wait()` calls to reap the orphaned child process and clears it from the process table.

---

## Key Takeaways

* **Terminated but unreaped:** A zombie process has completed its lifecycle but still holds an entry in the system process table.
* **The `Z` state:** Identifies a process as defunct or zombie.
* **Resource consumption:** Zombies consume no CPU time or memory, but they occupy system PID slots.
* **Parent accountability:** The parent process must collect the child's exit status; signals like `SIGKILL` have no effect on a dead process.
* **Transient vs. persistent:** Temporary zombies are a normal part of process lifecycles; long-lived or accumulating zombies indicate a bug or stall in the parent application.