# 🐧 Linux Filesystem Architecture, Permissions & File Operations Deep Dive

> Comprehensive hands-on notes exploring the Linux Filesystem Hierarchy Standard (FHS), core file operations, inode architecture, hard & soft links, octal permissions, file inspection tools (`less`, `tail -f`), shell globbing, and advanced search automation (`find`) — annotated with real-time DevOps production scenarios.

---

## Table of Contents
1. [1. Linux Filesystem Hierarchy (FHS) & "Everything is a File"](#1-linux-filesystem-hierarchy-fhs--everything-is-a-file)
   - [The "Everything is a File" Philosophy](#the-everything-is-a-file-philosophy)
   - [Single Root Namespace vs Windows Drive Letters](#single-root-namespace-vs-windows-drive-letters)
   - [Essential Standard Directories Deep Dive](#essential-standard-directories-deep-dive)
   - [Absolute vs Relative Paths](#absolute-vs-relative-paths)
   - [🚀 DevOps Real-Time Scenario: Full `/var` Partition Triage](#-devops-real-time-scenario-full-var-partition-triage)
2. [2. File & Directory Inspection (`ls`, `file`, File Types)](#2-file--directory-inspection-ls-file-file-types)
   - [Deconstructing the `ls -l` Long Listing Output](#deconstructing-the-ls--l-long-listing-output)
   - [Essential `ls` Flags & Power Combinations](#essential-ls-flags--power-combinations)
   - [The 7 Linux File Types Explained](#the-7-linux-file-types-explained)
   - [Magic Number Identification with `file`](#magic-number-identification-with-file)
   - [🚀 DevOps Real-Time Scenario: Detecting Masked Binaries in Web Uploads](#-devops-real-time-scenario-detecting-masked-binaries-in-web-uploads)
3. [3. Core File & Directory Operations (`touch`, `mkdir`, `cp`, `mv`, `rm`, `rmdir`)](#3-core-file--directory-operations-touch-mkdir-cp-mv-rm-rmdir)
   - [File Creation & Timestamp Mechanics (`touch`)](#file-creation--timestamp-mechanics-touch)
   - [Directory Creation (`mkdir`, `mkdir -p`)](#directory-creation-mkdir-mkdir--p)
   - [Copying Files & Preserving Attributes (`cp`)](#copying-files--preserving-attributes-cp)
   - [Atomic Renaming and Relocation (`mv`)](#atomic-renaming-and-relocation-mv)
   - [Safe Deletion Practices (`rm`, `rm -rf`, `rmdir`)](#safe-deletion-practices-rm-rm--rf-rmdir)
   - [🚀 DevOps Real-Time Scenario: Safe File Cleanups & Preventing `rm -rf /` in CI/CD](#-devops-real-time-scenario-safe-file-cleanups--preventing-rm--rf--in-cicd)
4. [4. Shell Wildcards, Globbing & Expansion Patterns](#4-shell-wildcards-globbing--expansion-patterns)
   - [Shell Globbing Syntax & Mechanics](#shell-globbing-syntax--mechanics)
   - [Globbing vs Bash Brace Expansion](#globbing-vs-bash-brace-expansion)
   - [🚀 DevOps Real-Time Scenario: Multi-Service Config Templating & Deployment](#-devops-real-time-scenario-multi-service-config-templating--deployment)
5. [5. Viewing & Paging File Contents (`cat`, `tac`, `less`, `head`, `tail`)](#5-viewing--paging-file-contents-cat-tac-less-head-tail)
   - [Concatenation & Line Numbering (`cat`, `tac`)](#concatenation--line-numbering-cat-tac)
   - [Paginated Navigation with `less`](#paginated-navigation-with-less)
   - [Top and Bottom Slicing (`head`, `tail`)](#top-and-bottom-slicing-head-tail)
   - [Live Log Streaming (`tail -f` vs `tail -F`)](#live-log-streaming-tail--f-vs-tail--f)
   - [🚀 DevOps Real-Time Scenario: Production Outage Live Log Triage](#-devops-real-time-scenario-production-outage-live-log-triage)
6. [6. Inodes, Hard Links, and Symbolic Links (`ln`, `ls -i`)](#6-inodes-hard-links-and-symbolic-links-ln-ls--i)
   - [Low-Level Storage Mechanics: Inodes vs Data Blocks](#low-level-storage-mechanics-inodes-vs-data-blocks)
   - [Hard Links Explained](#hard-links-explained)
   - [Symbolic (Soft) Links Explained](#symbolic-soft-links-explained)
   - [Comparison: Hard Link vs Symbolic Link](#comparison-hard-link-vs-symbolic-link)
   - [🚀 DevOps Real-Time Scenario: Zero-Downtime Blue/Green Deployment with Symlinks](#-devops-real-time-scenario-zero-downtime-bluegreen-deployment-with-symlinks)
7. [7. Linux Ownership & Permission Architecture (`chmod`, `chown`, `chgrp`)](#7-linux-ownership--permission-architecture-chmod-chown-chgrp)
   - [The Permission Triad: User, Group, and Others](#the-permission-triad-user-group-and-others)
   - [File Permissions vs Directory Permissions](#file-permissions-vs-directory-permissions)
   - [Octal (Numeric) vs Symbolic Notation](#octal-numeric-vs-symbolic-notation)
   - [Changing Ownership & Groups (`chown`, `chgrp`)](#changing-ownership--groups-chown-chgrp)
   - [Special Permissions Overview (SUID, SGID, Sticky Bit)](#special-permissions-overview-suid-sgid-sticky-bit)
   - [🚀 DevOps Real-Time Scenario: Production SSH Key Hardening & Shared Group Repositories](#-devops-real-time-scenario-production-ssh-key-hardening--shared-group-repositories)
8. [8. Advanced File Search & Batch Operations (`find`)](#8-advanced-file-search--batch-operations-find)
   - [Anatomical Structure of `find`](#anatomical-structure-of-find)
   - [Searching by Name, Type, and Size](#searching-by-name-type-and-size)
   - [Searching by Timestamps (`-mtime`, `-mmin`)](#searching-by-timestamps--mtime--mmin)
   - [Security & Permission Auditing (`-perm`)](#security--permission-auditing--perm)
   - [Executing Batch Actions (`-exec`, `-delete`)](#executing-batch-actions--exec--delete)
   - [🚀 DevOps Real-Time Scenario: Automated Log Rotation & Old Artifact Cleanup Cron](#-devops-real-time-scenario-automated-log-rotation--old-artifact-cleanup-cron)
9. [9. DevOps Production Troubleshooting Cheat Sheet](#9-devops-production-troubleshooting-cheat-sheet)

---

## 1. Linux Filesystem Hierarchy (FHS) & "Everything is a File"

### The "Everything is a File" Philosophy

In Unix and Linux operating systems, almost every system resource — whether a text file, a running process, a hard disk, an SSH terminal, or an inter-process communication socket — is abstracted as a **stream of bytes exposed through a file descriptor within the filesystem**.

```
+-------------------------------------------------------------------------+
|                    THE UNIFIED VIRTUAL FILESYSTEM (VFS)                 |
+-------------------------------------------------------------------------+
       │                     │                      │              │
       ▼                     ▼                      ▼              ▼
[ Regular Files ]    [ Block / Char Devs ]    [ Kernel State ] [ Sockets/Pipes ]
  /etc/nginx.conf      /dev/sda1 (SSD/Disk)     /proc/cpuinfo    /var/run/docker.sock
  /var/log/syslog      /dev/pts/1 (Terminal)    /proc/meminfo    /run/systemd/notify
```

Because everything is represented as a file, the standard POSIX system calls (`open()`, `read()`, `write()`, `close()`) can interact with hardware, inspect kernel metrics, and configure OS behavior uniformly without dedicated vendor APIs.

---

### Single Root Namespace vs Windows Drive Letters

Unlike Windows (which separates physical disks or volumes into drive letters like `C:\`, `D:\`, `E:\`), Linux organizes all physical and virtual storage beneath a **single unified hierarchical tree starting at the root directory (`/`)**.

```
                                 / (Root)
                                 │
     ┌──────────┬──────────┬─────┴────┬──────────┬──────────┬──────────┐
     │          │          │          │          │          │          │
   /bin       /etc       /home      /var       /usr       /proc      /dev
                           │          │          │          │          │
                       /kshitiz    /var/log   /usr/bin    (Kernel)   (Devices)
```

Extra physical disks, network shares (NFS/CIFS), and virtual memory filesystems are attached ("mounted") directly onto directory branch points (called **mount points**) anywhere within this tree (e.g., `/mnt/data`, `/var/lib/docker`).

---

### Essential Standard Directories Deep Dive

The **Filesystem Hierarchy Standard (FHS)** defines the official structure and purpose of directories across Linux distributions:

| Directory | Full Form / Concept | Type | Purpose & DevOps Significance |
| :--- | :--- | :--- | :--- |
| **`/`** | Root | Root | The top-level root node of the entire directory tree. Every file and directory stems from here. |
| **`/usr/bin`** | User Binaries | Physical | Standard executable commands accessible to all system users (e.g., `git`, `python3`, `curl`, `jq`, `grep`). |
| **`/usr/sbin`** | System Binaries | Physical | System administration commands requiring root/sudo privileges (e.g., `useradd`, `iptables`, `fdisk`, `reboot`). |
| **`/etc`** | Editable Text Configurations | Physical | System-wide configuration files (e.g., `/etc/hosts`, `/etc/ssh/sshd_config`, `/etc/nginx/nginx.conf`). Contains no executable binaries. |
| **`/var`** | Variable Data | Physical | Dynamic data generated at runtime that changes frequently: `/var/log` (logs), `/var/lib/docker` (container layers), `/var/www` (web assets). |
| **`/home`** | User Home Dirs | Physical | Personal directories for regular non-root accounts (e.g., `/home/kshitiz`, `/home/deployer`). |
| **`/root`** | Root Superuser Home | Physical | Dedicated `$HOME` directory for the `root` superuser (`UID 0`). Kept separate from `/home` to ensure root can log in even if `/home` fails to mount. |
| **`/usr`** | User System Resources | Physical | Read-only user software, libraries, documentation, and header files (`/usr/lib`, `/usr/share`). |
| **`/usr/local`** | Local Admin Binaries | Physical | Software installed manually by the administrator (e.g., compiled from source, custom Go/Rust binaries) to avoid clashing with package managers (`apt`, `yum`). |
| **`/tmp`** | Temporary Storage | RAM/Disk (`tmpfs`) | Temporary scratch files. Often mounted as RAM (`tmpfs`) or flushed automatically upon reboot or via `systemd-tmpfiles`. World-writable with sticky bit. |
| **`/proc`** | Process Information | Virtual (`procfs`) | Pseudo-filesystem generated in memory by the kernel on the fly. Contains real-time metrics for processes (`/proc/<PID>/`) and hardware (`/proc/cpuinfo`, `/proc/meminfo`). |
| **`/sys`** | System & Kernel Objects | Virtual (`sysfs`) | Exposes kernel subsystem parameters, cgroups (`/sys/fs/cgroup`), and hardware buses/devices for live runtime tuning. |
| **`/dev`** | Device Nodes | Virtual (`devtmpfs`) | Direct interfaces to hardware and virtual devices: storage drives (`/dev/nvme0n1`, `/dev/sda`), pseudo-terminals (`/dev/pts/*`), random entropy (`/dev/urandom`), null sink (`/dev/null`). |
| **`/mnt` & `/media`** | Mount Points | Physical/Mount | Temporary mount points for external filesystems, NFS shares, or removable storage. |
| **`/lib` & `/lib64`** | Shared Libraries | Physical | Essential shared libraries (`.so` files) and kernel modules required by executables in `/bin` and `/sbin`. |

---

### Absolute vs Relative Paths

Understanding path resolution is critical when writing automated Ansible playbooks, shell scripts, and Dockerfiles:

```
Current Working Directory: /home/kshitiz/cs/linux
```

1. **Absolute Path:**
   - Always begins with the root directory (`/`).
   - Resolves the exact location regardless of the current working directory.
   - *Example:* `/home/kshitiz/cs/linux/commands.md`
   - **DevOps Best Practice:** Always use absolute paths in Cron jobs, Systemd unit files, and automation scripts to eliminate path-dependency bugs.

2. **Relative Path:**
   - Evaluated relative to the directory you are currently in (`PWD`).
   - Does **not** start with a leading slash `/`.
   - Special symbols:
     - `.` (Current directory): `./build.sh`
     - `..` (Parent directory): `cd ../..`
     - `~` (Current user's home directory): `~/.ssh/id_rsa` $\rightarrow$ `/home/kshitiz/.ssh/id_rsa`
   - *Example:* If in `/home/kshitiz`, the relative path `cs/linux/commands.md` resolves to `/home/kshitiz/cs/linux/commands.md`.

---

### 🚀 DevOps Real-Time Scenario: Full `/var` Partition Triage

**The Situation:** Alert triggers: `DiskSpaceExhausted: /var on production worker node k8s-worker-04 is at 99% capacity`. Pods are failing to schedule with `DiskPressure` state.

```
Host Filesystem Mounts:
/dev/sda1   -> /     (20GB, 40% used)
/dev/sda2   -> /var  (100GB, 99% used)  <-- CRITICAL BOTTLENECK
```

#### Step-by-Step Triage & Resolution:

1. **Step 1 — Verify disk utilization across mount points:**
   ```bash
   df -h /var
   ```
2. **Step 2 — Identify the largest sub-directory consuming space inside `/var`:**
   ```bash
   # Sort top 10 largest folders inside /var in human-readable format
   sudo du -ah /var | sort -rh | head -n 10
   ```
   *Output shows runaway un-rotated application logs:*
   ```text
   85G    /var/log/nginx/access.log
   5.2G   /var/lib/docker/overlay2
   2.1G   /var/log/journal
   ```
3. **Step 3 — Truncate the file safely without breaking running processes:**
   > [!WARNING]
   > Never delete an open log file with `rm /var/log/nginx/access.log` while the service is running! The inode remains open in memory by the running process, disk space is **not** reclaimed, and `df` will continue to show 100% full.

   ```bash
   # Safely zero out the active file descriptor in-place
   sudo truncate -s 0 /var/log/nginx/access.log
   # OR:
   sudo : > /var/log/nginx/access.log
   ```
4. **Step 4 — Verify space is reclaimed:**
   ```bash
   df -h /var
   # Now shows 14% utilization!
   ```

---

## 2. File & Directory Inspection (`ls`, `file`, File Types)

### Deconstructing the `ls -l` Long Listing Output

The `ls -l` command prints detailed metadata stored in the filesystem for each directory entry.

```
-rw-r--r--   1   root   root   12   Jun 10 09:15   /etc/hostname
│└┬┘└┬┘└┬┘   │   └─┬─┘  └─┬─┘  │    └────┬────┘    └─────┬─────┘
│ │  │  │    │     │      │    │         │               │
│ │  │  │    │     │      │    │         │               └─ Filename / Entry Name
│ │  │  │    │     │      │    │         └─ Modification Timestamp (mtime)
│ │  │  │    │     │      │    └─ File size in bytes (12 bytes)
│ │  │  │    │     │      └─ Owning Group (`root`)
│ │  │  │    │     └─ Owning User (`root`)
│ │  │  │    └─ Hard Link Count (1 reference to this inode)
│ │  │  └─ Others (World) Permissions: Read only (`r--`)
│ │  └─ Group Permissions: Read only (`r--`)
│ └─ Owner (User) Permissions: Read & Write (`rw-`)
└─ File Type Indicator: Regular File (`-`)
```

#### Field-by-Field Reference Breakdown:

1. **File Type (1 character):** Defines the nature of the filesystem node (e.g., `-` for file, `d` for directory, `l` for symlink).
2. **Permissions (9 characters):** Split into three triplets:
   - User/Owner (`rw-` $\rightarrow$ Read + Write)
   - Group (`r--` $\rightarrow$ Read only)
   - Others (`r--` $\rightarrow$ Read only)
3. **Hard Link Count:** The total number of directory entries pointing to the underlying inode. For a regular new file, this is `1`. For a directory, it is at least `2` (the directory name itself and the `.` entry inside it).
4. **User Owner:** Username of the account that owns the file (`root` or `kshitiz`).
5. **Group Owner:** Group name associated with the file permissions (`root`, `docker`, `developers`).
6. **File Size:** Size of the file content in bytes (or human-readable units like `4.2K`, `12M` when using `-h`).
7. **Modification Time (`mtime`):** Date and time when the contents of the file were last written to.
8. **Filename:** Name of the file or directory.

---

### Essential `ls` Flags & Power Combinations

```bash
# Basic listing (names only)
ls

# Long listing with full permissions, ownership, size, and date
ls -l

# Show all files INCLUDING hidden files (files starting with a dot, e.g., .bashrc, .git)
ls -a

# Power Combination: Long listing + all hidden files (DevOps Standard)
ls -la

# Long listing with human-readable file sizes (KB, MB, GB instead of raw bytes)
ls -lh

# Sort files by modification time (newest files first)
ls -lt

# Reverse sort by modification time (oldest first, newest at bottom of terminal)
ls -ltr

# Sort by file size descending (largest files first)
ls -lS

# Recursive listing (lists all sub-directories and their nested files)
ls -lR

# List ONLY directories (ignoring files)
ls -d */

# Display Inode numbers alongside file details
ls -li
```

#### Terminal Inspection Example:
```bash
kshitiz@KZ:~/cs/linux$ ls -la
total 196
drwxr-xr-x 1 kshitiz kshitiz  4096 Sep 27 15:21 .
drwxrwxrwx 1 kshitiz kshitiz  4096 Sep 23 04:47 ..
drwxrwxrwx 1 kshitiz kshitiz  4096 Sep 27 10:05 .git
-rw------- 1 kshitiz kshitiz 12288 Sep 27 15:34 .notes_1.md.swp
-rwxrwxrwx 1 kshitiz kshitiz 12844 Sep 27 10:02 README.md
-rw-r--r-- 1 kshitiz kshitiz 25568 Sep 23 07:46 commands.md
-rw-r--r-- 1 kshitiz kshitiz 35396 Sep 25 06:16 commands_2.md
-rw-r--r-- 1 kshitiz kshitiz 26847 Sep 25 06:38 commands_3.md
-rw-r--r-- 1 kshitiz kshitiz 23267 Sep 25 07:37 commands_4.md
-rw-r--r-- 1 kshitiz kshitiz 44229 Sep 27 06:40 commands_5.md
-rw-r--r-- 1 kshitiz kshitiz  1461 Sep 27 10:06 commands_6.md
-rw-r--r-- 1 kshitiz kshitiz  9360 Sep 27 21:33 notes_1.md
```

> [!NOTE]
> - `.` refers to the **current directory**.
> - `..` refers to the **parent directory**.
> - Files prefixed with a dot (like `.git` or `.notes_1.md.swp`) are hidden from standard `ls` and only visible when `-a` is passed.

---

### The 7 Linux File Types Explained

In the first column of `ls -l`, the leading character tells you the exact kernel file type:

| Symbol | File Type | Description | Real Example |
| :---: | :--- | :--- | :--- |
| **`-`** | **Regular File** | Standard text file, compiled binary, image, or compressed archive. | `/etc/passwd`, `/bin/bash`, `app.tar.gz` |
| **`d`** | **Directory** | A special file containing a list of filenames and their corresponding inode numbers. | `/var/log`, `/home/kshitiz` |
| **`l`** | **Symbolic Link (Soft Link)** | A pointer file containing the path string to another target file or directory. | `/usr/bin/python -> python3` |
| **`c`** | **Character Device** | A device stream transmitting data character-by-character (unbuffered, serial). | `/dev/tty`, `/dev/pts/1`, `/dev/urandom` |
| **`b`** | **Block Device** | A storage device transferring data in fixed-size blocks (buffered, random-access). | `/dev/sda`, `/dev/nvme0n1p1` |
| **`s`** | **Local Domain Socket** | Inter-process communication (IPC) endpoint for bidirectional data exchange between local processes. | `/var/run/docker.sock`, `/run/systemd/notify` |
| **`p`** | **Named Pipe (FIFO)** | First-In-First-Out IPC mechanism allowing two independent processes to stream data. | `/var/run/app.pipe` |

---

### Magic Number Identification with `file`

On Windows, the operating system relies heavily on the file extension (`.exe`, `.txt`, `.png`) to identify file types. On Linux, **file extensions are purely cosmetic for human readability**. The kernel and applications inspect the header bytes (**Magic Numbers**) inside the file.

The `file` command determines the actual MIME type and encoding of any file:

```bash
# Check regular text/markdown file
file commands.md
# Output: commands.md: Unicode text, UTF-8 text, with very long lines (368)

# Check compiled ELF executable
file /bin/ls
# Output: /bin/ls: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked...

# Check compressed archive with disguised extension
mv backup.tar.gz backup.png
file backup.png
# Output: backup.png: gzip compressed data, from Unix, original size modulo 2^32 10485760
```

---

### 🚀 DevOps Real-Time Scenario: Detecting Masked Binaries in Web Uploads

**The Situation:** Security alert: An attacker bypassed a web application's upload filter by renaming an ELF rootkit executable to `avatar.jpg`.

#### Detection & Forensics Walkthrough:
```bash
# 1. Listing shows innocent looking file
ls -la /var/www/uploads/avatar.jpg
# -rw-r--r-- 1 www-data www-data 2048576 Sep 27 12:00 avatar.jpg

# 2. Inspect real file type using 'file' command:
file /var/www/uploads/avatar.jpg
# Output reveals malicious executable:
# /var/www/uploads/avatar.jpg: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, stripped

# 3. Prevent execution and quarantine the file immediately:
chmod 000 /var/www/uploads/avatar.jpg
sudo mv /var/www/uploads/avatar.jpg /root/quarantine/
```

---

## 3. Core File & Directory Operations (`touch`, `mkdir`, `cp`, `mv`, `rm`, `rmdir`)

### File Creation & Timestamp Mechanics (`touch`)

The `touch` command serves two core functions in Linux:
1. **Creates a 0-byte empty file** if the target file does not exist.
2. **Updates the access time (`atime`) and modification time (`mtime`)** to the current system clock without altering the file content if it already exists.

```bash
# Create a single empty file
touch app.log

# Create multiple empty files simultaneously
touch a.txt b.txt c.txt

# Update timestamp of an existing file to current time (triggers build tools like Make)
touch existing_config.yaml

# Set a specific custom timestamp (Format: [[CC]YY]MMDDhhmm[.ss])
touch -t 202609271200.00 release.lock
```

---

### Directory Creation (`mkdir`, `mkdir -p`)

```bash
# Create a single directory
mkdir logs

# Create deeply nested directory trees at once using -p (parents)
# Does not throw an error if the directory already exists!
mkdir -p projects/microservice/src/components
```

> [!TIP]
> Always use `mkdir -p` in automation scripts, Dockerfiles, and CI pipelines. It prevents scripts from crashing if intermediate directories are missing or if the target directory has already been created.

---

### Copying Files & Preserving Attributes (`cp`)

Copying files involves reading from the source inode and writing data to a new target inode:

| Command | Flag Meaning | DevOps Use Case |
| :--- | :--- | :--- |
| `cp file.txt backup.txt` | Standard copy | Duplicate a config file before editing. |
| `cp -r src/ dest/` | **`-r` (Recursive)** | Copy entire directory trees and sub-folders. |
| `cp -p file.txt backup.txt` | **`-p` (Preserve)** | Preserves file ownership, permissions, and timestamps. |
| `cp -a src/ dest/` | **`-a` (Archive mode)** | Recursively copies, preserves all permissions/timestamps/symlinks, and disables link dereferencing (`-dR --preserve=all`). **Gold Standard for DevOps backups**. |
| `cp -v file.txt /tmp/` | **`-v` (Verbose)** | Prints detailed stdout progress of what is being copied. |
| `cp -i file.txt target.txt` | **`-i` (Interactive)** | Prompts for confirmation before overwriting an existing destination file. |
| `cp -u src/*.log /backup/` | **`-u` (Update)** | Copies only files that are newer than the destination files or missing. |

```bash
# Full production directory backup preserving permissions & symlinks
cp -av /etc/nginx /etc/nginx_backup_$(date +%F)
```

---

### Atomic Renaming and Relocation (`mv`)

In Linux, **moving a file and renaming a file use the exact same command (`mv`)**. If the source and destination are on the same filesystem, `mv` simply updates the directory entry pointer without re-copying the underlying data blocks on disk — making it instantaneous and **atomic**.

```bash
# Rename a file in place
mv old_name.txt new_name.txt

# Move a file to a different directory
mv app.log /var/log/app/

# Move and rename simultaneously
mv config.dev.json /etc/app/config.json

# Interactive move (prompt before overwriting destination)
mv -i server.key /etc/ssl/certs/
```

---

### Safe Deletion Practices (`rm`, `rm -rf`, `rmdir`)

```bash
# Delete a single file
rm notes.txt

# Interactive deletion (asks for confirmation)
rm -i sensitive.key

# Remove an EMPTY directory (fails if directory contains any files)
rmdir empty_logs/

# Recursive directory deletion (deletes folder and all nested files)
rm -r build/

# Recursive + Force deletion (no prompts, ignores non-existent files)
rm -rf /tmp/build_cache/
```

> [!CAUTION]
> `rm -rf` deletes files permanently without moving them to a "Recycle Bin". It unlinks inodes directly. Double check all variables before executing commands like `rm -rf $BUILD_DIR/*` in bash scripts!

---

### 🚀 DevOps Real-Time Scenario: Safe File Cleanups & Preventing `rm -rf /` in CI/CD

**The Vulnerability:** An unquoted or undefined environment variable in a CI/CD pipeline script:
```bash
# DANGEROUS PIPELINE SCRIPT:
rm -rf $RELEASE_DIR/
```
If `$RELEASE_DIR` is empty or undefined due to a configuration failure, the shell expands the command to:
```bash
rm -rf /
```
This wipes the root filesystem of the server or container!

#### Production-Grade Defensive Bash Safeguard:
```bash
# 1. Enforce strict bash error checking at the top of scripts:
set -euo pipefail

# 2. Defensively check that the variable is non-empty before deleting:
if [ -n "${RELEASE_DIR:-}" ] && [ -d "${RELEASE_DIR}" ]; then
    rm -rf "${RELEASE_DIR:?RELEASE_DIR is unset or empty}"/*
fi
```

---

## 4. Shell Wildcards, Globbing & Expansion Patterns

### Shell Globbing Syntax & Mechanics

Wildcards (Globbing) are evaluated by the **shell itself (Bash/Zsh)** *before* passing the expanded list of matching arguments to the executed binary.

```
Shell Input:       ls *.txt
Shell Expansion:   ls notes.txt readme.txt todo.txt
Executed Binary:   /bin/ls receives argv["notes.txt", "readme.txt", "todo.txt"]
```

| Wildcard Pattern | Matches | Practical Example | Matches Examples | Does NOT Match |
| :--- | :--- | :--- | :--- | :--- |
| **`*`** | Zero or more characters | `*.log` | `app.log`, `error.log`, `.log` | `app.txt`, `log.gz` |
| **`?`** | Exactly **one** character | `app-v?.tar` | `app-v1.tar`, `app-v2.tar` | `app-v10.tar` |
| **`[abc]`** | Any one character from the set | `data_[abc].csv` | `data_a.csv`, `data_b.csv` | `data_d.csv` |
| **`[a-z]`** | Any character within the range | `node-[0-9].json` | `node-1.json`, `node-8.json` | `node-a.json` |
| **`[!0-9]`** or **`[^0-9]`** | Negation: Any character NOT in set | `test-[!0-9].sh` | `test-a.sh`, `test-z.sh` | `test-1.sh` |

---

### Globbing vs Bash Brace Expansion

Brace expansion is **not globbing**; it does not check if files exist on disk. It generates arbitrary string combinations according to a pattern:

```bash
# Sequence expansion
echo {1..5}
# Output: 1 2 3 4 5

# Alphabetic sequence
echo {A..E}
# Output: A B C D E

# Zero-padded sequence (great for numbering servers or logs)
echo server-{01..04}.prod
# Output: server-01.prod server-02.prod server-03.prod server-04.prod

# Multiple list expansion
echo {frontend,backend,db}-{dev,prod}.internal
# Output: frontend-dev.internal frontend-prod.internal backend-dev.internal backend-prod.internal db-dev.internal db-prod.internal

# Creating a complex directory structure in one single command:
mkdir -p project/{src,tests,docs,bin,config}
```

---

### 🚀 DevOps Real-Time Scenario: Multi-Service Config Templating & Deployment

**The Situation:** You are bootstrapping directory structures and placeholder configs for a microservices cluster consisting of `auth`, `orders`, `payments`, and `notifications` across `dev` and `prod` environments.

```bash
# Create complete directory layout in 1 line
mkdir -p /opt/deploy/{auth,orders,payments,notifications}/{dev,prod}/{logs,config}

# Generate placeholder .env files
touch /opt/deploy/{auth,orders,payments,notifications}/{dev,prod}/config/.env

# Verify directory tree structure
ls -d /opt/deploy/*/*
```

---

## 5. Viewing & Paging File Contents (`cat`, `tac`, `less`, `head`, `tail`)

### Concatenation & Line Numbering (`cat`, `tac`)

- **`cat` (Concatenate & Print):** Dumps the entire file content directly to standard output (`stdout`).
- **`tac` (Reverse `cat`):** Prints file content in reverse order (bottom line first).

```bash
# Display entire file contents
cat /etc/hosts

# Display contents with line numbers
cat -n /etc/nginx/nginx.conf

# Display non-blank line numbers only
cat -b server.js

# Display invisible characters (tabs as ^I, line ends with $)
cat -A config.env

# Inspect the most recent events from a log file (reverse order)
tac /var/log/auth.log | head -n 20
```

> [!WARNING]
> Never run `cat` on massive multi-gigabyte files (e.g., `cat /var/log/syslog`). It will flood your terminal, lock up your SSH session, and consume unnecessary CPU. Use `less` instead.

---

### Paginated Navigation with `less`

`less` is an interactive pager that loads files **lazily in chunks as you scroll**, meaning it opens a 50GB file instantly without consuming 50GB of RAM.

```bash
less /var/log/syslog
```

#### Essential `less` Interactive Navigation Shortcuts:

| Keybinding | Action / Navigation |
| :--- | :--- |
| **`Space`** or **`f`** | Scroll forward one full page / screen. |
| **`b`** | Scroll backward one full page / screen. |
| **`j`** / **`k`** or **`Down`** / **`Up`** | Scroll down / up by one single line. |
| **`/pattern`** | Search forward for `pattern` (e.g., `/ERROR`). |
| **`?pattern`** | Search backward for `pattern`. |
| **`n`** | Jump to next search match in current direction. |
| **`N`** | Jump to previous search match (opposite direction). |
| **`g`** | Jump immediately to the very top (Line 1) of the file. |
| **`G`** | Jump immediately to the very bottom (last line) of the file. |
| **`F`** | Live follow mode (works like `tail -f`; press `Ctrl + C` to return to interactive paging). |
| **`q`** | Quit and exit `less`. |

---

### Top and Bottom Slicing (`head`, `tail`)

```bash
# View the first 10 lines (default)
head /etc/passwd

# View the first 25 lines
head -n 25 /var/log/nginx/access.log

# View the last 10 lines (default)
tail /etc/passwd

# View the last 50 lines
tail -n 50 /var/log/nginx/error.log

# Output all lines starting from line 100 to the end of file
tail -n +100 app.log
```

---

### Live Log Streaming (`tail -f` vs `tail -F`)

`tail -f` keeps the file descriptor open and prints new log entries as they are written in real-time.

```bash
# Follow new lines as they are appended to the file
tail -f /var/log/nginx/access.log

# Follow multiple files simultaneously (headers indicate which file updated)
tail -f /var/log/nginx/access.log /var/log/nginx/error.log
```

#### The Critical Difference Between `-f` and `-F`:
- **`tail -f` (Follow by File Descriptor):** Tracks the open file descriptor. If logrotate rotates the file (renaming `access.log` $\rightarrow$ `access.log.1` and creating a new `access.log`), `tail -f` continues following the old rotated file and stops displaying new logs!
- **`tail -F` (Follow by Filename with Retry):** Tracks the filename. If logrotate truncates or recreates `access.log`, `tail -F` automatically re-opens the new file seamlessly.

> [!TIP]
> Always use `tail -F` (capital `F`) when monitoring production application logs that undergo automatic `logrotate` schedules.

---

### 🚀 DevOps Real-Time Scenario: Production Outage Live Log Triage

**The Situation:** Web application returns `HTTP 502 Bad Gateway`. You need to inspect incoming requests live while filtering for errors.

```bash
# Stream live production errors while filtering out routine 200 OK healthchecks:
tail -F /var/log/nginx/access.log | grep -E --color=auto " (500|502|503|504) "
```

---

## 6. Inodes, Hard Links, and Symbolic Links (`ln`, `ls -i`)

### Low-Level Storage Mechanics: Inodes vs Data Blocks

When a filesystem (such as `ext4` or `XFS`) is formatted, disk space is partitioned into two primary components:
1. **Data Blocks:** Fixed-size chunks of storage (usually 4KB) where actual user payload/file content is saved.
2. **Inodes (Index Nodes):** A table of metadata structures. Each file or directory on the filesystem corresponds to exactly **one Inode Number**.

```
+-------------------------------------------------------------------------+
|                              EXT4 DISK LAYOUT                           |
+-------------------------------------------------------------------------+
|  INODE TABLE (Metadata Storage)         |  DATA BLOCKS (Actual Content) |
|  ─────────────────────────────────────  |  ──────────────────────────── |
|  Inode #71213169:                       |                               |
|  ├── File Type: Regular file            |  Block #1048:                 |
|  ├── Owner UID: 1000 (kshitiz)          |  ["# 🐧 Linux Administration"]|
|  ├── Group GID: 1000 (kshitiz)          |  Block #1049:                 |
|  ├── Permissions: 0644 (rw-r--r--)      |  ["Comprehensive notes..."]   |
|  ├── Timestamps: atime, mtime, ctime    |                               |
|  ├── Link Count: 1                      |                               |
|  └── Block Pointers: [1048, 1049] ──────┼──► Point to storage blocks    |
+-------------------------------------------------------------------------+
```

> [!IMPORTANT]
> **What does an Inode NOT store?**
> An inode **never** stores the filename or the directory path! Filenames are stored inside **Directory Entries (Dentries)**, which simply act as a mapping table linking human-readable names to inode numbers:
> `{"commands.md": 71213169}`

```bash
# Check the inode number of a file using ls -i
ls -i commands.md
# Output: 71213169 commands.md
```

---

### Hard Links Explained

A **Hard Link** is an additional directory entry (filename) pointing to an **existing inode**.

```
Directory Entry "original.txt" ──┐
                                 ├────► Inode #450123 ────► [ Data Block on Disk ]
Directory Entry "hardlink.txt" ──┘      (Link Count = 2)
```

```bash
# Create original file
echo "Linux Kernel Architecture" > original.txt

# Create hard link
ln original.txt hardlink.txt

# Inspect inode numbers and link counts using ls -li
ls -li original.txt hardlink.txt
# Output:
# 450123 -rw-r--r-- 2 kshitiz kshitiz 27 Sep 27 12:00 hardlink.txt
# 450123 -rw-r--r-- 2 kshitiz kshitiz 27 Sep 27 12:00 original.txt

# Modify the hard link
echo "New updates added" >> hardlink.txt

# Read the original file (contents are identical!)
cat original.txt
# Output:
# Linux Kernel Architecture
# New updates added

# Delete the original file
rm original.txt

# Data is STILL PRESERVED because Inode Link Count is now 1!
cat hardlink.txt
# Output:
# Linux Kernel Architecture
# New updates added
```

#### Hard Link Limitations:
1. **Cannot span across different filesystems/partitions:** Inode numbers are only unique within a single filesystem instance.
2. **Cannot link directories:** Restricted to prevent infinite cyclic directory loops in filesystem traversal trees.

---

### Symbolic (Soft) Links Explained

A **Symbolic Link (Symlink)** is an entirely independent file with its **own unique inode** and file type `l`. Its data block contains only one thing: **the text string of the target path**.

```
Symlink "app.symlink" ──► Inode #991002 ──► [ Path: "/var/www/app_v1" ]
                                                        │
                                                        ▼
Target File/Directory ──► Inode #450123 ──► [ Actual Data Blocks ]
```

```bash
# Create symbolic link: ln -s <TARGET_PATH> <LINK_NAME>
ln -s /mnt/c/Users/kshit/cs/linux ~/linux_shortcut

# Inspect the symlink
ls -l ~/linux_shortcut
# lrwxrwxrwx 1 kshitiz kshitiz 25 Sep 27 12:00 /home/kshitiz/linux_shortcut -> /mnt/c/Users/kshit/cs/linux
```

> [!WARNING]
> **Dangling / Broken Symlink:**
> If the destination target file is deleted or renamed, the symbolic link continues to point to the non-existent path. Attempting to read it results in `No such file or directory`.

---

### Comparison: Hard Link vs Symbolic Link

| Feature | Hard Link (`ln file link`) | Symbolic Link (`ln -s target link`) |
| :--- | :--- | :--- |
| **Inode Number** | **Same** inode number as the target. | **Different**, independent inode number. |
| **Link Count** | Increments the inode's link counter. | Does not change target's link count. |
| **Across Filesystems** | ❌ **No** (limited to same partition/disk). | ✅ **Yes** (can point to any mount point or network disk). |
| **Can Link Directories** | ❌ **No** (prevent filesystem recursion loops). | ✅ **Yes** (frequently used for directory shortcuts). |
| **If Target is Deleted** | ✅ Data remains fully intact via the link. | ❌ Becomes a broken / dangling pointer. |
| **File Type (`ls -l`)** | Shows as regular file (`-`). | Shows with file type `l` and arrow (`->`). |
| **Permissions** | Shared with the target. | Symlink itself has `777` permissions; target's permissions govern access. |

---

### 🚀 DevOps Real-Time Scenario: Zero-Downtime Blue/Green Deployment with Symlinks

**The Architecture:** In modern web application hosting (and tools like Capistrano or custom CI/CD pipelines), zero-downtime releases are achieved by pointing a web server root to a symbolic link.

```
/var/www/
  ├── releases/
  │     ├── release_2026-09-20_v1.0.0/   (Old / Blue)
  │     └── release_2026-09-27_v1.1.0/   (New / Green)
  └── current ──► /var/www/releases/release_2026-09-27_v1.1.0/   (Live Symlink)
```

#### Production Deployment Script:
```bash
# 1. Clone & build the new version in a new releases folder
mkdir -p /var/www/releases/v1.1.0
tar -xzf app_v1.1.0.tar.gz -C /var/www/releases/v1.1.0/

# 2. Run database migrations & warm up caches inside v1.1.0
cd /var/www/releases/v1.1.0 && npm ci --production

# 3. ATOMIC SWITCH: Atomically update symlink using -sfn
# -s = symbolic, -f = force, -n = treat symlink target as normal file
ln -sfn /var/www/releases/v1.1.0 /var/www/current

# 4. Reload Nginx without dropping active TCP connections
sudo systemctl reload nginx
```
*Result:* Web traffic instantly points to version 1.1.0 with **zero seconds of downtime**. If an error occurs, rolling back to `v1.0.0` takes 1 millisecond by flipping the symlink back!

---

## 7. Linux Ownership & Permission Architecture (`chmod`, `chown`, `chgrp`)

### The Permission Triad: User, Group, and Others

Every file and directory in Linux has an owner (`UID`) and a primary group (`GID`). Permissions are evaluated in order:

```
                      [ Incoming Process Request ]
                                   │
              Does Process UID == File Owner UID?
                   ├── YES ──► Check [ USER Permissions ]
                   └── NO
                        │
             Does Process GID match File Group GID?
                   ├── YES ──► Check [ GROUP Permissions ]
                   └── NO
                        │
                        └────► Check [ OTHER / WORLD Permissions ]
```

---

### File Permissions vs Directory Permissions

The meaning of Read (`r`), Write (`w`), and Execute (`x`) differs fundamentally between files and directories:

| Permission | Meaning on a **FILE** | Meaning on a **DIRECTORY** |
| :---: | :--- | :--- |
| **`r` (Read)** | View/read the text or binary content of the file (`cat`, `less`). | List the files inside the directory (`ls`). |
| **`w` (Write)** | Modify, overwrite, or truncate the file contents. | **Create, delete, or rename files inside the directory** (even files owned by someone else!). |
| **`x` (Execute)** | Execute the file as a program or shell script (`./script.sh`). | **Traverse/Enter the directory (`cd`) and access file inodes inside it**. |

> [!IMPORTANT]
> If a user has `r` permission on a directory but lacks `x` (execute) permission, they can list the filenames (`ls`), but they cannot `cd` into it, nor can they view file sizes, permissions, or read file contents!

---

### Octal (Numeric) vs Symbolic Notation

Permissions are represented mathematically as a 3-bit binary octal number per entity:

| Permission | Binary | Bit Value | Numeric Total |
| :---: | :---: | :---: | :---: |
| **`r`** (Read) | `100` | $2^2$ | **4** |
| **`w`** (Write) | `010` | $2^1$ | **2** |
| **`x`** (Execute) | `001` | $2^0$ | **1** |
| **`-`** (None) | `000` | $0$ | **0** |

```
Octal Calculation Formula:
rwx = 4 + 2 + 1 = 7
rw- = 4 + 2 + 0 = 6
r-x = 4 + 0 + 1 = 5
r-- = 4 + 0 + 0 = 4
--- = 0 + 0 + 0 = 0
```

#### Standard Production Permission Sets:
- **`755` (`rwxr-xr-x`):** Owner has full access; Group and Others can read and execute. Standard for public binaries, scripts, and directories.
- **`644` (`rw-r--r--`):** Owner can read/write; Group and Others can read only. Standard for web files, configs, and documentation.
- **`600` (`rw-------`):** Owner read/write only; zero access for everyone else. **Mandatory for private keys (`id_rsa`), certificates, and database credentials**.
- **`700` (`rwx------`):** Owner full access; zero access for others. **Mandatory for `~/.ssh` directory**.
- **`777` (`rwxrwxrwx`):** Read, write, execute for everyone. **Dangerous security risk; avoid in production**.

#### Changing Permissions with `chmod`:
```bash
# Octal syntax
chmod 755 deploy.sh
chmod 600 ~/.ssh/id_ed25519
chmod 700 ~/.ssh

# Symbolic syntax: [u/g/o/a] [+ / - / =] [r/w/x]
chmod u+x deploy.sh          # Add execute to user
chmod g-w config.yaml        # Remove write from group
chmod o=r public.txt         # Set others to read-only
chmod a+r index.html         # Add read for all (user, group, other)

# Recursive permission change
chmod -R 644 /var/www/html/*.html
```

---

### Changing Ownership & Groups (`chown`, `chgrp`)

```bash
# Change user ownership only
sudo chown alex server.js

# Change both user and group owner simultaneously (syntax: user:group)
sudo chown deployer:docker /var/run/docker.sock

# Change group ownership only
sudo chgrp developers /opt/shared_project

# Change ownership recursively across an entire directory tree
sudo chown -R www-data:www-data /var/www/html
```

---

### Special Permissions Overview (SUID, SGID, Sticky Bit)

Beyond standard `rwx`, Linux provides three special permission bits:

```
[ Special Bits ] [ Owner ] [ Group ] [ Others ]
     4/2/1         7/6/5     7/6/5     7/6/5
```

1. **SUID (Set User ID — Value `4000` / `u+s`):**
   - When an executable with SUID runs, it executes with the privileges of the **file owner**, not the user executing it.
   - *Example:* `/usr/bin/passwd` has `-rwsr-xr-x` owned by `root`. It allows standard users to update their password in `/etc/shadow`.

2. **SGID (Set Group ID — Value `2000` / `g+s`):**
   - On executable files: Runs with the permissions of the owning group.
   - **On directories:** Any new file created inside this directory automatically inherits the **group owner of the parent directory**, rather than the primary group of the user who created it. Crucial for team collaboration directories!

3. **Sticky Bit (Value `1000` / `+t`):**
   - When set on a world-writable directory (like `/tmp`), **only the file owner or root can delete or rename files inside it**, preventing users from deleting each other's temporary files.
   - Represented as `drwxrwxrwt`.

```bash
# Set SGID on shared team directory
sudo chmod 2775 /opt/team_collab

# Set Sticky Bit on temporary folder
sudo chmod 1777 /tmp/scratch
```

---

### 🚀 DevOps Real-Time Scenario: Production SSH Key Hardening & Shared Group Repositories

#### Scenario A: SSH Key Security Violation
**The Issue:** Running `ssh -i id_rsa deployer@10.0.1.20` fails with:
```text
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@         WARNING: UNPROTECTED PRIVATE KEY FILE!          @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
Permissions 0644 for 'id_rsa' are too open.
It is required that your private key files are NOT accessible by others.
```
**The Fix:**
```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_rsa.pub
chmod 600 ~/.ssh/authorized_keys
```

#### Scenario B: Multi-User Collaboration Folder with SGID
**The Requirement:** Developers `alex` and `kshitiz` belong to group `devteam`. When either creates a file in `/srv/code`, both must be able to edit it.
```bash
# 1. Create shared directory and set group ownership
sudo mkdir -p /srv/code
sudo chown -R root:devteam /srv/code

# 2. Assign read/write/execute to owner & group, plus SGID (2775)
sudo chmod 2775 /srv/code

# Now, every file created in /srv/code will automatically belong to group 'devteam'!
```

---

## 8. Advanced File Search & Batch Operations (`find`)

### Anatomical Structure of `find`

The `find` utility traverses directory trees in real time, evaluating criteria and executing actions against matched filesystem nodes.

```
find  [Starting Directory]  [Search Criteria / Tests]  [Actions / Operations]
                │                       │                         │
find         /var/log             -name "*.log"                -delete
```

---

### Searching by Name, Type, and Size

```bash
# Search by exact filename (case-sensitive)
find /etc -name "nginx.conf"

# Case-insensitive search (matches App.log, APP.LOG, app.log)
find /var/log -iname "*.log"

# Filter by file type:
find /var/run -type s    # Find local UNIX sockets
find /usr/bin -type l    # Find symbolic links
find /home -type d       # Find directories only
find /tmp -type f        # Find regular files only

# Filter by size:
find /var/log -size +100M               # Files LARGER than 100 Megabytes
find /var/log -size -10k                # Files SMALLER than 10 Kilobytes
find / -size +1G 2>/dev/null            # Find 1GB+ files (suppress permission errors)
```

> [!NOTE]
> `2>/dev/null` redirects `stderr` (Standard Error) to the null device. When running searches as a regular user from root `/`, this cleanly silences hundreds of noisy `Permission denied` output lines.

---

### Searching by Timestamps (`-mtime`, `-mmin`)

Linux tracks three primary timestamps for every file:
- **`mtime` (Modification Time):** When file *content* was last altered.
- **`atime` (Access Time):** When file was last *read*.
- **`ctime` (Change Time):** When file *metadata/inode* (permissions, owner) last changed.

```bash
# Modified in the last 24 hours (less than 1 day ago)
find /var/log -mtime -1

# Modified MORE than 30 days ago (great for archiving/cleanup)
find /var/log/myapp -mtime +30

# Modified in the last 60 minutes
find /etc -mmin -60

# Find empty files or empty directories
find ~/playground -empty
```

---

### Security & Permission Auditing (`-perm`)

```bash
# Find files owned by a specific user
find /home -user deployer

# Find world-writable files (Major Security Audit Check!)
find / -perm -002 -type f 2>/dev/null

# Find SUID binaries on the system (Security Audit for Privilege Escalation vectors)
find / -perm -4000 -type f 2>/dev/null
```

---

### Executing Batch Actions (`-exec`, `-delete`)

```bash
# 1. Direct Delete:
find /var/log/myapp -name "*.tmp" -mtime +7 -delete

# 2. Execute command individually for EACH match ({} replaced by filename, \; ends command):
find ~/scripts -name "*.sh" -exec chmod +x {} \;

# 3. Execute command by BATCHING all matches together (+ runs command once with multiple args):
find /var/log -name "*.log" -size +50M -exec ls -lh {} +

# 4. Interactive execution (prompts y/n before executing action on each file):
find /tmp -name "*.bak" -ok rm {} \;
```

---

### 🚀 DevOps Real-Time Scenario: Automated Log Rotation & Old Artifact Cleanup Cron

**The Situation:** CI/CD runners generate hundreds of build artifacts and log files under `/opt/ci-runner/work/` daily. You need a maintenance cron job that:
1. Purges uncompressed `.log` files older than 14 days.
2. Archives `.tar.gz` artifacts older than 30 days.

```bash
#!/usr/bin/env bash
# /usr/local/bin/ci-cleanup.sh
set -euo pipefail

TARGET_DIR="/opt/ci-runner/work"

echo "[$(date +'%Y-%m-%d %H:%M:%S')] Starting automated cleanup..."

# 1. Delete .log files older than 14 days
find "${TARGET_DIR}" -type f -name "*.log" -mtime +14 -delete

# 2. Move build archives older than 30 days to cold storage mount
find "${TARGET_DIR}" -type f -name "*.tar.gz" -mtime +30 -exec mv -t /mnt/cold_storage/ {} +

# 3. Remove orphaned empty directories
find "${TARGET_DIR}" -type d -empty -delete

echo "[$(date +'%Y-%m-%d %H:%M:%S')] Cleanup completed successfully."
```

#### Crontab Entry (`/etc/cron.d/ci-cleanup`):
```cron
# Run every night at 02:00 AM UTC
0 2 * * * root /usr/local/bin/ci-cleanup.sh >> /var/log/ci-cleanup.log 2>&1
```

---

## 9. DevOps Production Troubleshooting Cheat Sheet

| Operation / Task | Command Syntax | Real-World DevOps Context |
| :--- | :--- | :--- |
| **Inspect Directory Details** | `ls -la /path` | Inspect permissions, ownership, and hidden dotfiles. |
| **Check Human-Readable Sizes** | `ls -lhS` | Quickly identify the largest files in a directory. |
| **Detect Real MIME Type** | `file <filename>` | Verify file type via magic bytes regardless of extension. |
| **Create Directory Tree** | `mkdir -p /path/to/nested/dir` | Safely create nested directories in CI/CD and Dockerfiles. |
| **Preserve Attributes on Copy** | `cp -a src/ dest/` | Backup directories while preserving permissions, symlinks, and timestamps. |
| **Atomic Symlink Update** | `ln -sfn <new_release> <symlink>` | Zero-downtime blue/green application version switching. |
| **Check File Inode** | `ls -i <file>` / `ls -li <file>` | Troubleshoot hard links and disk inode exhaustion. |
| **Set Secure Key Permissions** | `chmod 600 ~/.ssh/id_rsa` | Resolve SSH "unprotected private key file" authentication errors. |
| **Set Team Shared Directory** | `chmod 2775 /shared && chown :devs /shared` | Ensure all newly created files inherit the shared group ID automatically. |
| **Live Follow Active Logs** | `tail -F /var/log/app.log` | Stream production logs across logrotate rotations seamlessly. |
| **Search Massive Log Files** | `less /var/log/syslog` | Memory-efficient lazy-loaded log navigation and pattern searching. |
| **Find Files Larger Than 500MB** | `find / -type f -size +500M 2>/dev/null` | Triage disk exhaustion alerts on production nodes. |
| **Purge Logs Older Than 30 Days**| `find /var/log/app -name "*.log" -mtime +30 -delete` | Automated disk retention and log cleanup policies. |
| **Batch Make Scripts Executable** | `find . -name "*.sh" -exec chmod +x {} +` | Make all repository scripts executable in one command. |

---

*Authored for Linux System Administrators & DevOps Engineers.*
