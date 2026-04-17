# Multi-Container Runtime — OS-Jackfruit

A lightweight Linux container runtime written in C, featuring a long-running parent supervisor, isolated container execution using Linux namespaces, a bounded-buffer logging pipeline, and a kernel-space memory monitor implemented as a Linux Kernel Module (LKM).

---

## Table of Contents

- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [Setup & Installation](#setup--installation)
- [Build](#build)
- [Usage](#usage)
- [Kernel Module & Memory Monitor](#kernel-module--memory-monitor)
- [Design Decisions](#design-decisions)
- [Screenshots](#screenshots)

---

## Project Overview

This project builds two integrated components:

**1. User-Space Runtime + Supervisor (`engine.c`)**
Launches and manages multiple isolated containers concurrently. Handles container lifecycle (start, run, stop, ps, logs), IPC via UNIX domain sockets, and captures container output through a producer-consumer bounded-buffer logging system.

**2. Kernel-Space Monitor (`monitor.c`)**
A Linux Kernel Module that tracks container processes by PID, enforces soft and hard memory limits by periodically polling RSS, and integrates with the user-space runtime through `ioctl`. Soft limit violations are logged to `dmesg`; hard limit violations trigger `SIGKILL`.

---

## Architecture

The runtime is a single binary (`engine`) used in two distinct modes:

- **Supervisor daemon** — started once with `engine supervisor <rootfs>`. Stays alive, owns the UNIX domain socket at `/tmp/mini_runtime.sock`, manages container records, and drives the logging pipeline.
- **CLI client** — each invocation (e.g. `engine start`, `engine ps`) is a short-lived process that connects to the supervisor socket, sends a request, receives a response, and exits.

### IPC Paths

| Path | Direction | Mechanism |
|------|-----------|-----------|
| Control | CLI → Supervisor | UNIX domain socket (`/tmp/mini_runtime.sock`) |
| Logging | Container stdout/stderr → Supervisor | Pipes → bounded ring buffer |

### Container Isolation

Each container is spawned via `clone()` with isolated `PID`, `UTS`, and `mount` namespaces. The container process `chroot`s into its own writable rootfs copy before executing the target command.

### Memory Monitoring

The kernel module uses a periodic timer (1-second interval) to poll each tracked container's RSS via `get_task_mm()`. Limits are registered and unregistered from user-space through `ioctl` calls (`MONITOR_REGISTER` / `MONITOR_UNREGISTER`). The module maintains a mutex-protected linked list of monitored entries.

---

## Setup & Installation

### Requirements

- Ubuntu 22.04 or 24.04 in a VM (**not WSL**)
- Secure Boot **OFF** (required for kernel module loading)
- `build-essential` and `linux-headers` for your running kernel

### Install Dependencies

```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r)
```

### Run Environment Check

```bash
cd boilerplate
chmod +x environment-check.sh
sudo ./environment-check.sh
```

Fix any issues reported before proceeding.

### Prepare Root Filesystem

The runtime uses an Alpine Linux mini rootfs. Because the system is ARM64 (aarch64), use the `aarch64` tarball:

```bash
mkdir rootfs-base
cd rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/aarch64/alpine-minirootfs-3.20.3-aarch64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-aarch64.tar.gz
cd ..

# Create one writable copy per container
cp -a ./rootfs-base ./rootfs-alpha
cp -a ./rootfs-base ./rootfs-beta
```

> **Note:** Do not commit `rootfs-base/` or `rootfs-*` directories to the repository.

---

## Build

```bash
cd boilerplate
make
```

This produces:
- `engine` — the supervisor/CLI binary
- `memory_hog`, `cpu_hog`, `io_pulse` — statically linked test workloads
- `monitor.ko` — the kernel module

For CI (no kernel headers required):

```bash
make -C boilerplate ci
```

---

## Usage

### 1. Load the Kernel Module

```bash
sudo insmod monitor.ko
```

Verify it loaded:

```bash
sudo dmesg | grep container_monitor
# [ 486.486545] [container_monitor] Module loaded. Device: /dev/container_monitor
```

### 2. Start the Supervisor

```bash
sudo ./engine supervisor rootfs-base
# [supervisor] Ready. Listening on /tmp/mini_runtime.sock
```

### 3. CLI Commands

```bash
# Start a long-running container (non-blocking)
sudo ./engine start <id> <rootfs> <command> [--soft-mib N] [--hard-mib N]

# Run a container and block until it exits
sudo ./engine run <id> <rootfs> <command>

# List all containers
sudo ./engine ps

# Print logs for a container
sudo ./engine logs <id>

# Stop a container
sudo ./engine stop <id>
```

> Running `engine` commands without `sudo` will fail with a permission error on the socket — always use `sudo`.

### Memory Limit Flags

| Flag | Default | Description |
|------|---------|-------------|
| `--soft-mib N` | 40 MiB | Log warning when RSS exceeds this |
| `--hard-mib N` | 64 MiB | SIGKILL the container when RSS exceeds this |

---

## Kernel Module & Memory Monitor

The `monitor.ko` kernel module exposes a character device at `/dev/container_monitor`. The user-space supervisor opens this device and uses `ioctl` to register/unregister containers.

### ioctl Commands (defined in `monitor_ioctl.h`)

| Command | Action |
|---------|--------|
| `MONITOR_REGISTER` | Register a container PID with soft/hard limits |
| `MONITOR_UNREGISTER` | Remove a container from tracking |

### Limit Enforcement

- **Soft limit:** Logged once to `dmesg` when RSS first exceeds the threshold. Container continues running.
- **Hard limit:** `SIGKILL` sent immediately; entry removed from the monitored list.

Monitor events are visible via:

```bash
sudo dmesg | grep container_monitor
```

---

## Design Decisions

**`clone()` over `fork()`** — `clone()` allows fine-grained namespace flags (`CLONE_NEWPID`, `CLONE_NEWUTS`, `CLONE_NEWNS`) in a single call, giving each container its own PID tree, hostname, and mount namespace without a separate `unshare()` step.

**Mutex over spinlock in the kernel module** — The timer callback and `ioctl` paths both call `get_task_mm()`, which can block. A spinlock would be unsafe here; a `DEFINE_MUTEX` is the correct choice.

**Bounded ring buffer for logging** — A fixed-capacity producer-consumer ring buffer (capacity 16, chunk size 4096 bytes) decouples container I/O from log file writes. This prevents a slow disk from stalling container stdout and avoids unbounded memory growth.

**UNIX domain socket for IPC** — Simpler and faster than TCP for local process communication. The socket path `/tmp/mini_runtime.sock` is created by the supervisor and requires `sudo` to access (root-owned), which also prevents unprivileged CLI invocations from accidentally reaching the daemon.

**Alpine aarch64 rootfs** — The VM runs on ARM64 (`uname -m` → `aarch64`), so the x86_64 rootfs from the README template would fail with `Exec format error`. The aarch64 tarball is used instead.

---

## Screenshots

### Build Output

Successful `make` showing compilation of user-space binaries and the kernel module:

```
gcc -O2 -Wall -static -o memory_hog memory_hog.c
gcc -O2 -Wall -static -o cpu_hog cpu_hog.c
gcc -O2 -Wall -static -o io_pulse io_pulse.c
make -C /lib/modules/6.8.0-107-generic/build M=/home/arya/OS-Jackfruit/boilerplate modules
make[1]: Entering directory '/usr/src/linux-headers-6.8.0-107-generic'
  CC [M]  /home/arya/OS-Jackfruit/boilerplate/monitor.o
  MODPOST /home/arya/OS-Jackfruit/boilerplate/Module.symvers
  CC [M]  /home/arya/OS-Jackfruit/boilerplate/monitor.mod.o
  LD [M]  /home/arya/OS-Jackfruit/boilerplate/monitor.ko
  BTF [M] /home/arya/OS-Jackfruit/boilerplate/monitor.ko
make[1]: Leaving directory '/usr/src/linux-headers-6.8.0-107-generic'
```

### Root Filesystem Setup

Downloading the aarch64 Alpine rootfs and creating per-container copies:

```
arya@pes1ug24am054:~/OS-Jackfruit/boilerplate$ rm -rf rootfs-base rootfs-alpha rootfs-beta
arya@pes1ug24am054:~/OS-Jackfruit/boilerplate$ mkdir rootfs-base
cd rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/aarch64/alpine-minirootfs-3.20.3-aarch64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-aarch64.tar.gz
cd ..
--2026-04-16 16:18:18--  https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/aarch64/alpine-minirootfs-3.20.3-aarch64.tar.gz
Resolving dl-cdn.alpinelinux.org... 151.101.158.132
Connecting to dl-cdn.alpinelinux.org|151.101.158.132|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3947906 (3.8M) [application/octet-stream]
Saving to: 'alpine-minirootfs-3.20.3-aarch64.tar.gz'
alpine-minirootfs-3 100%[===================>]   3.76M  11.0MB/s    in 0.3s
2026-04-16 16:18:18 (11.0 MB/s) - 'alpine-minirootfs-3.20.3-aarch64.tar.gz' saved [3947906/3947906]
arya@pes1ug24am054:~/OS-Jackfruit/boilerplate$ cp -a rootfs-base rootfs-alpha
cp -a rootfs-base rootfs-beta
arya@pes1ug24am054:~/OS-Jackfruit/boilerplate$
```

### Module Load Confirmed in dmesg

```
arya@pes1ug24am054:~/OS-Jackfruit/boilerplate$ sudo dmesg | grep container_monitor
[ 486.486545] [container_monitor] Module loaded. Device: /dev/container_monitor
arya@pes1ug24am054:~/OS-Jackfruit/boilerplate$
```

### Supervisor Start

Supervisor listening on `/tmp/mini_runtime.sock` after module load:

```
[ 486.486545] [container_monitor] Module loaded. Device: /dev/container_monitor
arya@pes1ug24am054:~/OS-Jackfruit/boilerplate$ sudo ./engine supervisor rootfs-base
[supervisor] Ready. Listening on /tmp/mini_runtime.sock
```

### Starting a Container

```
arya@pes1ug24am054:~/OS-Jackfruit/boilerplate$ sudo ./engine start alpha rootfs-alpha /bin/sh
[sudo] password for arya:
Started container 'alpha' pid=5725
arya@pes1ug24am054:~/OS-Jackfruit/boilerplate$
```

### Listing Running Containers (`engine ps`)

```
arya@pes1ug24am054:~/OS-Jackfruit/boilerplate$ sudo ./engine ps
ID          PID     STATE       EXIT    SOFT-MIB    HARD-MIB
alpha       5725    running     0       40          64

arya@pes1ug24am054:~/OS-Jackfruit/boilerplate$
```

### Multiple Containers Running

```
arya@pes1ug24am054:~/OS-Jackfruit/boilerplate$ sudo ./engine start alpha rootfs-alpha /bin/sh
sudo ./engine start beta rootfs-beta /bin/sh
Started container 'alpha' pid=6982
Started container 'beta' pid=6990
arya@pes1ug24am054:~/OS-Jackfruit/boilerplate$ sudo ./engine ps
ID          PID     STATE       EXIT    SOFT-MIB    HARD-MIB
beta        6990    running     0       40          64
alpha       6982    running     0       40          64
```

### Running a Short-Lived Container (`engine run`)

```
arya@pes1ug24am054:~/OS-Jackfruit/boilerplate$ sudo ./engine run test rootfs-alpha "/bin/echo hello"
Running container 'test' pid=6495 (waiting...)
Container 'test' finished with exit_code=0
arya@pes1ug24am054:~/OS-Jackfruit/boilerplate$
```

### Viewing Logs

```
arya@pes1ug24am054:~/OS-Jackfruit/boilerplate$ sudo ./engine logs test
hello
arya@pes1ug24am054:~/OS-Jackfruit/boilerplate$
```

### Memory Limit Enforcement — Hard Kill

Running `memory_hog` inside a container triggers the kernel module to detect RSS exceeding the hard limit and send SIGKILL (exit code 137 = 128 + SIGKILL):

```
arya@pes1ug24am054:~/OS-Jackfruit/boilerplate$ sudo ./engine run memtest rootfs-alpha /memory_hog
Running container 'memtest' pid=6596 (waiting...)
Container 'memtest' finished with exit_code=137
137 arya@pes1ug24am054:~/OS-Jackfruit/boilerplate$
```

### dmesg — Full Monitor Event Log

Shows registration, soft-limit warning, hard-limit kill, and unregister events across the full session:

```
arya@pes1ug24am054:~/OS-Jackfruit/boilerplate$ sudo dmesg | grep container_monitor
[  486.486545] [container_monitor] Module loaded. Device: /dev/container_monitor
[  614.363937] [container_monitor] Registering container=alpha pid=5725 soft=41943040 hard=67108864
[  947.137578] [container_monitor] Registering container=test pid=6495 soft=41943040 hard=67108864
[  947.139767] [container_monitor] Unregister request container=test pid=6495
[ 1031.045980] [container_monitor] Registering container=memtest pid=6596 soft=41943040 hard=67108864
[ 1035.351381] [container_monitor] SOFT LIMIT container=memtest pid=6596 rss=42336256 limit=41943040
[ 1038.423465] [container_monitor] HARD LIMIT container=memtest pid=6596 rss=67502080 limit=67108864
[ 1038.426385] [container_monitor] Unregister request container=memtest pid=6596
arya@pes1ug24am054:~/OS-Jackfruit/boilerplate$
```

### Permission Error Without sudo

Attempting to run `engine` without root privileges results in a socket permission error:

```
arya@pes1ug24am054:~/OS-Jackfruit/boilerplate$ ./engine ps
connect (is the supervisor running?): Permission denied
1 arya@pes1ug24am054:~/OS-Jackfruit/boilerplate$
```

---

## File Structure

```
boilerplate/
├── engine.c              # User-space runtime and supervisor
├── monitor.c             # Kernel module
├── monitor_ioctl.h       # Shared ioctl definitions
├── Makefile              # Builds both user-space and kernel targets
├── cpu_hog.c             # CPU-bound test workload
├── io_pulse.c            # I/O-bound test workload
├── memory_hog.c          # Memory-consuming test workload
├── loop.c                # Loop test workload
└── environment-check.sh  # VM preflight check
```
