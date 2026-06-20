# Ansible — Day 03 Reference Guide

> **Installation, Inventory, Conditionals, LAMP Stack, Lookup, Roles, Vault & Async/Polling**

---

## Table of Contents

1. [Ansible Installation on Master Node](#1-ansible-installation-on-master-node)
2. [Ansible Configuration File](#2-ansible-configuration-file)
3. [Inventory File Setup](#3-inventory-file-setup)
4. [Server Types: Homogenous vs Heterogenous](#4-server-types-homogenous-vs-heterogenous)
5. [Conditionals — The `when` Module](#5-conditionals--the-when-module)
6. [LAMP Stack Deployment](#6-lamp-stack-deployment)
7. [Lookup Module](#7-lookup-module)
8. [Management UIs](#8-management-uis)
9. [Ansible Strategies](#9-ansible-strategies)
10. [Ansible Roles](#10-ansible-roles)
11. [Ansible Galaxy](#11-ansible-galaxy)
12. [Ansible Vault](#12-ansible-vault)
13. [Asynchronous Execution & Polling](#13-asynchronous-execution--polling)

---

## 1. Ansible Installation on Master Node

Before Ansible can manage any nodes, it must be installed on the **control/master node**. Ansible is a Python-based tool, so `pip` is used to install it.

```bash
# Install Python's package manager
sudo dnf install python3-pip -y

# Install Ansible via pip
pip install ansible

# Verify the installation and check the version
ansible --version
```

> **Note:** Running `ansible --version` also confirms the location of the default config file and Python interpreter being used — useful for troubleshooting later.

---

## 2. Ansible Configuration File

Ansible's behavior (inventory location, SSH settings, etc.) is controlled by a central configuration file, conventionally stored at `/etc/ansible/ansible.cfg`.

### Step 1 — Create the `/etc/ansible/` directory

```bash
sudo mkdir -p /etc/ansible
```

### Step 2 — Create the config file

```bash
sudo nano /etc/ansible/ansible.cfg
```

Paste the following minimal configuration:

```ini
[defaults]
inventory=/etc/ansible/hosts
host_key_checking=False
```

Save & exit.

| Directive | Purpose |
|-----------|---------|
| `inventory=/etc/ansible/hosts` | Tells Ansible where to find the default inventory file |
| `host_key_checking=False` | Disables SSH host key prompts, useful for automation/testing environments |

---

## 3. Inventory File Setup

The **inventory file** lists all the managed nodes (servers) Ansible will control, optionally organized into groups.

### Step 3 — Create the hosts inventory file

```bash
sudo nano /etc/ansible/hosts
```

Add your hosts, grouped under named sections in square brackets:

```ini
[dev-servers]
your-ec2-private-ip-here

[test-servers]
your-ec2-private-ip-here
```

Save & exit.

### Verifying Connectivity

```bash
# Ping all hosts in the inventory — a healthy response returns "pong"
ansible all -m ping

# List all hosts Ansible currently recognizes
ansible all --list-hosts

# View the full inventory structure
ansible-inventory --list
```

> You should see a `"pong"` response from each worker node if connectivity and authentication are correctly configured.

### Example Grouped Inventory

```ini
[dev-server]
172.31.10.218

[test-server]
172.31.12.98
```

---

## 4. Server Types: Homogenous vs Heterogenous

When managing infrastructure, servers generally fall into one of two categories based on their OS and configuration:

| Server Type | Description |
|--------------|-------------|
| **Homogenous Servers** | Servers with the same OS and flavour (e.g., all RedHat-based) |
| **Heterogenous Servers** | Servers with different OS and flavour (e.g., a mix of RedHat and Debian-based) |

> This distinction matters because tasks like package installation use **different modules** depending on the OS family — which is exactly the problem the `when` conditional (Section 5) solves.

---

## 5. Conditionals — The `when` Module

In a **homogenous** environment, a single module works fine for every host:

```yaml
- hosts: all
  tasks:
    - name: git installation on AL
      yum: name=git state=present
    - name: git installation on Ubuntu
      apt: name=git state=present
```

> The playbook above runs **both** tasks on **every** host regardless of OS — `yum` will simply fail on Ubuntu and `apt` will fail on Amazon Linux. This is inefficient and throws unnecessary errors.

In a **heterogenous** environment, the `when` conditional lets you target tasks to the correct OS family using Ansible facts, so only the relevant task executes on each host.

```bash
vi when_playbook.yaml
```

```yaml
- hosts: all
  tasks:
    - name: git installation on RedHat
      yum: name=git state=present
      when: ansible_os_family == "RedHat"

    - name: git installation on Debian
      apt: name=git state=present
      when: ansible_os_family == "Debian"
```

### Run Command

```bash
ansible-playbook when_playbook.yaml
```

> **Key Point:** `ansible_os_family` is a fact automatically gathered by Ansible (via the Setup module covered in Day 02). The `when` clause evaluates this fact on **each host individually**, so the right task runs on the right OS — making the playbook safe to run across heterogenous infrastructure.

---

## 6. LAMP Stack Deployment

**LAMP** is a classic web application stack made up of four components:

| Letter | Component | Role |
|--------|-----------|------|
| **L** | Linux | Operating System |
| **A** | Apache | Web server |
| **M** | MySQL / MariaDB | Database |
| **P** | PHP | Server-side scripting language |

### Task: Setup LAMP Stack with a Custom PHP Page

This playbook installs and configures the full LAMP stack on Amazon Linux 2023, then deploys a sample PHP page to confirm everything works end-to-end.

```bash
vi lamp.yaml
```

```yaml
- name: LAMP Stack Setup on Amazon Linux 2023
  hosts: lamp
  become: yes
  tasks:
    - name: Update all packages
      dnf:
        name: "*"
        state: latest

    - name: Install Apache
      package:
        name: httpd
        state: present

    - name: Start and enable Apache
      service:
        name: httpd
        state: started
        enabled: yes

    - name: Install MariaDB 10.5 Server
      package:
        name: mariadb105-server
        state: present

    - name: Start and enable MariaDB
      service:
        name: mariadb
        state: started
        enabled: yes

    - name: Install PHP and extensions
      package:
        name:
          - php
          - php-mysqlnd
        state: present

    - name: Restart Apache after PHP install
      service:
        name: httpd
        state: restarted

    - name: Deploy sample PHP application
      copy:
        dest: /var/www/html/index.php
        owner: apache
        group: apache
        mode: '0644'
        content: |
          <?php
          echo "<h1>LAMP Stack Working on Amazon Linux 2023</h1>";
          echo "<p>MariaDB + Apache + PHP</p>";
          ?>
```

### Run Command

```bash
ansible-playbook lamp.yaml
```

> **Why restart Apache after installing PHP?** Apache needs to reload its modules so it recognizes the newly installed PHP handler — without the restart, `.php` files would be served as plain text or fail to render.

> **Reference walkthrough:** [ChatGPT conversation link](https://chatgpt.com/share/6982119a-26d4-8002-83e0-2018a6b6776e) (shared LAMP setup discussion).

---

## 7. Lookup Module

The **Lookup Module** allows a playbook to **read external data** — such as the contents of a file on the control node — and inject it into a variable for use within the play.

> **Use case:** Suppose you have a file containing some data (credentials, configuration values, etc.) and you want your playbook to read it. This is exactly what the `lookup` module is for.

```bash
vi lookup.yaml
```

```yaml
- name: lookup playbook
  hosts: all
  vars:
    credentials: "{{ lookup('file', '/home/ec2-user/day3/kartik.txt') }}"
  tasks:
    - debug:
        msg: My Credentials Are {{ credentials }}
```

### Run Command

```bash
ansible-playbook lookup.yaml
```

| Element | Description |
|---------|--------------|
| `lookup('file', path)` | Reads the contents of the specified file on the **control node** |
| `credentials` | A variable holding the file's contents, available for use anywhere in the play |

> **Caution:** Since the file contents are printed via `debug`, avoid using this pattern on files containing sensitive secrets in production — consider **Ansible Vault** (Section 12) for secret management instead.

---

## 8. Management UIs

While Ansible and Docker are primarily CLI-driven, both have popular web-based UIs for visual management:

| Tool | UI Name |
|------|---------|
| **Docker** | Docker Portainer |
| **Ansible** | Ansible Tower (also known as AWX in its open-source form) |

---

## 9. Ansible Strategies

A **strategy** controls the **order and concurrency** in which Ansible executes tasks across the hosts in a play.

| Strategy | Behavior |
|----------|----------|
| `linear` (default) | All hosts run the **same task** before moving to the next task, in lockstep |
| `free` | Each host runs through **all tasks** independently, as fast as it can, without waiting for other hosts |

### Example: Using the `free` Strategy

```bash
vi strategy.yaml
```

```yaml
- hosts: all
  strategy: free
  tasks:
    - name: git installation on RedHat
      yum: name=git state=present
      when: ansible_os_family == "RedHat"

    - name: git installation on Debian
      apt: name=git state=present
      when: ansible_os_family == "Debian"
```

### Run Command

```bash
ansible-playbook strategy.yaml
```

> **When to use `free`:** Useful when hosts vary significantly in speed or workload, and you don't need tasks executed in strict lockstep across all of them — for example, large fleets where a few slow hosts shouldn't bottleneck the rest.

---

## 10. Ansible Roles

So far, multiple modules/tasks have been written into a **single YAML file**. As playbooks grow, calling or reusing one specific piece of functionality becomes difficult to manage.

**Ansible Roles** solve this by letting you split a playbook into **separate, reusable playbooks** — one per responsibility — which can then be called individually or together as needed.

> Ansible roles help organize and structure playbooks by breaking them into reusable components. Roles divide a playbook into a standardized **directory structure**, commonly including folders such as:

```
packages/
users/
webservers/
```

> **Benefit:** Each role (e.g., `webservers`) is self-contained and can be reused across multiple projects or playbooks — instead of copy-pasting the same tasks everywhere.

---

## 11. Ansible Galaxy

*(Reserved for future notes — Ansible Galaxy is Ansible's official hub for sharing and downloading community-built roles.)*

---

## 12. Ansible Vault

**Ansible Vault** is used to **encrypt** YAML files, allowing sensitive data (passwords, API keys, credentials) to be stored securely within version-controlled playbooks.

> Ansible Vault encrypts your `.yml` files so they can be stored and shared in a secured way — without exposing plaintext secrets.

---

## 13. Asynchronous Execution & Polling

By default, Ansible runs tasks **synchronously** — it waits for each task to finish before moving to the next, and if a task doesn't complete within its allotted time, the playbook execution stops.

**Asynchronous mode** changes this: a task is allowed to run in the background for up to a defined time limit (`async`), while Ansible periodically checks back on its progress (`poll`).

```bash
vi async.yaml
```

```yaml
- name: async and polling playbook
  hosts: all
  become: yes
  ignore_errors: yes
  tasks:
    - name: install git
      yum:
        name: git
        state: present
        async: 20
        poll: 10
```

### Run Command

```bash
ansible-playbook async.yaml
```

| Directive | Description |
|-----------|--------------|
| `async: 20` | Maximum time (in seconds) the task is allowed to run before being considered timed out |
| `poll: 10` | Interval (in seconds) at which Ansible checks back on the task's status |
| `ignore_errors: yes` | Tells Ansible to continue running the rest of the playbook even if this task fails/errors out |

> **Key Point:** For every task, you can set a time limit. If the task isn't completed within that limit, Ansible normally stops the playbook execution — **async + poll** is the mechanism that allows long-running tasks to execute in the background instead of blocking the whole play.

---

## Quick Reference Summary

| Concept | Key Command / Directive |
|---------|--------------------------|
| Install Ansible | `pip install ansible` |
| Check Ansible version | `ansible --version` |
| Default config file | `/etc/ansible/ansible.cfg` |
| Default inventory file | `/etc/ansible/hosts` |
| Test connectivity | `ansible all -m ping` |
| List inventory hosts | `ansible all --list-hosts` |
| View full inventory | `ansible-inventory --list` |
| Conditional execution | `when: ansible_os_family == "RedHat"` |
| Read external file into a variable | `lookup('file', '/path/to/file')` |
| Set execution strategy | `strategy: free` or `strategy: linear` |
| Reusable playbook components | Ansible **Roles** |
| Share/download community roles | Ansible **Galaxy** |
| Encrypt sensitive YAML files | Ansible **Vault** |
| Run task in background with timeout | `async: <seconds>` |
| Polling interval for async task | `poll: <seconds>` |
| Continue despite task failure | `ignore_errors: yes` |

---

*Ansible Documentation: [https://docs.ansible.com](https://docs.ansible.com)*
