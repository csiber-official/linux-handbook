# Chapter 5: System Security and Hardening

## 5.1 Introduction to System Security and Hardening

Welcome to the fortress of Linux security! As a system administrator, securing your servers is like guarding a digital castle, protecting critical data and services from intruders. In enterprise environments, where a single breach can cost millions or compromise sensitive information, system security and hardening are non-negotiable skills. This chapter dives deep into securing Red Hat Enterprise Linux 9 (RHEL 9), guiding you from foundational security practices to advanced techniques like SELinux, AppArmor, PAM, intrusion detection, and system auditing. By blending theory with hands-on examples, we’ll equip you to fortify your systems, meet compliance requirements, and prepare for the Red Hat Certified System Administrator (RHCSA) exam.

Security is about layers—each tool and technique adds a barrier to protect your system. Imagine locking your doors, installing alarms, hiring guards, and checking surveillance footage. In Linux, this translates to configuring mandatory access controls, restricting user privileges, detecting unauthorized access, and auditing system integrity. Whether you’re securing a web server hosting an e-commerce site or a database storing customer data, these skills ensure your systems remain robust and trustworthy. Let’s embark on this journey to make your RHEL 9 servers impenetrable, while embracing the open-source community’s collaborative spirit.

## 5.2 Recapping Security Basics

Before we dive into advanced hardening, let’s ensure you’re comfortable with core security concepts from your prior Linux experience:

- **User and Group Management**: You should know how to create users (`useradd`), set passwords (`passwd`), and manage groups (`usermod -aG`). For example, `sudo useradd -m alice` creates a user with a home directory.
- **Permissions**: Understand file permissions (`chmod 640 file.txt` for owner-read/write, group-read) and ownership (`chown alice:devteam file.txt`).
- **Firewall Basics**: Familiarity with `firewalld` (e.g., `firewall-cmd --add-service=http`) to control network access.
- **SSH Security**: Basic SSH configuration (e.g., disabling root login in `/etc/ssh/sshd_config`) and key-based authentication.
- **System Updates**: Keeping software current with `dnf update`.

If these feel rusty, practice in a RHEL 9 virtual machine (VM) using VirtualBox or KVM. This chapter assumes you’re ready to build on these foundations to implement enterprise-grade security.

## 5.3 SELinux: Mandatory Access Control

Security-Enhanced Linux (SELinux) is a mandatory access control (MAC) system, enforcing strict policies to limit what processes and users can do, even with elevated privileges. Unlike discretionary access controls (DAC) like file permissions, SELinux uses labels and policies to confine actions, making it a cornerstone of RHEL 9 security and a key RHCSA objective.

### 5.3.1 Understanding SELinux Concepts

SELinux operates in three modes:

- **Enforcing**: Policies are strictly applied; violations are blocked and logged.
- **Permissive**: Violations are logged but not blocked, useful for testing.
- **Disabled**: SELinux is turned off (not recommended in production).

Each file, process, and user has an SELinux context (e.g., `user:role:type:level`), defining allowed actions. For example, Apache (`httpd`) can only access files labeled `httpd_sys_content_t`. Policies are managed via booleans, ports, and file contexts.

### 5.3.2 Configuring SELinux

Check SELinux status:

```bash
sestatus
```

Output shows the mode (e.g., enforcing) and policy (e.g., targeted). To set enforcing mode:

```bash
sudo setenforce 1
```

Edit `/etc/selinux/config` for persistence:

```bash
SELINUX=enforcing
SELINUXTYPE=targeted
```

Restart to apply: `sudo reboot`.

To allow Apache to serve custom content from `/var/www/custom`:

1. Create directory: `sudo mkdir /var/www/custom`.
2. Set context: `sudo semanage fcontext -a -t httpd_sys_content_t "/var/www/custom(/.*)?"`.
3. Apply context: `sudo restorecon -R /var/www/custom`.
4. Verify: `ls -Z /var/www/custom` (shows `httpd_sys_content_t`).

Enable HTTP network connections:

```bash
sudo setsebool -P httpd_can_network_connect 1
```

List booleans: `getsebool -a | grep httpd`.

### 5.3.3 Troubleshooting SELinux

- **Denial Errors**: Check logs with `sudo ausearch -m avc -ts recent`. Example denial: `httpd` can’t access `/var/www/custom`.
- **Fix Denials**: Generate a custom policy with `audit2allow`:

  ```bash
  sudo ausearch -m avc -ts recent | audit2allow -M myhttpd
  sudo semodule -i myhttpd.pp
  ```

- **Context Issues**: Use `chcon` for temporary fixes (e.g., `chcon -t httpd_sys_content_t /var/www/custom/file.html`), but prefer `semanage` for permanence.

## 5.4 AppArmor: Application Confinement

AppArmor is another MAC system, focusing on confining applications via profiles. While less common in RHEL 9 (which favors SELinux), it’s used in some distributions and worth understanding for completeness.

### 5.4.1 Understanding AppArmor

AppArmor restricts programs (e.g., Apache, MySQL) to specific files, directories, and capabilities. Profiles operate in **enforce** (block violations) or **complain** (log violations) modes. In RHEL 9, AppArmor is not installed by default but can be added.

### 5.4.2 Installing and Configuring AppArmor

Install AppArmor:

```bash
sudo dnf install apparmor-profiles apparmor-utils -y
```

Start the service:

```bash
sudo systemctl enable --now apparmor
```

Create a profile for a custom script `/usr/local/bin/myscript.sh`:

1. Generate profile: `sudo aa-genprof /usr/local/bin/myscript.sh`.
2. Run the script to log activity, then follow prompts to allow/deny actions.
3. Set to enforce: `sudo aa-enforce /usr/local/bin/myscript.sh`.
4. Verify: `sudo aa-status`.

### 5.4.3 Troubleshooting AppArmor

- **Denials**: Check logs in `/var/log/audit/audit.log` or `/var/log/messages` for AppArmor messages.
- **Fix Profiles**: Edit `/etc/apparmor.d/usr.local.bin.myscript.sh` to allow specific paths (e.g., `/var/log/mylog rw`). Reload: `sudo apparmor_parser -r /etc/apparmor.d/usr.local.bin.myscript.sh`.

## 5.5 Advanced User Security

Controlling user access is critical for security. Pluggable Authentication Modules (PAM) and `sudoers` provide fine-grained control, aligning with RHCSA objectives.

### 5.5.1 Configuring PAM

PAM manages authentication tasks (e.g., login, sudo). Configuration files in `/etc/pam.d/` or `/etc/pam.conf` define rules. To enforce strong passwords:

1. Edit `/etc/security/pwquality.conf`:

   ```bash
   minlen = 12
   dcredit = -1  # Require at least one digit
   ucredit = -1  # Require at least one uppercase
   ```

2. Update `/etc/pam.d/passwd` to include `pam_pwquality`:

   ```bash
   password requisite pam_pwquality.so retry=3
   ```

Test by changing a password: `sudo passwd testuser` (must meet criteria).

### 5.5.2 Configuring sudoers

The `/etc/sudoers` file controls sudo access. Edit with `visudo` for syntax safety:

```bash
sudo visudo
```

Add a user to run specific commands:

```bash
alice ALL=(ALL) /usr/bin/systemctl restart httpd, /usr/bin/dnf update
```

Create a group alias for admins:

```bash
%sysadmins ALL=(ALL) ALL
```

Test: `sudo -u alice systemctl restart httpd`.

### 5.5.3 Troubleshooting PAM and sudoers

- **PAM Errors**: Check `/var/log/secure` for authentication failures. Verify module paths in `/etc/pam.d/`.
- **sudoers Syntax Errors**: If `visudo` fails, recover by editing `/etc/sudoers.tmp` as root and re-run `visudo`.

## 5.6 Intrusion Detection with fail2ban

Intrusion detection and prevention systems (IDS/IPS) monitor for suspicious activity. `fail2ban` is a host-based IPS that protects against brute-force attacks, a key RHCSA skill.

### 5.6.1 Installing and Configuring fail2ban

Install:

```bash
sudo dnf install fail2ban -y
```

Copy default config:

```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

Edit `/etc/fail2ban/jail.local`:

```bash
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 5

[sshd]
enabled = true
port = 22
logpath = /var/log/secure
backend = auto
```

Start service:

```bash
sudo systemctl enable --now fail2ban
```

Check status: `fail2ban-client status sshd`.

### 5.6.2 Monitoring and Managing Bans

View banned IPs:

```bash
fail2ban-client status sshd
```

Unban an IP:

```bash
fail2ban-client set sshd unbanip 192.168.1.100
```

Check logs: `sudo less /var/log/fail2ban.log`.

### 5.6.3 Troubleshooting fail2ban

- **No Bans Applied**: Verify log path (`/var/log/secure`) and ensure `sshd` logs failed attempts (`journalctl -u sshd`).
- **Firewall Issues**: Ensure `firewalld` allows fail2ban to modify rules (`firewall-cmd --add-service=ssh`).

## 5.7 Auditing System Integrity with AIDE

Advanced Intrusion Detection Environment (AIDE) monitors file system changes to detect unauthorized modifications, crucial for compliance and security auditing.

### 5.7.1 Installing and Configuring AIDE

Install:

```bash
sudo dnf install aide -y
```

Initialize database:

```bash
sudo aide --init
sudo mv /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz
```

Edit `/etc/aide.conf` to monitor critical files:

```bash
/etc PASSWD+SHADOW
/var/www/html CONTENT+SHA256
```

Run a check:

```bash
sudo aide --check
```

### 5.7.2 Automating AIDE Checks

Schedule daily checks with `cron`:

```bash
sudo crontab -e
```

Add:

```bash
0 3 * * * /usr/bin/aide --check > /var/log/aide-report-$(date +%F).txt
```

### 5.7.3 Troubleshooting AIDE

- **Database Errors**: Reinitialize if corrupted (`aide --init`).
- **False Positives**: Update database after legitimate changes (`aide --update; mv /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz`).

## 5.8 Best Practices for System Hardening

- **Enable SELinux in Enforcing Mode**: Always use enforcing mode in production for robust MAC.
- **Minimize Attack Surface**: Disable unused services (`systemctl disable <service>`) and close unnecessary ports.
- **Regular Auditing**: Schedule AIDE and `fail2ban` checks to catch issues early.
- **Secure Backups**: Encrypt backups (covered in Chapter 9) to protect sensitive data.
- **Contribute to Open-Source**: Explore `selinux`, `fail2ban`, and `aide` repositories on GitHub (e.g., `github.com/fail2ban/fail2ban`) to report bugs or improve documentation.

## 5.9 Try This: Harden a Web Server with SELinux and fail2ban

Let’s apply your skills by hardening a RHEL 9 web server with SELinux and `fail2ban`, simulating an enterprise scenario.

**Objective**: Configure Apache with SELinux policies, protect SSH with `fail2ban`, and audit integrity with AIDE.

**Prerequisites**: RHEL 9 VM with 2 GB RAM, 20 GB disk, registered subscription, and `httpd` installed (`sudo dnf install httpd -y`).

**Steps**:

1. **Ensure SELinux is Enforcing**:

   ```bash
   sudo setenforce 1
   sudo sed -i 's/SELINUX=.*/SELINUX=enforcing/' /etc/selinux/config
   ```

2. **Configure Apache with SELinux**:
   Create a custom directory: `sudo mkdir /var/www/custom`.
   Set context: `sudo semanage fcontext -a -t httpd_sys_content_t "/var/www/custom(/.*)?"; sudo restorecon -R /var/www/custom`.
   Create `/var/www/custom/index.html`: `echo "<h1>Test</h1>" | sudo tee /var/www/custom/index.html`.
   Update `/etc/httpd/conf/httpd.conf`:

   ```bash
   <Directory "/var/www/custom">
       Require all granted
   </Directory>
   ```

   Start Apache: `sudo systemctl enable --now httpd`.
3. **Install and Configure fail2ban**:

   ```bash
   sudo dnf install fail2ban -y
   sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
   ```

   Edit `/etc/fail2ban/jail.local`:

   ```bash
   [sshd]
   enabled = true
   port = 22
   logpath = /var/log/secure
   maxretry = 3
   bantime = 3600
   ```

   Start: `sudo systemctl enable --now fail2ban`.
4. **Set Up AIDE**:

   ```bash
   sudo dnf install aide -y
   sudo aide --init
   sudo mv /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz
   ```

   Edit `/etc/aide.conf`:

   ```bash
   /var/www/custom CONTENT+SHA256
   ```

   Test: `sudo aide --check`.
5. **Verify**:
   - Access `http://<server-ip>/index.html` to confirm Apache serves content.
   - Test SSH brute-force (from another system, simulate failed logins) and check bans: `fail2ban-client status sshd`.
   - Modify `/var/www/custom/index.html` and run `sudo aide --check` to detect changes.

**Expected Outcome**: Apache serves content securely under SELinux, `fail2ban` blocks brute-force SSH attempts, and AIDE detects file changes.

**Troubleshooting**:

- **SELinux Denials**: Check `ausearch -m avc -ts recent` and apply custom policies with `audit2allow`.
- **fail2ban Not Banning**: Verify log path (`/var/log/secure`) and restart `fail2ban` (`systemctl restart fail2ban`).
- **AIDE Errors**: Ensure database is initialized and permissions are correct (`ls -Z /var/lib/aide`).

**Open-Source Contribution**: Visit the `fail2ban` GitHub page (`github.com/fail2ban/fail2ban`). Find a “good first issue” (e.g., documentation typo) and comment on how you’d contribute.

## 5.10 Glossary

- **SELinux**: Security-Enhanced Linux, a mandatory access control system using policies and contexts.
- **AppArmor**: An application confinement framework using profiles to restrict programs.
- **PAM**: Pluggable Authentication Modules, managing authentication tasks.
- **fail2ban**: A host-based IPS that bans IPs after repeated failed login attempts.
- **AIDE**: Advanced Intrusion Detection Environment, a file integrity checker.

## 5.11 Conclusion

You’ve now fortified your RHEL 9 system with advanced security techniques, from SELinux and AppArmor for mandatory access control to PAM and `sudoers` for user restrictions, `fail2ban` for intrusion prevention, and AIDE for integrity auditing. The “Try This” exercise simulated a secure web server, preparing you for enterprise challenges and RHCSA certification.

These skills build on prior chapters (e.g., firewall management in Chapter 3, networking in Chapter 4) and prepare you for high availability (Chapter 7) and automation (Chapter 8). Continue exploring open-source tools like `selinux` and `fail2ban` to deepen your expertise and contribute to the Linux community. Next, Chapter 6 explores virtualization and containers, where you’ll leverage your secure systems to deploy scalable environments.

---

<!-- markdownlint-disable MD033 -->
<nav class="chapter-nav" aria-label="Chapter navigation">
  <div style="display:flex;justify-content:space-between;align-items:center">
    <a href="/volume-2/chapter_4.md" rel="prev">← Chapter 4: Networking Concepts</a>
    <a href="/volume-2/chapter_6.md" rel="next">Chapter 6: Virtualization and Containers →</a>
  </div>
</nav>
<!-- markdownlint-enable MD033 -->

---
