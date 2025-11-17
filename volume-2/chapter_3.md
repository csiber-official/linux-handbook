# Chapter 3: Linux Server Administration

## 3.1 Introduction to Linux Server Administration

Welcome to the heart of enterprise Linux management—server administration! As a Linux system administrator, you’re the backbone of IT infrastructure, ensuring servers run smoothly, securely, and efficiently. Whether hosting a website, managing a database, or sharing files across a network, your skills keep critical systems operational. This chapter dives deep into Linux server administration using Red Hat Enterprise Linux 9 (RHEL 9), a leading enterprise distribution. We’ll cover setting up servers, managing services, monitoring performance, securing systems, and automating tasks, all while aligning with the Red Hat Certified System Administrator (RHCSA) objectives.

Server administration is both an art and a science. It requires understanding system components, anticipating failures, and automating repetitive tasks to maintain uptime and security. In enterprise environments, you might deploy a web server for a corporate website, monitor database performance for a financial application, or secure a file server for a remote team. This chapter builds on your foundational knowledge of Linux commands, file systems, and networking, guiding you through practical, real-world scenarios. By the end, you’ll be equipped to manage RHEL 9 servers with confidence, contribute to open-source tools, and prepare for RHCSA certification.

## 3.2 Recapping Server Administration Basics

Before diving into advanced concepts, let’s ensure you’re comfortable with core server administration skills, likely familiar from your prior Linux experience:

- **Command-Line Proficiency**: You should navigate the terminal with ease, using commands like `ls`, `cd`, `find`, and `grep` to locate files and filter output.
- **Service Management**: Understand basic service control with `systemctl` (e.g., `systemctl start httpd`) and process monitoring with `ps` or `top`.
- **File System Management**: Be familiar with mounting file systems (e.g., `mount /dev/sdb1 /mnt`) and managing permissions (`chmod`, `chown`).
- **Basic Networking**: Know how to check network status (`ip a`, `ss -tuln`) and configure simple firewall rules (e.g., `firewall-cmd --add-port=80/tcp`).
- **Automation Basics**: Have experience scheduling tasks with `cron` (e.g., `crontab -e` to add a backup job).

If these concepts feel shaky, practice in a RHEL 9 virtual machine (VM) using tools like VirtualBox or KVM. This chapter assumes you’re ready to build on these skills to manage enterprise-grade servers.

## 3.3 Setting Up a Linux Server

Setting up a server involves configuring hardware, installing software, and tailoring the system for specific roles like web, database, or file sharing. RHEL 9, with its stability and enterprise focus, is ideal for these tasks. Let’s explore setting up a LAMP (Linux, Apache, MySQL/MariaDB, PHP) stack, a common web server configuration, as it aligns with RHCSA objectives for managing services and software.

### 3.3.1 Preparing the System

Start with a minimal RHEL 9 installation, which conserves resources for server tasks. Ensure your system is registered with Red Hat Subscription Management to access repositories:

```bash
sudo subscription-manager register --username <your-username> --password <your-password>
sudo subscription-manager attach --auto
```

Verify with `subscription-manager status`. Update the system:

```bash
sudo dnf update -y
```

Install essential tools:

```bash
sudo dnf install vim nano wget curl -y
```

### 3.3.2 Installing the LAMP Stack

A LAMP stack powers dynamic websites. Here’s how to set it up on RHEL 9:

1. **Install Apache (httpd)**:

   ```bash
   sudo dnf install httpd -y
   ```

   Apache is the web server, handling HTTP requests. Enable and start it:

   ```bash
   sudo systemctl enable --now httpd
   ```

   Verify with `curl http://localhost`, which should return HTML content.

2. **Install MariaDB (MySQL-compatible)**:

   ```bash
   sudo dnf install mariadb-server -y
   ```

   Enable and start MariaDB:

   ```bash
   sudo systemctl enable --now mariadb
   ```

   Secure the installation:

   ```bash
   sudo mysql_secure_installation
   ```

   Follow prompts to set a root password, remove anonymous users, and disable remote root login.

3. **Install PHP**:

   ```bash
   sudo dnf install php php-mysqlnd -y
   ```

   Test PHP by creating `/var/www/html/info.php`:

   ```bash
   echo "<?php phpinfo(); ?>" | sudo tee /var/www/html/info.php
   ```

   Access `http://<server-ip>/info.php` in a browser to confirm PHP is working.

4. **Configure Permissions**:
   Ensure Apache can access web files:

   ```bash
   sudo chown -R apache:apache /var/www/html
   sudo chmod -R 755 /var/www/html
   ```

### 3.3.3 Troubleshooting Setup

- **Apache Fails to Start**: Check `systemctl status httpd` for errors (e.g., port 80 in use). Use `ss -tuln | grep 80` to identify conflicts and stop conflicting services.
- **MariaDB Access Denied**: Verify the root password with `mysql -u root -p`. Reset if needed via `sudo mysqladmin -u root password 'new-password'`.
- **PHP Not Rendering**: Ensure `php` module is loaded in `/etc/httpd/conf.d/php.conf`. Restart Apache: `sudo systemctl restart httpd`.

## 3.4 Managing Server Services and Daemons

Services (daemons) are background processes that provide functionality, like `httpd` for web serving or `sshd` for SSH. RHEL 9 uses `systemd` to manage services, a key RHCSA skill.

### 3.4.1 Understanding Systemd

Systemd organizes services into units (e.g., `.service`, `.timer`). Key commands:

- **List Services**: `systemctl list-units --type=service`
- **Check Status**: `systemctl status httpd`
- **Enable/Disable**: `systemctl enable httpd` (starts on boot), `systemctl disable httpd`
- **Start/Stop/Restart**: `systemctl start httpd`, `systemctl stop httpd`, `systemctl restart httpd`

### 3.4.2 Creating Custom Services

For custom tasks (e.g., a monitoring script), create a service file. Example: Create `/etc/systemd/system/monitor.service`:

```bash
[Unit]
Description=System Monitoring Script
After=network.target

[Service]
ExecStart=/usr/local/bin/monitor.sh
Restart=always

[Install]
WantedBy=multi-user.target
```

Write `/usr/local/bin/monitor.sh`:

```bash
#!/bin/bash
while true; do
  echo "Monitoring $(date)" >> /var/log/monitor.log
  sleep 60
done
```

Make executable: `sudo chmod +x /usr/local/bin/monitor.sh`. Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now monitor
```

### 3.4.3 Troubleshooting Services

- **Service Fails to Start**: Check logs with `journalctl -u httpd`. Look for errors like missing files or syntax issues in configuration.
- **Dependency Issues**: Use `systemctl list-dependencies httpd` to verify dependencies. Adjust `After=` in service files if needed.
- **Resource Limits**: If a service crashes, check `systemctl status <service>` for OOM (Out of Memory) errors. Adjust limits in the service file (e.g., `MemoryLimit=512M`).

## 3.5 Monitoring Server Performance and Logs

Monitoring ensures servers run efficiently and helps identify issues before they escalate. RHEL 9 provides robust tools for performance and log analysis, critical for RHCSA.

### 3.5.1 Performance Monitoring

Key tools:

- **top**: Displays real-time process details. Run `top`, press `f` to manage fields, and monitor CPU/memory usage.
- **htop**: A more user-friendly alternative (install: `sudo dnf install htop -y`). Use arrow keys to sort processes.
- **free**: Shows memory usage: `free -h`. Check “used” vs. “available” memory.
- **df**: Displays disk usage: `df -h`. Monitor free space on mounted file systems.

Example: To identify a memory-hogging process:

```bash
htop
```

Sort by `MEM%` and kill problematic processes with `F9` or `kill <pid>`.

### 3.5.2 Log Monitoring

Systemd’s `journalctl` is the primary log tool:

- **View All Logs**: `journalctl`
- **Filter by Service**: `journalctl -u httpd`
- **Follow Real-Time**: `journalctl -f`
- **Filter by Time**: `journalctl --since "2025-08-08 10:00" --until "2025-08-08 11:00"`

Example: Check Apache errors:

```bash
journalctl -u httpd --since "1 hour ago" | grep -i error
```

### 3.5.3 Troubleshooting Monitoring

- **High CPU Usage**: Use `top` or `htop` to identify the process. Check if it’s expected (e.g., database query) or a runaway script.
- **Disk Full**: If `df -h` shows 100% usage, find large files with `du -sh /var/* | sort -hr` and clean up (e.g., `rm /var/log/old.log`).
- **Log Errors**: If `journalctl` shows no logs, verify `systemd-journald` is running: `systemctl status systemd-journald`.

## 3.6 Basic Server Security

Security is paramount in server administration. RHEL 9’s tools, like `firewalld` and SSH hardening, align with RHCSA security objectives.

### 3.6.1 Configuring Firewalld

`firewalld` manages firewall rules dynamically. Key commands:

- **Check Status**: `firewall-cmd --state`
- **List Zones**: `firewall-cmd --get-zones`
- **Add Service**: `firewall-cmd --permanent --add-service=http`
- **Add Port**: `firewall-cmd --permanent --add-port=3306/tcp` (for MariaDB)
- **Reload Rules**: `firewall-cmd --reload`

Example: Allow HTTP and MariaDB traffic:

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-port=3306/tcp
sudo firewall-cmd --reload
```

### 3.6.2 Hardening SSH

Secure SSH (`sshd`) to prevent unauthorized access:

1. **Edit Configuration**: Modify `/etc/ssh/sshd_config`:

   ```bash
   PermitRootLogin no
   PasswordAuthentication no
   Port 2222
   ```

2. **Set Up Key-Based Authentication**:
   Generate a key pair on the client:

   ```bash
   ssh-keygen -t rsa -b 4096
   ```

   Copy to server:

   ```bash
   ssh-copy-id -i ~/.ssh/id_rsa.pub user@server-ip -p 2222
   ```

3. **Restart SSH**: `sudo systemctl restart sshd`

### 3.6.3 Automating Updates

Ensure security patches are applied automatically:

```bash
sudo dnf install dnf-automatic -y
```

Edit `/etc/dnf/automatic.conf`:

```bash
[commands]
upgrade_type = security
apply_updates = yes
```

Enable: `sudo systemctl enable --now dnf-automatic.timer`

### 3.6.4 Troubleshooting Security

- **Firewall Blocks Traffic**: Verify rules with `firewall-cmd --list-all`. Add missing services/ports.
- **SSH Connection Refused**: Check `sshd` status (`systemctl status sshd`) and ensure the port (e.g., 2222) is open in `firewalld`.
- **Update Failures**: If `dnf-automatic` fails, check logs in `/var/log/dnf.log` and ensure subscription is active.

## 3.7 Automating Routine Server Tasks

Automation saves time and reduces errors. RHEL 9 supports `logrotate` for log management and `cron` for scheduled tasks, both RHCSA requirements.

### 3.7.1 Configuring Log Rotation

`logrotate` manages log file growth. Edit `/etc/logrotate.d/httpd`:

```bash
/var/log/httpd/*.log {
    daily
    rotate 7
    compress
    missingok
}
```

Test rotation: `sudo logrotate -f /etc/logrotate.conf`

### 3.7.2 Scheduling Backups

Create a backup script `/usr/local/bin/backup.sh`:

```bash
#!/bin/bash
tar -czf /backup/web-$(date +%F).tar.gz /var/www/html
```

Make executable: `sudo chmod +x /usr/local/bin/backup.sh`. Schedule with `cron`:

```bash
sudo crontab -e
```

Add:

```bash
0 2 * * * /usr/local/bin/backup.sh
```

### 3.7.3 Troubleshooting Automation

- **Logrotate Fails**: Check `/var/log/logrotate.log` for errors (e.g., permission issues). Fix with `chown root:root /etc/logrotate.d/httpd`.
- **Cron Job Not Running**: Verify with `journalctl -u crond`. Ensure the script is executable and paths are absolute.

## 3.8 Try This: Set Up a LAMP Server and Configure Firewalld

Let’s apply your skills by setting up a LAMP server on RHEL 9, securing it with `firewalld`, and automating log rotation, simulating a real-world web server deployment.

**Objective**: Deploy a LAMP stack, secure it with firewall rules, and configure log rotation.

**Prerequisites**: A RHEL 9 VM with 2 GB RAM, 20 GB disk, and a registered subscription. Network access for testing.

**Steps**:

1. **Install LAMP Components**:

   ```bash
   sudo dnf install httpd mariadb-server php php-mysqlnd -y
   sudo systemctl enable --now httpd mariadb
   ```

2. **Secure MariaDB**:

   ```bash
   sudo mysql_secure_installation
   ```

   Set a root password and follow prompts to remove test databases.
3. **Test PHP**:
   Create `/var/www/html/info.php`:

   ```bash
   echo "<?php phpinfo(); ?>" | sudo tee /var/www/html/info.php
   sudo chown apache:apache /var/www/html/info.php
   sudo chmod 644 /var/www/html/info.php
   ```

4. **Configure Firewalld**:

   ```bash
   sudo firewall-cmd --permanent --add-service=http
   sudo firewall-cmd --permanent --add-port=3306/tcp
   sudo firewall-cmd --reload
   ```

   Test: `curl http://localhost/info.php` or access via a browser.
5. **Set Up Log Rotation**:
   Create `/etc/logrotate.d/httpd`:

   ```bash
   /var/log/httpd/*.log {
       daily
       rotate 7
       compress
       missingok
   }
   ```

   Test: `sudo logrotate -f /etc/logrotate.conf`
6. **Verify**:
   - Check services: `systemctl status httpd mariadb`
   - View logs: `journalctl -u httpd --since "10 minutes ago"`
   - Confirm firewall: `firewall-cmd --list-all`

**Expected Outcome**: The LAMP server hosts a PHP page, is secured with `firewalld`, and logs rotate daily. Access `http://<server-ip>/info.php` to see PHP info.

**Troubleshooting**:

- **Apache Fails**: Check `journalctl -u httpd` for errors (e.g., port conflicts). Resolve with `ss -tuln | grep 80`.
- **MariaDB Access Issues**: Reset password with `mysqladmin -u root password 'new-password'`.
- **Firewall Blocks**: Verify rules with `firewall-cmd --list-services`. Add missing services.
- **Logrotate Errors**: Check permissions (`ls -l /etc/logrotate.d/httpd`) and logs (`/var/log/logrotate.log`).

**Open-Source Contribution**: Explore the `httpd` repository (`github.com/apache/httpd`). Find a “good first issue” (e.g., documentation update) and comment on how you’d contribute.

## 3.9 Glossary

- **Daemon**: A background process providing system services (e.g., `httpd` for web serving).
- **LAMP Stack**: A software bundle (Linux, Apache, MariaDB/MySQL, PHP) for web applications.
- **Firewalld**: A dynamic firewall management tool using zones and services.
- **Systemd**: A system and service manager for Linux, controlling daemons and boot processes.

## 3.10 Conclusion

You’ve now mastered the essentials of Linux server administration on RHEL 9, from deploying a LAMP stack to managing services with `systemd`, monitoring with `htop` and `journalctl`, securing with `firewalld` and SSH hardening, and automating with `logrotate` and `cron`. The “Try This” exercise simulated a real-world web server setup, preparing you for enterprise tasks and RHCSA certification.

These skills lay the groundwork for later chapters, such as advanced networking (Chapter 4), security hardening (Chapter 5), and automation with Ansible (Chapter 8). Continue exploring open-source tools like `httpd` and `mariadb` to deepen your expertise and contribute to the Linux community. Next, Chapter 4 dives into advanced networking, where you’ll configure DNS servers and VPNs to enhance your server’s connectivity.

---

<!-- markdownlint-disable MD033 -->
<nav class="chapter-nav" aria-label="Chapter navigation">
  <div style="display:flex;justify-content:space-between;align-items:center">
    <a href="/volume-2/chapter_2.md" rel="prev">← Chapter 2: Advanced File Systems and Storage</a>
    <a href="/volume-2/chapter_4.md" rel="next">Chapter 4: Advanced Networking →</a>
  </div>
</nav>
<!-- markdownlint-enable MD033 -->

---
