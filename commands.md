# 🐧 Linux Administration & DevOps Internal Mechanics

> Comprehensive hands-on notes exploring Linux process tracking, session management (`systemd-logind`), user/group permissions, and resource controls (`cgroups` & `systemd slices/scopes`) — annotated with real-time DevOps production scenarios.

---

## Table of Contents
1. [1. `w` — Real-Time User Activity & System Load](#1-w--real-time-user-activity--system-load)
   - [Command Output & Field-by-Field Breakdown](#command-output--field-by-field-breakdown)
   - [Deep Dive: Understanding Load Averages](#deep-dive-understanding-load-averages)
   - [🚀 DevOps Real-Time Scenario: Production Server Triage](#-devops-real-time-scenario-production-server-triage)
2. [2. `loginctl` — Systemd Login & Session Management](#2-loginctl--systemd-login--session-management)
   - [Command Output Breakdown](#command-output-breakdown)
   - [Essential `loginctl` Commands](#essential-loginctl-commands)
   - [What is a Session?](#what-is-a-session)
   - [What is a Scope?](#what-is-a-scope)
   - [🚀 DevOps Real-Time Scenario: CI/CD Runners & Lingering Sessions](#-devops-real-time-scenario-cicd-runners--lingering-sessions)
3. [3. User & Group Management (`useradd`, `passwd`, `usermod`)](#3-user--group-management-useradd-passwd-usermod)
   - [Low-Level Mechanics of `sudo useradd xyz`](#low-level-mechanics-of-sudo-useradd-xyz)
   - [Common Flags & Options Breakdown](#common-flags--options-breakdown)
   - [User Creation Walkthrough & Verification](#user-creation-walkthrough--verification)
   - [`useradd` vs `adduser` Explained](#useradd-vs-adduser-explained)
   - [Supplementary Group Management (`usermod`, `sudoers`)](#supplementary-group-management-usermod-sudoers)
   - [🚀 DevOps Real-Time Scenario: Least Privilege & Non-Root Containers](#-devops-real-time-scenario-least-privilege--non-root-containers)
4. [4. Linux Kernel, Cgroups & Systemd Hierarchy](#4-linux-kernel-cgroups--systemd-hierarchy)
   - [What is the Linux Kernel?](#what-is-the-linux-kernel)
   - [What are Control Groups (Cgroups)?](#what-are-control-groups-cgroups)
   - [Systemd Resource Hierarchy: Slices, Scopes, and Services](#systemd-resource-hierarchy-slices-scopes-and-services)
   - [Deconstructing `systemctl status session-1.scope`](#deconstructing-systemctl-status-session-1scope)
   - [Real-Time Monitoring with `systemd-cgtop`](#real-time-monitoring-with-systemd-cgtop)
   - [🚀 DevOps Real-Time Scenario: How Docker & Kubernetes Use Cgroups](#-devops-real-time-scenario-how-docker--kubernetes-use-cgroups)
5. [5. DevOps Production Troubleshooting Cheat Sheet](#5-devops-production-troubleshooting-cheat-sheet)

---

## 1. `w` — Real-Time User Activity & System Load

It displays information about the currently logged-in users and what process/command they are running on the system.

### Terminal Output
```bash
kshitiz@KZ:/mnt/c/Users/kshit$ w
 04:40:11 up 0 min,  1 user,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU  WHAT
kshitiz  pts/1    -                04:39   13.00s  0.03s  0.02s -bash
```

### Command Output & Field-by-Field Breakdown

#### Row 1: Header Line
The first line represents system-wide status:
- **`04:40:11`**: Current system clock time.
- **`up 0 min`**: System uptime (how long the system has been running since the last reboot).
- **`1 user`**: Total count of currently logged-in users.
- **`load average: 0.00, 0.00, 0.00`**: System load averages for the past **1 minute**, **5 minutes**, and **15 minutes**.

#### Row 2+: User Info Table
| Column | Meaning | Details & Context |
| :--- | :--- | :--- |
| **`USER`** | Username | The account currently logged in (`kshitiz`). |
| **`TTY`** | Pseudo-Terminal / Terminal | `pts/1` stands for *pseudo-terminal slave 1* (virtual terminal spawned via SSH, WSL, or terminal emulator). A physical terminal would show `tty1`. |
| **`FROM`** | Remote Host / IP | `-` indicates local host connection (e.g., WSL2 or local socket); if logged in remotely via SSH, this shows the client IP address (e.g., `192.168.1.50`). |
| **`LOGIN@`** | Login Time | The timestamp when this specific session was initialized (`04:39`). |
| **`IDLE`** | Idle Time | How long since the user last interacted with the terminal (`13.00s`). |
| **`JCPU`** | Joint CPU Time | Total CPU time consumed by **all processes** attached to this particular TTY/session. |
| **`PCPU`** | Process CPU Time | CPU time consumed specifically by the **currently active process** listed in the `WHAT` column. |
| **`WHAT`** | Current Process | The command or shell currently running (`-bash`). The leading `-` indicates a login shell. |

---

### Deep Dive: Understanding Load Averages

Load average is **not** purely CPU percentage — it measures the average number of processes that are in a **runnable state** (using CPU or waiting for CPU) plus processes in an **uninterruptible sleep state** (usually waiting for disk/network I/O).

```
Formula Rule of Thumb:
Target Load Average <= Total Number of CPU Cores
```

- **If you have 4 CPU cores:**
  - Load average = `2.00`: The system is at 50% capacity.
  - Load average = `4.00`: The system is at 100% capacity (fully utilized).
  - Load average = `8.00`: 4 processes are running, and 4 processes are queuing up waiting for CPU/Disk.

#### Trend Analysis:
- `load average: 4.50, 2.00, 0.80` $\rightarrow$ Load is **increasing** sharply (1m > 5m > 15m). Potential traffic spike or runaway thread.
- `load average: 0.50, 2.10, 4.20` $\rightarrow$ Load was high in the past, but the system is now **recovering** (1m < 5m < 15m).

---

### 🚀 DevOps Real-Time Scenario: Production Server Triage

**The Situation:** PagerDuty fires an alert: `NodeHighLoadAverage: Production Web Server 03`.

1. **Step 1 — Immediate inspection with `w`:**
   ```bash
   w
   ```
   *You notice a rogue developer or automated script running a heavy query:*
   ```text
   USER       TTY      FROM             LOGIN@   IDLE   JCPU    PCPU   WHAT
   deployer   pts/2    10.0.1.45        12:10    0.00s  12m40s  12m35s python3 migrate_db.py
   ```
2. **Step 2 — Identify & Resolve:**
   You immediately see who started the process (`deployer`), from where (`10.0.1.45`), how much CPU it has eaten (`12m35s`), and the exact script (`python3 migrate_db.py`). You can now isolate or kill the process without blindly rebooting the server.

---

## 2. `loginctl` — Systemd Login & Session Management

`loginctl` is part of `systemd` and provides inspection and control over user login sessions managed by the systemd login manager daemon (`systemd-logind`).

### Terminal Output
```bash
SESSION  UID USER    SEAT TTY   STATE  IDLE SINCE
      1 1000 kshitiz -    pts/1 active no   -

1 sessions listed.
```

### Command Output Breakdown

- **`SESSION`**: Session ID number (`1`) dynamically assigned by `systemd-logind`.
- **`UID`**: User ID (`1000` is the default standard UID for the first regular non-root user created on Linux).
- **`USER`**: Username attached to the session (`kshitiz`).
- **`SEAT`**: Physical hardware seat (keyboard, monitor, graphics card). `-` indicates that there is no physical hardware seat attached (e.g., remote SSH connection, headless server, or virtualized WSL environment).
- **`TTY`**: The allocated terminal device (`pts/1`).
- **`STATE`**: Current state (`active` = user is actively logged in and foregrounded; `online` = logged in but on another virtual terminal; `closing` = session terminating).
- **`IDLE`**: Whether the session is currently idle (`no` = user is actively sending input).
- **`SINCE`**: Timestamp when the user became idle (shows `-` when active).

---

### Essential `loginctl` Commands

```bash
# List all active login sessions on the system
loginctl list-sessions

# Show detailed metadata, cgroup path, and process tree of session 1
loginctl show-session 1

# List all users who currently have active or lingering sessions
loginctl list-users

# Inspect detailed status, cgroup slice, and child processes of a specific user
loginctl user-status kshitiz

# Lock a specific desktop/terminal session
loginctl lock-session 1

# Forcefully terminate/kill a session and all its spawned processes
loginctl terminate-session 1

# Terminate all sessions belonging to a specific user
loginctl terminate-user kshitiz
```

---

### What is a Session?

> **Definition:** A **Session** in Linux is a collection of processes associated with an interactive or non-interactive login instance of a user. 

When you log into Linux (via SSH, GUI, or WSL), PAM (`Pluggable Authentication Modules`) contacts `systemd-logind`. `systemd-logind` assigns a unique Session ID, allocates a pseudo-terminal (`pts`), sets up environment variables (`XDG_RUNTIME_DIR`), and creates a dedicated **cgroup scope** (e.g., `session-1.scope`) to track every process you start during that login.

---

### What is a Scope?

> **Definition:** A **Scope** is a transient (temporary) `systemd` unit that groups and manages processes that were started **externally** (not started directly by systemd from a configuration file, but spawned by user logins, SSH sessions, or container runtimes).

- **Difference between Service and Scope:**
  - **`.service`**: Systemd reads a unit file from `/etc/systemd/system/` and starts/manages the daemon itself (e.g., `nginx.service`, `docker.service`).
  - **`.scope`**: Systemd did **not** start the process directly; rather, an external program (like SSH daemon or `logind`) created processes, and systemd registers them into a `.scope` on the fly to monitor and apply resource limits.
  - As soon as the user logs out or the process exits, the `.scope` automatically terminates and cleans up.

---

### 🚀 DevOps Real-Time Scenario: CI/CD Runners & Lingering Sessions

**The Situation:** You run self-hosted GitHub Actions runners or Jenkins agents on an Ubuntu server under a dedicated service user `github-runner`.

By default, when `github-runner` logs out or SSH disconnects, systemd terminates all background user processes. This can kill running docker builds or background processes!

#### The Fix: Enable User Lingering
```bash
# Allow user processes to run in the background even after SSH logout
sudo loginctl enable-linger github-runner

# Verify lingering status
loginctl show-user github-runner --property=Linger
# Output: Linger=yes
```

> [!TIP]
> `enable-linger` tells `systemd-logind` to keep a persistent `user@UID.service` running at boot, allowing background tasks, cron jobs, and rootless Podman/Docker containers to survive without an active interactive login session.

---

## 3. User & Group Management (`useradd`, `passwd`, `usermod`)

Managing Linux users and permissions is a core foundation of system security and automation.

### Low-Level Mechanics of `sudo useradd xyz`

When you run the raw command:
```bash
sudo useradd xyz
```

Here is what Linux does under the hood:
1. **Creates an entry in `/etc/passwd`** for `xyz` (contains username, UID, GID, home dir path, login shell).
2. **Creates an entry in `/etc/shadow`** with a locked/disabled password hash (`!`), preventing login until a password or SSH key is set.
3. **Creates a private group in `/etc/group`** with the same name (`xyz`) and GID matching the UID (on standard modern Linux distributions).
4. **Assigns a UID automatically** $\ge 1000$ (reserving $0-999$ for system daemons).
5. **Does NOT** create a `/home/xyz` directory by default unless `-m` is explicitly passed.
6. **Does NOT** prompt interactively for a password.

---

### Common Flags & Options Breakdown

| Flag | Description | Command Example | DevOps Use Case |
| :--- | :--- | :--- | :--- |
| **`-m`** | Create user's home directory (`/home/username`) | `sudo useradd -m xyz` | Standard developer interactive account. |
| **`-d`** | Specify custom home directory path | `sudo useradd -d /var/www/app xyz` | Pointing an app worker to its project root. |
| **`-s`** | Specify default login shell | `sudo useradd -s /bin/bash xyz` | Granting Bash access vs disabling shell access. |
| **`-g`** | Specify primary group name or GID | `sudo useradd -g developers xyz` | Assigning team-wide default file permissions. |
| **`-G`** | Specify supplementary (secondary) groups | `sudo useradd -G sudo,docker xyz` | Granting Docker and Sudo privileges. |
| **`-u`** | Specify an exact, custom UID | `sudo useradd -u 1500 xyz` | Ensuring identical UIDs across a Kubernetes cluster / NFS share. |
| **`-c`** | Add a comment / GECOS info (Full Name/Dept) | `sudo useradd -c "CI Runner Service" xyz` | Documenting account ownership and purpose. |
| **`-e`** | Set account expiration date (`YYYY-MM-DD`) | `sudo useradd -e 2026-12-31 xyz` | Creating temporary access for external contractors. |
| **`-r`** | Create a system account (UID < 1000, no home) | `sudo useradd -r -s /usr/sbin/nologin app` | Creating non-login daemons for security. |

---

### User Creation Walkthrough & Verification

#### Step 1: Create the User with Home Directory and Bash Shell
```bash
sudo useradd -m -s /bin/bash testuser
```

#### Step 2: Set the User Password
```bash
sudo passwd testuser
```
*(You will be prompted to enter and confirm the new password).*

#### Step 3: Switch/Login to the New User
```bash
su - testuser
```
> [!NOTE]
> The `-` (or `-l` / `--login`) flag is crucial: it starts a complete **login shell**, loading `testuser`'s own environment variables, `$HOME`, and `.bashrc`.

#### Step 4: Verify with `loginctl`
```bash
loginctl list-users
```
*Output will now show both logged-in users:*
```text
 UID USER
1000 kshitiz
1001 testuser

2 users listed.
```

#### Step 5: Switch Back
Type `exit` or press `Ctrl + D` to return to the original user session (`kshitiz`).

---

### `useradd` vs `adduser` Explained

| Feature | `useradd` (Low-Level Binary) | `adduser` (High-Level Interactive Wrapper) |
| :--- | :--- | :--- |
| **Type** | Native compiled C binary (`/usr/sbin/useradd`). | High-level Perl script (standard on Debian/Ubuntu). |
| **Behavior** | Non-interactive. Requires explicit flags (`-m`, `-s`). | Friendly interactive wizard. Prompts for password, name, phone, etc. |
| **Home Dir** | Not created by default unless `-m` is passed. | Automatically created by default from `/etc/skel`. |
| **Best For** | **DevOps automation, Dockerfiles, Ansible scripts, CI/CD pipelines**. | **Manual interactive system administration on desktop/servers**. |

---

### Supplementary Group Management (`usermod`, `sudoers`)

To modify existing users and grant permissions:

```bash
# Add an existing user to the docker group (without removing existing groups using -a)
sudo usermod -aG docker testuser

# Add an existing user to the sudoers group
sudo usermod -aG sudo testuser

# Lock a user's account (prevents login)
sudo usermod -L testuser

# Unlock a user's account
sudo usermod -U testuser

# Delete a user and remove their home directory
sudo userdel -r testuser
```

> [!WARNING]
> Always use the `-a` (append) flag together with `-G` (`usermod -aG groupname username`). If you omit `-a`, the user will be removed from all other supplementary groups!

---

### 🚀 DevOps Real-Time Scenario: Least Privilege & Non-Root Containers

**The Golden Rule of Container Security:** Never run production workloads as `root`. If an attacker exploits a remote code execution vulnerability in your app, they gain root access to the host kernel if container breakout occurs.

#### Real-World Dockerfile Example:
```dockerfile
FROM node:20-alpine

# 1. Create a dedicated system group and user without root privileges
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# 2. Set up application directory
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

COPY . .

# 3. Change file ownership to non-root user
RUN chown -R appuser:appgroup /app

# 4. Switch from root to non-root user
USER appuser

EXPOSE 3000
CMD ["node", "server.js"]
```

---

## 4. Linux Kernel, Cgroups & Systemd Hierarchy

```
+-------------------------------------------------------------+
|                        LINUX KERNEL                         |
|     (Memory Management, CPU Scheduling, Cgroups, Namespaces)|
+-------------------------------------------------------------+
                              |
+-------------------------------------------------------------+
|                        SYSTEMD (PID 1)                      |
|                  Organizes system into Slices               |
+-------------------------------------------------------------+
                              |
         +--------------------+--------------------+
         |                                         |
+-------------------+                     +-------------------+
|    user.slice     |                     |   system.slice    |
| (All User Logins) |                     | (System Daemons)  |
+-------------------+                     +-------------------+
         |                                         |
+--------------------+                    +-------------------+
|  user-1000.slice   |                    |   docker.service  |
|  (UID 1000 Slice)  |                    |   nginx.service   |
+--------------------+                    +-------------------+
         |
+--------------------+
|  session-1.scope   |
| (Interactive WSL / |
|   SSH Terminal)    |
+--------------------+
```

---

### What is the Linux Kernel?

> **Definition:** The **Kernel** is the core engine of the operating system that sits directly between the computer hardware and software applications.

- **Primary Responsibilities:**
  1. **CPU Scheduling:** Deciding which process gets time on CPU cores.
  2. **Memory Management:** Allocating virtual memory pages and preventing processes from overwriting each other.
  3. **Device Drivers:** Communicating with disks, network interfaces, GPUs, keyboards.
  4. **System Calls (Syscalls):** Providing secure APIs (`read`, `write`, `fork`, `clone`) so user applications can request hardware actions.
  5. **Resource Control & Isolation:** Implementing **Cgroups** (resource limits) and **Namespaces** (network, mount, PID, IPC isolation).

---

### What are Control Groups (Cgroups)?

> **Definition:** **Cgroups (Control Groups)** is a Linux kernel feature that allows you to allocate, prioritize, limit, and monitor resource usage (CPU, RAM, Disk I/O, Network, number of processes) for a collection of processes.

Using Cgroups, we can restrict users and processes:
- Limit User A to max 2GB of RAM and 1 CPU core.
- Prevent a runaway script or fork bomb from freezing the entire operating system.
- Enforce different resource limits across multiple sessions using **systemd scopes**.

---

### Systemd Resource Hierarchy: Slices, Scopes, and Services

Systemd maps cgroups directly into a hierarchical tree using three primary unit types:

1. **`.slice` (Resource Partition Container):**
   - Slices don't run processes directly; they represent folders/trees in the cgroup hierarchy that define resource limits for everything underneath them.
   - `user.slice`: Top-level slice containing all interactive user sessions.
   - `user-1000.slice`: Specific sub-slice dedicated to all processes spawned by UID `1000`.
   - `system.slice`: Default slice for all background system services and daemons.

2. **`.scope` (Transient External Process Groups):**
   - Created on the fly for processes started outside of systemd (e.g., SSH sessions, WSL sessions, user login shells).

3. **`.service` (Managed Daemons):**
   - Background services started and monitored by systemd according to static unit files on disk (e.g., `nginx.service`, `k3s.service`).

---

### Deconstructing `systemctl status session-1.scope`

#### Terminal Output
```bash
systemctl status session-1.scope
● session-1.scope - Session 1 of User kshitiz
     Loaded: loaded (/run/systemd/transient/session-1.scope; transient)
  Transient: yes
     Active: active (running) since Wed 2026-09-23 04:39:59 UTC; 2h 24min ago
      Tasks: 2
     Memory: 2.3M (peak: 7.6M)
        CPU: 95ms
     CGroup: /user.slice/user-1000.slice/session-1.scope
             ├─257 /bin/login -f
             └─325 -bash

Sep 23 04:39:59 KZ systemd[1]: Started session-1.scope - Session 1 of User kshitiz.
```

#### Line-by-Line Breakdown:

- **`Loaded: loaded (/run/systemd/transient/session-1.scope; transient)` & `Transient: yes`:**
  - It means the unit file was **not** loaded from a permanent configuration file stored on disk (e.g., `/etc/systemd/system/`).
  - Instead, it was generated dynamically in volatile memory (`/run/systemd/transient/`) at runtime by `systemd-logind` when the user session started.
  - **Lifecycle:** As soon as the user logs out or the session ends, the scope and its transient unit file are completely destroyed.

- **`Tasks: 2`:**
  - Total number of active threads/processes running inside this cgroup.
  - In WSL, pseudo-terminals (`pts/0`, `pts/1`, etc.) are managed under the main WSL host bridge, so `Tasks: 2` represents the login wrapper and the current Bash shell.

- **`Memory: 2.3M (peak: 7.6M)`:**
  - Real-time RAM consumption by processes in this scope (currently `2.3 MB`, with a lifetime peak of `7.6 MB`).

- **`CPU: 95ms`:**
  - Total cumulative CPU time spent executing code across all processes inside this scope since initialization.

- **`CGroup: /user.slice/user-1000.slice/session-1.scope`:**
  - Shows the exact control group hierarchy in the kernel:
    - `/user.slice` $\rightarrow$ The top-level container for all user sessions.
    - `/user-1000.slice` $\rightarrow$ The resource container specific to UID 1000 (`kshitiz`).
    - `/session-1.scope` $\rightarrow$ The container isolating this specific login session.

- **`├─257 /bin/login -f`:**
  - The login process. The `-f` flag stands for **fast login** (the user was already authenticated via WSL/SSH keys, so it bypassed the interactive password prompt).

- **`└─325 -bash`:**
  - The interactive Bash login shell running under PID `325`.

---

### Real-Time Monitoring with `systemd-cgtop`

To view real-time CPU, Memory, and Disk I/O consumption organized by control group:

```bash
# Monitor the entire system cgroup hierarchy in real time (like htop, but for cgroups)
systemd-cgtop

# Monitor only a specific slice or scope
systemd-cgtop /user.slice/user-1000.slice/session-1.scope
```

#### Key Metrics Reported by `systemd-cgtop`:
- **`Tasks`**: Active process count in that slice/scope.
- **`%CPU`**: Current CPU percentage utilization.
- **`Memory`**: Current RAM consumption.
- **`Input/s` & `Output/s`**: Disk read/write throughput per cgroup.

---

### 🚀 DevOps Real-Time Scenario: How Docker & Kubernetes Use Cgroups

Containers are not separate virtual machines; **a container is simply a standard Linux process isolated with Namespaces and bounded by Cgroups**.

```
Kubernetes Pod Resource Limits
             │
             ▼
Docker / Containerd Runtime
             │
             ▼
Linux Kernel Cgroups (cgroupv2)
  ├── cpu.max        (CPU throttling / limits)
  ├── memory.max     (Hard RAM limit -> triggers OOM Killer if exceeded)
  └── pids.max       (Prevents fork bombs)
```

#### What Happens When a Container Hits Its Memory Limit?
1. In Kubernetes, you define:
   ```yaml
   resources:
     limits:
       memory: "512Mi"
       cpu: "500m"
   ```
2. Container runtime sets the cgroup property: `memory.max = 536870912` (512MB).
3. If the app inside leaks memory and reaches 513MB, the Linux Kernel Cgroup manager steps in:
   - It invokes the **Kernel OOM (Out Of Memory) Killer**.
   - It sends `SIGKILL` (Signal 9) to the process.
   - Kubernetes marks the Pod status as **`OOMKilled`** (Exit Code 137).

#### Setting Dynamic Cgroup Limits with Systemd:
You can dynamically restrict any running service on a production Linux host without restarting it:
```bash
# Limit Nginx service to maximum 1GB of RAM and 50% of one CPU core
sudo systemctl set-property nginx.service MemoryMax=1G CPUQuota=50%
```

---

## 5. DevOps Production Troubleshooting Cheat Sheet

| Task | Linux Command | Why DevOps Engineers Use It |
| :--- | :--- | :--- |
| **Check System Load & Logged-in Users** | `w` or `uptime` | Quick check during high CPU alerts to see who is on the box and load trends. |
| **Inspect User Sessions** | `loginctl list-sessions` | Audit open SSH connections on bastion/jump servers. |
| **Kill Rogue SSH Session** | `sudo loginctl terminate-session <ID>` | Instantly kill all processes started by a stuck or compromised session. |
| **Keep Background Jobs Alive After Logout** | `sudo loginctl enable-linger <username>` | Crucial for self-hosted GitHub Runners, Jenkins agents, and rootless Podman. |
| **Create Automation Service User** | `sudo useradd -r -s /usr/sbin/nologin -d /app appuser` | Follows Principle of Least Privilege for microservices and apps. |
| **Grant Docker Access to User** | `sudo usermod -aG docker <username>` | Allows CI runner to build containers without needing full `sudo` privileges. |
| **Inspect Process Cgroup Tree** | `systemctl status <unit>.service` | View RAM usage, task count, and child PIDs grouped cleanly. |
| **Real-Time Resource Breakdown by CGroup** | `systemd-cgtop` | Discover which container, pod, or user is monopolizing CPU/RAM. |
| **Adjust Resource Limits on the Fly** | `sudo systemctl set-property <unit> MemoryMax=2G` | Prevent memory leaks from taking down critical production nodes. |

---

*Authored for Linux System Administrators & DevOps Engineers.*
