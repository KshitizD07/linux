# 🐧 Linux Administration & DevOps Internal Mechanics (Part 4)

> Comprehensive hands-on notes exploring terminal multiplexing (`tmux`), advanced process introspection (`ps -o` / `ps -eo`), and low-level Linux memory architecture (`VSZ`, `RSS`, `SHR`, `PSS`, `USS`, `/proc/[PID]/status`) — annotated with real-time DevOps production scenarios.

---

## Table of Contents
1. [1. Terminal Multiplexing with `tmux`](#1-terminal-multiplexing-with-tmux)
   - [What is a Terminal Multiplexer & Why Engineers Use It](#what-is-a-terminal-multiplexer--why-engineers-use-it)
   - [Architecture: Server, Sessions, Windows, and Panes](#architecture-server-sessions-windows-and-panes)
   - [Installation & Essential CLI Commands](#installation--essential-cli-commands)
   - [The Prefix Key (`Ctrl + b`) & Keyboard Shortcuts](#the-prefix-key-ctrl--b--keyboard-shortcuts)
     - [Window Management](#window-management)
     - [Pane Management (Splitting & Navigation)](#pane-management-splitting--navigation)
     - [Session Detachment & Scrollback Buffer](#session-detachment--scrollback-buffer)
   - [Customizing `~/.tmux.conf` for DevOps Productivity](#customizing-tmuxconf-for-devops-productivity)
   - [🚀 DevOps Real-Time Scenario: Multi-Pane Incident Response & Safe Remote Deployments](#-devops-real-time-scenario-multi-pane-incident-response--safe-remote-deployments)
2. [2. Custom Process Introspection with `ps -o` / `ps -eo`](#2-custom-process-introspection-with-ps--o--ps--eo)
   - [Why Custom Formatting (`-o`) is Superior for Automation](#why-custom-formatting--o-is-superior-for-automation)
   - [Core Output Specifiers & Formatting Tokens](#core-output-specifiers--formatting-tokens)
   - [High-Yield Production Query Recipes](#high-yield-production-query-recipes)
3. [3. Deep Dive: Linux Memory Internals (`VSZ`, `RSS`, `PSS`, `USS`)](#3-deep-dive-linux-memory-internals-vsz-rss-pss-uss)
   - [The Memory Hierarchy: Virtual vs Physical Address Spaces](#the-memory-hierarchy-virtual-vs-physical-address-spaces)
   - [Metric Comparison: `VSZ` vs `RSS` vs `SHR` vs `PSS` vs `USS`](#metric-comparison-vsz-vs-rss-vs-shr-vs-pss-vs-uss)
   - [Extracting Memory Metrics via `/proc/<PID>/status` & `/proc/<PID>/smaps`](#extracting-memory-metrics-via-procpidstatus--procpidsmaps)
   - [Step-by-Step Walkthrough: Tracing Memory Allocation Under the Hood](#step-by-step-walkthrough-tracing-memory-allocation-under-the-hood)
   - [🚀 DevOps Real-Time Scenario: Diagnosing Silent Memory Leaks in Containers & Pods](#-devops-real-time-scenario-diagnosing-silent-memory-leaks-in-containers--pods)
4. [4. DevOps Production Troubleshooting Cheat Sheet](#4-devops-production-troubleshooting-cheat-sheet)

---

## 1. Terminal Multiplexing with `tmux`

### What is a Terminal Multiplexer & Why Engineers Use It

> **Definition:** **`tmux` (Terminal Multiplexer)** is a software application that allows you to create, access, and control multiple terminal sessions from a single console window.

```
+-------------------------------------------------------------------------------+
|                            TMUX SERVER (Daemon)                               |
+-------------------------------------------------------------------------------+
                                        │
           ┌────────────────────────────┴────────────────────────────┐
           ▼                                                         ▼
+─────────────────────────────+           +─────────────────────────────────────+
|     Session: 'prod-api'     |           |        Session: 'db-migrate'        |
|                             |           |                                     |
|  +-----------------------+  |           |  +-------------------------------+  |
|  | Window 1: 'logs'      |  |           |  | Window 1: 'migration-script'  |  |
|  | ┌──────────┬────────┐ |  |           |  +-------------------------------+  |
|  | │ Pane 1   │ Pane 2 │ |  |           +─────────────────────────────────────+
|  | │ Nginx    │ App    │ |  |
|  | ├──────────┴────────┤ |  |
|  | │ Pane 3: htop      │ |  |
|  | └───────────────────┘ |  |
|  +-----------------------+  |
+─────────────────────────────+
```

#### The 3 Superpowers of `tmux`:
1. **Session Persistence (Immunity to SSH Drops):** Processes running inside a `tmux` session run under a persistent background server daemon. If your SSH connection drops or your laptop sleeps, the session keeps running uninterrupted on the remote server.
2. **Terminal Splitting (Multi-Tasking in One Window):** Split a single terminal into multiple horizontal and vertical panes to monitor logs, run build commands, and inspect system metrics side by side.
3. **Session Sharing (Pair Programming):** Two engineers can connect to the same remote `tmux` session simultaneously over SSH and view/type into the exact same terminal in real time.

---

### Architecture: Server, Sessions, Windows, and Panes

1. **Server (`tmux server`):** Background process that manages all active sessions and stays alive as long as at least one session exists.
2. **Session:** A collection of one or more windows managed together under a specific name (e.g., `prod-deploy`).
3. **Window:** Equivalent to a "tab" in a graphical web browser. Only one window is displayed at a time within a session.
4. **Pane:** A sub-division of a window. A single window can be split into multiple vertical and horizontal panes, each running its own independent shell.

---

### Installation & Essential CLI Commands

#### Installation:
```bash
# Ubuntu / Debian
sudo apt update && sudo apt install tmux -y
```

#### Managing Sessions from the Shell:
```bash
# 1. Start a new named session
tmux new -s deploy

# 2. List all active sessions
tmux ls
# Output: deploy: 1 windows (created Fri Sep 25 12:30:00 2026)

# 3. Attach (reconnect) to an existing session
tmux attach -t deploy
# (Shorthand: tmux a -t deploy)

# 4. Detach from inside the session
# (Press Ctrl+b then d)

# 5. Kill / Delete a specific session
tmux kill-session -t deploy

# 6. Kill all tmux sessions and terminate server daemon
tmux kill-server
```

---

### The Prefix Key (`Ctrl + b`) & Keyboard Shortcuts

Every interactive command in `tmux` is triggered by pressing the **Prefix Key** (`Ctrl + b`), releasing both keys, and then pressing an **Action Key**.

```
Step 1: Press [ Ctrl ] + [ b ]
Step 2: Release both keys
Step 3: Press [ Action Key ] (e.g., %, ", c, d, z)
```

---

#### Window Management

| Shortcut | Action | DevOps Usage |
| :---: | :--- | :--- |
| **`Ctrl+b` $\rightarrow$ `c`** | **Create** a new window (tab). | Opens a clean, dedicated full-screen shell. |
| **`Ctrl+b` $\rightarrow$ `,`** | **Rename** current window. | Label tabs clearly (e.g., `logs`, `k8s-pod`, `db`). |
| **`Ctrl+b` $\rightarrow$ `n`** | Switch to **Next** window. | Rapid tab switching. |
| **`Ctrl+b` $\rightarrow$ `p`** | Switch to **Previous** window. | Rapid tab switching. |
| **`Ctrl+b` $\rightarrow$ `0-9`** | Jump directly to window number. | Instant navigation across complex setups. |
| **`Ctrl+b` $\rightarrow$ `w`** | Interactive **Visual Window List**. | Displays tree menu of all sessions and windows. |
| **`Ctrl+b` $\rightarrow$ `&`** | **Kill/Close** current window. | Closes window and all running child panes. |

---

#### Pane Management (Splitting & Navigation)

| Shortcut | Action | Visual Representation |
| :---: | :--- | :---: |
| **`Ctrl+b` $\rightarrow$ `%`** | **Split Vertically** (Left / Right) | `[ Pane 1 \| Pane 2 ]` |
| **`Ctrl+b` $\rightarrow$ `"`** | **Split Horizontally** (Top / Bottom) | `[ Pane 1 ] / [ Pane 2 ]` |
| **`Ctrl+b` $\rightarrow$ `Arrow Key`** | Move focus to adjacent pane ($\leftarrow, \rightarrow, \uparrow, \downarrow$). | Moves active cursor border. |
| **`Ctrl+b` $\rightarrow$ `z`** | **Toggle Zoom** (Maximize/Restore active pane). | Temporarily fullscreens a pane to read wide log lines. |
| **`Ctrl+b` $\rightarrow$ `x`** | **Kill/Close** active pane. | Prompts `kill-pane? (y/n)` to safely terminate. |
| **`Ctrl+b` $\rightarrow$ `Space`** | Cycle through preset pane layouts. | Re-arranges split grids evenly. |
| **`Ctrl+b` $\rightarrow$ `q`** | Show pane index numbers briefly. | Useful when managing 4+ splits. |

---

#### Session Detachment & Scrollback Buffer

- **Detach Safely:** **`Ctrl+b` $\rightarrow$ `d`**
  - Leaves everything running in the background and returns you to your normal shell.
- **Enter Scroll / Copy Mode:** **`Ctrl+b` $\rightarrow$ `[`**
  - Allows you to scroll up through historical terminal output using PageUp/PageDown or arrow keys.
  - Press **`q`** to exit copy mode and return to live interactive typing.

---

### Customizing `~/.tmux.conf` for DevOps Productivity

Create `~/.tmux.conf` to enable mouse scrolling, 256-color support, and vi keybindings:

```ini
# ~/.tmux.conf

# 1. Enable mouse control (click to select panes, drag to resize, scroll wheel)
set -g mouse on

# 2. Increase scrollback buffer history limit (default is 2000 lines)
set -g history-limit 50000

# 3. Set terminal color mode
set -g default-terminal "screen-256color"

# 4. Use vi keybindings in copy mode (navigate with h, j, k, l)
setw -g mode-keys vi

# 5. Make pane splitting intuitive using | and -
bind | split-window -h -c "#{pane_current_path}"
bind - split-window -v -c "#{pane_current_path}"
```
To reload configuration without restarting tmux:
```bash
tmux source-file ~/.tmux.conf
```

---

### 🚀 DevOps Real-Time Scenario: Multi-Pane Incident Response & Safe Remote Deployments

**The Situation:** You are on-call and an incident triggers: `API Gateway Latency > 2500ms`. You SSH into the production bastion host.

#### Step 1: Open a Named Incident Session
```bash
tmux new -s incident-404
```

#### Step 2: Split Panes for Real-Time Triage
1. In Pane 1: Tail the live application error logs:
   ```bash
   tail -f /var/log/nginx/error.log
   ```
2. Press **`Ctrl+b` then `%`** (split vertically):
   In Pane 2: Monitor system hardware and CPU states:
   ```bash
   htop
   ```
3. Press **`Ctrl+b` then `"`** (split bottom right):
   In Pane 3: Run debugging queries and curl health checks:
   ```bash
   curl -w "@curl-format.txt" -o /dev/null -s http://localhost:8080/health
   ```
4. Even if your VPN drops, all 3 monitors remain active and intact on the server. You can reconnect instantly with `tmux a -t incident-404`.

---

## 2. Custom Process Introspection with `ps -o` / `ps -eo`

### Why Custom Formatting (`-o`) is Superior for Automation

While `ps aux` prints fixed columns, the `-o` (or `--format`) flag allows you to construct custom, streamlined tables containing only the exact process parameters required for monitoring scripts, Prometheus textfile collectors, or incident investigation.

```bash
# Syntax
ps -eo <field1>,<field2>,<field3>... --sort=<[-]field>
```

- **`-e`**: Select all processes on the system.
- **`-o`**: Specify user-defined output format.
- **`--sort=-<field>`**: Sort output descending (`-`) or ascending (`+`).

---

### Core Output Specifiers & Formatting Tokens

| Specifier Code | Header Name | Meaning & Details |
| :--- | :--- | :--- |
| **`pid`** | `PID` | Process ID. |
| **`ppid`** | `PPID` | Parent Process ID (essential for diagnosing orphaned/zombie processes). |
| **`user`** | `USER` | Effective username running the process. |
| **`ruser`** | `RUSER` | Real username (identifies who originally launched it before `setuid`). |
| **`stat`** | `STAT` | Multi-character process state (`R`, `S`, `D`, `Z`, `T`, `s`, `l`, `+`). |
| **`%cpu`** | `%CPU` | Real-time CPU percentage utilization. |
| **`%mem`** | `%MEM` | Physical RAM percentage utilization. |
| **`vsz`** | `VSZ` | Virtual Memory Size in KiB. |
| **`rss`** | `RSS` | **Resident Set Size** (Actual physical RAM occupied) in KiB. |
| **`etime`** | `ELAPSED` | Total elapsed runtime since launch (`[[DD-]hh:]mm:ss`). |
| **`cputime` / `time`**| `TIME` | Cumulative CPU time consumed. |
| **`wchan`** | `WCHAN` | Name of the specific **Linux Kernel function** the process is sleeping in. |
| **`comm`** | `COMMAND` | Short executable name (e.g., `python3`). |
| **`args`** | `COMMAND` | Complete command line invocation including all flags and paths. |

---

### High-Yield Production Query Recipes

#### 1. Find Top 10 Memory-Consuming Processes (Ranked by Physical RAM `RSS`)
```bash
ps -eo pid,user,stat,%mem,rss,vsz,comm --sort=-rss | head -n 11
```
*Output:*
```text
  PID USER     STAT %MEM   RSS    VSZ COMMAND
 5412 postgres Ssl  12.4 982400 1250420 postgres
 3120 node     Sl    8.1 645120  980140 node
 1280 redis    Ssl   4.2 334100  450200 redis-server
```

#### 2. Find Process Uptime / Elapsed Execution Time (`etime`)
Determine exactly how long a worker or background script has been running:
```bash
ps -eo pid,user,etime,cputime,comm | grep python3
```
*Output:*
```text
 4510 kshitiz   02-04:15:22 00:32:10 python3
```
*(Indicates process `4510` has been alive for 2 days, 4 hours, 15 minutes, and 22 seconds, having used 32 minutes of active CPU execution time).*

#### 3. Inspect Kernel Wait Channels (`wchan`) for Stuck/Frozen Processes
If a process is unresponsive in `D` or `S` state, `wchan` reveals the exact kernel function it is blocked on:
```bash
ps -eo pid,user,stat,wchan:20,comm
```
*Output:*
```text
  PID USER     STAT WCHAN                COMMAND
 6120 postgres D    nfs_wait_bit_uninter nfs_backup
 3180 kshitiz  S    poll_schedule_timeou bash
```
*(Instantly shows process `6120` is frozen waiting on an NFS network storage response).*

---

## 3. Deep Dive: Linux Memory Internals (`VSZ`, `RSS`, `PSS`, `USS`)

### The Memory Hierarchy: Virtual vs Physical Address Spaces

When a process asks the Linux kernel for memory (e.g., via `malloc()` or `mmap()`), the kernel does **not** allocate physical hardware RAM immediately.

Instead, the kernel allocates **Virtual Memory (`VSZ`)** and updates the process's page table. Physical RAM pages (**`RSS`**) are only allocated lazily when the process actually writes data to that address, triggering a **Page Fault**.

```
+─────────────────────────────────────────────────────────────────────────────+
|                         VIRTUAL MEMORY (VSZ)                                |
|  (Total address space requested: Mallocs, file mappings, shared libraries)   |
+─────────────────────────────────────────────────────────────────────────────+
                                       │
                         [ Lazy Allocation / Paging ]
                                       │
                                       ▼
+─────────────────────────────────────────────────────────────────────────────+
|                     PHYSICAL HARDWARE RAM OCCUPIED                          |
|                                                                             |
|   +──────────────────────────────────+  +────────────────────────────────+  |
|   |    Unique Set Size (USS)         |  |    Shared Memory (SHR)         |  |
|   |    (Private unshared RAM pages   |  |    (Shared libraries, libc,    |  |
|   |     exclusive to this PID)       |  |     shared DB buffer pools)    |  |
|   +──────────────────────────────────+  +────────────────────────────────+  |
|   └────────────────────────────────┬─────────────────────────────────────┘  |
|                                    ▼                                        |
|                       Resident Set Size (RSS)                               |
+─────────────────────────────────────────────────────────────────────────────+
```

---

### Metric Comparison: `VSZ` vs `RSS` vs `SHR` vs `PSS` vs `USS`

| Metric | Name | Definition | Diagnostic Significance |
| :---: | :--- | :--- | :--- |
| **`VSZ`** | **Virtual Memory Size** | Total virtual address space mapped by the process (including code, data, shared libs, allocated but untouched pages). | Often inflated (e.g. Java JVM or Go runtime pre-allocating heaps). Not an indicator of RAM exhaustion. |
| **`RSS`** | **Resident Set Size** | **Total physical RAM** pages currently mapped into hardware RAM for this process. | Standard metric reported by `ps` and `top`. **Note:** Overcounts shared libraries when summing across multiple processes. |
| **`SHR`** | **Shared Memory** | Portion of `RSS` that is shared with other processes (e.g., `libc.so`, shared memory segments). | Helps determine how much of a process's footprint is reusable across forks. |
| **`PSS`** | **Proportional Set Size** | `USS` + (Shared Memory $\div$ Number of processes sharing it). | **Most accurate representation** of actual RAM cost per worker in multi-process architectures (like Nginx, Gunicorn, PostgreSQL). |
| **`USS`** | **Unique Set Size** | Private physical RAM belonging **exclusively** to this process. | If you kill this process, **this is the exact amount of RAM instantly freed** back to the kernel. |

---

### Extracting Memory Metrics via `/proc/<PID>/status` & `/proc/<PID>/smaps`

The `/proc` virtual filesystem provides direct, kernel-level introspection into process memory without running third-party utilities.

#### 1. Inspecting `/proc/<PID>/status`
```bash
# Inspect process PID 318 (e.g., bash)
cat /proc/318/status | grep -iE 'VmPeak|VmSize|VmRSS|RssAnon|RssFile|RssShmem|VmSwap'
```

*Output Breakdown:*
```text
VmPeak:     6312 kB    # Peak virtual memory size lifetime
VmSize:     6312 kB    # Current total virtual memory (VSZ)
VmRSS:      5120 kB    # Current physical RAM occupied (RSS)
RssAnon:    1664 kB    # Anonymous memory (heap/stack private to process)
RssFile:    3456 kB    # File-backed memory (executable code, dynamic libraries)
RssShmem:      0 kB    # Shared memory segments
VmSwap:        0 kB    # Amount of memory swapped out to disk
```

#### 2. Detailed Memory Accounting via `/proc/<PID>/smaps_rollup` (Kernel 4.14+)
`smaps_rollup` gives an instant, lightweight summary of `PSS`, `USS`, and `RSS`:
```bash
sudo cat /proc/318/smaps_rollup
```
*Output:*
```text
Rss:                5120 kB
Pss:                2340 kB
Pss_Anon:           1664 kB
Pss_File:            676 kB
Shared_Clean:       3456 kB
Private_Dirty:      1664 kB    # USS (Unique Set Size)
Swap:                  0 kB
```

---

### Step-by-Step Walkthrough: Tracing Memory Allocation Under the Hood

Let's observe how physical RAM (`RSS`) behaves differently from Virtual Memory (`VSZ`):

1. When a program allocates a 1GB array in C/Python:
   ```python
   # Memory allocated in virtual address space
   data = [0] * (1024 * 1024 * 256) # Requests ~1GB
   ```
2. **At Allocation Time:**
   - `VSZ` spikes by `+1 GB`.
   - `RSS` stays nearly unchanged (`~10 MB`) because the kernel has not yet mapped physical pages.
3. **At Write Time (When data is populated):**
   - As each page is written to, the CPU raises a minor page fault.
   - The kernel allocates a physical 4KB memory frame.
   - `RSS` rises to `1 GB`.

---

### 🚀 DevOps Real-Time Scenario: Diagnosing Silent Memory Leaks in Containers & Pods

**The Situation:** A Python Gunicorn worker pool inside a Kubernetes pod has a memory limit of `1Gi`. Over 12 hours, the pod is repeatedly killed with `Exit Code 137` (**OOMKilled**).

#### Root Cause Analysis Workflow:
1. **The Trap:** Looking at `VSZ` shows `1.8GB` even at startup because Python pre-allocates virtual heap address space.
2. **The Correct Inspection:** Query `RSS` and `Private_Dirty` (`USS`) across all worker PIDs:
   ```bash
   ps -eo pid,ppid,user,rss,vsz,comm --sort=-rss | grep gunicorn
   ```
3. **Automate Memory Tracking in a Loop:**
   ```bash
   while true; do
     date +"%H:%M:%S" >> mem.log
     ps -o pid,rss,comm -p $(pgrep -d',' gunicorn) >> mem.log
     sleep 60
   done
   ```
4. **Finding:** Each worker's `RSS` increases by 5MB for every 100 requests handled because global cache dictionaries are never cleared.
5. **Mitigation:**
   - Code fix: Implement bounded LRU caches (`functools.lru_cache(maxsize=1000)`).
   - Operational workaround: Configure Gunicorn worker recycling (`--max-requests 1000 --max-requests-jitter 100`) to restart workers before reaching the pod's OOM threshold.

---

## 4. DevOps Production Troubleshooting Cheat Sheet

| Task / Problem | Command / Technique | Why DevOps Engineers Use It |
| :--- | :--- | :--- |
| **Start Named Tmux Session** | `tmux new -s <name>` | Create isolated workspace for specific task/incident. |
| **Attach to Running Session** | `tmux a -t <name>` | Reconnect to persistent session after SSH drop. |
| **Split Pane Vertically** | Press `Ctrl+b` then `%` | Put code/command on left and output/logs on right. |
| **Split Pane Horizontally** | Press `Ctrl+b` then `"` | Put active shell on top and monitoring dashboard below. |
| **Zoom/Fullscreen Active Pane**| Press `Ctrl+b` then `z` | Inspect wide stack traces without messing up pane layout. |
| **Detach from Session Safely** | Press `Ctrl+b` then `d` | Exit terminal while leaving long-running jobs active. |
| **Find Top 10 RAM Users (RSS)**| `ps -eo pid,user,%mem,rss,comm --sort=-rss \| head -n 11` | Identify physical memory hogs instantly. |
| **Find Process Elapsed Time** | `ps -eo pid,user,etime,comm \| grep <app>` | Check exactly how long a service has been alive. |
| **Check Process Wait Channel** | `ps -eo pid,stat,wchan:20,comm` | Discover which kernel syscall/lock a stuck process is blocked on. |
| **Inspect Kernel Memory Status**| `cat /proc/<PID>/status \| grep -i vm` | Read exact kernel memory counters (`VmRSS`, `VmSize`, `VmSwap`). |
| **Extract True Unique Memory** | `sudo cat /proc/<PID>/smaps_rollup \| grep Private_Dirty` | Determine exact RAM freed if process is killed (`USS`). |

---

*Authored for Linux System Administrators & DevOps Engineers.*
