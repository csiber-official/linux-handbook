# Chapter 4: Advanced Networking

## 4.1 Introduction to Advanced Networking

Welcome to the world of advanced Linux networking. As a system administrator, networking is the foundation that connects servers, services, and users. It enables everything from simple file sharing to full cloud infrastructures.

This chapter introduces advanced Linux networking concepts using Red Hat Enterprise Linux 9 (RHEL 9). You will learn how to set up DNS servers, create VPNs, strengthen network security, monitor traffic, and troubleshoot issues. These skills align with the Red Hat Certified System Administrator (RHCSA) objectives and provide a bridge toward RHCE-level knowledge.

In enterprise environments, networking responsibilities may include configuring DNS for an internal domain, establishing a VPN for remote workers, or inspecting network traffic for anomalies. This chapter builds on basic networking knowledge such as IP addressing, routing, and basic firewalld rules, and expands into real-world administrative scenarios.

By the end, you will be able to design, deploy, and maintain robust networks on RHEL 9, contribute to open-source networking tools, and prepare for real-world production challenges.

---

## 4.2 Recapping Networking Basics

Before moving into advanced topics, ensure that the following skills are familiar:

* **IP addressing and interfaces**
  Understand IPv4/IPv6, subnets (for example `192.168.1.0/24`), and configuring interfaces using `nmcli` or `ip addr`.

* **Routing and connectivity**
  Know how to configure gateways and static routes using `ip route add`, and how to diagnose issues using tools such as `ping`, `traceroute`, `netstat`, or `ss`.

* **Firewall fundamentals**
  Be comfortable allowing ports and services using `firewall-cmd`.
  Example:

  ```bash
  sudo firewall-cmd --add-port=80/tcp
  ```

* **DNS resolution**
  Understand how to edit `/etc/resolv.conf` and use `dig` or `nslookup`.

If any of these topics feel rusty, practice by configuring a static IP address and testing network connectivity in a RHEL 9 VM.

---

## 4.3 Configuring a DNS Server

The Domain Name System (DNS) translates domain names into IP addresses. RHEL 9 uses BIND (bind9), a widely used DNS server. DNS is essential for both internal and external name resolution.

### 4.3.1 Understanding DNS Concepts

DNS is hierarchical:

* **Zones**
  Each DNS zone contains resource records such as A, CNAME, or MX entries.

* **Forwarding and caching**
  DNS servers may forward unresolved queries to external resolvers and cache responses.

* **Master/slave (primary/secondary) servers**
  Secondary servers pull zone data from the primary to provide redundancy.

Enterprises often run internal DNS zones (for example `intranet.local`) while forwarding public queries to resolvers like `8.8.8.8`.

### 4.3.2 Installing and Configuring BIND

Install BIND and utilities:

```bash
sudo dnf install bind bind-utils -y
sudo systemctl enable --now named
```

Edit `/etc/named.conf` to configure a forwarding DNS server:

```conf
options {
    listen-on port 53 { any; };
    allow-query { localhost; 192.168.1.0/24; };
    forwarders { 8.8.8.8; 8.8.4.4; };
    dnssec-validation no;
};
```

To create an authoritative zone, add:

```conf
zone "example.local" IN {
    type master;
    file "/var/named/example.local.zone";
};
```

Create the zone file `/var/named/example.local.zone`:

```text
$ORIGIN example.local.
$TTL 86400
@   IN SOA ns.example.local. admin.example.local. (
        2025010101
        3600
        1800
        604800
        86400 )
@   IN NS ns.example.local.
ns  IN A 192.168.1.10
www IN A 192.168.1.20
```

Restart and test:

```bash
sudo systemctl restart named
dig @localhost www.example.local
```

### 4.3.3 Advanced DNS Features

* **Secondary (slave) servers**
  Configure a secondary with:

  ```conf
  type slave;
  masters { 192.168.1.10; };
  ```

* **DNSSEC**
  Improve integrity using `dnssec-keygen` and `dnssec-signzone` to sign zones.

### 4.3.4 Troubleshooting DNS

* **Queries fail**

  ```bash
  journalctl -u named
  named-checkzone example.local /var/named/example.local.zone
  ```

* **Forwarding issues**

  ```bash
  sudo firewall-cmd --add-service=dns
  dig google.com @localhost
  ```

* **Zone transfer failures**
  Check `allow-transfer` rules and SELinux contexts:

  ```bash
  restorecon -Rv /var/named
  ```

---

## 4.4 Setting Up a VPN

A Virtual Private Network (VPN) protects data in transit by encrypting communication across public networks. RHEL 9 supports OpenVPN for remote-access and site-to-site VPNs.

### 4.4.1 Understanding VPN Principles

OpenVPN uses TLS to encrypt sessions and authenticate clients using certificates or keys. In an enterprise environment, VPNs provide secure access to internal resources and reduce risks such as eavesdropping or man-in-the-middle attacks.

### 4.4.2 Installing and Configuring OpenVPN

Install OpenVPN and Easy-RSA:

```bash
sudo dnf install openvpn easy-rsa -y
```

Set up Easy-RSA:

```bash
sudo mkdir /etc/openvpn/easy-rsa
sudo cp -r /usr/share/easy-rsa/3/* /etc/openvpn/easy-rsa/
cd /etc/openvpn/easy-rsa
sudo ./easyrsa init-pki
sudo ./easyrsa build-ca nopass
sudo ./easyrsa gen-dh
sudo ./easyrsa build-server-full server nopass
sudo ./easyrsa build-client-full client nopass
```

Server configuration `/etc/openvpn/server.conf`:

```conf
port 1194
proto udp
dev tun

ca /etc/openvpn/easy-rsa/pki/ca.crt
cert /etc/openvpn/easy-rsa/pki/issued/server.crt
key /etc/openvpn/easy-rsa/pki/private/server.key
dh /etc/openvpn/easy-rsa/pki/dh.pem

server 10.8.0.0 255.255.255.0
push "route 192.168.1.0 255.255.255.0"

keepalive 10 120
cipher AES-256-CBC
```

Enable the server:

```bash
sudo systemctl enable --now openvpn-server@server
```

Clients connect using a generated `.ovpn` profile.

### 4.4.3 Advanced VPN Configurations

* **Two-factor authentication** using PAM modules
* **Split tunneling** to send only specific traffic over the VPN
* **High availability** using Keepalived (covered in Chapter 7)

### 4.4.4 Troubleshooting VPN

* **Connection refused**

  ```bash
  journalctl -u openvpn-server@server
  sudo firewall-cmd --add-port=1194/udp
  ```

* **Certificate errors**
  Check file paths and permissions.

* **Routing issues**

  ```bash
  ip route
  push "redirect-gateway def1"
  ```

---

## 4.5 Network Security

RHEL 9 offers several tools for securing networks, including firewalld, iptables, and SELinux.

### 4.5.1 Firewalld and iptables

Firewalld provides zone-based dynamic firewall management. Iptables is more granular and lower level.

Example translation:

```bash
sudo firewall-cmd --add-rich-rule='rule family="ipv4" source address="192.168.1.50" port port=22 protocol=tcp accept'
```

Equivalent iptables rule:

```bash
sudo iptables -A INPUT -s 192.168.1.50 -p tcp --dport 22 -j ACCEPT
```

### 4.5.2 Basic SELinux Networking

Examples:

```bash
sudo semanage port -a -t http_port_t -p tcp 8080
sudo setsebool -P httpd_can_network_connect on
```

### 4.5.3 Troubleshooting Security

* **Rules not applying**

  ```bash
  sudo firewall-cmd --reload
  iptables -L -v -n
  ```

* **SELinux denials**

  ```bash
  ausearch -m avc -ts recent
  audit2allow -a -M mypolicy
  sudo semodule -i mypolicy.pp
  ```

---

## 4.6 Network Monitoring and Packet Analysis

Monitoring helps detect failures and performance issues. Tools include `tcpdump` and Wireshark.

### 4.6.1 Using tcpdump

Example capture:

```bash
sudo tcpdump -i eth0 -nn -s0 -v port 80
```

### 4.6.2 Wireshark Integration

Install Wireshark:

```bash
sudo dnf install wireshark -y
```

Capture using TShark:

```bash
sudo tshark -i eth0 -f "tcp port 80" -w capture.pcap
```

### 4.6.3 Troubleshooting Monitoring

* **No packets captured**

  ```bash
  tcpdump -D
  ip link set eth0 promisc on
  ```

* **Too much data**
  Use limits:

  ```bash
  -c 100
  ```

---

## 4.7 Troubleshooting Complex Network Issues

Use a layered approach:

1. Check interfaces:

   ```bash
   ip link show
   ip addr show
   ```

2. Test connectivity:

   ```bash
   ping -c 4 gateway
   traceroute destination
   ```

3. Review firewall and SELinux:

   ```bash
   firewall-cmd --list-all
   sealert -a /var/log/audit/audit.log
   ```

4. Capture traffic:

   ```bash
   tcpdump
   ```

5. Inspect logs:

   ```bash
   journalctl -u NetworkManager
   ```

Common scenarios:

* **Intermittent connectivity**
  Use:

  ```bash
  mtr
  ```

* **DNS failures**

  ```bash
  dig +trace example.com
  ```

---

## 4.8 Try This: Set Up a DNS Server and Analyze Traffic

### Objective

Deploy a forwarding DNS server, add a custom zone, and capture DNS queries.

### Prerequisites

A RHEL 9 VM with network access.

### Steps

1. Install BIND:

   ```bash
   sudo dnf install bind bind-utils -y
   sudo systemctl enable --now named
   ```

2. Configure forwarding in `/etc/named.conf`.

3. Create `/var/named/test.local.zone` with sample records.

4. Open firewall access:

   ```bash
   sudo firewall-cmd --permanent --add-service=dns
   sudo firewall-cmd --reload
   ```

5. Test DNS:

   ```bash
   dig @localhost www.test.local
   ```

6. Analyze traffic:

   ```bash
   sudo tcpdump -i any -nn -s0 port 53 -c 10
   ```

### Expected Outcome

Your DNS server resolves internal names and forwards external queries. `tcpdump` displays DNS packets with source and destination IPs.

### Troubleshooting

* **BIND errors**

  ```bash
  journalctl -u named
  named-checkzone test.local /var/named/test.local.zone
  ```

* **No packet capture**
  Ensure interface is active.

* **SELinux denials**

  ```bash
  restorecon -Rv /var/named
  ```

### Open-Source Contribution

Explore BIND at:
[https://github.com/isc-projects/bind9](https://github.com/isc-projects/bind9)

---

## 4.9 Conclusion

You explored advanced networking concepts on RHEL 9, including DNS with BIND, VPNs with OpenVPN, layered security with firewalld/iptables/SELinux, and traffic analysis with tcpdump and Wireshark.

These skills support earlier chapters on server administration and prepare you for security hardening (Chapter 5) and high availability (Chapter 7). Continue experimenting with open-source tools such as BIND and OpenVPN to build confidence and contribute to the Linux community.

---

<!-- markdownlint-disable MD033 -->
<nav class="chapter-nav" aria-label="Chapter navigation">
  <div style="display:flex;justify-content:space-between;align-items:center">
    <a href="chapter_3.md" rel="prev">← Chapter 3: Networking Fundamentals</a>
    <a href="chapter_5.md" rel="next">Chapter 5: System Security and Hardening →</a>
  </div>
</nav>
<!-- markdownlint-enable MD033 -->
---
