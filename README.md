# 🐧 Linux Administration & DevOps Internal Mechanics

> A deep-dive reference guide and knowledge base covering low-level Linux kernel concepts, process lifecycle, memory architecture, resource control (`cgroups`), terminal auditing, and system call tracing (`strace`) — annotated with real-world DevOps production triage scenarios.

---

## 📚 Module Overview & Curated Roadmap

| Part | Document | Primary Focus Areas | Key Commands & Concepts |
| :---: | :--- | :--- | :--- |
| **01** | [**`commands.md`**](file:///C:/Users/kshit/cs/linux/commands.md) | **Sessions, Users & Cgroup Slices** | `w`, `loginctl`, `useradd`, `usermod`, `systemd-logind`, `user.slice`, `session-*.scope` |
| **02** | [**`commands_2.md`**](file:///C:/Users/kshit/cs/linux/commands_2.md) | **Job Control, Signals & Task Limits** | GNU Readline (`Ctrl+U`/`K`), `tlog`, `jobs`, `nohup`, `kill` (`SIGTERM`/`SIGKILL`), `systemctl edit`, `TasksMax` |
| **03** | [**`commands_3.md`**](file:///C:/Users/kshit/cs/linux/commands_3.md) | **Process Introspection & Scheduling** | `ps aux`, `top` (CPU states `us`/`sy`/`wa`/`st`), `STAT` codes (`R`, `S`, `D`, `Z`), `nice`, `renice` |
| **04** | [**`commands_4.md`**](file:///C:/Users/kshit/cs/linux/commands_4.md) | **Multiplexing & Memory Internals** | `tmux` architecture, `ps -eo` custom formatting, `VSZ` vs `RSS` vs `PSS` vs `USS`, `/proc/[PID]/smaps` |
| **05** | [**`commands_5.md`**](file:///C:/Users/kshit/cs/linux/commands_5.md) | **Syscalls, Kernel Space & `strace`** | Kernel vs User Space (Ring 0 vs 3), Context Switching, `glibc` vs Syscalls, `strace`, `strace -c`, I/O buffering |

---

## 🏛️ High-Level Architectural Flow

```
+───────────────────────────────────────────────────────────────────────────+
|                                USER SPACE                                 |
|  CLI Tools (date, ps)      Applications (Go/Node/Python)     Multiplexer  |
|  [ commands.md, 3.md ]        [ commands_5.md ]            [ commands_4.md ]
|  ───────────────────────────────────────────────────────────────────────  |
|          Standard C Library / POSIX Wrappers (glibc, musl)                |
+───────────────────────────────────────────────────────────────────────────+
                                      │
                         SYSCALL TRAP (`syscall`)
                         [ Traced via `strace` in 5.md ]
                                      │
+───────────────────────────────────────────────────────────────────────────+
|                               KERNEL SPACE                                |
|   ┌────────────────────────┐  ┌───────────────────────┐  ┌──────────────┐ |
|   │ Task & CPU Scheduler   │  │ Virtual Memory & Page │  │ File Systems │ |
|   │ (nice/renice - 3.md)   │  │ Tables (PSS/RSS-4.md) │  │ & VFS Driver │ |
|   └────────────────────────┘  └───────────────────────┘  └──────────────┘ |
|   ┌────────────────────────┐  ┌─────────────────────────────────────────┐ |
|   │ Cgroups & Resource     │  │ Systemd Hierarchy (PID 1)               │ |
|   │ Slices (1.md & 2.md)   │  │ (system.slice, user.slice, scopes)      │ |
|   └────────────────────────┘  └─────────────────────────────────────────┘ |
+───────────────────────────────────────────────────────────────────────────+
                                      │
+───────────────────────────────────────────────────────────────────────────+
|                                 HARDWARE                                  |
|            [ CPU Rings ]      [ Physical RAM ]      [ NVMe / NIC ]        |
+───────────────────────────────────────────────────────────────────────────+
```

---

## 🔍 Detailed Chapter Summaries

### 📖 [Part 1: Sessions, User Administration & Cgroups](file:///C:/Users/kshit/cs/linux/commands.md)
- **User Activity & Load Averages (`w`):** Deconstructing system clock, uptime, joint CPU (`JCPU`), process CPU (`PCPU`), and the 1/5/15-minute load average formulas.
- **Session & Seat Management (`loginctl`):** Lifecycle of sessions, scopes vs services, and user lingering (`enable-linger`) for CI/CD runners and rootless containers.
- **User/Group Provisioning (`useradd`, `usermod`, `passwd`):** Internal `/etc/passwd` and `/etc/shadow` mechanics, flag matrices (`-m`, `-s`, `-aG`), and non-root container security.
- **Kernel Resource Hierarchy (`cgroups` & `systemd`):** Demystifying `user.slice`, `system.slice`, dynamic scope units, and real-time monitoring via `systemd-cgtop`.

### 📖 [Part 2: Efficiency, Session Auditing, Signals & Task Limits](file:///C:/Users/kshit/cs/linux/commands_2.md)
- **Terminal Productivity:** GNU Readline navigation and kill-ring shortcuts (`Ctrl+U`, `Ctrl+K`, `Ctrl+W`).
- **Terminal Session Auditing (`tlog`):** Bastion jump-host recording (`tlog-rec`), JSON payload parsing, and playback (`tlog-play`) for SOC2/ISO compliance.
- **Process & Job Control:** Background execution (`&`), job management (`jobs`, `fg`, `bg`), and process detachment (`nohup`, `disown`).
- **Linux Signals:** Signal architecture, trapping, and the critical distinction between graceful drains (`SIGTERM` / signal 15) vs kernel-forced aborts (`SIGKILL` / signal 9).
- **Hard Resource Limits (`systemctl edit`, `TasksMax`):** Drop-in service overrides and hands-on fork bomb mitigation.

### 📖 [Part 3: Snapshot Introspection, CPU States & Scheduling](file:///C:/Users/kshit/cs/linux/commands_3.md)
- **Process Snapshot Introspection (`ps aux` / `ps -ef`):** Comprehensive field breakdown, process hierarchy navigation with `pstree`, and custom formatting.
- **Linux Memory Metrics:** Deconstructing Virtual Size (`VSZ`), Resident Set Size (`RSS`), and Shared Memory (`SHR`).
- **Process State Codes (`STAT`):** Understanding `R` (Running), `S` (Interruptible Sleep), `D` (Uninterruptible Disk Sleep), `Z` (Zombie), and auxiliary flags (`+`, `s`, `l`, `<`).
- **Real-Time Monitoring (`top`):** CPU time categorization (`us`, `sy`, `ni`, `id`, `wa`, `hi`, `si`, `st`), interactive triage hotkeys, and AWS/GCP CPU steal (`st%`) diagnosis.
- **Kernel Scheduling & Priorities (`nice`, `renice`):** Dynamic priority calculation (`PR = 20 + NI`) and launching background batch tasks safely.

### 📖 [Part 4: Multiplexing, Custom Queries & Memory Architecture](file:///C:/Users/kshit/cs/linux/commands_4.md)
- **Terminal Multiplexing (`tmux`):** Client-server architecture, sessions, windows, panes, scrollback buffers, prefix shortcuts (`Ctrl+b`), and production configuration recipes.
- **Automated Process Queries (`ps -eo`):** Building targeted CLI reports, sorting memory/CPU consumers, and formatting custom token lists.
- **Deep-Dive Memory Architecture:** Demystifying Proportional Set Size (`PSS`) and Unique Set Size (`USS`), page tables, shared library deduping, and inspecting `/proc/[PID]/smaps`.

### 📖 [Part 5: Architecture, System Calls & `strace` Profiling](file:///C:/Users/kshit/cs/linux/commands_5.md)
- **System Architecture & Hardware Rings:** User Space (Ring 3) vs Kernel Space (Ring 0), driver abstraction layers, and physical network packet ingress.
- **System Calls vs System APIs:** Low-level `syscall` register ABI (`RAX`, `RDI`, `RSI`), `glibc` standard wrappers, and user-to-kernel mode transitions.
- **Context Switching & Performance Optimization:** Mode switches vs process switches, cache thrashing, and why unbuffered I/O loops degrade algorithms regardless of Big-O complexity.
- **Syscall Tracing (`strace` & `strace -c`):** Phase-by-phase trace dissection (`execve`, ELF dynamic linker `ld.so`, locale fallback, timezone reading, and standard output `write`), plus real-time profiling metrics.

---

## 🛠️ DevOps Production Incident Response Playbook

| Scenario / Production Symptom | Diagnostic Command | Root Cause & Resolution Path |
| :--- | :--- | :--- |
| **High CPU Alert (System Time Spike `%sy`)** | `top` $\rightarrow$ `pidstat -w 1` $\rightarrow$ `sudo strace -c -p <PID>` | Detects excessive system calls / mode switching. Fix by adding I/O buffering. |
| **Process Frozen / 0% CPU Consumption** | `sudo strace -p <PID> -f -tt -T` | Identifies blocking system calls (`connect()`, `futex()`, `read()`) waiting on dead sockets or unreleased mutexes. |
| **Container OOM Killed (Exit Code 137)** | `grep -i oom /var/log/syslog` $\rightarrow$ inspect `USS` in `/proc/<PID>/smaps` | Identifies true unshared memory growth rather than shared library allocations. |
| **Unkillable Process in `D` State** | `ps -eo pid,stat,wchan:20,cmd \| grep D` | Process is stuck waiting on physical hardware I/O or a frozen NFS mount. Kill the underlying I/O, not the process. |
| **Silent App Startup Failure / Missing Config** | `strace -e trace=openat,access ./app` | Pinpoints exactly which file paths the binary probed and distinguishes `ENOENT` (not found) from `EACCES` (permission denied). |
| **Rogue Fork Bomb / Process Exhaustion** | `systemctl status user-1000.slice` | Systemd cgroup `TasksMax` threshold reached; prevents host kernel panic and isolates runaway scripts to the user slice. |

---

## 💡 How to Use These Notes

1. **Sequential Study:** Read from Part 1 through Part 5 for a structured progression from Linux user-space management down to kernel system call mechanics.
2. **On-Call Reference:** Jump directly to the **DevOps Real-Time Scenarios** and **Troubleshooting Cheat Sheets** at the end of each module during production incidents.
3. **Hands-On Verification:** Run the included commands in a local Linux/WSL2 environment to observe real-time system behaviors, cgroup hierarchies, and process states.

---

*Authored for Linux System Administrators, SREs, and DevOps Engineers.*
