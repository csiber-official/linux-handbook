# Chapter 11: Troubleshooting and Debugging

## 11.1 Introduction to Troubleshooting and Debugging

Welcome to **Troubleshooting and Debugging**, the **art and science of turning chaos into clarity**. In the real world, systems don’t just *work*—they break, crash, hang, leak, and occasionally set themselves on fire (figuratively). The difference between a junior admin and a **Linux ninja** is not how many commands they know, but **how fast and methodically they can solve problems under pressure**.

This chapter is your **battlefield playbook**. You’ll learn to diagnose everything from boot failures to network blackouts, memory leaks to service deadlocks—using only the tools built into **RHEL 9**. No magic, no guesswork—just **systematic, repeatable workflows** that turn panic into progress.

Think of troubleshooting as **detective work**:  

- **Clues** are in logs, metrics, and system state  
- **Suspects** are processes, configs, hardware, and users  
- **Motive** is always *why did this break?*  
- **Alibi** is a working rollback or fix

Building on **Chapter 10: Performance Tuning**, where you made systems fast, this chapter ensures they **stay alive**. Together, they form the **complete admin mindset**: build it strong, keep it running, fix it fast.

We’ll cover:

- **Boot Forensics** with GRUB and `journalctl`
- **Log Analysis** with `journalctl`, `grep`, and `awk`
- **Network Diagnostics** with `tcpdump`, `ss`, `nmap`
- **Process Debugging** with `strace`, `lsof`, `gdb`
- **The 5-Step Troubleshooting Workflow**

By the end, you’ll **recover a dead server from a corrupted initramfs in under 10 minutes** and document it like a pro.

## 11.2 Recapping Core Diagnostic Skills

Before diving in, ensure you're fluent with:

- **Systemd and Services** (Chapter 3): `journalctl`, `systemctl`
- **File Systems** (Chapter 2): `fsck`, `xfs_repair`
- **Networking** (Chapter 4): `ip`, `ss`, `firewall-cmd`
- **Processes** (Chapter 3): `ps`, `top`, `kill`
- **SELinux** (Chapter 5): `sealert`, `semanage`

Troubleshooting is **80% observation, 20% action**.

## 11.3 The 5-Step Troubleshooting Workflow

| Step | Action | Tool |
|------|------|------|
| 1. **Reproduce** | Can you make it happen again? | Manual test, script |
| 2. **Isolate** | Is it network? Disk? App? Kernel? | `systemctl isolate`, `chroot` |
| 3. **Collect** | Gather logs, metrics, state | `journalctl`, `dmesg`, `sosreport` |
| 4. **Analyze** | Find patterns, errors, anomalies | `grep`, `awk`, `jq` |
| 5. **Resolve & Verify** | Fix, test, document | Patch, config, rollback |

> **Rule**: Never change more than one thing at a time.

## 11.4 Boot Failures: GRUB and Initramfs

### 11.4.1 When the System Won’t Boot

| Symptom | Likely Cause |
|--------|-------------|
| GRUB menu missing | Bootloader corruption |
| `initramfs` errors | Missing modules, wrong UUID |
| Kernel panic | Bad kernel, hardware failure |
| Black screen | Graphics driver |

### 11.4.2 Boot into Rescue Mode

1. Boot from RHEL 9 ISO → **Troubleshooting** → **Rescue a Red Hat system**
2. Or use `rd.break` in GRUB:

   ```bash
   # At GRUB, press 'e'
   linux ... rd.break
   Ctrl+X to boot
   ```

3. You land in `switch_root` emergency shell

### 11.4.3 Fix Common Boot Issues

#### Case 1: Wrong fstab UUID

```bash
# Remount rw
mount -o remount,rw /

# Check real UUID
ls -l /dev/disk/by-uuid/

# Fix /etc/fstab
vi /etc/fstab

# Regenerate initramfs
dracut -f /boot/initramfs-$(uname -r).img $(uname -r)

# Exit and reboot
exit
```

#### Case 2: Corrupted GRUB

```bash
# From rescue shell
mount /dev/sda1 /mnt
grub2-install /dev/sda
grub2-mkconfig -o /mnt/boot/grub2/grub.cfg
```

## 11.5 Advanced Log Analysis with `journalctl`

### 11.5.1 Powerful Queries

| Query | Use |
|------|-----|
| `journalctl -u httpd` | Service logs |
| `journalctl -b -1` | Previous boot |
| `journalctl --since "1 hour ago"` | Time range |
| `journalctl -p err` | Errors only |
| `journalctl /usr/bin/sshd` | Binary logs |

### 11.5.2 Real-Time Streaming

```bash
journalctl -f -u nginx
```

### 11.5.3 Export for Analysis

```bash
journalctl -b > boot.log
journalctl --since "2025-11-12 08:00" --until "09:00" > outage.log
```

### 11.5.4 Parse with `awk` and `grep`

```bash
# Find failed SSH logins
journalctl -u sshd | grep "Failed password" | awk '{print $11}' | sort | uniq -c | sort -nr

# Top error sources
journalctl -p err | awk '{print $5}' | sort | uniq -c | sort -nr | head
```

## 11.6 Network Troubleshooting

### 11.6.1 Layered Approach

| Layer | Tool |
|------|------|
| Physical | `ethtool`, cable check |
| Link | `ip link`, `nmcli` |
| IP | `ip addr`, `ping` |
| TCP | `ss`, `netstat` |
| DNS | `dig`, `nslookup` |
| Application | `curl`, `telnet` |

### 11.6.2 Capture with `tcpdump`

```bash
# Capture HTTP to file
sudo tcpdump -i eth0 port 80 -w http.pcap

# Filter SSH failures
sudo tcpdump -i any port 22 and 'tcp[tcpflags] & tcp-syn != 0'

# Read later
tcpdump -r http.pcap -nn | head
```

### 11.6.3 Analyze with `tshark` (Wireshark CLI)

```bash
sudo dnf install wireshark-cli -y
tshark -r http.pcap -Y "http.request" -T fields -e http.host -e http.request.uri
```

## 11.7 Process and Application Debugging

### 11.7.1 Find Open Files and Ports

```bash
# What is using port 80?
sudo ss -tlnp | grep :80

# What files does PID 1234 have open?
sudo lsof -p 1234

# Or by name
sudo lsof -i :22
```

### 11.7.2 Trace System Calls with `strace`

```bash
# Trace a command
strace -f -e trace=openat,read,write -o trace.log ping google.com

# Attach to running process
sudo strace -p 1234

# Common errors: ENOENT, EACCES, ETIMEDOUT
```

### 11.7.3 Core Dumps and `gdb`

Enable core dumps:

```bash
ulimit -c unlimited
echo "/tmp/core.%e.%p" > /proc/sys/kernel/core_pattern
```

Debug crash:

```bash
gcc -g -o crashme crashme.c
./crashme
gdb ./crashme /tmp/core.crashme.1234
(gdb) bt
(gdb) frame 2
(gdb) print variable
```

## 11.8 Generating Diagnostic Reports with `sosreport`

```bash
sudo dnf install sos -y
sudo sosreport
```

Generates `/tmp/sosreport-*.tar.xz` with:

- Logs
- Configs
- `dmesg`, `ps`, `ip`, `df`
- SELinux status

Perfect for Red Hat support or peer review.

## 11.9 Best Practices for Troubleshooting

| Practice | Why |
|--------|-----|
| **Document Everything** | `README.md` per incident |
| **Use Version Control** | `git` for configs |
| **Test in Staging** | Never fix in prod first |
| **Ask "What Changed?"** | 90% of issues |
| **Automate Checks** | Nagios, Prometheus alerts |
| **Know When to Escalate** | Don’t fight kernel bugs alone |

## 11.10 Try This: Recover a Server with a Corrupted Initramfs

**Objective**: Simulate and fix a boot failure due to missing initramfs.

**Prerequisites**: RHEL 9 VM

### Step-by-Step

#### 1. Simulate Failure

```bash
sudo rm /boot/initramfs-$(uname -r).img
sudo reboot
```

→ System hangs at `dracut` or kernel panic

#### 2. Boot into Rescue Mode

- Boot from RHEL 9 ISO → **Troubleshooting** → **Rescue**

#### 3. Mount the System

```bash
# Find root partition
lsblk

# Mount
mount /dev/sda2 /mnt
mount /dev/sda1 /mnt/boot
mount --bind /dev /mnt/dev
mount --bind /proc /mnt/proc
mount --bind /sys /mnt/sys
mount --bind /run /mnt/run
```

#### 4. Chroot and Fix

```bash
chroot /mnt

# Regenerate initramfs
dracut -f /boot/initramfs-$(uname -r).img $(uname -r)

# Verify
ls -l /boot/initramfs*

# Exit and reboot
exit
umount /mnt/{dev,proc,sys,run,boot,}
umount /mnt
reboot
```

#### 5. Verify Boot

```bash
uname -r
ls /boot/initramfs*
```

**Expected Outcome**: System boots normally in < 10 minutes.

**Troubleshooting**:

- Wrong partition? Use `blkid`
- SELinux relabel: `touch /.autorelabel` before reboot
- GRUB update: `grub2-mkconfig -o /boot/grub2/grub.cfg`

**Open-Source Contribution**:  
Improve `dracut` [wiki](https://github.com/dracutdevs/dracut/wiki) with a RHEL 9 rescue guide.

## 11.11 Conclusion

You’ve mastered **troubleshooting and debugging on RHEL 9**:

- **Boot forensics** with rescue mode and `dracut`
- **Log mastery** with `journalctl` and `awk`
- **Network sleuthing** with `tcpdump` and `ss`
- **Process tracing** with `strace` and `gdb`
- **The 5-Step Workflow** for any failure

These skills make you **the one they call at 3 AM**.

This foundation powers:

- **Chapter 12: Real-World Projects** – Build systems that survive chaos

> **Next: Chapter 12 – Bring It All Together: Real-World Projects**  
> Deploy a full-stack, HA, secure, backed-up, monitored app.

---

<!-- markdownlint-disable MD033 -->
<nav class="chapter-nav" aria-label="Chapter navigation">
  <div style="display:flex;justify-content:space-between;align-items:center">
    <a href="/volume-2/chapter_10.md" rel="prev">← Chapter 10: Performance Tuning</a>
    <a href="/volume-2/chapter_12.md" rel="next">Chapter 12: Real-World Projects →</a>
  </div>
</nav>
---
