# 00. Linux Fundamentals

[← Назад к списку тем](README.md)

---

## Process Management

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Process States                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Running (R)     — Currently executing or ready to run              │
│  Sleeping (S)    — Waiting for event (interruptible)                │
│  Disk Sleep (D)  — Waiting for I/O (uninterruptible)                │
│  Stopped (T)     — Suspended (SIGSTOP/SIGTSTP)                      │
│  Zombie (Z)      — Terminated, waiting for parent to read exit      │
│                                                                     │
│  Process hierarchy:                                                 │
│  - init (PID 1) — parent of all processes                           │
│  - fork() — create child process                                    │
│  - exec() — replace process image                                   │
│  - wait() — parent waits for child                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Commands

```bash
# View processes
ps aux                    # All processes with details
ps -ef                    # Full format listing
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%mem | head  # Top by memory

# Real-time monitoring
top                       # Process monitor
htop                      # Better top (if installed)

# Process tree
pstree -p                 # Show process tree with PIDs

# Find process
pgrep -f "pattern"        # Find by pattern
pidof nginx               # Find PID by name

# Send signals
kill -9 PID               # SIGKILL (force kill)
kill -15 PID              # SIGTERM (graceful)
kill -HUP PID             # SIGHUP (reload config)
killall nginx             # Kill by name

# Background jobs
command &                 # Run in background
nohup command &           # Survive terminal close
jobs                      # List background jobs
fg %1                     # Bring job to foreground
bg %1                     # Continue job in background
```

### Signals

```
┌─────────────────┬─────────────────────────────────────────────────┐
│ Signal          │ Description                                     │
├─────────────────┼─────────────────────────────────────────────────┤
│ SIGTERM (15)    │ Graceful termination (default kill)             │
│ SIGKILL (9)     │ Force kill (cannot be caught)                   │
│ SIGHUP (1)      │ Hangup / reload config                          │
│ SIGINT (2)      │ Interrupt (Ctrl+C)                              │
│ SIGSTOP (19)    │ Stop process (cannot be caught)                 │
│ SIGCONT (18)    │ Continue stopped process                        │
│ SIGUSR1/2       │ User-defined signals                            │
└─────────────────┴─────────────────────────────────────────────────┘
```

---

## Memory Management

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Linux Memory                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Physical Memory                                                    │
│  ├── Used                                                           │
│  │   ├── Active — Recently accessed                                 │
│  │   └── Inactive — Candidate for reclaim                           │
│  ├── Buffers — Block device cache                                   │
│  ├── Cached — File system cache                                     │
│  └── Free — Immediately available                                   │
│                                                                     │
│  Virtual Memory                                                     │
│  - Each process has own address space                               │
│  - Page tables map virtual → physical                               │
│  - Page size typically 4KB                                          │
│                                                                     │
│  Swap                                                               │
│  - Extension of RAM on disk                                         │
│  - Used when physical memory is low                                 │
│  - Very slow compared to RAM                                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Commands

```bash
# Memory overview
free -h                   # Human-readable memory stats
# Note: "available" is what can be allocated, not just "free"

# Detailed memory info
cat /proc/meminfo         # Full memory statistics

# Per-process memory
ps aux --sort=-%mem | head -10    # Top memory consumers
pmap -x PID               # Memory map for process

# OOM Killer
dmesg | grep -i "out of memory"   # Check OOM kills
cat /proc/PID/oom_score           # Process OOM score

# Swap
swapon --show             # Show swap usage
vmstat 1                  # Virtual memory stats every 1 sec
```

### Memory Metrics to Know

```
┌─────────────────┬──────────────────────────────────────────────────┐
│ Metric          │ Meaning                                          │
├─────────────────┼──────────────────────────────────────────────────┤
│ RSS             │ Resident Set Size — physical memory used         │
│ VSZ             │ Virtual Size — total virtual memory              │
│ Shared          │ Memory shared with other processes               │
│ Cache/Buffers   │ Memory used for caching (can be reclaimed)       │
│ Available       │ Memory available for allocation                  │
└─────────────────┴──────────────────────────────────────────────────┘
```

---

## Networking

### Network Commands

```bash
# Interface info
ip addr                   # Show IP addresses
ip link                   # Show interfaces
ifconfig                  # Legacy, still common

# Routing
ip route                  # Show routing table
netstat -rn               # Legacy routing display
traceroute google.com     # Trace packet path

# Connections
netstat -tuln             # TCP/UDP listening ports
ss -tuln                  # Modern netstat replacement
ss -tunp                  # Show process using ports
lsof -i :8080             # What's using port 8080

# DNS
nslookup google.com       # DNS lookup
dig google.com            # Detailed DNS query
host google.com           # Simple DNS lookup
cat /etc/resolv.conf      # DNS servers configured

# Connectivity
ping host                 # ICMP ping
telnet host port          # Test TCP connection
nc -zv host port          # Netcat port test
curl -v http://host       # HTTP request with details

# Traffic capture
tcpdump -i eth0           # Capture packets
tcpdump port 80           # Capture port 80 traffic
tcpdump -w capture.pcap   # Save to file
```

### Network Configuration Files

```bash
/etc/hosts                # Local hostname mapping
/etc/resolv.conf          # DNS configuration
/etc/nsswitch.conf        # Name service order
/etc/sysctl.conf          # Kernel network parameters
```

### Common Network Issues

```
┌─────────────────────────────────────────────────────────────────────┐
│                  Network Troubleshooting                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Connection refused                                                 │
│  → Service not running or wrong port                                │
│  → Check: ss -tuln | grep PORT                                      │
│                                                                     │
│  Connection timeout                                                 │
│  → Firewall blocking or host unreachable                            │
│  → Check: iptables -L, security groups                              │
│                                                                     │
│  DNS resolution fails                                               │
│  → Wrong DNS servers or DNS unreachable                             │
│  → Check: cat /etc/resolv.conf, nslookup                            │
│                                                                     │
│  Slow network                                                       │
│  → Bandwidth, latency, or packet loss                               │
│  → Check: ping, mtr, iperf                                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## File System

### Key Commands

```bash
# Disk usage
df -h                     # Filesystem disk space
du -sh /path              # Directory size
du -sh * | sort -h        # Sort by size
ncdu /path                # Interactive disk usage (if installed)

# Find files
find /path -name "*.log"  # Find by name
find /path -size +100M    # Find large files
find /path -mtime -1      # Modified in last day
find /path -type f -exec rm {} \;  # Find and execute

# File permissions
ls -la                    # List with permissions
chmod 755 file            # Change permissions
chown user:group file     # Change ownership
```

### Important Directories

```
/                         # Root
/etc                      # Configuration files
/var/log                  # Log files
/var/run                  # Runtime data (PIDs, sockets)
/tmp                      # Temporary files
/proc                     # Process information (virtual)
/sys                      # System information (virtual)
/home                     # User directories
/opt                      # Optional software
```

### File Descriptors

```bash
# View open files
lsof                      # All open files
lsof -p PID               # Files opened by process
lsof -i                   # Network connections

# Check limits
ulimit -n                 # Max open files (per process)
cat /proc/sys/fs/file-max # System-wide limit
cat /proc/PID/limits      # Limits for specific process
```

---

## Systemd

```bash
# Service management
systemctl status nginx        # Service status
systemctl start nginx         # Start service
systemctl stop nginx          # Stop service
systemctl restart nginx       # Restart service
systemctl reload nginx        # Reload config (if supported)
systemctl enable nginx        # Start on boot
systemctl disable nginx       # Don't start on boot

# View logs
journalctl -u nginx           # Logs for service
journalctl -u nginx -f        # Follow logs
journalctl -u nginx --since "1 hour ago"
journalctl -b                 # Logs since boot

# System targets
systemctl get-default         # Current default target
systemctl list-units          # All active units
systemctl list-unit-files     # All installed units
```

### Service Unit File Example

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Application
After=network.target

[Service]
Type=simple
User=appuser
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/bin/server
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

---

## Performance Debugging

### CPU Issues

```bash
# Find CPU-intensive processes
top -o %CPU               # Sort by CPU
pidstat 1                 # Per-process CPU stats
mpstat -P ALL 1           # Per-CPU stats

# Profile CPU
perf top                  # Real-time profiling
perf record -p PID        # Record process
perf report               # Analyze recording
```

### I/O Issues

```bash
# Disk I/O
iostat -x 1               # Extended disk stats
iotop                     # Top for I/O (if installed)

# Per-process I/O
pidstat -d 1              # I/O per process
cat /proc/PID/io          # Process I/O stats
```

### General Debugging

```bash
# System load
uptime                    # Load averages
cat /proc/loadavg         # Load and running processes

# What's the system doing
strace -p PID             # System calls
ltrace -p PID             # Library calls
```

---

## На интервью

### Типичные вопросы

1. **Process high CPU — как диагностировать?**
   ```bash
   top -o %CPU                  # Find process
   strace -p PID                # What's it doing
   perf top -p PID              # Where is CPU spent
   ```

2. **Out of memory — что произойдёт?**
   - OOM killer выбирает процесс по oom_score
   - Kills процесс, освобождает память
   - Check: `dmesg | grep -i "out of memory"`

3. **Порт занят — как узнать кем?**
   ```bash
   ss -tulnp | grep :PORT
   lsof -i :PORT
   netstat -tulnp | grep :PORT  # if no ss
   ```

4. **Файловая система full — что делать?**
   ```bash
   df -h                        # Which filesystem
   du -sh /* | sort -h          # Where is space used
   find / -type f -size +100M   # Large files
   lsof +D /path                # Deleted but open files
   ```

5. **Process zombie — что это?**
   - Завершённый процесс, ждущий wait() от родителя
   - Занимает только запись в процесс-таблице
   - Решение: kill parent или дождаться его завершения

---

## Quick Reference

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Debugging Cheatsheet                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  High CPU:      top, perf top, strace                               │
│  High Memory:   free -h, ps aux --sort=-%mem, pmap                  │
│  Disk Full:     df -h, du -sh, find -size, lsof +D                  │
│  I/O Issues:    iostat, iotop, pidstat -d                           │
│  Network:       ss -tuln, netstat, tcpdump, traceroute              │
│  Logs:          journalctl, /var/log/*                              │
│  Process:       ps, pstree, strace, lsof                            │
│                                                                     │
│  Remember: /proc and /sys contain live system information           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

[← Назад к списку тем](README.md)
