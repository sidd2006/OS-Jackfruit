# Multi-Container Runtime

## 1. Team Information

Name - Siddhant Khurdiya | PES1UG24AM275
       Shubham Bhat | PES1UG24AM273

---

## 2. Build, Load, and Run Instructions

### Prerequisites
- Ubuntu 22.04/24.04 VM with Secure Boot OFF
- Dependencies: `sudo apt install -y build-essential linux-headers-$(uname -r)`

### Build
```bash
cd boilerplate
make
```

### Prepare Root Filesystem
```bash
cd ~/OS-Jackfruit
mkdir rootfs
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs
```

### Load Kernel Module
```bash
sudo insmod boilerplate/monitor.ko
ls -l /dev/container_monitor
```

### Start Supervisor
```bash
sudo ./boilerplate/engine supervisor ./rootfs
```

### Launch Containers (in another terminal)
```bash
sudo ./boilerplate/engine start alpha ./rootfs /bin/sh
sudo ./boilerplate/engine start beta ./rootfs /bin/sh
sudo ./boilerplate/engine ps
sudo ./boilerplate/engine logs alpha
sudo ./boilerplate/engine stop alpha
```

### Memory Limit Test
```bash
cp boilerplate/memory_hog rootfs/
sudo ./boilerplate/engine start memtest ./rootfs "/memory_hog 5 500" --soft-mib 8 --hard-mib 25
sleep 15
sudo dmesg | grep -E "SOFT|HARD"
```

### Scheduler Experiment
```bash
cp boilerplate/cpu_hog rootfs/
sudo ./boilerplate/engine start cpulow ./rootfs "/cpu_hog" --nice 10
sudo ./boilerplate/engine start cpuhigh ./rootfs "/cpu_hog" --nice -10
sleep 10
sudo ./boilerplate/engine logs cpulow
sudo ./boilerplate/engine logs cpuhigh
```

### Cleanup
```bash
sudo ./boilerplate/engine stop alpha
sudo ./boilerplate/engine stop beta
# Ctrl+C to stop supervisor
sudo rmmod monitor
```

---

## 3. Demo with Screenshots

### Screenshot 1 — Multi-container supervision
Two containers (alpha, beta) started and running under one supervisor process.
![Screenshot 1a](ss-1a.png)
![Screenshot 1b](ss-1b.png)

### Screenshot 2 — Metadata tracking
Output of `ps` command showing container ID, PID, state, exit code, and signal.

![Screenshot 2](ss-2.png)



### Screenshot 3 — Bounded-buffer logging
Log file contents captured through the logging pipeline.

![Screenshot 3](ss-3.png)

### Screenshot 4 — CLI and IPC
CLI command (`stop`) issued and supervisor responding with state update.

![Screenshot 4](ss-4.png)

### Screenshot 5 — Soft-limit warning
`dmesg` showing SOFT LIMIT warning when container exceeds soft memory limit.

![Screenshot 5,6](ss-5&6.png)

### Screenshot 6 — Hard-limit enforcement
`dmesg` showing HARD LIMIT kill and `ps` showing container state as `killed` with signal 9.


### Screenshot 7 — Scheduling experiment
Two CPU-bound containers with different nice values showing different CPU throughput.

![Screenshot 7](ss-7.png)

### Screenshot 8 — Clean teardown
All containers reaped, supervisor clean exit, module unloaded with no zombies.

![Screenshot 8a](ss-8a.png)

![Screenshot 8b](ss-8b.png)

---

## 4. Engineering Analysis

### 1. Isolation Mechanisms
The runtime uses `clone()` with `CLONE_NEWPID`, `CLONE_NEWUTS`, and `CLONE_NEWNS` flags to create isolated namespaces for each container. PID namespace isolation gives each container its own PID 1, preventing it from seeing host processes. UTS namespace isolation allows each container to have its own hostname set via `sethostname()`. Mount namespace isolation enables `chroot()` into the Alpine rootfs, giving each container its own filesystem view. The host kernel is still shared — the same kernel handles all system calls from all containers. Resources like network interfaces, the host clock, and kernel memory are shared unless additional namespaces are added.

### 2. Supervisor and Process Lifecycle
A long-running supervisor is necessary because containers are child processes that must be reaped when they exit — orphaned children become zombies without a waiting parent. The supervisor uses `clone()` to create containers, maintains a linked list of `container_record_t` metadata for each, and installs a `SIGCHLD` handler to reap children asynchronously with `waitpid(WNOHANG)`. `SIGINT`/`SIGTERM` trigger orderly shutdown: running containers are sent `SIGTERM`, threads are joined, and all heap resources are freed.

### 3. IPC, Threads, and Synchronization
The project uses two IPC mechanisms. A pipe per container carries stdout/stderr from the container to a producer thread in the supervisor. A UNIX domain socket (`/tmp/mini_runtime.sock`) serves as the control channel between CLI clients and the supervisor. The bounded buffer between producer threads and the logger consumer thread uses a `pthread_mutex_t` for mutual exclusion and two `pthread_cond_t` variables (`not_full`, `not_empty`) to block producers when the buffer is full and consumers when it is empty. Without the mutex, concurrent producers could corrupt the buffer's head/tail indices. Without condition variables, threads would busy-wait, wasting CPU. The container metadata list uses a separate `metadata_lock` mutex because it is accessed by both the SIGCHLD handler and the command handler concurrently.

### 4. Memory Management and Enforcement
RSS (Resident Set Size) measures the amount of physical RAM currently mapped and used by a process. It does not measure swap usage, shared library pages counted multiple times, or memory allocated but not yet touched. Soft and hard limits represent different enforcement policies: a soft limit triggers a warning log but allows the process to continue, giving it a chance to free memory. A hard limit terminates the process immediately. Enforcement belongs in kernel space because a user-space monitor could be killed or paused by the scheduler, creating a window where a process exceeds limits undetected. The kernel timer fires reliably every second regardless of user-space state.

### 5. Scheduling Behavior
Linux uses the Completely Fair Scheduler (CFS), which assigns CPU time proportional to each task's weight. Nice values map to weights: a lower nice value means higher weight and more CPU time. In our experiment, `cpuhigh` (nice=-10) and `cpulow` (nice=10) ran concurrently. CFS gave `cpuhigh` significantly more CPU time per scheduling period, resulting in a higher accumulator value after 10 seconds. This demonstrates that CFS does not enforce strict time-sharing equality — it favors higher-priority tasks while still ensuring lower-priority tasks make progress.

---

## 5. Design Decisions and Tradeoffs

**Namespace isolation:** Used `CLONE_NEWPID | CLONE_NEWUTS | CLONE_NEWNS` with `chroot`. The tradeoff is that network isolation (`CLONE_NEWNET`) is not implemented, so containers share the host network stack. This was the right call given time constraints — the project focuses on process and filesystem isolation.

**Supervisor architecture:** Single-process supervisor with a UNIX domain socket for control. The tradeoff is that the supervisor is a single point of failure. A crash kills all container tracking. This is justified because it keeps the design simple and matches the project scope.

**IPC/logging:** Pipe per container feeding a bounded buffer consumed by one logger thread. The tradeoff is that a slow logger can block producers if the buffer fills. This is acceptable because the buffer size (16 chunks) is large enough for normal workloads.

**Kernel monitor:** Used `find_get_pid()` at registration time to store a `struct pid *` pointer rather than looking up PIDs by number in the timer. The tradeoff is that the stored pointer must be released with `put_pid()` on removal to avoid reference count leaks. This design correctly handles PID namespace boundaries.

**Scheduling experiments:** Used nice values rather than cgroups CPU quotas. The tradeoff is that nice values only influence CFS weights, not hard CPU limits. This is sufficient to demonstrate observable scheduling differences without requiring cgroup setup.

---

## 6. Scheduler Experiment Results

| Container | Nice Value | Final Accumulator | Duration |
|-----------|------------|-------------------|----------|
| cpuhigh   | -10        | 163986464650570990065 | 10s |
| cpulow    | +10        | 45623213588543582​60 | 10s |

`cpuhigh` achieved a higher accumulator value in the same wall-clock time, demonstrating that CFS allocated more CPU time to the higher-priority process. The ratio reflects the CFS weight difference between nice=-10 and nice=+10, which is approximately 110:1 in the kernel's weight table. In practice the observed ratio is lower due to I/O, scheduling overhead, and the single-CPU VirtualBox environment. The results confirm that Linux scheduling is not purely equal time-sharing — priority influences CPU allocation in a measurable and predictable way.
```

Save as `README.md`, scp it:
```
