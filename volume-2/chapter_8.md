# Chapter 8: Linux Automation and Configuration Management

## 8.1 Introduction to Automation and Configuration Management

Welcome to Linux automation and configuration management—where systems do what you want, when you want, without manual effort. In modern IT environments with hundreds or thousands of servers, manual configuration is no longer sustainable. Automation is essential for consistency, speed, security, and scalability.

Imagine seasoning one hundred dishes by hand—one pinch at a time. That is manual server administration. Automation tools like **Ansible**, **Puppet**, **Chef**, and **shell scripting** act like robotic arms that apply the correct settings perfectly every time.

This chapter builds on concepts from **Chapter 7: High Availability**, demonstrating how automation can deploy and manage entire HA clusters using a single command. You will learn how to:

* Write idempotent automation
* Manage server inventory
* Use playbooks to orchestrate complex tasks
* Enforce security policies at scale
* Automate updates, backups, monitoring, and configuration

By the end, you will be able to deploy a fully configured, secure, highly available web stack across multiple servers in minutes—with zero typos.

---

## 8.2 Why Automate? The Business Case

| Without Automation                | With Automation                 |
| --------------------------------- | ------------------------------- |
| Human error (typos, missed steps) | Consistent, repeatable results  |
| Hours to deploy one server        | Minutes to deploy hundreds      |
| Hard to audit changes             | Full change history             |
| “Works on my machine”             | Works everywhere, every time    |
| Unique “snowflake” servers        | Identical, replaceable “cattle” |

**Open-Source Philosophy:** Automation embodies the Linux ethos of being scriptable, reusable, and community-driven. Ansible, for example, is widely adopted, lightweight, and included in RHEL repositories.

---

## 8.3 Automation Tools Landscape on RHEL 9

| Tool    | Type | Agent? | Language | Best For                     |
| ------- | ---- | ------ | -------- | ---------------------------- |
| Ansible | Push | No     | YAML     | Simplicity, RHEL-native      |
| Puppet  | Pull | Yes    | Ruby     | Large enterprise deployments |
| Chef    | Pull | Yes    | Ruby     | Complex workflows            |
| Shell   | Push | No     | Bash     | Quick tasks, scripting glue  |

**Recommendation:** Start with **Ansible**. It is agentless, uses SSH, and integrates easily with RHEL 9.

---

## 8.4 Getting Started with Ansible

### 8.4.1 Installing Ansible

```bash
sudo dnf install ansible -y
ansible --version
```

### 8.4.2 Creating an Inventory File

Create `/etc/ansible/hosts`:

```ini
[webservers]
web1 ansible_host=192.168.122.101
web2 ansible_host=192.168.122.102

[loadbalancers]
lb1 ansible_host=192.168.122.100

[all:vars]
ansible_user=admin
ansible_ssh_private_key_file=/home/admin/.ssh/id_rsa
```

**Tip:** Use DNS names in production. Avoid hardcoding IP addresses when possible.

---

## 8.5 Ad-Hoc Commands

Test connectivity:

```bash
ansible all -m ping
```

Run a command:

```bash
ansible webservers -m command -a "uptime"
```

Install a package:

```bash
ansible webservers -m dnf -a "name=httpd state=present" --become
```

Gather system facts:

```bash
ansible web1 -m setup | less
```

---

## 8.6 Playbooks: The Heart of Ansible

A playbook is a YAML file containing tasks, roles, variables, and desired state definitions.

### 8.6.1 Example Playbook: Deploy Apache

Create `deploy-web.yml`:

```yaml
---
- name: Deploy Apache Web Server
  hosts: webservers
  become: yes
  tasks:
    - name: Install Apache
      dnf:
        name: httpd
        state: present

    - name: Enable and start httpd
      systemd:
        name: httpd
        enabled: yes
        state: started

    - name: Deploy index.html
      copy:
        content: "Welcome to {{ inventory_hostname }}!\n"
        dest: /var/www/html/index.html
        mode: '0644'
```

Run:

```bash
ansible-playbook deploy-web.yml
```

**Idempotency:** Running the playbook multiple times will always yield the same final state.

---

## 8.7 Variables, Templates, and Conditionals

### 8.7.1 Variables

In the inventory:

```ini
[webservers]
web1 app_env=prod
web2 app_env=dev
```

In a playbook:

```yaml
- name: Set message based on environment
  copy:
    content: "This is {{ app_env }} environment\n"
    dest: /var/www/html/env.txt
```

### 8.7.2 Jinja2 Templates

Create `index.html.j2`:

```html
<!DOCTYPE html>
<html>
<head>
  <title>{{ inventory_hostname }}</title>
</head>
<body>
  <h1>Welcome to {{ inventory_hostname }}</h1>
  <p>Environment: <strong>{{ app_env | default('unknown') }}</strong></p>
  <p>Deployed on: {{ ansible_date_time.date }}</p>
</body>
</html>
```

Use the template:

```yaml
- name: Deploy dynamic homepage
  template:
    src: index.html.j2
    dest: /var/www/html/index.html
    mode: '0644'
```

---

## 8.8 Roles: Modular Automation

Directory structure:

```plaintext
roles/
  webserver/
    tasks/
    templates/
    vars/
    handlers/
```

### 8.8.1 Creating a Role

Initialize:

```bash
ansible-galaxy init roles/webserver
```

Example `roles/webserver/tasks/main.yml`:

```yaml
- name: Install httpd
  dnf:
    name: httpd
    state: present

- name: Deploy config
  template:
    src: index.html.j2
    dest: /var/www/html/index.html
  notify: restart httpd

- name: Ensure httpd is running
  systemd:
    name: httpd
    state: started
    enabled: yes
```

Example `roles/webserver/handlers/main.yml`:

```yaml
- name: restart httpd
  systemd:
    name: httpd
    state: restarted
```

Use the role:

```yaml
---
- name: Deploy web servers using role
  hosts: webservers
  become: yes
  roles:
    - webserver
```

---

## 8.9 Advanced Ansible Techniques

### 8.9.1 Loops

```yaml
- name: Create multiple users
  user:
    name: "{{ item }}"
    state: present
  loop:
    - alice
    - bob
    - charlie
```

### 8.9.2 Conditionals

```yaml
- name: Install MySQL only on DB servers
  dnf:
    name: mariadb-server
    state: present
  when: "'dbservers' in group_names"
```

### 8.9.3 Error Handling

```yaml
- name: Test web server
  uri:
    url: "http://{{ inventory_hostname }}"
    return_content: yes
  register: webpage
  ignore_errors: yes

- name: Report failure
  fail:
    msg: "Web server not responding!"
  when: webpage.failed
```

---

## 8.10 Integrating Automation with High Availability

### 8.10.1 Automating HAProxy Deployment

Example role: `roles/haproxy/tasks/main.yml`

```yaml
- name: Install HAProxy
  dnf:
    name: haproxy
    state: present

- name: Deploy HAProxy config
  template:
    src: haproxy.cfg.j2
    dest: /etc/haproxy/haproxy.cfg
  notify: restart haproxy

- name: Enable HAProxy
  systemd:
    name: haproxy
    enabled: yes
    state: started
```

Example `haproxy.cfg.j2`:

```jinja
frontend web
  bind *:80
  default_backend webs

backend webs
  balance roundrobin
  {% for host in groups['webservers'] %}
  server {{ host }} {{ hostvars[host].ansible_host }}:80 check
  {% endfor %}
```

Deploy the full HA stack using one playbook.

---

## 8.11 Best Practices for Ansible

| Practice    | Reason                            |
| ----------- | --------------------------------- |
| Use Git     | Version control for playbooks     |
| Idempotency | Safe to rerun playbooks           |
| Roles       | Reusable, modular, maintainable   |
| Vault       | Encrypt secrets                   |
| Linting     | `ansible-lint` improves quality   |
| Testing     | Use Molecule for role testing     |
| Tags        | Run only what you need (`--tags`) |

---

## 8.12 Try This: Log Cleanup Script + Package Installation

### Objective

Combine shell scripting with Ansible automation.

### Part 1: Shell Script for Log Cleanup

Create `cleanup-logs.sh`:

```bash
#!/bin/bash
LOGDIR="/var/log"
MAX_AGE=30

echo "Cleaning logs older than $MAX_AGE days in $LOGDIR..."

find "$LOGDIR" -type f -name "*.log" -mtime +$MAX_AGE -print -delete

journalctl --vacuum-time=30d
logrotate -f /etc/logrotate.conf

echo "Log cleanup completed at $(date)"
```

Make executable:

```bash
chmod +x cleanup-logs.sh
```

### Part 2: Ansible Playbook

Create `site.yml`:

```yaml
---
- name: System Maintenance Automation
  hosts: all
  become: yes
  tasks:
    - name: Install htop
      dnf:
        name: htop
        state: present

    - name: Deploy log cleanup script
      copy:
        src: cleanup-logs.sh
        dest: /usr/local/bin/cleanup-logs.sh
        mode: '0755'

    - name: Schedule daily cleanup
      cron:
        name: "daily log cleanup"
        minute: "0"
        hour: "2"
        job: "/usr/local/bin/cleanup-logs.sh >> /var/log/cleanup.log 2>&1"
```

Run:

```bash
ansible-playbook site.yml
```

### Expected Outcome

* `htop` installed on all servers
* Log cleanup script deployed
* Daily cron job created
* Logs older than 30 days removed automatically

### Troubleshooting

| Issue              | Fix                           |
| ------------------ | ----------------------------- |
| Permission denied  | Ensure `become: yes` is set   |
| Script not running | Verify path and `chmod +x`    |
| Cron not firing    | Check `cronie` service status |

### Open-Source Contribution

Contribute an improved HAProxy + Podman example to Ansible’s official documentation.

---

## 8.13 Conclusion

You now understand how to automate Linux systems at scale on RHEL 9 using Ansible. You can:

* Run ad-hoc commands
* Build structured playbooks
* Use roles for modular automation
* Apply templates and variables
* Integrate automation with HA, security, and containers

These skills prepare you for **Chapter 12: Real-World Projects**, where you will combine automation, HA, security, and infrastructure-as-code into production-ready deployments.

Next up: **Chapter 9 – Advanced Backup and Recovery**.

---

<!-- markdownlint-disable MD033 -->
<nav class="chapter-nav" aria-label="Chapter navigation">
  <div style="display:flex;justify-content:space-between;align-items:center">
    <a href="/volume-2/chapter_7.md" rel="prev">← Chapter 7: High Availability and Clustering</a>
    <a href="/volume-2/chapter_9.md" rel="next">Chapter 9: Advanced Backup and Recovery→</a>
  </div>
</nav>
<!-- markdownlint-enable MD033 -->

---
