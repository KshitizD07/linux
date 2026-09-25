# 🐧 Linux Administration & DevOps Internal Mechanics (Part 3)

> Comprehensive hands-on notes exploring Linux process introspection (`ps`), real-time performance monitoring (`top`), CPU time distributions (`us`, `sy`, `wa`, `st`), memory metrics (`VSZ`, `RSS`, `SHR`), process state codes (`STAT`), and priority scheduling (`nice`, `renice`) — annotated with real-time DevOps production scenarios.

---

## Table of Contents
1. [1. `ps` — Snapshot Process Introspection & Analysis](#1-ps--snapshot-process-introspection--analysis)
   - [Basic Shell Snapshot (`ps`)](#basic-shell-snapshot-ps)
   - [Comprehensive System-Wide View (`ps aux` / `ps -ef`)](#comprehensive-system-wide-view-ps-aux--ps--ef)
   - [Field-by-Field Column Breakdown](#field-by-field-column-breakdown)
   - [Custom Formats & Tree Navigation (`ps -eo`, `pstree`)](#custom-formats--tree-navigation-ps--eo-pstree)
2. [2. Deep Dive: Memory Metrics & Process State Codes (`STAT`)](#2-deep-dive-memory-metrics--process-state-codes-stat)
   - [Demystifying Linux Memory: `VSZ` vs `RSS` vs `SHR`](#demystifying-linux-memory-vsz-vs-rss-vs-shr)
   - [Comprehensive Process State Code Matrix (`STAT`)](#comprehensive-process-state-code-matrix-stat)
   - [Deciphering Real-World STAT Combinations (`S<s`, `Ss`, `R+`, `Ssl`)](#deciphering-real-world-stat-combinations-ss-ss-r-ssl)
3. [3. `top` — Real-Time System & Performance Monitoring](#3-top--real-time-system--performance-monitoring)
   - [Terminal Output Breakdown](#terminal-output-breakdown)
   - [System Summary Area (Header Breakdown)](#system-summary-area-header-breakdown)
     - [Line 1: Uptime & Load Averages](#line-1-uptime--load-averages)
     - [Line 2: Task States Breakdown](#line-2-task-states-breakdown)
     - [Line 3: Deep Dive into CPU States (`us`, `sy`, `wa`, `st`, etc.)](#line-3-deep-dive-into-cpu-states-us-sy-wa-st-etc)
     - [Line 4 & 5: Physical RAM & Swap Allocation](#line-4--5-physical-ram--swap-allocation)
   - [Process List Area (Body Breakdown)](#process-list-area-body-breakdown)
   - [Essential Interactive Navigation Keys](#essential-interactive-navigation-keys)
   - [`ps` vs `top` vs `htop` Architectural Comparison](#ps-vs-top-vs-htop-architectural-comparison)
4. [4. Process Priority & Kernel Scheduling (`nice`, `renice`)](#4-process-priority--kernel-scheduling-nice-renice)
   - [How Linux CPU Scheduling Works: `PR` vs `NI`](#how-linux-cpu-scheduling-works-pr-vs-ni)
   - [Launching Low/High-Priority Tasks with `nice`](#launching-lowhigh-priority-tasks-with-nice)
   - [Modifying Running Process Priority with `renice`](#modifying-running-process-priority-with-renice)
5. [5. 🚀 DevOps Real-Time Scenarios](#5--devops-real-time-scenarios)
   - [Scenario A: Diagnosing High Disk I/O Wait (`wa%`) & Unkillable `D`-State Processes](#scenario-a-diagnosing-high-disk-io-wait-wa-unkillable-d-state-processes)
   - [Scenario B: Cloud VM CPU Throttling & Steal Time (`st%`) on AWS/GCP](#scenario-b-cloud-vm-cpu-throttling--steal-time-st-on-awsgcp)
   - [Scenario C: Identifying Production Memory Leaks Before the OOM Killer Strikes](#scenario-c-identifying-production-memory-leaks-before-the-oom-killer-strikes)
6. [6. DevOps Production Troubleshooting Cheat Sheet](#6-devops-production-troubleshooting-cheat-sheet)

---

## 1. `ps` — Snapshot Process Introspection & Analysis

The `ps` (Process Status) utility captures a static point-in-time snapshot of the active processes running on a Linux system.

```
                                  ps (Process Status)
                                          │
                     ┌────────────────────┴────────────────────┐
                     ▼                                         ▼
            Standard POSIX Style                      BSD Style (No Dashes)
           (e.g., ps -ef, ps -u user)                (e.g., ps aux, ps axjf)
```

---

### Basic Shell Snapshot (`ps`)

Running raw `ps` with no flags shows only processes attached to the **current terminal session (`pts`)** and owned by the **current user**:

```bash
kshitiz@KZ:~$ ps
    PID TTY          TIME CMD
    318 pts/0    00:00:00 bash
   1283 pts/0    00:00:00 ps
```

- **`PID`**: Process ID (kernel-level unique numeric identifier).
- **`TTY`**: Controlling terminal attached to the process (`pts/0` = Pseudo-Terminal Slave 0).
- **`TIME`**: Cumulative CPU execution time consumed by the process so far (`00:00:00`).
- **`CMD`**: Exact command/executable name (`bash`, `ps`).

---

### Comprehensive System-Wide View (`ps aux` / `ps -ef`)

To inspect all running system daemons, user sessions, kernel helper threads, and background workers:

#### BSD Syntax: `ps aux`
```bash
kshitiz@KZ:~$ ps aux
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.1  21596 12264 ?        Ss   05:56   0:00 /sbin/init
root          39  0.0  0.2  58644 18812 ?        S<s  05:56   0:00 /usr/lib/systemd/systemd-journald
systemd+     145  0.0  0.1  21344 12672 ?        Ss   05:56   0:00 /usr/lib/systemd/systemd-resolved
kshitiz      318  0.0  0.0   6312  5120 pts/0    Ss   05:56   0:00 -bash
kshitiz     1284  0.0  0.0   9588  4864 pts/0    R+   06:15   0:00 ps aux
```

#### Breakdown of the `aux` Flag Syntax:
- **`a`**: Displays processes across **all users** on the system (not just your own).
- **`u`**: Displays the output in **user-oriented format** (adds CPU%, MEM%, RSS, VSZ, and username).
- **`x`**: Includes processes **without a controlling terminal** (`TTY = ?`), such as background system daemons, database servers, and Docker containers.

> [!NOTE]
> In BSD-style `ps` syntax, flags are passed **without a leading dash** (`ps aux`). If you run `ps -aux`, Linux interprets `-u` as "filter by user named 'ax'", which might raise a warning or behave unexpectedly on some Unix distributions.

---

### Field-by-Field Column Breakdown

| Column | Meaning | Low-Level Internal Details |
| :--- | :--- | :--- |
| **`USER`** | Effective User | The Linux user account whose permissions govern the process execution. |
| **`PID`** | Process ID | The unique integer assigned by the kernel's process table. |
| **`%CPU`** | CPU Utilization | Percentage of total CPU time used divided by the process lifetime. |
| **`%MEM`** | Memory Utilization | Percentage of the machine's total physical RAM occupied by the process's **RSS**. |
| **`VSZ`** | Virtual Memory Size | Total virtual memory mapped by the process (in KiB), including swap, allocated buffers, and shared libraries. |
| **`RSS`** | Resident Set Size | **Actual physical RAM** held by the process in non-swapped memory pages (in KiB). |
| **`TTY`** | Controlling Terminal | `pts/0` = virtual terminal; `tty1` = physical console; `?` = detached daemon/service. |
| **`STAT`** | Process State | Multi-character encoding describing the current kernel execution state and scheduling priority. |
| **`START`** | Launch Timestamp | Time or date when the process was originally spawned. |
| **`TIME`** | Cumulative CPU Time | Total execution time the process has actually spent executing on CPU cores. |
| **`COMMAND`** | Executable / CLI Args | The binary invoked plus all initial command-line arguments passed at startup. |

---

### Custom Formats & Tree Navigation (`ps -eo`, `pstree`)

#### 1. Custom Formatting with `ps -eo`
Filter out unnecessary noise and extract only exact metrics needed for automated scripts:
```bash
# Print only PID, User, CPU%, Memory%, and Command sorted by memory consumption
ps -eo pid,user,%cpu,%mem,vsz,rss,comm --sort=-%mem | head -n 10
```

#### 2. Visual Parent-Child Tree (`pstree`)
```bash
pstree -p -u kshitiz
```
*Output:*
```text
systemd(1)───systemd(366)───(sd-pam)(368)
          └──login(319)───bash(390)───pstree(1410)
```

---

## 2. Deep Dive: Memory Metrics & Process State Codes (`STAT`)

### Demystifying Linux Memory: `VSZ` vs `RSS` vs `SHR`

One of the most frequent mistakes in systems engineering is confusing **Virtual Memory (`VSZ`)** with **Actual RAM Usage (`RSS`)**.

```
+--------------------------------------------------------------------------+
|                      PROCESS VIRTUAL ADDRESS SPACE                       |
|                                  (VSZ)                                   |
|                                                                          |
|  +-------------------------------------+  +---------------------------+  |
|  |       Resident Set Size (RSS)       |  |     Unallocated / Swap    |  |
|  |     (Pages Loaded in Physical RAM)  |  |   (MALLOC'd but untouched |  |
|  |                                     |  |    or swapped to disk)    |  |
|  |  +-------------------------------+  |  +---------------------------+  |
|  |  | Shared Memory (SHR)           |  |                                 |
|  |  | (libc.so, shared memory IPC)  |  |                                 |
|  |  +-------------------------------+  |                                 |
|  +-------------------------------------+                                 |
+--------------------------------------------------------------------------+
```

1. **`VSZ` (Virtual Memory Size):**
   - Represents all memory the process can address.
   - Includes allocated but untouched memory (e.g., when a program reserves 4GB with `malloc` but only writes 50MB), shared dynamic libraries (`libc.so`), and memory mapped to files.
   - *High VSZ does NOT mean high physical RAM consumption.*

2. **`RSS` (Resident Set Size):**
   - **The true measure of physical RAM occupied.**
   - The actual number of kilobytes of physical RAM pages currently mapped and resident in hardware memory for this process.

3. **`SHR` (Shared Memory):**
   - Subset of RSS that represents code libraries and shared memory regions shared across multiple processes (e.g., standard C libraries or shared database buffer pools).

---

### Comprehensive Process State Code Matrix (`STAT`)

The `STAT` column contains a **primary state code** (1st character) followed by optional **scheduling/session modifiers**:

#### Primary State Codes (First Character):
| Code | Kernel State | Description |
| :---: | :--- | :--- |
| **`R`** | **Running / Runnable** | Actively executing on a CPU core or placed in the scheduler's run-queue. |
| **`S`** | **Interruptible Sleep** | Blocked waiting for an event/signal (e.g., waiting for network packets, timer, or user input). |
| **`D`** | **Uninterruptible Sleep** | Blocked waiting directly on hardware I/O (disk read/write, NFS response). **Cannot be killed by `kill -9`** until I/O completes. |
| **`Z`** | **Zombie (Defunct)** | Child process has finished executing (`exit()`), but its parent has not yet read its return status via `wait()`. |
| **`T`** | **Stopped / Traced** | Process was paused by a signal (`SIGSTOP`, `SIGTSTP` via `Ctrl+Z`) or attached to a debugger (`gdb`, `strace`). |

#### Secondary Modifier Characters:
| Modifier | Meaning | Technical Context |
| :---: | :--- | :--- |
| **`<`** | **High Priority** | Process has a negative nice value (given more CPU time slices by kernel). |
| **`N`** | **Low Priority** | Process has a positive nice value ("nice" to other tasks, gets fewer time slices). |
| **`L`** | **Pages Locked** | Process has memory pages locked into RAM via `mlock()` (prevents swapping; used by crypto/databases). |
| **`s`** | **Session Leader** | The process is the parent of a session (e.g., login shell `bash` that manages child processes). |
| **`l`** | **Multi-Threaded** | The process has spawned multiple threads (cloned with `CLONE_THREAD`). |
| **`+`** | **Foreground Process** | Member of the foreground process group attached to the terminal (receives keyboard input). |

---

### Deciphering Real-World STAT Combinations (`S<s`, `Ss`, `R+`, `Ssl`)

Let's dissect real outputs from a running Linux host:

- **`systemd-journald` $\rightarrow$ `STAT = S<s`**:
  - `S`: Sleeping (waiting for log events).
  - `<`: High priority (critical logging service; given preferential CPU scheduling).
  - `s`: Session leader.
- **`-bash` $\rightarrow$ `STAT = Ss`**:
  - `S`: Interruptible sleep (waiting for user to type a command).
  - `s`: Session leader (owns the terminal session).
- **`ps aux` $\rightarrow$ `STAT = R+`**:
  - `R`: Running on CPU right now to generate the process report.
  - `+`: Foreground process group (currently controlling terminal output).
- **`polkitd` / `node` $\rightarrow$ `STAT = Ssl`**:
  - `S`: Sleeping.
  - `s`: Session leader.
  - `l`: Multi-threaded application utilizing a thread pool.

---

## 3. `top` — Real-Time System & Performance Monitoring

While `ps` provides a static point-in-time snapshot, `top` provides a dynamic, self-refreshing dashboard of hardware utilization, kernel scheduler activity, and active processes.

```bash
kshitiz@KZ:/mnt/c/Users/kshit$ top
top - 06:25:15 up 28 min,  1 user,  load average: 0.00, 0.00, 0.00
Tasks:  27 total,   1 running,  26 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem :   7848.2 total,   7255.9 free,    524.3 used,    224.8 buff/cache
MiB Swap:   2048.0 total,   2048.0 free,      0.0 used.   7323.9 avail Mem

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
      1 root      20   0   21596  12264   9320 S   0.0   0.2   0:00.70 systemd
     39 root      19  -1   66840  18940  17916 S   0.0   0.2   0:00.29 systemd-journal
    318 kshitiz   20   0    6312   5120   3456 S   0.0   0.1   0:00.07 bash
```

---

### Terminal Output Breakdown

The `top` display is architecturally split into two primary segments:
1. **System Summary Area (Lines 1 to 5):** Global operating system metrics, CPU state distributions, and memory counters.
2. **Process List Area (Body):** Real-time list of individual processes ranked dynamically by resource usage.

---

### System Summary Area (Header Breakdown)

#### Line 1: Uptime & Load Averages
```text
top - 06:25:15 up 28 min,  1 user,  load average: 0.00, 0.00, 0.00
```
- **`06:25:15`**: Current system clock time.
- **`up 28 min`**: System uptime duration since last kernel boot.
- **`1 user`**: Total count of active interactive user sessions.
- **`load average: 0.00, 0.00, 0.00`**: Exponential moving average of runnable and uninterruptible processes over **1 min**, **5 min**, and **15 min**.

#### Line 2: Task States Breakdown
```text
Tasks:  27 total,   1 running,  26 sleeping,   0 stopped,   0 zombie
```
- **`total`**: Total number of processes and threads in the kernel process table.
- **`running`**: Number of tasks actively executing or queued for CPU cores.
- **`sleeping`**: Tasks waiting on events/signals or I/O.
- **`stopped`**: Tasks halted by signals (`Ctrl+Z` / `SIGSTOP`).
- **`zombie`**: Dead child processes awaiting parent cleanup.

---

#### Line 3: Deep Dive into CPU States (`us`, `sy`, `wa`, `st`, etc.)
```text
%Cpu(s):  0.0 us,  0.0 sy,  0.0 ni, 100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
```

Understanding these CPU state metrics is essential for diagnosing production performance bottlenecks:

| Metric | Full Name | Description & DevOps Diagnostic Meaning |
| :--- | :--- | :--- |
| **`us`** | **User Time** | CPU time spent executing non-kernel user-space processes (e.g., Python apps, Node.js, Nginx, Go binaries). |
| **`sy`** | **System Time** | CPU time spent executing Linux Kernel system calls (`read`, `write`, `fork`, network socket handling). If high, check for excessive context switching or syscall loops. |
| **`ni`** | **Nice Time** | CPU time spent executing user-space processes with positive nice values (manually lowered priority). |
| **`id`** | **Idle Time** | Percentage of CPU time when processor cores have no work and are waiting. |
| **`wa`** | **I/O Wait** | **Critical Metric:** Percentage of time CPU is idle **waiting for disk or network I/O operations to finish**. High `wa%` (>10-15%) indicates a severe disk bottleneck or failing storage array. |
| **`hi`** | **Hardware Interrupts** | CPU time spent servicing physical hardware interrupts (NIC packet receipt, disk controller events). |
| **`si`** | **Software Interrupts** | CPU time spent handling software interrupts (network stack processing, packet filtering via iptables). |
| **`st`** | **Steal Time** | **Critical Cloud Metric:** Percentage of CPU cycles stolen from your Virtual Machine by the hypervisor (AWS Nitro, KVM, VMware) to service other "noisy neighbor" VMs on the same physical host. |

---

#### Line 4 & 5: Physical RAM & Swap Allocation
```text
MiB Mem :   7848.2 total,   7255.9 free,    524.3 used,    224.8 buff/cache
MiB Swap:   2048.0 total,   2048.0 free,      0.0 used.   7323.9 avail Mem
```

- **`total`**: Total hardware RAM installed.
- **`free`**: Completely unallocated memory.
- **`used`**: Memory consumed by active applications and kernel space.
- **`buff/cache`**: Linux uses unused RAM to cache disk blocks (`page cache`) and buffer I/O. **This is normal and beneficial.**
- **`avail Mem`**: The true available memory estimate without causing the system to swap.

> [!TIP]
> **"Free memory" vs "Available memory":**
> In Linux, unused RAM is wasted RAM. The Linux kernel automatically borrows unused RAM for disk caching (`buff/cache`) to accelerate read speeds. When an application requests memory, the kernel drops the cache instantly. Always look at **`avail Mem`**, not `free Mem`!

---

### Process List Area (Body Breakdown)

```text
    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
      1 root      20   0   21596  12264   9320 S   0.0   0.2   0:00.70 systemd
     39 root      19  -1   66840  18940  17916 S   0.0   0.2   0:00.29 systemd-journal
```

- **`PR`**: Kernel dynamic scheduling priority (e.g., `20` = standard default, `rt` = real-time).
- **`NI`**: User-configurable **Nice Value** (ranges from `-20` to `+19`).
- **`VIRT`**: Total virtual memory requested by process.
- **`RES`**: Actual non-swapped physical RAM held.
- **`SHR`**: Shared memory pages.
- **`S`**: Process state (`R`, `S`, `D`, `Z`, `T`).
- **`%CPU`**: Current CPU core usage share.
- **`%MEM`**: Fraction of physical RAM occupied by `RES`.
- **`TIME+`**: Total cumulative CPU execution time down to hundredths of a second.
- **`COMMAND`**: Program name.

---

### Essential Interactive Navigation Keys

While `top` is running, use these interactive single-key commands:

| Key | Function | DevOps Usage Context |
| :---: | :--- | :--- |
| **`q`** | **Quit** | Exit `top` and return to the shell prompt. |
| **`P`** | **Sort by CPU%** | Default sorting; instantly brings CPU-hogging processes to the top. |
| **`M`** | **Sort by MEM%** | Sorts by physical RAM consumption (`RES`); ideal for memory leak troubleshooting. |
| **`T`** | **Sort by TIME+** | Sorts by longest cumulative execution time; useful for finding persistent rogue loops. |
| **`k`** | **Kill Process** | Prompts for a PID and signal number (default 15 `SIGTERM`) to kill directly from `top`. |
| **`r`** | **Renice Process** | Prompts for a PID and new nice value to adjust priority without leaving `top`. |
| **`1`** | **Toggle CPU Cores** | Expands Line 3 to display individual utilization graphs for every CPU core (`Cpu0`, `Cpu1`, etc.). |
| **`u`** | **Filter by User** | Prompts for a username to filter process list (e.g., type `kshitiz` or `www-data`). |
| **`c`** | **Toggle Full Command** | Displays full executable paths and CLI arguments instead of just the binary name. |
| **`z`** | **Toggle Color** | Highlights active running processes and summary data in color for easier scanning. |
| **`d`** or **`s`** | **Change Delay** | Changes refresh interval (default is 3.0 seconds; set to `1.0` for faster updates). |

---

### `ps` vs `top` vs `htop` Architectural Comparison

| Feature | `ps` | `top` | `htop` |
| :--- | :--- | :--- | :--- |
| **Type** | Point-in-time snapshot. | Real-time terminal dashboard. | Enhanced interactive visual tool. |
| **Interaction** | None (prints to stdout). | Keyboard commands (`P`, `M`, `k`). | Mouse support, visual menus, hotkeys. |
| **Default Availability** | Pre-installed everywhere. | Pre-installed everywhere. | Requires manual install (`apt install htop`). |
| **Best Used For** | **Shell scripts, CI/CD pipes, exact regex grep queries.** | **Quick triage on minimal server installations without extra packages.** | **Interactive human debugging & multi-core CPU monitoring.** |

---

## 4. Process Priority & Kernel Scheduling (`nice`, `renice`)

In Linux, the kernel's Completely Fair Scheduler (CFS) dynamically decides how much CPU time each thread receives. Engineers can influence this scheduling priority using **Nice values**.

```
Highest Priority (Least Nice)                      Lowest Priority (Most Nice)
       -20 <─────────────────────── 0 ───────────────────────> +19
 (Gets max CPU time)            (Default)              (Yields CPU to others)
```

---

### How Linux CPU Scheduling Works: `PR` vs `NI`

- **`NI` (Nice Value):** The user-space priority metric ranging from **`-20` (highest priority)** to **`+19` (lowest priority)**.
- **`PR` (Priority Value):** The actual dynamic priority calculated internally by the kernel:
  $$\text{PR} = 20 + \text{NI}$$
  - Default: $\text{NI} = 0 \implies \text{PR} = 20$.
  - Minimum Nice: $\text{NI} = -20 \implies \text{PR} = 0$ (maximum priority).
  - Maximum Nice: $\text{NI} = +19 \implies \text{PR} = 39$ (lowest priority).

> [!IMPORTANT]
> - Any regular non-root user can make their processes **nicer** (increase value from `0` to `+19`).
> - **Only `root` / `sudo`** can assign negative nice values (`-1` to `-20`) to give a process higher priority.

---

### Launching Low/High-Priority Tasks with `nice`

```bash
# Launch a background backup script with lowest CPU priority (nice = 19)
nice -n 19 tar -czf backup.tar.gz /var/www/data &

# Launch a critical real-time processing worker with high priority (requires sudo)
sudo nice -n -10 python3 high_freq_trading.py &
```

---

### Modifying Running Process Priority with `renice`

To alter the priority of already running processes on the fly:

```bash
# Lower priority of a running process (PID 4510) to nice value +10
renice -n 10 -p 4510

# Elevate priority of a running database daemon to -5 (requires sudo)
sudo renice -n -5 -p 1280

# Renice ALL processes owned by user 'developer' to +15
sudo renice -n 15 -u developer
```

---

## 5. 🚀 DevOps Real-Time Scenarios

### Scenario A: Diagnosing High Disk I/O Wait (`wa%`) & Unkillable `D`-State Processes

**The Situation:** Alert fires: `API Server P99 Response Time Degraded`. You run `top` and observe:
```text
%Cpu(s):  1.2 us,  0.5 sy,  0.0 ni, 42.1 id, 56.2 wa,  0.0 hi,  0.0 si,  0.0 st
```
The CPU is 56.2% stuck in **`wa` (I/O Wait)**.

#### Troubleshooting Steps:
1. **Locate processes stuck in `D` state (Uninterruptible Sleep):**
   ```bash
   ps aux | awk '$8 ~ /D/'
   ```
   *Output:*
   ```text
   postgres  6120  0.1  8.4 1250420 680210 ?  D   04:12   0:15 postgres: writer process
   ```
2. **Understand why `kill -9 6120` will NOT work:**
   The kernel does not deliver signals to processes in state `D` until the outstanding hardware I/O request finishes.
3. **Identify the storage bottleneck:**
   Run `iostat -xz 1` or `iotop` to discover which disk volume or NFS mount has saturated its throughput or IOPS limits.

---

### Scenario B: Cloud VM CPU Throttling & Steal Time (`st%`) on AWS/GCP

**The Situation:** Your application running on an AWS EC2 `t3.medium` instance experiences sudden severe latency spikes. You inspect `top`:
```text
%Cpu(s):  8.0 us,  2.1 sy,  0.0 ni, 20.0 id,  0.0 wa,  0.0 hi,  0.0 si, 69.9 st
```
Notice **`69.9 st` (Steal Time)**!

#### Analysis & Fix:
- **Root Cause:** Burstable cloud instances (like AWS `t3`/`t4g` or GCP shared-core `e2-micro`) operate on CPU credits. Once the instance exhausts its baseline CPU credit bucket, the hypervisor artificially starves the VM of physical CPU clock cycles.
- **Resolution:**
  1. Upgrade to a non-burstable compute instance (e.g., AWS `c6i.large` or `m6i.large`).
  2. Or enable *Unlimited CPU Credits* in AWS CloudWatch / EC2 configuration.

---

### Scenario C: Identifying Production Memory Leaks Before the OOM Killer Strikes

**The Situation:** A Node.js backend service crashes intermittently once every 24 hours under production traffic.

#### Diagnostic Workflow:
1. **Track `RSS` vs `VSZ` growth over time:**
   ```bash
   ps -eo pid,user,vsz,rss,comm --sort=-rss | head -n 5
   ```
2. Monitor memory growth interactively in `top` by pressing **`M`** (Sort by Resident Memory).
3. If `RSS` steadily grows from `200MB` to `4GB` while traffic remains flat, the application has an uncollected object/cache leak.
4. Catching this early prevents the kernel **OOM Killer** from abruptly sending `SIGKILL` to critical database and gateway daemons on the host.

---

## 6. DevOps Production Troubleshooting Cheat Sheet

| Diagnostic Task | Command / Technique | What It Tells You |
| :--- | :--- | :--- |
| **List Top 10 RAM Consumers** | `ps -eo pid,user,%mem,rss,comm --sort=-rss \| head -n 11` | Pinpoint exact physical memory hogs without running interactive dashboards. |
| **List Top 10 CPU Consumers** | `ps -eo pid,user,%cpu,comm --sort=-%cpu \| head -n 11` | Discover rogue computational loops or runaway worker threads. |
| **Find Uninterruptible `D`-State Tasks** | `ps aux \| awk '$8 ~ /D/'` | Identify processes hung waiting on disk/NFS I/O hardware failures. |
| **Identify Zombie Processes** | `ps aux \| grep 'Z'` | Locate dead child processes awaiting parent status collection. |
| **Display CPU Core Breakdown** | Launch `top` $\rightarrow$ press `1` | View load distribution across every individual CPU core. |
| **Filter `top` by Specific User** | Launch `top` $\rightarrow$ press `u` $\rightarrow$ type `nginx` | Filter process listing exclusively to targeted service user. |
| **Kill Stuck Process from `top`** | Launch `top` $\rightarrow$ press `k` $\rightarrow$ enter PID | Send `SIGTERM` or `SIGKILL` directly without switching terminal windows. |
| **Launch Background Task with Low Priority** | `nice -n 19 <command> &` | Run heavy indexing or backup jobs without impacting production web traffic. |
| **Lower Priority of Running Task** | `sudo renice -n 15 -p <PID>` | Dynamically throttle resource-heavy batch workers during business hours. |
| **Elevate Priority of Critical Service** | `sudo renice -n -10 -p <PID>` | Ensure real-time trading or streaming workers receive prioritized CPU cycles. |

---

*Authored for Linux System Administrators & DevOps Engineers.*
