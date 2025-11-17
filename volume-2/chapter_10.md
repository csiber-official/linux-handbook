# Chapter 10: Performance Tuning and Optimization

## 10.1 Introduction to Performance Tuning and Optimization

Welcome to **Performance Tuning and Optimization**, where you transform good Linux systems into **blazing-fast, resource-efficient powerhouses** that deliver maximum throughput under pressure. In today’s world of microservices, AI workloads, and real-time applications, performance isn’t a luxury—it’s a **competitive advantage**.

Every millisecond saved in a web response, every CPU cycle reclaimed from a bloated process, and every I/O operation optimized directly impacts user experience, cloud costs, and business outcomes. This chapter arms you with **enterprise-grade tools**, **scientific methodologies**, and **RHEL 9-specific techniques** to identify bottlenecks, eliminate waste, and scale efficiently.

Think of performance tuning as **sculpting**: you start with a raw block of stone (your system) and chisel away inefficiencies—misconfigured kernels, runaway processes, disk thrashing—until only peak performance remains. Building on **Chapter 9: Backup and Recovery**, which ensured your data survives, this chapter ensures your systems **thrive** under load.

We’ll cover:

- **Kernel Tuning** with sysctl
- **Web Server Optimization** (Apache, Nginx)
- **Database Tuning** (MySQL/MariaDB)
- **Resource Control** with cgroups and systemd
- **Monitoring Bottlenecks** with `vmstat`, `iostat`, `sar`
- **Scaling Strategies** for high-traffic workloads

By the end, you’ll **tune Apache to handle 10,000 concurrent users** on modest hardware and prove it with real metrics.

## 10.2 Recapping Performance Fundamentals

Before optimizing, ensure mastery of:

- **Process Management** (Chapter 3): `ps`, `top`, `nice`, `kill`
- **File Systems and I/O** (Chapter 2): ext4, XFS, I/O schedulers
- **Networking** (Chapter 4): `tcpdump`, `ss`, firewall impact
- **Systemd Services** (Chapter 3): Resource limits, timers
- **Virtualization/Containers** (Chapter 6): Overhead awareness

Performance tuning is **data-driven**—never guess, always measure.

## 10.3 Understanding Linux Performance Metrics

### 10.3.1 The Four Core Resources

| Resource | Tool | Key Metric |
|--------|------|------------|
| **CPU** | `top`, `mpstat` | Load average, %user, %system |
| **Memory** | `free`, `vmstat` | Available RAM, swap usage |
| **Disk I/O** | `iostat`, `iotop` | await, %util, tps |
| **Network** | `sar -n DEV`, `nload` | rxkB/s, txkB/s, errors |

### 10.3.2 Load Average Explained

```bash
uptime
# 14:32:01 up 5 days,  load average: 1.20, 0.85, 0.60
```

- **1-minute, 5-minute, 15-minute averages**
- **Ideal**: < number of CPU cores
- **> cores**: System is overloaded

## 10.4 Kernel Tuning with sysctl

The Linux kernel is highly tunable via `/etc/sysctl.conf` and `sysctl -p`.

### 10.4.1 Critical Performance Parameters

Edit `/etc/sysctl.d/99-performance.conf`:

```conf
# Increase TCP buffer sizes
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216

# Enable TCP Fast Open
net.ipv4.tcp_fastopen = 3

# Reduce swappiness (prefer RAM over swap)
vm.swappiness = 10

# Increase file descriptors
fs.file-max = 2097152

# Optimize CPU scheduler
kernel.sched_autogroup_enabled = 0
kernel.sched_migration_cost_ns = 500000
```

Apply:

```bash
sudo sysctl -p /etc/sysctl.d/99-performance.conf
```

### 10.4.2 I/O Scheduler Tuning

Check current scheduler:

```bash
cat /sys/block/sda/queue/scheduler
# [mq-deadline] kyber bfq none
```

Set `none` for NVMe, `mq-deadline` for SSD/HDD:

```bash
echo none | sudo tee /sys/block/nvme0n1/queue/scheduler
```

Make persistent with `udev` rules or `GRUB_CMDLINE_LINUX` in `/etc/default/grub`.

## 10.5 Optimizing Web Servers (Apache)

Apache is flexible but can become bloated. Let’s harden and speed it up.

### 10.5.1 Install and Baseline

```bash
sudo dnf install httpd -y
sudo systemctl enable --now httpd
```

### 10.5.2 Disable Unneeded Modules

```bash
sudo httpd -M | grep -E "(status|info|autoindex)"
# Disable in /etc/httpd/conf.modules.d/00-base.conf
```

### 10.5.3 Tune MPM Prefork (`/etc/httpd/conf.modules.d/00-mpm.conf`)

```conf
<IfModule mpm_prefork_module>
    StartServers             5
    MinSpareServers          5
    MaxSpareServers         10
    ServerLimit            250
    MaxRequestWorkers      250
    MaxConnectionsPerChild 10000
</IfModule>
```

Use **MPM Event** for high concurrency:

```conf
LoadModule mpm_event_module modules/mod_mpm_event.so
```

### 10.5.4 Enable Caching and Compression

```apache
LoadModule cache_module modules/mod_cache.so
LoadModule cache_disk_module modules/mod_cache_disk.so
LoadModule deflate_module modules/mod_deflate.so

<IfModule mod_deflate.c>
    AddOutputFilterByType DEFLATE text/html text/css application/javascript
</IfModule>

CacheRoot "/var/cache/httpd"
CacheEnable disk /
```

### 10.5.5 Test with `ab` (Apache Benchmark)

```bash
sudo dnf install httpd-tools -y
ab -n 1000 -c 100 http://localhost/
```

**Goal**: > 1000 req/sec on 2 vCPU

## 10.6 Optimizing Databases (MySQL/MariaDB)

Poorly tuned databases are the #1 performance killer.

### 10.6.1 Install MariaDB

```bash
sudo dnf install mariadb-server -y
sudo systemctl enable --now mariadb
sudo mysql_secure_installation
```

### 10.6.2 Tune `my.cnf` (`/etc/my.cnf.d/performance.cnf`)

```ini
[mysqld]
innodb_buffer_pool_size = 2G        # 70% of RAM
innodb_log_file_size = 512M
innodb_flush_log_at_trx_commit = 2  # Faster, less durable
query_cache_type = 0                # Disabled in MySQL 8+
tmp_table_size = 64M
max_connections = 200
thread_cache_size = 50
table_open_cache = 4000
```

### 10.6.3 Use `mysqltuner` for Recommendations

```bash
curl -L https://raw.githubusercontent.com/major/MySQLTuner-perl/master/mysqltuner.pl | perl -
```

## 10.7 Resource Management with cgroups and systemd

Control runaway processes and ensure fairness.

### 10.7.1 Limit Apache Memory

Create override:

```bash
sudo systemctl edit httpd
```

```ini
[Service]
MemoryMax=1G
MemoryHigh=800M
CPUQuota=80%
```

### 10.7.2 Create a cgroup for Backup Jobs

```bash
sudo mkdir /sys/fs/cgroup/backup
echo "+memory +cpu" | sudo tee /sys/fs/cgroup/backup/cgroup.subtree_control
echo 500M | sudo tee /sys/fs/cgroup/backup/memory.max
```

Run backup inside:

```bash
systemd-run --scope -p MemoryMax=500M -p CPUQuota=50% /usr/local/bin/backup-rsync.sh
```

## 10.8 Monitoring Performance Bottlenecks

### 10.8.1 Real-Time Tools

| Tool | Use |
|------|-----|
| `vmstat 1` | CPU, memory, swap |
| `iostat -xz 1` | Disk I/O per device |
| `iotop` | Top-like for I/O |
| `htop` | Enhanced process view |
| `nethogs` | Bandwidth per process |

### 10.8.2 Historical Analysis with `sar`

Install:

```bash
sudo dnf install sysstat -y
sudo systemctl enable --now sysstat
```

View CPU:

```bash
sar -u
```

Generate HTML report:

```bash
sadc -F -O /var/log/sa/sa$(date +%d)
```

## 10.9 Best Practices for Scalability

| Practice | Why |
|--------|-----|
| **Use CDN** | Offload static assets |
| **Enable HTTP/2 or HTTP/3** | Multiplexing, header compression |
| **Tune KeepAlive** | Reuse TCP connections |
| **Use Reverse Proxy (Nginx)** | SSL termination, caching |
| **Scale Horizontally** | Add nodes + load balancer |
| **Monitor Everything** | Prometheus + Grafana |

## 10.10 Try This: Tune Apache for Better Performance Under Load

**Objective**: Optimize Apache to handle 10,000 requests with < 100ms latency.

**Prerequisites**: RHEL 9 VM with 4 vCPU, 8 GB RAM

### Step-by-Step

#### 1. Install Apache and Tools

```bash
sudo dnf install httpd httpd-tools -y
sudo systemctl enable --now httpd
```

#### 2. Create Test Page

```bash
echo "<h1>Welcome to High-Performance Linux</h1>" | sudo tee /var/www/html/index.html
```

#### 3. Baseline Performance

```bash
ab -n 1000 -c 50 http://localhost/
# Record: Requests/sec, Time per request
```

#### 4. Apply Optimizations

- Switch to **MPM Event**
- Set `KeepAlive On`, `KeepAliveTimeout 5`
- Enable `mod_deflate`, `mod_expires`
- Apply sysctl tuning (Section 10.4.1)

#### 5. Re-test

```bash
ab -n 10000 -c 200 http://localhost/
```

#### 6. Monitor During Test

In another terminal:

```bash
vmstat 1
iostat -xz 1
```

#### 7. Analyze

Compare:

- Before: ~500 req/sec
- After: **> 3000 req/sec**
- CPU < 80%, no swap, disk %util < 50%

**Expected Outcome**: 5x performance gain with zero hardware upgrade.

**Troubleshooting**:

- `journalctl -u httpd` for errors
- `ss -tlnp` to check connections
- `systemctl status httpd` for MPM

**Open-Source Contribution**:  
Submit a performance benchmark to Apache’s [wiki](https://httpd.apache.org/docs/current/platform/performance.html) using RHEL 9.

## 10.11 Conclusion

You’ve mastered **performance tuning on RHEL 9**:

- **Kernel tuning** with sysctl
- **Web server optimization** (Apache MPM, caching)
- **Database tuning** (InnoDB, query cache)
- **Resource control** with cgroups
- **Monitoring** with `vmstat`, `iostat`, `sar`

These skills turn **good systems into great ones**—faster, cheaper, more reliable.

This foundation powers:

- **Chapter 11: Troubleshooting** – Diagnose slow systems
- **Chapter 12: Real-World Projects** – Build a high-traffic app

> **Next: Chapter 11 – Troubleshooting and Debugging**  
> When things break, you’ll fix them—fast.

---

<!-- markdownlint-disable MD033 -->
<nav class="chapter-nav" aria-label="Chapter navigation">
  <div style="display:flex;justify-content:space-between;align-items:center">
    <a href="/volume-2/chapter_9.md" rel="prev">← Chapter 9: Backup and Recovery</a>
    <a href="/volume-2/chapter_11.md" rel="next">Chapter 11: Performance Tuning →</a>
  </div>
</nav>
---
