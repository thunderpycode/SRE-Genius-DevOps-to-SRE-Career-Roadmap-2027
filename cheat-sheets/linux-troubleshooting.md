# 🐧 Linux Performance & Troubleshooting Master Reference for SREs

A production-grade diagnostic manual and reference guide covering system load, CPU/memory profiling, disk I/O bottlenecks, network latency, kernel logs, and live incident triage.

---

## 📑 Quick Navigation
1. [The USE Method & 60-Second Triage Checklist](#1-the-use-method--60-second-triage-checklist)
2. [CPU Saturation & Load Average Diagnostics](#2-cpu-saturation--load-average-diagnostics)
3. [Memory Bottlenecks, Swapping & OOM Triage](#3-memory-bottlenecks-swapping--oom-triage)
4. [Storage, Filesystems & Disk I/O Saturation](#4-storage-filesystems--disk-io-saturation)
5. [Network Latency, Sockets & Packet Diagnostics](#5-network-latency-sockets--packet-diagnostics)
6. [Process State, Tracing & System Calls](#6-process-state-tracing--system-calls)
7. [Systemd, Services & Journald Auditing](#7-systemd-services--journald-auditing)
8. [Kernel Errors, Dmesg & Hardware Checks](#8-kernel-errors-dmesg--hardware-checks)
9. [Production Incident Triage Matrix](#9-production-incident-triage-matrix)

---

## 1. The USE Method & 60-Second Triage Checklist

When paged for a host incident, run these commands in sequence to isolate the constrained subsystem (CPU, Memory, Disk, or Network):

```bash
# 1. System uptime and 1, 5, 15 minute load averages
uptime

# 2. Kernel ring buffer errors or OOM events
dmesg -T | tail -n 50

# 3. Virtual memory statistics (run queue, context switches, swap in/out)
vmstat 1 5

# 4. CPU breakdown per core (%user, %system, %iowait, %steal)
mpstat -P ALL 1 2

# 5. Process-level CPU/Memory consumption
pidstat 1 3

# 6. Disk device utilization (%util, wait times, r/w IOPS)
iostat -xz 1 3

# 7. Physical and swap memory utilization
free -h

# 8. Network interface throughput and dropped packet counters
sar -n DEV 1 3

# 9. Network TCP transmission error rates and retransmissions
sar -n TCP,ETCP 1 3

# 10. Live interactive view sorted by resource hog
top -c
```

---

## 2. CPU Saturation & Load Average Diagnostics

### Understanding Load Average vs. CPU Cores
* **Load average** represents processes running, waiting to run (runnable), or in uninterruptible sleep (`D` state, usually disk/NFS I/O).
* Compare load to physical core count: `nproc`. If `load > nproc`, jobs are queuing.

```bash
# Print total number of available processing cores
nproc

# Profile processes consuming the highest CPU %
ps -eo pid,ppid,user,%cpu,%mem,time,comm --sort=-%cpu | head -n 15

# Real-time process CPU profiling with threads and commands
top -H -p <PID>

# Detect runaway context switching (indicates high lock contention or thread churn)
vmstat 1 5
# Look at the 'cs' (context switches) and 'in' (interrupts) columns
```

---

## 3. Memory Bottlenecks, Swapping & OOM Triage

### Memory Breakdown & Leak Detection
```bash
# Detailed memory footprint (Buffers, Cached, Available)
free -m -w

# Check detailed kernel slab, dirty cache, and anonymous memory
cat /proc/meminfo | head -n 20

# Identify top 10 memory-consuming processes
ps -eo pid,user,%mem,rss,vsz,comm --sort=-rss | head -n 11

# Inspect memory usage for a specific process PID in human-readable units
cat /proc/<PID>/status | grep -E 'VmSize|VmRSS|VmSwap'
```

### OOM Killer Investigation
```bash
# Check if kernel OOM-killer killed a process recently
dmesg -T | grep -i -E 'killed process|out of memory'

# Inspect system logs for past OOM events
journalctl -k --grep="Out of memory" --since "2 hours ago"

# View OOM score adjustment of a critical process (higher score = killed first)
cat /proc/<PID>/oom_score
```

---

## 4. Storage, Filesystems & Disk I/O Saturation

### Filesystem & Inode Exhaustion
```bash
# Check available filesystem space
df -hT

# Check inode usage (prevents "No space left on device" when disk % is not 100%)
df -ih

# Find top 10 largest directories on a filesystem
du -xh /var | sort -rh | head -n 10

# Find and list deleted files still held open by processes (unreleased disk space)
lsof +L1
```

### Disk Latency & I/O Saturation
```bash
# Detailed block device statistics (1-second intervals)
# Check: %util (near 100% means saturated), await (latency in ms)
iostat -xz 1 5

# Trace live disk I/O activity by process (requires root)
iotop -oPa

# Flush write buffers to disk manually during diagnostics
sync
```

---

## 5. Network Latency, Sockets & Packet Diagnostics

### Socket Allocation & Port Listening
```bash
# List all active listening TCP/UDP ports with PID and program name
ss -tulpn

# Display summary of all open socket states (ESTAB, TIME-WAIT, CLOSE-WAIT)
ss -s

# Check for socket buffer backlog or dropped packets on sockets
ss -tina
```

### Connectivity, DNS & Route Diagnostics
```bash
# Trace full path with per-hop latency and packet loss (TCP SYN mode)
traceroute -T -p 443 <destination-ip-or-domain>

# Perform deep DNS resolution chain trace
dig +trace +question <domain-name>

# Ping with timestamp and interval to detect intermittent jitter
ping -D -c 10 -i 0.5 <gateway-or-target>

# Test TCP socket handshake without telnet
nc -zv -w 3 <destination-host> <port>
```

### Live Traffic Capture
```bash
# Capture and inspect live traffic on eth0 for a specific port
sudo tcpdump -i eth0 -nn 'tcp port 443' -c 50

# Capture dropped or reset TCP packets (RST flag set)
sudo tcpdump -i any -nn 'tcp[tcpflags] & (tcp-rst) != 0'
```

---

## 6. Process State, Tracing & System Calls

### Process States & Thread Hierarchies
* `R`: Running / Runnable
* `S`: Interruptible sleep (waiting for event/timer)
* `D`: Uninterruptible sleep (usually waiting on Disk/NFS I/O)
* `Z`: Zombie (terminated, parent has not reaped exit code)

```bash
# Find all processes stuck in uninterruptible sleep ('D' state)
ps -eo state,pid,user,cmd | grep "^D"

# List all Zombie processes and their parent PIDs
ps -eo ppid,pid,user,stat,comm | grep -w "Z"

# Print hierarchical process tree
pstree -p -a
```

### Live Process Tracing (`strace` & `lsof`)
```bash
# Trace syscalls and execution time of a running, stuck process
strace -p <PID> -T -f -s 256

# Count syscall execution frequencies and total time spent
strace -p <PID> -c

# List all open files, sockets, and pipes held by a process
lsof -p <PID>

# Find which process is locking a specific port or file
lsof -i :8080
lsof /var/log/app.log
```

---

## 7. Systemd, Services & Journald Auditing

```bash
# Check status, exit code, and recent logs of a unit
systemctl status <service-name>

# List all failed systemd units on the host
systemctl --failed

# Tail logs of a specific unit in real-time
journalctl -u <service-name> -f -n 100

# View priority errors (crit, err, alert) logged since last boot
journalctl -p err -b

# Clean up journald logs if /var/log/journal is consuming excessive disk
journalctl --vacuum-size=500M
```

---

## 8. Kernel Errors, Dmesg & Hardware Checks

```bash
# Inspect human-readable kernel ring buffer for hardware/driver failures
dmesg -T -l err,crit,alert

# Check hardware temperature, fan speeds, and voltage
sensors

# Inspect PCI devices, disk controllers, and NIC hardware
lspci -nnk
ethtool <interface-name>
```

---

## 9. Production Incident Triage Matrix

| Symptom / Alert | Likely Cause | Diagnostic Command | Immediate Mitigation Action |
| :--- | :--- | :--- | :--- |
| **High Load Avg with Low CPU %** | Processes blocked on disk or network storage (NFS) | `ps -eo state,pid,cmd \| grep "^D"` | Check `iostat -xz 1 2`; unmount hanging remote mounts or fix unresponsive SAN/EBS. |
| **High CPU %Steal** | Noisy neighbor on cloud virtual host / oversubscribed hypervisor | `mpstat 1 5` (check `%steal`) | Resize instance, migrate VM, or report hypervisor degradation to cloud provider. |
| **Disk 100% Full but No Files Found** | Deleted files still held open by un-restarted processes | `lsof +L1` | Restart the application process holding the deleted file descriptor. |
| **Cannot Allocate Memory / OOM** | Memory leak or aggressive process allocations | `dmesg -T \| grep -i oom` | Restart offending service; tune heap/worker pools or increase swap/RAM. |
| **Too Many Open Files (EMFILE)** | File descriptor limit reached for process or system | `cat /proc/sys/fs/file-nr` & `lsof -p <PID> \| wc -l` | Increase `nofile` limit in `/etc/security/limits.conf` or `systemd` service file. |
| **High CLOSE_WAIT Sockets** | App received remote FIN but failed to close socket connection | `ss -tan \| grep CLOSE-WAIT` | Application bug in socket handling; restart application and report connection pool bug. |

---

*Authored for the [SRE Genius](https://www.youtube.com/@SREGENIUS) community.*