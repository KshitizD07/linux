# 🐧 Linux System Call Auditing, Kernel Sandboxing, & Process Internals

> Comprehensive hands-on notes exploring Linux system call profiling (`strace`), security access control models (DAC vs. MAC vs. `seccomp`), file integrity monitoring (`inotify` vs. `auditd`), and `/proc` virtual pseudo-filesystem internals — annotated with real-time DevOps production scenarios.

---

## Table of Contents
1. [1. `strace` — System Call Tracing & Resource Profiling](#1-strace--system-call-tracing--resource-profiling)
   - [Profiling Execution: `strace -c date`](#profiling-execution-strace--c-date)
   - [Dissecting Socket & Network Syscalls: `strace -c nc -l 1234`](#dissecting-socket--network-syscalls-strace--c-nc--l-1234)
   - [Deep Dive: Essential Linux Syscalls Reference](#deep-dive-essential-linux-syscalls-reference)
   - [🚀 DevOps Real-Time Scenario: Diagnosing Stalled Microservices with `strace`](#-devops-real-time-scenario-diagnosing-stalled-microservices-with-strace)
2. [2. Linux Security Models: DAC vs. MAC vs. Seccomp](#2-linux-security-models-dac-vs-mac-vs-seccomp)
   - [Discretionary Access Control (DAC)](#discretionary-access-control-dac)
   - [Mandatory Access Control (MAC)](#mandatory-access-control-mac)
   - [Syscall Sandboxing with `seccomp` (Secure Computing Mode)](#syscall-sandboxing-with-seccomp-secure-computing-mode)
   - [Comparative Matrix: Security Access Layers](#comparative-matrix-security-access-layers)
   - [🚀 DevOps Real-Time Scenario: Container Hardening & Seccomp Profiles](#-devops-real-time-scenario-container-hardening--seccomp-profiles)
3. [3. File Auditing: `inotifywait` vs. Linux Audit Subsystem (`auditd`)](#3-file-auditing-inotifywait-vs-linux-audit-subsystem-auditd)
   - [Real-Time Inode Monitoring with `inotifywait`](#real-time-inode-monitoring-with-inotifywait)
   - [Enterprise Auditing with `auditctl` & `auditd`](#enterprise-auditing-with-auditctl--auditd)
   - [WSL2 vs. Native Linux Audit Subsystem Mechanics](#wsl2-vs-native-linux-audit-subsystem-mechanics)
   - [🚀 DevOps Real-Time Scenario: Detecting Unauthorized Configuration Drift](#-devops-real-time-scenario-detecting-unauthorized-configuration-drift)
4. [4. Process Anatomy & The `/proc` Pseudo-Filesystem](#4-process-anatomy--the-proc-pseudo-filesystem)
   - [What is a PID (Process Identifier)?](#what-is-a-pid-process-identifier)
   - [How the Kernel Mounts and Generates `/proc` (Procfs)](#how-the-kernel-mounts-and-generates-proc-procfs)
   - [Deconstructing the `/proc/<PID>/` Anatomy](#deconstructing-the-procpid-anatomy)
   - [🚀 DevOps Real-Time Scenario: Zero-Downtime Troubleshooting via `/proc`](#-devops-real-time-scenario-zero-downtime-troubleshooting-via-proc)
5. [5. DevOps Production Troubleshooting & Security Cheat Sheet](#5-devops-production-troubleshooting--security-cheat-sheet)

---

## 1. `strace` — System Call Tracing & Resource Profiling

In Linux architecture, user-space applications (Node.js, Go, Python, Nginx, or standard binaries like `date`) cannot directly access physical hardware (CPU, RAM, Disks, Network Interfaces). Every request to read a file, allocate heap memory, or open a network connection must go through a **System Call (Syscall)** to the Linux Kernel.

`strace` is a diagnostic, debugging, and instructional userspace utility used to intercept, record, and profile all system calls made by a process and the signals received by it.

---

### Profiling Execution: `strace -c date`

The `-c` (or `--summary-only`) flag counts time, calls, and errors for each system call and presents an aggregated statistical summary upon program completion.

#### Terminal Output
```bash
kshitiz@KZ:/mnt/c/Users/kshit/cs/linux$ strace -c date
Mon Sep 28 06:13:00 UTC 2026
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 24.77    0.001042          33        31        13 openat
 17.81    0.000749          35        21           mmap
 15.45    0.000650         650         1           execve
 13.27    0.000558          27        20           fstat
 11.84    0.000498          24        20           close
  2.85    0.000120          24         5           read
  2.52    0.000106          35         3           mprotect
  1.93    0.000081          27         3           brk
  1.40    0.000059          59         1           set_robust_list
  1.26    0.000053          53         1           write
  1.00    0.000042          21         2           pread64
  0.90    0.000038          38         1           munmap
  0.74    0.000031          31         1         1 access
  0.74    0.000031          31         1           set_tid_address
  0.59    0.000025          25         1           arch_prctl
  0.59    0.000025          25         1           futex
  0.59    0.000025          25         1           prlimit64
  0.59    0.000025          25         1           getrandom
  0.57    0.000024          24         1           lseek
  0.57    0.000024          24         1           rseq
------ ----------- ----------- --------- --------- ----------------
100.00    0.004206          35       117        14 total
```

#### Field-by-Field Breakdown

| Column | Meaning | Details & Context |
| :--- | :--- | :--- |
| **`% time`** | Percentage of System Time | The percentage of total CPU kernel time spent executing this specific syscall. |
| **`seconds`** | Total Accumulated Time | Total elapsed time (in seconds) spent handling this syscall during the run. |
| **`usecs/call`** | Microseconds Per Call | Average latency (in $\mu s$, microseconds) taken by each individual invocation. |
| **`calls`** | Invocation Count | Total number of times this syscall was executed by the binary and dynamic linkers. |
| **`errors`** | Failed Syscalls | Number of calls that returned an error code (e.g., `-ENOENT` when searching dynamic library search paths). |
| **`syscall`** | Kernel Syscall Name | The standard Linux kernel system call function executed. |

---

### Dissecting Socket & Network Syscalls: `strace -c nc -l 1234`

Running `nc -l 1234` creates an Inter-Process Communication (IPC) TCP listener on port 1234.

#### Terminal Output
```bash
kshitiz@KZ:/mnt/c/Users/kshit/cs/linux$ strace -c nc -l 1234
^Cstrace: Process 1393 detached
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
  0.00    0.000000           0         4           read
  0.00    0.000000           0         5           close
  0.00    0.000000           0         5           fstat
  0.00    0.000000           0        22           mmap
  0.00    0.000000           0         6           mprotect
  0.00    0.000000           0         1           munmap
  0.00    0.000000           0         3           brk
  0.00    0.000000           0         1           rt_sigaction
  0.00    0.000000           0         2           pread64
  0.00    0.000000           0         1         1 access
  0.00    0.000000           0         1           socket
  0.00    0.000000           0         1           bind
  0.00    0.000000           0         1           listen
  0.00    0.000000           0         2           setsockopt
  0.00    0.000000           0         1           execve
  0.00    0.000000           0         1           arch_prctl
  0.00    0.000000           0         1           set_tid_address
  0.00    0.000000           0         5           openat
  0.00    0.000000           0         1           set_robust_list
  0.00    0.000000           0         1         1 accept4
  0.00    0.000000           0         1           prlimit64
  0.00    0.000000           0         1           getrandom
  0.00    0.000000           0         1           rseq
------ ----------- ----------- --------- --------- ----------------
100.00    0.000000           0        68         2 total
```

---

### Deep Dive: Essential Linux Syscalls Reference

```
+------------------------------------------------------------------------+
|                          SOCKET LIFECYCLE                              |
+------------------------------------------------------------------------+
|  User Space (Netcat / Web Application Server)                          |
|     │                                                                  |
|     ├── 1. socket()     ──> Create network communication endpoint      |
|     ├── 2. setsockopt() ──> Set socket flags (e.g. SO_REUSEADDR)       |
|     ├── 3. bind()       ──> Bind socket to IP 0.0.0.0 & Port 1234      |
|     ├── 4. listen()     ──> Mark socket as passive listener queue      |
|     └── 5. accept4()    ──> Block until new client TCP connection      |
|                                                                        |
|  Kernel Space (Networking Stack & VFS Layer)                           |
+------------------------------------------------------------------------+
```

| Syscall | Subsystem | Low-Level Kernel Mechanics |
| :--- | :--- | :--- |
| **`execve`** | Process Control | Replaces current process memory with the new executable binary, initializing the stack, heap, and registers. |
| **`openat`** | Virtual File System (VFS) | Opens a file path relative to a directory file descriptor (e.g., dynamically loading `libc.so.6` or `/etc/localtime`). |
| **`mmap` / `munmap`** | Memory Subsystem | Maps files or anonymous memory pages into the process virtual address space. |
| **`mprotect`** | Memory Security | Changes access protections (`PROT_READ`, `PROT_WRITE`, `PROT_EXEC`) on memory regions to prevent code injection. |
| **`brk`** | Memory Allocation | Changes the location of the program break, allocating or deallocating heap memory. |
| **`socket`** | Network / IPC | Allocates a socket descriptor in the kernel network subsystem. |
| **`bind`** | Network / IPC | Binds an address (IP and Port) to the allocated socket descriptor. |
| **`listen`** | Network / IPC | Flags socket to accept incoming connection requests with a configured backlog limit. |
| **`accept4`** | Network / IPC | Extracts the first connection request on the queue of pending connections and returns a new socket descriptor. |
| **`read` / `write`** | I/O Subsystem | Reads or writes raw bytes to/from file descriptors (disk files, pipes, or sockets). |

---

### 🚀 DevOps Real-Time Scenario: Diagnosing Stalled Microservices with `strace`

**The Situation:** A Golang payment microservice suddenly stops serving incoming requests on Kubernetes, hanging with 0% CPU consumption and no logs emitted.

1. **Step 1 — Identify the PID inside the container/node:**
   ```bash
   ps aux | grep payment-service
   ```
2. **Step 2 — Attach `strace` to the running process without restarting:**
   ```bash
   sudo strace -p 4210 -f -e trace=network,file,poll,futex
   ```
   *Terminal captures where the thread is blocked:*
   ```text
   [pid  4212] futex(0x13d8090, FUTEX_WAIT_PRIVATE, 0, NULL <unfinished ...>
   [pid  4213] connect(7, {sa_family=AF_INET, sin_port=htons(5432), sin_addr=inet_addr("10.0.4.15")}, 16) = -1 ETIMEDOUT (Connection timed out)
   ```
3. **Step 3 — Identify & Resolve:**
   The output immediately proves the service is blocked on a deadlocked PostgreSQL connection attempting to reach `10.0.4.15:5432`. You resolve the network security group rule without taking down other containers.

---

## 2. Linux Security Models: DAC vs. MAC vs. Seccomp

Linux organizes access control into three complementary layers:

```
                            User / Process Action
                                     │
                                     ▼
                     [ 1. Syscall Filter: Seccomp ]
                                     │
                      (Allowed Syscall? e.g. openat)
                                     │
                                     ▼
             [ 2. Discretionary Access Control (DAC: rwx) ]
                                     │
                        (UID/GID ownership match?)
                                     │
                                     ▼
            [ 3. Mandatory Access Control (MAC: SELinux/AppArmor) ]
                                     │
                      (Enforces strict system policy)
                                     │
                                     ▼
                               Hardware / File
```

---

### Discretionary Access Control (DAC)

- **Definition:** The standard Unix permission model where file owners determine permissions (`rwx`) for user, group, and others.
- **Commands:** `chmod`, `chown`, `chgrp`.
- **Limitation:** DAC is based purely on UID/GID identity. If a compromised service runs as `root` (UID `0`), DAC checks are bypassed entirely.

```bash
# Removing all permissions from sensitive file
sudo chmod 000 /etc/confidential_data.txt
```

---

### Mandatory Access Control (MAC)

- **Definition:** A centralized, kernel-enforced security framework where access decisions are governed by system-wide security labels, regardless of user ownership or root status.
- **Implementations:** **SELinux** (Security-Enhanced Linux) and **AppArmor**.
- **DevOps Mechanism:** Even if an attacker gains root access inside a vulnerable web server, MAC policies prevent that process from reading `/etc/shadow` or executing binaries outside its authorized profile.

---

### Syscall Sandboxing with `seccomp` (Secure Computing Mode)

> **Definition:** **Seccomp** is a Linux kernel security facility that acts as a packet filter for system calls using Berkeley Packet Filter (`BPF`) programs.

Seccomp inspects syscall numbers and arguments *before* the kernel executes them:
- **`SCMP_ACT_ALLOW`**: Allows the syscall to proceed.
- **`SCMP_ACT_ERRNO`**: Blocks the syscall and returns an error code (e.g., `EPERM`) without terminating the process.
- **`SCMP_ACT_KILL`**: Immediately terminates the process with `SIGSYS`.

---

### Comparative Matrix: Security Access Layers

| Feature | DAC (`chmod` / `chown`) | MAC (SELinux / AppArmor) | Seccomp (Syscall Filtering) |
| :--- | :--- | :--- | :--- |
| **Enforcement Point** | File / Inode access | System objects & processes | Kernel Syscall boundary |
| **Managed By** | File Owner / Root | System Administrator Policy | Application / Container Runtime |
| **Protects Against** | Unauthorized user reads/writes | Privileged root process escalation | Kernel exploits & unauthorized syscalls |
| **Primary DevOps Context** | File/directory permissions | Host hardening & RHEL/Ubuntu security | Docker/Kubernetes pod security profiles |

---

### 🚀 DevOps Real-Time Scenario: Container Hardening & Seccomp Profiles

**The Situation:** Hardening a production microservice to prevent container breakouts and zero-day kernel exploits.

#### Step 1: Create a Custom Seccomp Profile (`seccomp-profile.json`)
```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": [
    "SCMP_ARCH_X86_64",
    "SCMP_ARCH_X86",
    "SCMP_ARCH_AARCH64"
  ],
  "syscalls": [
    {
      "names": [
        "accept4",
        "bind",
        "close",
        "epoll_create1",
        "epoll_ctl",
        "epoll_wait",
        "exit_group",
        "fstat",
        "futex",
        "listen",
        "mmap",
        "mprotect",
        "openat",
        "read",
        "setsockopt",
        "socket",
        "write"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

#### Step 2: Run Container with the Seccomp Profile
```bash
docker run --security-opt seccomp=seccomp-profile.json -d -p 8080:8080 my-api:v1.0
```

> [!TIP]
> Standard container runtimes (Docker, containerd, CRI-O) apply a default Seccomp profile that blocks ~44 dangerous system calls (such as `reboot`, `kexec_load`, `mount`, `acct`) out of 450+ available Linux syscalls.

---

## 3. File Auditing: `inotifywait` vs. Linux Audit Subsystem (`auditd`)

### Real-Time Inode Monitoring with `inotifywait`

`inotifywait` is part of the `inotify-tools` package and hooks directly into the Linux Kernel's `inotify` subsystem to watch filesystem events.

#### Terminal Execution
```bash
# Terminal 1: Watch /etc/passwd continuously for modifications or attribute changes
sudo inotifywait -m -e modify -e attrib /etc/passwd
```
```bash
# Terminal 2: Touch file to update metadata timestamp
sudo touch /etc/passwd
```

#### Output Captured in Terminal 1
```text
Setting up watches.
Watches established.
/etc/passwd ATTRIB
```

#### Event Breakdown:
- **`Setting up watches. Watches established.`**: The `inotify_init()` syscall created an inotify instance, and `inotify_add_watch()` bound a watch descriptor to the inode of `/etc/passwd`.
- **`-m` (`--monitor`)**: Keeps the process running continuously to capture multiple events instead of exiting after the first event.
- **`-e modify`**: Triggers when file content is written to or truncated.
- **`-e attrib`**: Triggers when file metadata (timestamps, permissions, extended attributes) is modified without changing content.

---

### Enterprise Auditing with `auditctl` & `auditd`

In enterprise environments requiring regulatory compliance (SOC 2, ISO 27001, PCI-DSS), file integrity monitoring must log **who executed the action**, **the process binary**, **parent PID**, and **timestamp**.

```bash
# 1. Add persistent audit rule watching /etc/passwd for read, write, attribute change
sudo auditctl -w /etc/passwd -p rwa -k identity_audit

# 2. View audit logs filtered by custom key
sudo ausearch -k identity_audit --interpret
```

---

### WSL2 vs. Native Linux Audit Subsystem Mechanics

| Capability | Standard WSL2 Environment | Native Linux Host / Production Cloud VM |
| :--- | :--- | :--- |
| **Audit Subsystem (`auditd`)** | **Disabled by default** in Microsoft's minimal custom kernel build to minimize virtualization overhead. | **Active & running** by default with netlink audit socket support. |
| **`auditctl` Command Status** | Returns: `Error sending add rule data request (No such file or directory)`. | Successfully registers audit hooks directly in kernel VFS layer. |
| **Recommended Tool** | **`inotifywait`** (Native inotify kernel subsystem is fully supported in WSL2). | **`auditd` / `auditctl` / Falco / Wazuh**. |

---

### 🚀 DevOps Real-Time Scenario: Detecting Unauthorized Configuration Drift

**The Situation:** In a Kubernetes node or production server, critical configuration files (e.g., `/etc/resolv.conf`, `/etc/nginx/nginx.conf`, `/etc/pam.d/common-auth`) should never be manually edited outside CI/CD pipelines.

#### Automated Inotify Drift Detection Script:
```bash
#!/usr/bin/env bash
TARGET_FILE="/etc/nginx/nginx.conf"

echo "[INFO] Monitoring ${TARGET_FILE} for unauthorized changes..."
sudo inotifywait -m -e modify -e attrib -e delete "${TARGET_FILE}" | while read -r path action file; do
    TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
    echo "[ALERT ${TIMESTAMP}] Unauthorized file modification detected: ${action} on ${path}${file}"
    # Send alert webhook to Slack / PagerDuty
    curl -s -X POST -H 'Content-type: application/json' \
      --data "{\"text\":\"[SECURITY ALERT] ${TARGET_FILE} modified via ${action}\"}" \
      https://hooks.slack.com/services/T00/B00/X00
done
```

---

## 4. Process Anatomy & The `/proc` Pseudo-Filesystem

### What is a PID (Process Identifier)?

> **Definition:** A **PID** is a unique positive integer allocated by the Linux kernel to represent an active execution context managed by a kernel `task_struct` structure in RAM.

---

### How the Kernel Mounts and Generates `/proc` (Procfs)

```
                 RAM (Physical / Virtual Memory)
                              │
  [ Kernel task_struct ] <────┼─────> [ Dynamic Process Heap/Stack ]
             │
             ▼
 [ Virtual File System (VFS) Layer ]
             │
             ▼
    Mount Point: /proc/
             ├── /proc/1/       (PID 1: systemd)
             ├── /proc/1393/    (PID 1393: nc listener)
             └── /proc/cpuinfo  (Hardware state)
```

1. **/proc occupies 0 bytes on disk:** `/proc` is a virtual in-memory pseudo-filesystem (`procfs`). It exists strictly in RAM and kernel data structures.
2. **On-Demand Generation:** When a user executes `cat /proc/<PID>/status`, the kernel intercepts the VFS read call and dynamically generates text from the process `task_struct`.
3. **Automatic Lifecycle Management:** When a process exits, the kernel frees its `task_struct`, and its corresponding `/proc/<PID>/` folder instantly vanishes.

---

### Deconstructing the `/proc/<PID>/` Anatomy

For any running PID (e.g., your current shell PID `$$`):

```bash
ls -la /proc/$$/
```

| File / Subfolder in `/proc/<PID>/` | Internal Purpose & Data Stored | DevOps Inspection Command |
| :--- | :--- | :--- |
| **`cmdline`** | The complete executable path and arguments passed at startup. | `cat /proc/<PID>/cmdline \| tr '\0' ' '` |
| **`environ`** | Complete environment variables exported to the process. | `cat /proc/<PID>/environ \| tr '\0' '\n'` |
| **`fd/`** | Directory containing symbolic links to all open File Descriptors (sockets, logs, pipes). | `ls -l /proc/<PID>/fd` |
| **`status`** | Process metadata: UID/GID, memory footprints (`VmRSS`, `VmSize`), thread count. | `grep -E 'Name\|State\|VmRSS\|Threads' /proc/<PID>/status` |
| **`maps`** | Virtual memory address layout showing mapped binaries, heap, stack, and `.so` libraries. | `head -n 25 /proc/<PID>/maps` |
| **`exe`** | Symbolic link pointing to the original binary on disk. | `readlink -f /proc/<PID>/exe` |
| **`cwd`** | Symbolic link pointing to the current working directory of the process. | `readlink -f /proc/<PID>/cwd` |
| **`net/`** | Directory exposing socket states, TCP connection tables, and routing tables for that network namespace. | `cat /proc/<PID>/net/tcp` |

---

### 🚀 DevOps Real-Time Scenario: Zero-Downtime Troubleshooting via `/proc`

**The Situation:** A production container is misbehaving. The container image has no debugging tools installed (no `curl`, `netstat`, `lsof`, `env`).

#### Production Troubleshooting using `/proc`:
1. **Find the Container PID on the host node:**
   ```bash
   docker inspect --format '{{ .State.Pid }}' production-api
   # Output: 14820
   ```
2. **Inspect Runtime Environment Variables:**
   ```bash
   cat /proc/14820/environ | tr '\0' '\n' | grep DATABASE_URL
   ```
3. **Inspect All Active Network Connections:**
   ```bash
   ls -la /proc/14820/fd | grep socket
   ```
4. **Recover a Deleted Log File Held Open by the Process:**
   ```bash
   # If a log file was accidentally deleted with rm while the service is running:
   cp /proc/14820/fd/3 /var/log/recovered-application.log
   ```

---

## 5. DevOps Production Troubleshooting & Security Cheat Sheet

| Task | Linux Command | Why DevOps Engineers Use It |
| :--- | :--- | :--- |
| **Profile Syscall Overhead** | `strace -c <command>` | Pinpoint execution bottlenecks and excessive disk/network syscalls. |
| **Attach Syscall Tracer to Running PID** | `sudo strace -f -p <PID> -e trace=network,file` | Debug hung or unresponsive production processes without restarting them. |
| **Monitor File Integrity in Real-Time** | `sudo inotifywait -m -e modify,attrib /etc/nginx/nginx.conf` | Detect manual configuration tampering and trigger automatic rollbacks. |
| **Audit File Access via Kernel Audit Subsystem** | `sudo auditctl -w /etc/shadow -p rwa -k shadow_tamper` | Enterprise SOC2 compliance auditing for identity and credential access. |
| **Inspect Environment of Running Container** | `cat /proc/<PID>/environ \| tr '\0' '\n'` | Validate injected secrets and runtime configs without `docker exec`. |
| **Identify Rogue Process Binary Location** | `readlink -f /proc/<PID>/exe` | Security forensics: locate hidden or obfuscated malware binaries. |
| **Recover Accidentally Deleted Open File** | `cp /proc/<PID>/fd/<FD_NUM> /backup/recovered_file` | Restore active database or log files deleted from disk while process runs. |
| **Verify Seccomp Support in Host Kernel** | `grep -i CONFIG_SECCOMP= /boot/config-$(uname -r)` | Ensure host kernel supports container sandboxing profiles. |

---

*Authored for Linux System Administrators, Security Engineers, & DevOps/SRE Professionals.*
