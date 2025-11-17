# Chapter 7: High Availability and Clustering

## 7.1 Introduction to High Availability and Clustering

Welcome to the world of **high availability (HA)** and **clustering**, where Linux systems transform from standalone servers into resilient, self-healing infrastructures capable of surviving hardware failures, network outages, and even human error. In today’s digital landscape—where downtime can cost millions and user expectations demand 24/7 access—ensuring service continuity is no longer optional; it’s a fundamental requirement. This chapter explores the principles, tools, and techniques used to build fault-tolerant systems on **Red Hat Enterprise Linux 9 (RHEL 9)**, empowering you to design environments that stay online no matter what.

Imagine your web application as a bustling restaurant. A single chef (server) can handle lunch rush, but what happens if they get sick? The entire operation halts. Now imagine a kitchen with multiple chefs, backup ingredients, and automated alerts when supplies run low—that’s high availability. Clustering takes this further, creating a coordinated team of servers that share workloads, monitor each other, and seamlessly take over if one fails. These technologies are the backbone of cloud platforms, financial systems, e-commerce sites, and any mission-critical service.

Building on your knowledge from **Chapter 6: Virtualization and Containers**, you’ll now learn how to make those virtual machines and containers *survive* in production. We’ll cover **load balancing**, **failover**, **shared storage**, and **cluster communication**, all using open-source tools deeply integrated with RHEL 9. By the end, you’ll be able to deploy a fully redundant web stack that automatically recovers from failure—skills that define modern system administration.

## 7.2 Recapping Prerequisites for High Availability

Before diving into HA, ensure you’re comfortable with:

- **Networking (Chapter 4)**: IP addressing, routing, firewall rules (`firewalld`), and network namespaces.
- **Storage Management (Chapter 2)**: LVM, partitions, and shared block devices.
- **Security (Chapter 5)**: SELinux contexts, firewall zones, and user permissions.
- **Virtualization (Chapter 6)**: KVM VMs, libvirt networking, and storage pools.
- **Containerization**: Running services in Podman containers.

If any of these feel shaky, revisit them—HA systems rely heavily on proper network, storage, and security foundations.

---

## 7.3 Understanding High Availability: Concepts and Goals

### 7.3.1 What Is High Availability?

High Availability refers to a system design that ensures a predefined level of operational performance, usually **uptime greater than 99.9%** ("three nines"). The goal is **zero or near-zero unplanned downtime**, achieved through:

- **Redundancy**: Multiple components (servers, network paths, power supplies)
- **Failover**: Automatic transfer of workload to a standby system
- **Monitoring**: Continuous health checks and alerts
- **Self-healing**: Automated recovery without human intervention

| Uptime % | Downtime per Year |
|--------|-------------------|
| 99%    | ~3.65 days        |
| 99.9%  | ~8.76 hours       |
| 99.99% | ~52 minutes       |
| 99.999%| ~5 minutes        |

RHEL 9 supports HA through the **Red Hat High Availability Add-On**, but we’ll focus on **open-source equivalents** like **HAProxy**, **Keepalived**, **Pacemaker**, and **Corosync**—all freely available and widely used.

### 7.3.2 Key Components of an HA System

| Component         | Role |
|-------------------|------|
| **Load Balancer** | Distributes traffic across healthy nodes |
| **Virtual IP (VIP)** | Floating IP that follows the active node |
| **Cluster Manager** | Coordinates node membership and resource state |
| **Shared Storage** | Ensures data consistency across nodes |
| **Fencing** | Isolates failed nodes to prevent data corruption |

We’ll explore each in depth.

---

## 7.4 Load Balancing with HAProxy

**HAProxy** (High Availability Proxy) is a fast, reliable TCP/HTTP load balancer and proxy solution. It sits in front of your application servers, distributing incoming requests based on rules, health checks, and algorithms.

### 7.4.1 Installing and Configuring HAProxy

Install on RHEL 9:

```bash
sudo dnf install haproxy -y
```

Edit the configuration file `/etc/haproxy/haproxy.cfg`:

```hcl
global
    log /dev/log local0
    maxconn 4096
    user haproxy
    group haproxy
    daemon

defaults
    log     global
    mode    http
    option  httplog
    timeout connect 5000
    timeout client  30000
    timeout server  30000

frontend web_frontend
    bind *:80
    default_backend web_servers

backend web_servers
    balance roundrobin
    server web1 192.168.122.101:80 check
    server web2 192.168.122.102:80 check
```

- `check`: Enables active health checking (probes `/` every 2s)
- `roundrobin`: Distributes requests evenly
- `bind *:80`: Listens on port 80

Start and enable:

```bash
sudo systemctl enable --now haproxy
```

Test with:

```bash
curl http://<haproxy-ip>
```

### 7.4.2 Advanced HAProxy Features

- **SSL Termination**:

  ```hcl
  bind *:443 ssl crt /etc/ssl/certs/mycert.pem
  ```

- **Sticky Sessions**:

  ```hcl
  cookie SERVERID insert
  server web1 ... cookie web1
  ```

- **ACLs and Routing**:

  ```hcl
  acl is_api path_beg /api
  use_backend api_servers if is_api
  ```

---

## 7.5 Failover with Keepalived and Virtual IP

**Keepalived** uses the **VRRP (Virtual Router Redundancy Protocol)** to provide a floating **Virtual IP (VIP)** that moves between nodes. If the primary node fails, the backup takes over the VIP in seconds.

### 7.5.1 Installing Keepalived

```bash
sudo dnf install keepalived -y
```

### 7.5.2 Configuring Master Node (`/etc/keepalived/keepalived.conf`)

```conf
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 150
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass secret123
    }
    virtual_ipaddress {
        192.168.122.100/24
    }
    notify_master "/usr/bin/systemctl start haproxy"
    notify_backup "/usr/bin/systemctl stop haproxy"
}
```

### 7.5.3 Configuring Backup Node

Same config, but:

```conf
state BACKUP
priority 100
```

Enable and start:

```bash
sudo systemctl enable --now keepalived
```

Now, `192.168.122.100` is the VIP. If the master fails, the backup claims it and starts HAProxy.

---

## 7.6 Clustering with Pacemaker and Corosync

For **active-active** or **complex resource management**, use **Pacemaker** (cluster resource manager) with **Corosync** (messaging layer).

### 7.6.1 Theory: How Pacemaker Works

- **Corosync**: Handles node communication via multicast or unicast
- **Pacemaker**: Manages resources (VIPs, services, filesystems) and enforces policies
- **CRM (Cluster Resource Manager)**: Core engine
- **STONITH (Shoot The Other Node In The Head)**: Fencing to prevent split-brain

### 7.6.2 Setting Up a Two-Node Cluster

On **both nodes**:

```bash
sudo dnf install pcs pacemaker corosync fence-agents-all -y
sudo systemctl enable --now pcsd
```

Set password for `hacluster` user:

```bash
sudo passwd hacluster
```

On **node1**, authenticate and create cluster:

```bash
sudo pcs host auth node1 node2
sudo pcs cluster setup mycluster node1 node2
sudo pcs cluster start --all
```

Enable cluster:

```bash
sudo pcs cluster enable --all
```

### 7.6.3 Adding Resources

```bash
# Virtual IP
sudo pcs resource create vip ocf:heartbeat:IPaddr2 ip=192.168.122.100 cidr_netmask=24 op monitor interval=30s

# HAProxy service
sudo pcs resource create haproxy systemd:haproxy op monitor interval=30s

# Colocation: VIP and HAProxy on same node
sudo pcs constraint colocation add haproxy with vip INFINITY

# Order: Start VIP before HAProxy
sudo pcs constraint order vip then haproxy
```

View status:

```bash
sudo pcs status
```

### 7.6.4 Fencing Configuration (Critical!)

Without fencing, a failed node might continue writing to shared storage → **data corruption**.

Use a fence device (e.g., IPMI, VMware, AWS):

```bash
sudo pcs stonith create fence_node1 fence_ipmilan pcmk_host_list=node1 \
    ipaddr=192.168.122.11 login=admin passwd=secret lanplus=1 \
    action=reboot
```

Repeat for node2.

---

## 7.7 Shared Storage for Clustering

For stateful applications (databases, file servers), use **shared block storage**:

- **iSCSI**: Network block storage
- **NFS**: File-level (not ideal for DBs)
- **GlusterFS / Ceph**: Distributed storage

### 7.7.1 Simple iSCSI Target (on Storage Node)

```bash
sudo dnf install targetcli -y
sudo targetcli
/> /backstores/fileio create disk1 /var/lib/iscsi/disk1.img 1G
/> /iscsi create iqn.2025-09.com.example:storage
/> /iscsi/iqn.../tpg1/luns create /backstores/fileio/disk1
/> /iscsi/iqn.../tpg1 set attribute authentication=0
/> exit
```

### 7.7.2 iSCSI Initiator (on Cluster Nodes)

```bash
sudo dnf install iscsi-initiator-utils -y
echo "InitiatorName=iqn.2025-09.com.example:node1" > /etc/iscsi/initiatorname.iscsi
sudo systemctl enable --now iscsid
sudo iscsiadm --mode discovery -t sendtargets -p <storage-ip>
sudo iscsiadm --mode node -T iqn... --login
```

Now `/dev/sdb` is shared. Use in LVM or as a clustered filesystem (OCFS2, GFS2).

---

## 7.8 Monitoring and Testing HA Configurations

### 7.8.1 Tools

- `pcs status`: Cluster state
- `crm_mon -r`: Real-time resource view
- `journalctl -u haproxy`, `-u keepalived`
- `ip addr show`: Check VIP location

### 7.8.2 Testing Failover

1. Stop HAProxy on master → Keepalived should move VIP
2. Reboot master node → Pacemaker should migrate resources
3. Simulate network partition → Fencing should isolate node

---

## 7.9 Best Practices for High Availability

| Practice | Why |
|--------|-----|
| **Use Fencing** | Prevent split-brain and data corruption |
| **Separate Networks** | Control (Corosync), Storage (iSCSI), Frontend |
| **Monitor Everything** | Use Prometheus + Node Exporter |
| **Test Regularly** | Schedule chaos drills |
| **Document Failover** | Runbooks for operators |
| **Avoid Single Points of Failure** | Dual PSUs, NIC teaming, redundant switches |

---

## 7.10 Try This: Configure HAProxy for Load Balancing Across Two Web Servers

**Objective**: Build a fault-tolerant web stack with automatic failover.

**Prerequisites**: Three RHEL 9 VMs:

- `lb` (Load Balancer): HAProxy + Keepalived
- `web1`, `web2`: Apache servers
- All in same network (e.g., `192.168.122.0/24`)

---

### Step-by-Step

#### 1. Set Up Web Servers

On `web1` and `web2`:

```bash
sudo dnf install httpd -y
echo "Welcome to Web1" | sudo tee /var/www/html/index.html
sudo systemctl enable --now httpd
```

On `web1`: change message to "Welcome to Web1"  
On `web2`: "Welcome to Web2"

#### 2. Install HAProxy on `lb`

```bash
sudo dnf install haproxy -y
```

Edit `/etc/haproxy/haproxy.cfg`:

```hcl
frontend web
    bind *:80
    default_backend webs

backend webs
    balance roundrobin
    server web1 192.168.122.101:80 check
    server web2 192.168.122.102:80 check
```

```bash
sudo systemctl enable --now haproxy
```

#### 3. Set Up Keepalived (Master on `lb`, Backup on `web1`)

**On `lb` (priority 150)**:

```conf
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 150
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass mysecret
    }
    virtual_ipaddress {
        192.168.122.200
    }
}
```

**On `web1` (priority 100)**:

```conf
state BACKUP
priority 100
```

Start Keepalived on both.

#### 4. Test

```bash
curl http://192.168.122.200
# Should alternate between Web1 and Web2
```

#### 5. Simulate Failure

```bash
sudo systemctl stop httpd  # on web1
# HAProxy should stop sending traffic to web1
```

#### 6. Test VIP Failover

```bash
sudo systemctl stop keepalived  # on lb
# VIP should move to web1
ping 192.168.122.200
```

**Expected Outcome**: Traffic continues via surviving node; VIP follows active load balancer.

**Troubleshooting**:

- Check `journalctl -u haproxy`, `-u keepalived`
- Use `ip addr show` to verify VIP
- Firewall: Allow VRRP (`firewall-cmd --add-protocol=vrrp --permanent`)

**Open-Source Contribution Idea**:  
Improve HAProxy’s [official documentation](https://github.com/haproxy/haproxy) with a RHEL 9-specific guide. Submit a pull request!

---

## 7.11 Conclusion

You’ve now mastered the core of **high availability and clustering** on RHEL 9:

- **Load balancing** with HAProxy
- **Failover** using Keepalived and VIPs
- **Full clustering** with Pacemaker + Corosync
- **Shared storage** via iSCSI
- **Fencing** and monitoring best practices

These skills allow you to build systems that **don’t just work—they survive**.

This foundation connects directly to:

- **Chapter 8: Automation** – Use Ansible to deploy HA clusters
- **Chapter 9: Backup** – Protect clustered data
- **Chapter 12: Real-World Projects** – Build a complete HA web app

Keep practicing in lab environments. Break things. Fix them. That’s how you become a **Linux resilience engineer**.

> **Next: Chapter 8 – Linux Automation and Configuration Management**  
> Automate your HA deployments with Ansible, and scale to hundreds of nodes.

---

<!-- markdownlint-disable MD033 -->
<nav class="chapter-nav" aria-label="Chapter navigation">
  <div style="display:flex;justify-content:space-between;align-items:center">
    <a href="/volume-2/chapter_6.md" rel="prev">← Chapter 6: Virtualization and Containers</a>
    <a href="/volume-2/chapter_8.md" rel="next">Chapter 8: Automation →</a>
  </div>
</nav>
---
