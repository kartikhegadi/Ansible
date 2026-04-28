# Ansible — Day 02 Reference Guide

> **Playbooks, Modules, Variables, Loops, Tags, Notify & Handlers**

---

## Table of Contents

1. [Playbook: Apache Installation & Configuration](#1-playbook-apache-installation--configuration)
2. [Playbook: Nginx Installation & Configuration](#2-playbook-nginx-installation--configuration)
3. [Setup Module](#3-setup-module)
4. [Debug Module with Facts](#4-debug-module-with-facts)
5. [Ansible Tags](#5-ansible-tags)
6. [Variables](#6-variables)
   - [Static Variables](#61-static-variables)
   - [Dynamic Variables (Extra Vars)](#62-dynamic-variables-extra-vars)
7. [Loops](#7-loops)
8. [Ad-hoc Command Modules in Playbooks](#8-ad-hoc-command-modules-in-playbooks)
9. [Notify & Handlers](#9-notify--handlers)

---

## 1. Playbook: Apache Installation & Configuration

This playbook performs a full Apache (`httpd`) setup — installing the web server, starting and enabling the service, installing additional tools like Git and Docker, copying a file to a managed node, and creating a user. Each task is followed by a `debug` message to confirm successful execution.

```bash
vi l1.yaml
```

```yaml
---
- name: Apache Installation and Configuration Playbook
  hosts: all
  become: yes
  tasks:
    - name: Install Apache Web Server
      yum:
        name: httpd
        state: present

    - name: Print Apache Installation Success Message
      debug:
        msg: "Apache Web Server has been installed successfully."

    - name: Start and Enable Apache Service
      service:
        name: httpd
        state: started
        enabled: yes

    - name: Installing git
      yum:
        name: git
        state: present

    - name: Print Git Installation Success Message
      debug:
        msg: "Git has been installed successfully."

    - name: Installation Docker
      yum:
        name: docker
        state: present

    - name: Print Docker Installation Success Message
      debug:
        msg: "Docker has been installed successfully."

    - name: Start and Enable Docker Service
      service:
        name: docker
        state: started
        enabled: yes

    - name: Copying the files
      copy:
        src: kartik.txt
        dest: /home/ec2-user/

    - name: Print File Copy Success Message
      debug:
        msg: "File has been copied successfully to /home/ec2-user/"

    - name: Creating user "kartik"
      user:
        name: kartik
        state: present
        shell: /bin/bash
...
```

### Run & Utility Commands

```bash
# Execute the playbook
ansible-playbook l1.yaml

# Replace all 'present' with 'absent' to uninstall everything
sed -i "s/present/absent/g" 01.yaml

# Verify the change
cat 01.yaml
```

> **`sed -i "s/present/absent/g"`** — This is a powerful Linux command that performs an in-place find-and-replace inside the YAML file. It's a quick way to flip a playbook from installation mode to uninstallation mode without manually editing each task.

---

## 2. Playbook: Nginx Installation & Configuration

This playbook installs the Nginx web server, starts and enables the service, copies a custom `index.html` file to Nginx's default serving directory, and restarts the service to apply the changes.

> **Note:** After running this playbook, open port **80** in your server's security group (AWS) or firewall rules so users can access the page from a browser.

```bash
vi l2.yaml
```

```yaml
---
- name: Nginx Installation and Configuration Playbook
  hosts: all
  become: yes
  tasks:
    - name: Install Nginx Web Server
      yum:
        name: nginx
        state: present

    - name: Print Nginx Installation Success Message
      debug:
        msg: "Nginx Web Server has been installed successfully."

    - name: Start and Enable Nginx Service
      service:
        name: nginx
        state: started
        enabled: yes

    - name: Copying the files to nginx default directory
      copy:
        src: index.html
        dest: /usr/share/nginx/html/

    - name: Print File Copy Success Message
      debug:
        msg: "File has been copied successfully to /usr/share/nginx/html/"

    - name: Restart Nginx Service
      service:
        name: nginx
        state: restarted
...
```

### Run Commands

```bash
ansible-playbook l2.yaml
cat 02.yaml
```

---

## 3. Setup Module

The **Setup Module** is a built-in Ansible module that automatically collects detailed system information (called **facts**) from all managed nodes. This includes details about memory, CPU, OS, network interfaces, and much more.

Ansible runs the Setup module automatically at the start of every playbook (this is called **fact gathering**). You can also run it manually using ad-hoc commands.

```bash
# Show all system facts from all managed nodes
ansible all -m setup

# Filter facts by memory-related info
ansible all -m setup | grep -i mem

# Filter facts by CPU-related info
ansible all -m setup | grep -i cpu

# Filter facts by OS family (e.g., RedHat, Debian)
ansible all -m setup | grep -i family
```

---

## 4. Debug Module with Facts

The **Debug Module** is used to print messages or variable values during playbook execution. When combined with Ansible Facts (collected by the Setup module), it becomes a powerful tool to display live system information.

The `gather_facts: yes` directive tells Ansible to collect system facts before running tasks. All collected facts are accessible via the `ansible_facts` dictionary.

```bash
vi 03.yaml
```

```yaml
---
- name: Printing System Information
  hosts: all
  become: yes
  gather_facts: yes

  tasks:

    - name: Print All System Information
      debug:
        msg: "System Information: {{ ansible_facts }}"

    - name: Print Hostname
      debug:
        msg: "Hostname: {{ ansible_facts['hostname'] }}"

    - name: Print IP Address
      debug:
        msg: "IP Address: {{ ansible_facts['default_ipv4']['address'] }}"

    - name: Print Operating System
      debug:
        msg: "Operating System: {{ ansible_facts['os_family'] }}"
...
```

### Commonly Used Ansible Facts

| Fact Key | Description |
|----------|-------------|
| `ansible_facts['hostname']` | Server hostname |
| `ansible_facts['default_ipv4']['address']` | Primary IP address |
| `ansible_facts['os_family']` | OS family (e.g., RedHat, Debian) |
| `ansible_facts['processor_count']` | Number of CPUs |
| `ansible_facts['memtotal_mb']` | Total RAM in MB |
| `ansible_facts['pkg_mgr']` | Package manager (e.g., yum, apt) |

### Run Commands

```bash
ansible-playbook playbook.yml --syntax-check
ansible-playbook -i inventory playbook.yml
```

---

## 5. Ansible Tags

In large playbooks with many tasks, you often need to run only a specific subset of tasks — for example, just the user creation task or just the git installation — without executing the entire playbook. **Tags** solve this problem.

By assigning a tag to each task, you can selectively **run** or **skip** specific tasks at execution time using the `--tags` or `--skip-tags` flags.

```bash
vi 04.yaml
```

```yaml
---
- name: TAGS Playbook
  hosts: all
  become: yes

  tasks:

    - name: Installation of git
      yum:
        name: git
        state: present
      tags: git

    - name: Installation of maven
      yum:
        name: maven
        state: present
      tags: maven

    - name: Creation of user
      user:
        name: anamcara
        state: present
      tags: user
...
```

### Tags Commands

```bash
# Run only the task tagged 'user'
ansible-playbook 04.yaml --tags user

# Skip the 'user' and 'maven' tasks, run everything else
ansible-playbook 04.yaml --skip-tags "user,maven"
```

> **When to use Tags:** Tags are especially useful in CI/CD pipelines and large infrastructure playbooks where you want fine-grained control over which tasks execute on a given run — without splitting your playbook into multiple files.

---

## 6. Variables

A **variable** in Ansible is a named container that holds a value. Instead of hardcoding software names, paths, or settings repeatedly across tasks, you define them once as variables and reference them throughout the playbook. This makes playbooks cleaner, more reusable, and easier to maintain.

### Types of Variables

| Type | Description |
|------|-------------|
| **Static** | Defined directly inside the playbook using the `vars` section |
| **Dynamic** | Not defined inside the playbook; passed in at runtime using `--extra-vars` |

---

### 6.1 Static Variables

Static variables are declared in the `vars` block at the top of the play. They are referenced inside tasks using the `{{ variable_name }}` syntax (Jinja2 templating).

```bash
vi 05.yaml
```

```yaml
---
- hosts: all
  vars:
    a: git
    b: maven
    c: docker
    d: nginx
    e: httpd
  tasks:
    - name: git installation
      yum: name={{a}} state=present

    - name: maven uninstallation
      yum: name={{b}} state=absent

    - name: docker installation
      yum: name={{c}} state=present

    - name: nginx installation
      yum: name={{d}} state=present

    - name: httpd installation
      yum: name={{e}} state=present
...
```

```bash
ansible-playbook 05.yaml
```

---

### 6.2 Dynamic Variables (Extra Vars)

Dynamic variables are **not declared inside the playbook**. Instead, they are passed in at runtime from the command line using the `--extra-vars` flag. This is useful when the value of a variable changes between runs (e.g., deploying different versions, installing different packages).

In the example below, `top` is used inside the playbook but never declared in `vars`. If you run the playbook without passing it, Ansible will throw an error. You must supply it explicitly at runtime.

```bash
vi 06.yaml
```

```yaml
---
- hosts: all
  vars:
    a: git
    b: maven
    c: docker
    d: nginx
    e: httpd
  tasks:
    - name: git installation
      yum: name={{a}} state=present

    - name: maven uninstallation
      yum: name={{b}} state=absent

    - name: docker installation
      yum: name={{c}} state=present

    - name: nginx installation
      yum: name={{d}} state=present

    - name: httpd installation
      yum: name={{e}} state=present

    - name: top command installation
      yum: name={{top}} state=present
...
```

```bash
# Pass the undeclared variable 'top' at runtime
ansible-playbook 05.yaml --extra-vars "top=top"
```

> **Key Point:** `top` is not defined inside the `vars` block. It must be passed using `--extra-vars` at the command line. This pattern is very useful in pipelines where a CI/CD tool (like Jenkins) passes parameters dynamically into an Ansible playbook.

---

## 7. Loops

When you need to perform the same task on multiple items (e.g., install 5 packages), writing a separate task for each one is repetitive and inefficient. **Loops** allow you to iterate over a list of items and apply the same task to each one — with just a few lines of code.

### `loop` vs `with_items`

| Method | Status | Recommendation |
|--------|--------|----------------|
| `with_items` | Older syntax | Not recommended for new playbooks |
| `loop` | Modern syntax | **Recommended** — more flexible and readable |

Inside a loop, the current item is accessed using the special variable `{{ item }}`.

```bash
vi 07.yaml
```

```yaml
---
- name: loop example
  hosts: all
  become: yes
  gather_facts: False
  tasks:
    - name: Installation of multiple packages
      yum:
        name: "{{ item }}"
        state: present
      loop:
        - git
        - maven
        - docker
        - nginx
        - httpd
...
```

```bash
ansible-playbook 07.yaml
```

> **`state: present`** means "ensure this package is installed." If it is already installed, Ansible will skip it without making any changes — this is what makes Ansible **idempotent**.

---

## 8. Ad-hoc Command Modules in Playbooks

While Ansible modules like `yum` and `service` are the preferred way to write tasks, there are times when you need to run raw shell commands directly on managed nodes. Ansible provides three modules for this purpose, each with a different level of control and compatibility.

### The Three Command Modules

| Module | Description | Use Case |
|--------|-------------|----------|
| `shell` | Runs commands through the system shell (`/bin/sh`) | When you need shell features like pipes (`|`), redirection (`>`), or environment variables |
| `command` | Runs commands directly without invoking a shell | Safer and more predictable; use when shell features are not needed |
| `raw` | Sends commands over raw SSH without Python | Use on systems where Python is not installed (bare-metal bootstrap) |

> **Best Practice:** Prefer `command` over `shell` when possible, as it avoids shell injection risks. Use `raw` only when the managed node has no Python interpreter available.

```bash
vi 08.yaml
```

```yaml
---
- name: playbook using ad-hoc commands
  hosts: all
  tasks:
    - name: git installation
      shell: yum install git -y

    - name: maven install
      command: yum install maven -y

    - name: docker install
      raw: yum install docker -y
...
```

```bash
ansible-playbook 08.yaml
```

---

## 9. Notify & Handlers

### What are Handlers?

A **Handler** is a special type of task in Ansible that only runs when it is explicitly **notified** by another task. Handlers are defined separately from regular tasks and are typically used for actions that should only happen when something actually changes — such as restarting a service after its configuration file is updated.

### What is Notify?

**`notify`** is a directive you add to a regular task. When that task makes a change (e.g., a config file is created or modified), it sends a notification to the named handler. The handler is then queued to run at the **end of the play**, after all regular tasks have completed.

### Why Use Handlers?

Consider this scenario: you have 10 tasks in a playbook, and 3 of them might potentially trigger a service restart. Without handlers, you would either restart the service after every task (unnecessary restarts) or hard-code restart tasks in specific places (fragile and hard to maintain). With handlers, the service restarts **once at the end** — and only if at least one notifying task actually made a change.

### Execution Flow

```
Task 1 → Makes a change → Notifies handler (queued, not run yet)
Task 2 → Executes
Task 3 → Executes
Task 4 → Executes (debug message)
--- All regular tasks complete ---
Handler → Runs ONCE at the very end
```

> **Key Rule:** Even if multiple tasks notify the same handler, the handler runs **only once** at the end of the play. This prevents duplicate restarts.

```bash
vi 09.yaml
```

```yaml
---
- name: Demo of notify and handlers
  hosts: all
  become: yes

  tasks:
    - name: TASK 1 - Create application config file
      copy:
        dest: /tmp/app.conf
        content: |
          APP_NAME=DevOpsDemo
          VERSION=1.0
      notify: Restart application service   # Notifies the handler if this task changes anything

    - name: TASK 2 - Install nginx
      dnf:
        name: nginx
        state: present

    - name: TASK 3 - Ensure nginx is running
      service:
        name: nginx
        state: started
        enabled: yes

    - name: TASK 4 - Print completion message
      debug:
        msg: "All main tasks executed successfully"

  handlers:
    - name: Restart application service     # This handler runs ONLY if notified
      service:
        name: nginx
        state: restarted
...
```

```bash
ansible-playbook 09.yaml
```

### Notify & Handler Execution Flow Explained

| Step | What Happens |
|------|-------------|
| **Task 1 runs** | Creates `/tmp/app.conf`. Since the file is new (a change occurred), the handler is **notified and queued** |
| **Task 2 runs** | Installs Nginx normally |
| **Task 3 runs** | Starts and enables Nginx |
| **Task 4 runs** | Prints the debug message |
| **Handler runs** | After all tasks complete, `Restart application service` executes and restarts Nginx |

> **Important:** If Task 1 is run again and the file content has **not changed**, Ansible will report `ok` (no change), the handler will **not** be notified, and Nginx will **not** be restarted. This is Ansible's idempotent behavior working as intended.

---

## Quick Reference Summary

| Concept | Key Command / Directive |
|---------|------------------------|
| Run a playbook | `ansible-playbook <file>.yml` |
| Collect system facts | `ansible all -m setup` |
| Filter facts | `ansible all -m setup \| grep -i mem` |
| Run only specific tags | `ansible-playbook <file>.yml --tags <tagname>` |
| Skip specific tags | `ansible-playbook <file>.yml --skip-tags "<tag1,tag2>"` |
| Pass dynamic variable | `ansible-playbook <file>.yml --extra-vars "var=value"` |
| Loop over items | `loop:` with `{{ item }}` |
| Notify a handler | `notify: <Handler Name>` inside a task |
| Define a handler | Under `handlers:` section at play level |

---

*Ansible Documentation: [https://docs.ansible.com](https://docs.ansible.com)*
