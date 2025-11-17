# Chapter 2: Advanced File Systems and Storage

## 2.1 Introduction to Advanced Storage

Welcome to the world of advanced Linux storage! As a Linux administrator, you’ll manage complex storage solutions that ensure data availability, scalability, and resilience in enterprise environments. Whether you're setting up a high-performance web server, a redundant file share, or a cloud-native database, understanding advanced file systems and storage technologies is critical.

This chapter builds on your foundational knowledge of Linux file systems (e.g., `ext4`, permissions) and introduces enterprise-grade tools like:

* Logical Volume Manager (**LVM**)
* Redundant Array of Independent Disks (**RAID**)
* Network file systems (**NFS**, **Samba**)

By the end, you’ll be equipped to configure, optimize, and troubleshoot storage in real-world environments.

Storage management is central to system administration. In enterprise settings, you may:

* Use LVM to dynamically resize volumes for a growing database.
* Configure RAID to ensure redundancy for critical applications.
* Implement NFS for cross-server file sharing.
* Monitor disks using `smartctl` to prevent failures.

This chapter walks through these tools with practical examples and hands-on exercises.

---

## 2.2 Recapping File System Basics

Before moving into advanced storage, review these core concepts:

* **File System Hierarchy:**
  The Linux directory structure (`/`, `/etc`, `/var`, `/home`) organizes system and user data.

* **File System Types:**
  Example: Format a partition and mount it:

  ```bash
  mkfs.ext4 /dev/sdb1
  mount /dev/sdb1 /mnt
  ```

* **Permissions and Ownership:**

  ```bash
  chmod 640 file.txt
  chown alice:devteam file.txt
  ```

* **Disk Management Tools:**

  * `fdisk /dev/sdb` (partitioning)
  * `df -h` (disk usage)

If these feel rusty, practice in a virtual machine before continuing.

---

## 2.3 Logical Volume Manager (LVM)

LVM provides flexible storage management, allowing you to create, resize, and snapshot volumes without downtime. It abstracts physical storage into logical layers.

### 2.3.1 Understanding LVM Components

LVM operates across three layers:

* **Physical Volumes (PVs):**
  Physical partitions or disks (e.g., `/dev/sdb1`), initialized using `pvcreate`.

* **Volume Groups (VGs):**
  Pools of PVs created with `vgcreate`.

* **Logical Volumes (LVs):**
  Virtual partitions created with `lvcreate`.

Example: Two 100 GB disks combined into one VG → a 200 GB storage pool.

### 2.3.2 Setting Up LVM

Assume two disks: `/dev/sdb`, `/dev/sdc`.

1. **Initialize Physical Volumes**

   ```bash
   sudo pvcreate /dev/sdb /dev/sdc
   ```

   Verify:

   ```bash
   pvs
   ```

2. **Create Volume Group**

   ```bash
   sudo vgcreate my_vg /dev/sdb /dev/sdc
   ```

   Check:

   ```bash
   vgs
   ```

3. **Create Logical Volume**

   ```bash
   sudo lvcreate -L 50G -n my_lv my_vg
   ```

   Validate:

   ```bash
   lvs
   ```

4. **Format and Mount**

   ```bash
   sudo mkfs.ext4 /dev/my_vg/my_lv
   sudo mkdir /mnt/my_data
   sudo mount /dev/my_vg/my_lv /mnt/my_data
   ```

   Add to `/etc/fstab`:

   ``` plaintext
   /dev/my_vg/my_lv /mnt/my_data ext4 defaults 0 0
   ```

### 2.3.3 Resizing and Snapshots

**Resize to 75 GB:**

```bash
sudo lvresize -L 75G /dev/my_vg/my_lv
sudo resize2fs /dev/my_vg/my_lv
```

**Create snapshot:**

```bash
sudo lvcreate -L 10G -s -n my_lv_snap /dev/my_vg/my_lv
```

**Remove snapshot:**

```bash
sudo lvremove /dev/my_vg/my_lv_snap
```

### 2.3.4 Troubleshooting LVM

* **pvcreate fails ("Device not found")**

  * Check disk visibility:

    ```bash
    lsblk
    ```

  * Ensure it's not mounted.

* **lvresize fails ("Insufficient space")**

  * Check VG free space:

    ```bash
    vgs
    ```

  * Add a new disk:

    ```bash
    vgextend my_vg /dev/sdd
    ```

---

## 2.4 Redundant Array of Independent Disks (RAID)

RAID enhances redundancy and/or performance.

Common RAID levels:

* **RAID 0:** Striping, high performance, zero redundancy.
* **RAID 1:** Mirroring, high redundancy.
* **RAID 5:** Striping + parity (3+ disks).
* **RAID 6:** Striping + dual parity (4+ disks).

### 2.4.1 Setting Up Software RAID

Using `mdadm`, create RAID 1 on `/dev/sdb` and `/dev/sdc`:

1. **Create Array**

   ```bash
   sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb /dev/sdc
   ```

   View progress:

   ```bash
   cat /proc/mdstat
   ```

2. **Format and Mount**

   ```bash
   sudo mkfs.ext4 /dev/md0
   sudo mkdir /mnt/raid
   sudo mount /dev/md0 /mnt/raid
   ```

   Add to `/etc/fstab`:

   ```text
   /dev/md0 /mnt/raid ext4 defaults 0 0
   ```

3. **Save Configuration**

```bash
   sudo mdadm --detail --scan >> /etc/mdadm/mdadm.conf
   ```

```bash
   sudo mdadm --detail --scan >> /etc/mdadm/mdadm.conf
   ```

### 2.4.2 Managing RAID (management)

```bash
   sudo mdadm --detail --scan >> /etc/mdadm/mdadm.conf
   ```

### 2.4.2 Managing RAID

```bash
   sudo mdadm --detail --scan >> /etc/mdadm/mdadm.conf
   ```

* **Check status**

  ```bash
  mdadm --detail /dev/md0
  ```

* **Replace failed disk**

  ```bash
  mdadm /dev/md0 -f /dev/sdb
  mdadm /dev/md0 -r /dev/sdb
  mdadm /dev/md0 -a /dev/sdd
  ```

---

## 2.5 Network File Systems

Network file systems allow servers to share data. This section covers **NFS** (Linux-Linux) and **Samba** (Linux-Windows).

### 2.5.1 Configuring NFS

**On Server:**

1. Install:

   ```bash
   sudo apt install nfs-kernel-server
   ```

2. Edit `/etc/exports`:

   ```plaintext
   /srv/nfs 192.168.1.0/24(rw,sync,no_subtree_check)
   ```

3. Export & start:

   ```bash
   sudo exportfs -a
   sudo systemctl enable --now nfs-kernel-server
   ```

**On Client:**

1. Install:

   ```bash
   sudo apt install nfs-common
   ```

2. Mount:

   ```bash
   sudo mkdir /mnt/nfs
   sudo mount 192.168.1.10:/srv/nfs /mnt/nfs
   ```

Add to `/etc/fstab`:

```plaintext
192.168.1.10:/srv/nfs /mnt/nfs nfs defaults 0 0
```

### 2.5.2 Configuring Samba

**On Server:**

1. Install:

   ```bash
   sudo apt install samba
   ```

2. Edit `/etc/samba/smb.conf`:

   ```plaintext
   [shared]
   path = /srv/samba
   writable = yes
   browsable = yes
   guest ok = yes
   ```

3. Start:

   ```bash
   sudo systemctl enable --now smbd
   ```

**Windows Client:**
Access using:

```plaintext
\\192.168.1.10\shared
```

### 2.5.3 Troubleshooting Network File Systems

* **NFS: "permission denied"**

  * Check `/etc/exports` IP range.
  * Reload:

    ```bash
    exportfs -ra
    ```

  * Allow port 2049 in firewall.

* **Samba: Windows cannot connect**

  * Verify service:

    ```bash
    systemctl status smbd
    ```

  * Add firewall rules:

    ```bash
    ufw allow Samba
    ```

---

## 2.6 Monitoring Disk Health

Use `smartctl` from **smartmontools**.

1. Install:

   ```bash
   sudo apt install smartmontools
   ```

2. Check health:

   ```bash
   sudo smartctl -a /dev/sdb
   ```

3. Enable email alerts in `/etc/smartd.conf`:

   ```plaintext
   /dev/sdb -m admin@example.com
   ```

   Start monitoring:

   ```bash
   sudo systemctl enable --now smartd
   ```

If SMART reports warnings (e.g., `Reallocated_Sector_Ct`), back up and replace the disk.

---

## 2.7 Best Practices for Storage Management

* Use **LVM** for flexibility.
* Use **RAID 1 or 5** for redundancy.
* Restrict NFS exports; use Samba authentication for sensitive data.
* Run `smartctl` regularly via cron.
* Always back up configurations and data.
* Explore and contribute to open-source storage tools.

---

## 2.8 Try This: Set Up an LVM Volume and NFS Share

**Objective:** Create an LVM volume, format it, and share it via NFS.

**Prerequisites:**
VM with two disks: `/dev/sdb`, `/dev/sdc`; client VM on same network.

### Steps

1. **Install LVM & NFS**

   ```bash
   sudo apt update
   sudo apt install lvm2 nfs-kernel-server
   ```

2. **Set Up LVM**

   ```bash
   sudo pvcreate /dev/sdb /dev/sdc
   sudo vgcreate storage_vg /dev/sdb /dev/sdc
   sudo lvcreate -L 20G -n data_lv storage_vg
   sudo mkfs.ext4 /dev/storage_vg/data_lv
   sudo mkdir /srv/nfs_data
   sudo mount /dev/storage_vg/data_lv /srv/nfs_data
   ```

3. **Configure NFS**
   Edit `/etc/exports`:

   ```plaintext
   /srv/nfs_data 192.168.1.0/24(rw,sync,no_subtree_check)
   ```

   Run:

   ```bash
   sudo exportfs -a
   sudo systemctl enable --now nfs-kernel-server
   ```

4. **Client Access**

   ```bash
   sudo apt install nfs-common
   sudo mount 192.168.1.10:/srv/nfs_data /mnt
   echo "Test" > /mnt/test.txt
   ```

5. **Check Disk Health**

   ```bash
   sudo smartctl -a /dev/sdb
   ```

**Expected Outcome:**
NFS share is mounted and writable on client; disk health validated.

---

## 2.9 Conclusion

You now understand advanced Linux storage concepts including LVM, RAID, NFS, Samba, and disk monitoring. These skills prepare you for enterprise storage management and future chapters on server administration, security, and backups.

---

<!-- markdownlint-disable MD033 -->
<nav class="chapter-nav" aria-label="Chapter navigation">
  <div style="display:flex;justify-content:space-between;align-items:center">
    <a href="chapter_1.md" rel="prev">← Chapter 1: Introduction to Advanced Linux Concepts</a>
    <a href="chapter_3.md" rel="next">Chapter 3: Linux Server Administration →</a>
  </div>
</nav>
<!-- markdownlint-enable MD033 -->

---
