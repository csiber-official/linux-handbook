# Chapter 12: Bring It All Together: Real-World Projects

## 12.1 Introduction to Real-World Projects

Welcome to the **grand finale**—the moment where theory becomes **battle-tested
infrastructure**. This chapter is your **capstone**, where you’ll deploy **four
production-grade projects** that integrate concepts from Volumes 1 and 2:
file systems, security, networking, containers, high availability, backup,
performance, and troubleshooting.

These aren’t toy labs. They represent **real-world scenarios** faced by Linux
engineers:

* A global web app resilient to server failures
* A secure file server in a DMZ
* A multi-server environment automated through Ansible
* A ransomware-resistant backup/recovery system

By the end, you’ll have **four portfolio-ready projects**.

We will build:

* **Project 1**: HA Web App (HAProxy + Docker + Ansible)
* **Project 2**: Secure NFS File Server with SELinux
* **Project 3**: Automated Multi-Server Deployment
* **Project 4**: Ransomware-Resistant Backup and Recovery

Each includes:

* Architecture diagram
* Deployment procedure
* Validation checklist
* Troubleshooting tips
* Open-source contribution idea

---

## 12.2 Project Prerequisites

| Requirement | Specification                                                  |
| ----------- | -------------------------------------------------------------- |
| **VMs**     | 6× RHEL 9 VMs (4 GB RAM, 2 vCPU)                               |
| **Network** | Internal LAN (`192.168.122.0/24`)                              |
| **Tools**   | `dnf`, `podman`, `ansible`, `haproxy`, `nfs`, `bacula`, `rear` |
| **Access**  | Root + SSH key authentication                                  |

VM labels:

* `loadbalancer`
* `web1`, `web2`
* `ansible-control`
* `fileserver`
* `backupserver`

---

## 12.3 Project 1: Deploy a Highly Available Web Application

### 12.3.1 Goal of Project 1

Deploy a **Dockerized web app** behind **HAProxy** with **zero downtime**.

### 12.3.2 Architecture Diagram

```text
[Client] → [HAProxy] → [web1 (Docker)]
                     → [web2 (Docker)]
```

### 12.3.3 Configuring Web Servers

Run on **web1** and **web2**:

```bash
sudo dnf install podman -y
sudo podman run -d --name web -p 8080:80 nginx
echo "<h1>Server: $(hostname)</h1>" | sudo tee /var/www/html/index.html
```

### 12.3.4 Configuring HAProxy

```bash
sudo dnf install haproxy -y
```

`/etc/haproxy/haproxy.cfg`:

```conf
global
    log /dev/log local0
    maxconn 4096

defaults
    log global
    mode http
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms

frontend http-in
    bind *:80
    default_backend webservers

backend webservers
    balance roundrobin
    server web1 192.168.122.101:8080 check
    server web2 192.168.122.102:8080 check
```

Enable HAProxy:

```bash
sudo systemctl enable --now haproxy
```

### 12.3.5 Testing High Availability

```bash
curl http://loadbalancer
```

Stop the web container on `web1`:

```bash
sudo podman stop web
```

Test again:

```bash
curl http://loadbalancer
```

**Result:** App still responds.

---

## 12.4 Project 2: Secure NFS File Server with SELinux

### 12.4.1 Goal of Project 2

Build an **SELinux-enforced NFS server** with a secure export.

### 12.4.2 Configure NFS

On `fileserver`:

```bash
sudo dnf install nfs-utils -y
sudo mkdir -p /exports/data
echo "Shared Data" | sudo tee /exports/data/file.txt
```

`/etc/exports`:

```text
/exports/data 192.168.122.0/24(rw,sync,sec=krb5p)
```

```bash
sudo exportfs -a
sudo systemctl enable --now nfs-server
```

### 12.4.3 Apply SELinux Contexts

```bash
sudo setsebool -P nfs_export_all_rw on
sudo semanage fcontext -a -t public_content_rw_t "/exports/data(/.*)?"
sudo restorecon -R /exports/data
```

### 12.4.4 Configure Firewall

```bash
sudo firewall-cmd --add-service=nfs --permanent
sudo firewall-cmd --reload
```

### 12.4.5 Mount from Client

On `web1`:

```bash
sudo dnf install nfs-utils -y
sudo mkdir -p /mnt/shared
sudo mount -t nfs fileserver:/exports/data /mnt/shared
echo "test" | sudo tee -a /mnt/shared/file.txt
```

**Success:** Secure NFS share available.

---

## 12.5 Project 3: Automated Deployment with Ansible

### 12.5.1 Goal of Project 3

Use Ansible to deploy **Apache + firewall rules**.

### 12.5.2 Configure Ansible Control Node

```bash
sudo dnf install ansible -y
ssh-keygen
ssh-copy-id root@web1
ssh-copy-id root@web2
```

### 12.5.3 Create Inventory

`/etc/ansible/hosts`:

```ini
[webservers]
web1 ansible_host=192.168.122.101
web2 ansible_host=192.168.122.102
```

### 12.5.4 Write Playbook

`deploy-web.yml`:

```yaml
---
- name: Deploy Apache and Firewall Rules
  hosts: webservers
  become: yes
  tasks:
    - name: Install Apache
      dnf:
        name: httpd
        state: present

    - name: Enable Apache
      systemd:
        name: httpd
        state: started
        enabled: yes

    - name: Open HTTP Port
      firewalld:
        service: http
        permanent: yes
        state: enabled
      notify: reload firewall

  handlers:
    - name: reload firewall
      service:
        name: firewalld
        state: reloaded
```

### 12.5.5 Run Playbook

```bash
ansible-playbook deploy-web.yml
```

---

## 12.6 Project 4: Recover System After Simulated Ransomware

### 12.6.1 Goal of Project 4

Simulate an attack → restore via **Bacula + ReaR**.

### 12.6.2 Simulate Ransomware

On `web1`:

```bash
find /var/www/html -type f -exec sh -c 'mv "$1" "$1.encrypted"' _ {} \;
```

### 12.6.3 Restore from Bacula

On `backupserver`:

```bash
bconsole
*restore
```

Follow prompts to restore to `/tmp/restore`.

### 12.6.4 Bare-Metal Restore with ReaR

```bash
sudo rear -v recover
```

---

## 12.7 Validation Checklist

| Project | Test                          | Expected |
| ------- | ----------------------------- | -------- |
| 1       | Fail `web1`, curl LB          | 200 OK   |
| 2       | `getsebool nfs_export_all_rw` | on       |
| 3       | `ansible webservers -m ping`  | success  |
| 4       | Files restored                | present  |

---

## 12.8 Open-Source Contribution Ideas

| Project | Contribution                                |
| ------- | ------------------------------------------- |
| 1       | Improve HAProxy health-check examples       |
| 2       | Write SELinux policy for home-directory NFS |
| 3       | Publish Ansible role to Galaxy              |
| 4       | Improve ReaR docs for LUKS encryption       |

---

## 12.9 Conclusion

You’ve built:

* HA web stack
* Secure NFS server
* Automated deployment pipeline
* Ransomware-resistant backup system

You mastered:

* Production workflows
* Troubleshooting
* Automation
* High availability

> **Next Steps**
> • Deploy to AWS/GCP
> • Add Prometheus + Grafana
> • Prepare for RHCE
> • Contribute to Linux

---

<!-- markdownlint-disable MD033 -->
<nav class="chapter-nav" aria-label="Chapter navigation">
  <div style="display:flex;justify-content:space-between;align-items:center">
    <a href="chapter_11.md" rel="prev">← Chapter 11: Backup and Recovery</a>
  </div>
</nav>
<!-- markdownlint-enable MD033 -->
---
