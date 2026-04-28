# Ansible — Complete Reference Guide

> **Configuration Management & IT Automation**

---

## Table of Contents

1. [Overview & Context](#1-overview--context)
2. [Ansible Architecture](#2-ansible-architecture)
3. [Installation & Setup](#3-installation--setup)
4. [Host Inventory & Configuration](#4-host-inventory--configuration)
5. [Ways of Working with Ansible](#5-ways-of-working-with-ansible)
   - [Ad-hoc Commands](#51-ad-hoc-commands)
   - [Modules](#52-ansible-modules)
   - [Playbooks](#53-ansible-playbooks)
6. [Playbook Examples](#6-playbook-examples)
7. [Useful Playbook Commands Reference](#7-useful-playbook-commands-reference)

---

## 1. Overview & Context

### DevOps Toolchain Context

| Tool | Purpose |
|------|---------|
| **Jenkins** | CI/CD (Continuous Integration & Continuous Delivery) |
| **Docker** | Containerization |
| **Kubernetes** | Container Orchestration |
| **Terraform** | Infrastructure Provisioning (creates infrastructure) |
| **Ansible** | Configuration Management & IT Automation (configures infrastructure) |
| **Chef / Puppet** | Alternative Configuration Management tools |

> **Key Distinction:** Terraform is used to *create* infrastructure (e.g., spin up EC2 instances), while Ansible is used to *configure* that infrastructure (e.g., install software, manage services).

---

### What is Ansible?

Ansible is a free and open-source IT automation tool used for configuration management, application deployment, and task automation across multiple servers simultaneously — without needing any agent software installed on the managed machines.

- **Inventor:** Michael DeHaan (2012–2013)
- **Current Maintainer:** Red Hat
- **License:** Free & Open Source

---

## 2. Ansible Architecture

```
  +-------+  +-------+  +-------+  +-------+  +-------+  +-------+
  | dev-1 |  | dev-2 |  | test-1|  | test-2|  | prod-1|  | prod-2|
  +-------+  +-------+  +-------+  +-------+  +-------+  +-------+
      \          \           |          /          /          /
       \          \          |         /          /          /
        \          \   [SSH Connection]          /          /
         \          \        |        /          /          /
          \          \       |       /          /          /
           \          \      |      /          /          /
            \          \     |     /          /          /
             +----------+----+----+----------+----------+
                               |
                     +===================+
                     |    MASTER NODE    |
                     |-------------------|
                     | >> Install Ansible|
                     | >> Install Python |
                     +===================+
                               |
              +----------------+----------------+
              |                                 |
   +--------------------+          +--------------------+
   | host inventory file|          |    ansible.cfg     |
   | (list of all nodes)|          | (main config file) |
   +--------------------+          +--------------------+
```

> **Important:** All Ansible commands must be executed from the **Master Node**.

---

### Architecture Components Explained

#### Master Node
The central control machine where Ansible and Python are installed. It sends all instructions to the managed nodes via SSH. You write and run all your playbooks and ad-hoc commands from here.

#### Managed Nodes
The remote servers that Ansible manages. No Ansible installation is required on these machines — only SSH access is needed.

| Group | Servers | Role |
|-------|---------|------|
| `dev-servers` | dev-1, dev-2 | Development Environment |
| `test-servers` | test-1, test-2 | Testing/QA Environment |
| `prod-servers` | prod-1, prod-2 | Production Environment |

#### SSH Connection
Ansible communicates with all managed nodes over SSH. This is a secure and lightweight approach that requires no additional agents or daemons running on the remote machines.

#### Agentless Design
Unlike Puppet or Chef — which require agent software to be installed on each managed node — **Ansible is completely agentless**. This greatly simplifies setup and reduces overhead.

#### Configuration Files (on Master Node)

| File | Purpose |
|------|---------|
| `host inventory file` | Lists all managed nodes (by IP or hostname) |
| `ansible.cfg` | Main Ansible configuration file |

#### Software Ansible Can Manage

Ansible can install, configure, and manage virtually any software on managed nodes, for example:

| Software | Description |
|----------|-------------|
| `tomcat 9.0.10` | Web Application Server |
| `java 21` | Programming Runtime |
| `maven 3.9.7` | Build Automation Tool |
| `httpd`, `nginx`, `docker`, etc. | Any other software |

---

## 3. Installation & Setup

### Step 1 — Install Ansible on the Master Node

```bash
sudo dnf install python3-pip -y
pip install ansible
ansible --version
```

---

### Step 2 — Set Hostname (on all servers)

Run this on each server to set a recognizable hostname:

```bash
sudo hostnamectl set-hostname <Server-Name>
```

---

### Step 3 — Enable SSH Password Authentication (on all servers)

Edit the SSH daemon config to allow password-based login (required so you can copy SSH keys):

```bash
vi /etc/ssh/sshd_config
```

Update the following lines:
- **Line 38:** `PermitRootLogin yes`
- **Line 63:** `PasswordAuthentication yes`

Then restart SSH:

```bash
systemctl restart sshd
systemctl status sshd
```

---

### Step 4 — Set Root Password (on all managed nodes)

Set a root password on each managed node (dev, test servers):

```bash
passwd root
```

---

### Step 5 — Copy SSH Key from Master to All Managed Nodes

This enables passwordless SSH access from the Master Node:

```bash
ssh-copy-id root@<Private-IP>
```

Repeat this for every managed node's private IP address.

---

## 4. Host Inventory & Configuration

By default, newer versions of Ansible do **not** create the `/etc/ansible/` directory automatically. You must create it manually.

### Step 1 — Create the Ansible Directory

```bash
sudo mkdir -p /etc/ansible
```

---

### Step 2 — Create the Main Config File

```bash
sudo nano /etc/ansible/ansible.cfg
```

Paste the following minimal configuration:

```ini
[defaults]
inventory=/etc/ansible/hosts
host_key_checking=False
```

Save and exit.

---

### Step 3 — Create the Hosts Inventory File

```bash
sudo nano /etc/ansible/hosts
```

Add your managed nodes grouped by environment:

```ini
[dev-server]
172.31.15.228

[test-server]
172.31.14.181
```

Save and exit.

---

### Step 4 — Verify Connectivity

Test that Ansible can reach all nodes:

```bash
ansible all -m ping
```

Expected output (you should see `pong` from each managed node):

```
172.31.14.181 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3.9"
    },
    "changed": false,
    "ping": "pong"
}
```

---

### Useful Inventory Commands

```bash
# List all hosts in the inventory
ansible all --list-hosts

# List hosts for a specific group
ansible dev-server --list-hosts
ansible test-server --list-hosts

# Show the full inventory in JSON format
ansible-inventory --list
```

---

## 5. Ways of Working with Ansible

There are **three primary ways** to work with Ansible:

| # | Method | Description |
|---|--------|-------------|
| 1 | **Ad-hoc Commands** | Quick one-liner commands for immediate tasks |
| 2 | **Modules** | Structured key-value commands for idempotent operations |
| 3 | **Playbooks** | YAML files that define multi-step automation workflows |

---

## 5.1 Ad-hoc Commands

Ad-hoc commands let you run quick, one-time tasks on managed nodes without writing a playbook. They use the `-a` flag to pass shell commands directly.

```bash
# Install git on all servers
ansible all -a "yum install git -y"

# Install maven only on dev-server group
ansible dev-server -a "yum install maven -y"

# Create a user on all servers
ansible all -a "useradd ming"

# View /etc/passwd on all servers
ansible all -a "cat /etc/passwd"

# Check git version (run after install)
git -v

# Check maven version (run after install)
mvn -v

# Remove maven from dev-server group
ansible dev-server -a "yum remove maven* -y"

# Remove git from all servers
ansible all -a "yum remove git* -y"

# Create a file on all servers
ansible all -a "touch file1.txt"

# Create a directory on all servers
ansible all -a "mkdir ansible-day01"
```

> **Note:** Ad-hoc commands with `-a` run raw shell commands and are useful for quick tasks. For repeatable, idempotent operations, use **Modules** or **Playbooks** instead.

---

## 5.2 Ansible Modules

Modules are pre-built units of work that Ansible provides. They work based on **key=value** pairs and are designed to be **idempotent** — running them multiple times produces the same result without causing unintended changes.

---

### Module States Reference

#### `yum` Module States

| Action | State Value |
|--------|-------------|
| Install a package | `present` |
| Uninstall a package | `absent` |
| Update a package | `latest` |

#### `service` Module States

| Action | State Value |
|--------|-------------|
| Start a service | `started` |
| Stop a service | `stopped` |
| Restart a service | `restarted` |
| Enable a service on boot | `enabled` |

---

### Module Command Examples

```bash
# Install docker on all servers using the yum module
ansible all -m yum -a "name=docker state=present"

# Install Apache (httpd) on all servers
ansible all -m yum -a "name=httpd state=present"

# Verify Apache was installed
ansible all -a "httpd -v"

# Remove docker from all servers
ansible all -m yum -a "name=docker state=absent"

# Check httpd service status (ad-hoc)
ansible all -a "systemctl status httpd"

# Start the httpd service using the service module
ansible all -m service -a "name=httpd state=started"

# Verify httpd is now running
ansible all -a "systemctl status httpd"

# Create a user named 'lambo' on all servers
ansible all -m user -a "name=lambo state=present"

# Verify the user was created
ansible all -a "cat /etc/passwd"

# Create a local file to copy
touch ansible.py

# Copy the file to /root on all managed nodes
ansible all -m copy -a "src=ansible.py dest=/root"

# Verify the file was copied
ansible all -a "ls /root"
ansible all -a "ls /tmp"
ansible all -a "ls /bin"
```

---

## 5.3 Ansible Playbooks

A **Playbook** is a YAML file that defines one or more **plays**, each of which contains a list of **tasks** to execute on a set of managed hosts. Playbooks are the most powerful and recommended way to use Ansible for real automation workflows.

---

### Playbook Syntax Rules

- Playbooks are written in **YAML format**
- Every YAML playbook **starts with `---`** and optionally ends with `...`
- File extensions: `.yml` or `.yaml`
- Playbooks are **case-sensitive**
- **Indentation is critical** — YAML relies on consistent spacing (use 2 spaces, never tabs)
- Always begin plays with a `- name:` description for readability

---

## 6. Playbook Examples

### Example 1 — Connectivity Check Playbook

This simple playbook tests whether the Master Node can reach all managed nodes using the `ping` module.

```bash
vi playbook1.yml
```

```yaml
---

- name: Playbook to check the connection between the servers
  hosts: all
  tasks:
    - name: Testing the connectivity
      ping:

...
```

---

### Example 2 — Install Apache Web Server

This playbook installs Apache (`httpd`) on all managed nodes and prints a confirmation message using the `debug` module.

```bash
vi playbook2.yml
```

```yaml
---

- name: installation of apache
  hosts: all
  become: yes
  tasks:
    - name: installation of apache webserver
      yum:
        name: httpd
        state: present

    - name: print a message
      debug:
        msg: "Apache is successfully installed"

...
```

> **`become: yes`** — This tells Ansible to run the tasks with elevated (root) privileges, equivalent to `sudo`.  
> **`debug` module** — Used to print custom messages during playbook execution. Helpful for logging status or variable values.

---

## 7. Useful Playbook Commands Reference

Use these commands when working with any playbook file. Replace `playbook1.yml` / `playbook2.yml` with your actual filename.

### Syntax Check — Validate YAML before running

```bash
ansible-playbook playbook1.yml --syntax-check
ansible-playbook playbook2.yml --syntax-check
```

### Connectivity Check — Verify all hosts are reachable

```bash
ansible all -m ping
```

### Dry Run — Simulate execution without making changes

```bash
ansible-playbook playbook1.yml --check
ansible-playbook playbook2.yml --check
```

### Execute the Playbook — Run it for real

```bash
ansible-playbook playbook1.yml
ansible-playbook playbook2.yml

# Or specify inventory file explicitly
ansible-playbook -i hosts playbook1.yml
ansible-playbook -i hosts playbook2.yml
```

### Verbose Output — Useful for debugging

```bash
# Level 1 verbosity
ansible-playbook playbook1.yml -v

# Level 3 verbosity (very detailed)
ansible-playbook playbook1.yml -vvv
```

### List Tasks — Preview tasks before execution

```bash
ansible-playbook playbook1.yml --list-tasks
ansible-playbook playbook2.yml --list-tasks
```

---

## Quick Reference Summary

| Concept | Command / Detail |
|--------|-----------------|
| Test all nodes | `ansible all -m ping` |
| List all hosts | `ansible all --list-hosts` |
| Run shell command | `ansible all -a "<command>"` |
| Use a module | `ansible all -m <module> -a "key=value"` |
| Run a playbook | `ansible-playbook <file>.yml` |
| Syntax check | `ansible-playbook <file>.yml --syntax-check` |
| Dry run | `ansible-playbook <file>.yml --check` |
| Debug output | `ansible-playbook <file>.yml -vvv` |

---

*Ansible Documentation: [https://docs.ansible.com](https://docs.ansible.com)*
