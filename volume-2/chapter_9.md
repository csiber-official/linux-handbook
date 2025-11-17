# Chapter 9: Advanced Backup and Recovery

## 9.1 Introduction to Advanced Backup and Recovery

Welcome to **Advanced Backup and Recovery**, where you transform from a system administrator into a **data guardian**—the unsung hero who ensures that no matter what disaster strikes—hardware failure, ransomware, human error, or a rogue `rm -rf /`—your systems and data can rise from the ashes, fully intact and operational.

In enterprise environments, **data is the crown jewel**. A single hour of downtime can cost thousands, and a lost database can spell the end of trust. This chapter equips you with **enterprise-grade strategies**, **open-source tools**, and **battle-tested workflows** to protect, preserve, and restore critical systems on **Red Hat Enterprise Linux 9 (RHEL 9)**.

Think of backup not as insurance, but as **time travel for your infrastructure**. With the right tools and planning, you can roll back to any point in time—before the crash, before the breach, before the mistake. Building on **Chapter 7: High Availability**, which kept services online, this chapter ensures **data survives**. Together, they form the unbreakable backbone of resilient systems.

We’ll cover:

- **Full, Incremental, and Differential Backups**
- **rsync for Efficient File Syncing**
- **Bacula: The Enterprise Backup Framework**
- **Disaster Recovery Planning and Testing**
- **Restoring Systems After Total Failure**

By the end, you’ll deploy a **fully automated, encrypted, offsite backup pipeline** and confidently restore a production server from bare metal.

## 9.2 Recapping Backup Fundamentals

Before diving deep, ensure you're solid on:

- **File Permissions and Ownership** (Chapter 1)
- **LVM Snapshots** (Chapter 2)
- **Systemd Services and Timers** (Chapter 3)
- **SELinux Contexts** (Chapter 5)
- **Networking and SSH** (Chapter 4)

If needed, revisit these—backup systems must respect security and storage constraints.

## 9.3 Backup Strategies: Full, Incremental, Differential

### 9.3.1 Understanding Backup Types

| Type | Description | Pros | Cons |
|------|-------------|------|------|
| **Full** | Complete copy of all data | Simple, complete | Slow, storage-heavy |
| **Incremental** | Only changes since *last backup* (any type) | Fast, low storage | Complex restore |
| **Differential** | Changes since *last full* | Faster restore than incremental | Growing size over time |

**Best Practice**: **Full + Incremental** (e.g., weekly full, daily incremental) balances speed and reliability.

### 9.3.2 The Backup Equation

```plaintext
Recovery Point Objective (RPO) = Max acceptable data loss
Recovery Time Objective (RTO) = Max acceptable downtime
```

Example:  

- **RPO = 1 hour** → Hourly incremental backups  
- **RTO = 15 mins** → Automated, tested restore scripts

## 9.4 Using rsync for Offsite Backups

**rsync** is the Swiss Army knife of file synchronization—fast, efficient, and perfect for incremental backups over SSH.

### 9.4.1 Installing and Securing rsync

```bash
sudo dnf install rsync openssh-server -y
sudo systemctl enable --now sshd
```

Generate SSH key for passwordless backup:

```bash
ssh-keygen -t ed25519 -C "backup-key"
ssh-copy-id backupuser@backup-server
```

### 9.4.2 Creating an Incremental rsync Backup Script

Create `/usr/local/bin/backup-rsync.sh`:

```bash
#!/bin/bash
SRC="/var/www /etc /home"
DEST="backupuser@backup-server:/backups/$(hostname)"
LOG="/var/log/backup-rsync.log"
DATE=$(date +%Y-%m-%d_%H%M)

rsync -avz --delete \
      --link-dest=/backups/$(hostname)/latest \
      -e "ssh -i /root/.ssh/backup-key" \
      $SRC $DEST/incremental-$DATE/ \
      >> $LOG 2>&1

# Update 'latest' symlink
ssh backupuser@backup-server "rm -f $DEST/latest && ln -s incremental-$DATE $DEST/latest"
```

Make executable:

```bash
sudo chmod +x /usr/local/bin/backup-rsync.sh
```

### 9.4.3 Schedule with systemd Timer

**`/etc/systemd/system/backup-rsync.service`**

```ini
[Unit]
Description=rsync Incremental Backup
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup-rsync.sh
```

**`/etc/systemd/system/backup-rsync.timer`**

```ini
[Unit]
Description=Run rsync backup daily at 2 AM

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

Enable:

```bash
sudo systemctl enable --now backup-rsync.timer
```

## 9.5 Enterprise Backup with Bacula

**Bacula** is a powerful, open-source, client-server backup system used by enterprises worldwide. It supports full, incremental, differential, encryption, deduplication, and tape libraries.

### 9.5.1 Bacula Architecture

| Component | Role |
|---------|------|
| **Director** | Orchestrates jobs, schedules |
| **Storage Daemon (SD)** | Manages storage (disk, tape) |
| **File Daemon (FD)** | Runs on clients, sends data |
| **Catalog** | MySQL/PostgreSQL database of backups |
| **Console** | `bconsole` for management |

### 9.5.2 Installing Bacula on RHEL 9

```bash
sudo dnf install bacula-director bacula-storage bacula-client bacula-console mariadb-server -y
```

Start MariaDB:

```bash
sudo systemctl enable --now mariadb
mysql_secure_installation
```

Create Bacula database:

```sql
mysql -u root -p
CREATE DATABASE bacula;
GRANT ALL ON bacula.* TO 'bacula'@'localhost' IDENTIFIED BY 'securepassword';
FLUSH PRIVILEGES;
```

### 9.5.3 Configuring Bacula Director (`/etc/bacula/bacula-dir.conf`)

```conf
Director {
  Name = bacula-dir
  DIRport = 9101
  Password = "directorpass"
}

Job {
  Name = "BackupClient"
  Client = client1-fd
  JobDefs = "DefaultJob"
  FileSet = "Full Set"
  Schedule = "WeeklyCycle"
  Storage = File
  Messages = Standard
  Pool = FullPool
  Full Backup Pool = FullPool
  Incremental Backup Pool = IncPool
  Differential Backup Pool = DiffPool
}

FileSet {
  Name = "Full Set"
  Include {
    Options { compression = GZIP }
    File = /var/www
    File = /etc
    File = /home
  }
  Exclude { File = *.tmp }
}

Client {
  Name = client1-fd
  Address = 192.168.122.101
  FDPort = 9102
  Password = "clientpass"
}

Storage {
  Name = File
  Address = backup-server
  SDPort = 9103
  Password = "storagepass"
  Device = FileStorage
  Media Type = File
}

Pool {
  Name = FullPool
  Pool Type = Backup
  Recycle = yes
  AutoPrune = yes
  Volume Retention = 6 months
  Maximum Volumes = 100
}
```

### 9.5.4 Configure Storage Daemon (`/etc/bacula/bacula-sd.conf`)

```conf
Storage {
  Name = backup-server
  SDPort = 9103
  Password = "storagepass"
}

Device {
  Name = FileStorage
  Media Type = File
  Archive Device = /bacula/backups
  LabelMedia = yes
  Random Access = yes
  AutomaticMount = yes
  RemovableMedia = no
  AlwaysOpen = no
}
```

### 9.5.5 Configure Client (`/etc/bacula/bacula-fd.conf`)

```conf
FileDaemon {
  Name = client1-fd
  FDport = 9102
  Password = "clientpass"
}

Director {
  Name = bacula-dir
  Password = "directorpass"
}
```

### 9.5.6 Start and Test

```bash
sudo systemctl enable --now bacula-dir bacula-sd bacula-fd
bconsole
*run
Job: BackupClient
*yes
*status director
*list jobs
```

## 9.6 Disaster Recovery Planning

### 9.6.1 Creating a DR Plan

1. **Inventory**: List all critical systems, data, apps
2. **RTO/RPO**: Define recovery goals
3. **Backup Strategy**: Full + Incremental + Offsite
4. **Restore Procedures**: Document step-by-step
5. **Test Quarterly**: Simulate failures

### 9.6.2 Bare-Metal Restore with Relax-and-Recover (ReaR)

**ReaR** creates bootable rescue images for full system recovery.

Install:

```bash
sudo dnf install rear -y
```

Configure `/etc/rear/site.conf`:

```conf
OUTPUT=ISO
BACKUP=RSYNC
BACKUP_URL="nfs://backup-server:/backups/rear"
ISO_PREFIX="rear-$(hostname)"
```

Create rescue image:

```bash
sudo rear mkbackup
```

Test in VM: Boot from ISO → `rear recover`

## 9.7 Best Practices for Backup Security

| Practice | Command |
|--------|--------|
| **Encrypt Backups** | `gpg --encrypt --recipient key backup.tar` |
| **Offsite + Offline** | 3-2-1 Rule: 3 copies, 2 media, 1 offsite |
| **Immutable Storage** | Use S3 Object Lock or WORM tapes |
| **Test Restores** | Monthly full restore in sandbox |
| **Monitor Backups** | `journalctl -u bacula-dir`, Nagios checks |

## 9.8 Try This: Set Up Incremental Backup with rsync and Test a Restore

**Objective**: Automate daily incremental backups and simulate data loss.

**Prerequisites**: Two RHEL 9 VMs:

- `webserver` (source)
- `backupserver` (destination with NFS or SSH)

### Step-by-Step

#### 1. Set Up Backup Server

```bash
sudo dnf install rsync openssh-server -y
sudo useradd -m backupuser
sudo mkdir /backups
sudo chown backupuser: /backups
sudo systemctl enable --now sshd
```

#### 2. On Web Server: Generate SSH Key

```bash
sudo -u backupuser ssh-keygen -t ed25519
sudo -u backupuser ssh-copy-id backupuser@backupserver
```

#### 3. Create Test Data

```bash
sudo mkdir -p /var/www/html
echo "<h1>Critical App v1.0</h1>" | sudo tee /var/www/html/index.html
```

#### 4. Create Backup Script

Use script from **Section 9.4.2**, adjust `SRC` and `DEST`.

#### 5. Run First Backup

```bash
sudo /usr/local/bin/backup-rsync.sh
```

#### 6. Modify Data and Run Again

```bash
echo "<h1>Critical App v1.1</h1>" | sudo tee /var/www/html/index.html
sudo /usr/local/bin/backup-rsync.sh
```

#### 7. Simulate Disaster

```bash
sudo rm -rf /var/www/html
```

#### 8. Restore from Latest

```bash
LATEST=$(ssh backupuser@backupserver "readlink /backups/$(hostname)/latest")
rsync -avz backupuser@backupserver:/backups/$(hostname)/$LATEST/var/www/html/ /var/www/html/
```

#### 9. Verify

```bash
curl localhost
# Should show v1.1
```

**Expected Outcome**: Data restored from latest incremental backup.

**Troubleshooting**:

- Check SSH keys: `ssh -i /home/backupuser/.ssh/id_ed25519 backupuser@backupserver`
- Permissions: `chown -R backupuser: /backups`
- SELinux: `restorecon -R /backups`

**Open-Source Contribution**:  
Improve ReaR’s [documentation](https://github.com/rear/rear) with a RHEL 9 + Podman guide.

## 9.9 Conclusion

You’ve mastered **enterprise backup and recovery** on RHEL 9:

- **rsync** for fast, secure incremental sync
- **Bacula** for centralized, scalable backup
- **ReaR** for bare-metal disaster recovery
- **3-2-1 Rule**, encryption, and testing

These skills ensure **data immortality**—no failure is final.

This foundation powers:

- **Chapter 10: Performance Tuning** – Monitor backup impact
- **Chapter 11: Troubleshooting** – Recover from corruption
- **Chapter 12: Real-World Projects** – Build a ransomware-resistant stack

Keep testing restores. **The backup that isn’t tested doesn’t exist.**

> **Next: Chapter 10 – Performance Tuning and Optimization**  
> Make your recovered systems *fast*.

---

<!-- markdownlint-disable MD033 -->
<nav class="chapter-nav" aria-label="Chapter navigation">
  <div style="display:flex;justify-content:space-between;align-items:center">
    <a href="/volume-2/chapter_8.md" rel="prev">← Chapter 8: Backup and Recovery</a>
    <a href="/volume-2/chapter_10.md" rel="next">Chapter 10: Performance Tuning →</a>
  </div>
</nav>
---
