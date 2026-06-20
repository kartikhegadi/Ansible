# Ansible 🚀

My personal notes and hands-on practice from a 3-day Ansible learning sprint — covering everything from core concepts and architecture to playbooks, variables, conditionals, roles, vault, and async execution.

> 📌 These notes are written for self-reference but should be useful to anyone learning Ansible from scratch.

---

## 📂 Repository Structure

| File | Topic |
|------|-------|
| [`ansible-notes-D1.md`](./ansible-notes-D1.md) | Fundamentals, architecture, installation, inventory, ad-hoc commands, modules, first playbooks |
| [`ansible-notes-D2.md`](./ansible-notes-D2.md) | Apache/Nginx playbooks, facts, tags, variables, loops, notify & handlers |
| [`ansible-notes-D3.md`](./ansible-notes-D3.md) | Conditionals, LAMP stack, lookup module, roles, Ansible Vault, async/polling |

---

## 🗒️ Day 1 — Fundamentals & First Steps

- DevOps toolchain context: where Ansible fits alongside Jenkins, Docker, Kubernetes, and Terraform
- What Ansible is, and why it's **agentless** (works over SSH, no agent software needed on managed nodes)
- Ansible architecture: Master Node vs Managed Nodes, `host inventory file`, `ansible.cfg`
- Installing Ansible on the master node (`pip install ansible`)
- Setting up passwordless SSH access (`ssh-copy-id`) between master and managed nodes
- Creating the inventory file (`/etc/ansible/hosts`) and config file (`/etc/ansible/ansible.cfg`)
- Verifying connectivity with `ansible all -m ping`
- **Three ways to work with Ansible:**
  - Ad-hoc commands (`ansible all -a "..."`)
  - Modules (`yum`, `service`, `user`, `copy`, etc.)
  - Playbooks (YAML-based automation)
- Writing and running first playbooks (connectivity check, Apache installation)
- Useful playbook commands: `--syntax-check`, `--check` (dry run), `-v` / `-vvv` (verbose), `--list-tasks`

📄 Full notes: [`ansible-notes-D1.md`](./ansible-notes-D1.md)

---

## 🗒️ Day 2 — Playbooks, Variables, Loops & Handlers

- Full Apache and Nginx installation/configuration playbooks (install → start/enable → copy files → debug confirmation)
- Using `sed -i` to flip a playbook between install (`present`) and uninstall (`absent`) modes
- **Setup module** — automatic fact-gathering from managed nodes (memory, CPU, OS family, etc.)
- **Debug module** combined with `ansible_facts` to print live system information
- **Tags** — running or skipping specific tasks with `--tags` / `--skip-tags`
- **Variables:**
  - Static variables (defined in the `vars` block)
  - Dynamic variables passed at runtime via `--extra-vars`
- **Loops** — iterating over a list of packages/items with `loop` (preferred over the older `with_items`)
- Ad-hoc command modules inside playbooks: `shell`, `command`, and `raw`
- **Notify & Handlers** — triggering a service restart only when a task actually makes a change, running once at the end of the play

📄 Full notes: [`ansible-notes-D2.md`](./ansible-notes-D2.md)

---

## 🗒️ Day 3 — Conditionals, LAMP Stack, Roles, Vault & Async

- Re-cap of Ansible installation, `ansible.cfg`, and inventory setup
- **Homogenous vs Heterogenous** server environments
- **Conditionals (`when`)** — targeting tasks to the correct OS family (`ansible_os_family == "RedHat"` vs `"Debian"`)
- **LAMP stack deployment** — Apache + MariaDB + PHP on Amazon Linux 2023, with a sample PHP page to verify the stack
- **Lookup module** — reading external file contents into a variable (`lookup('file', path)`)
- Management UIs: Ansible Tower / AWX, Docker Portainer
- **Ansible Strategies** — `linear` (lockstep, default) vs `free` (independent host execution)
- **Ansible Roles** — breaking large playbooks into reusable, structured components (`packages`, `users`, `webservers`)
- **Ansible Galaxy** — community hub for sharing/downloading roles
- **Ansible Vault** — encrypting YAML files to protect sensitive data
- **Async & Polling** — running long tasks in the background (`async`, `poll`) without blocking the whole play

📄 Full notes: [`ansible-notes-D3.md`](./ansible-notes-D3.md)

---

## ⚡ Quick Command Reference

```bash
# Test connectivity to all managed nodes
ansible all -m ping

# List all hosts in inventory
ansible all --list-hosts

# Run an ad-hoc shell command
ansible all -a "<command>"

# Run a module against all hosts
ansible all -m <module> -a "key=value"

# Run a playbook
ansible-playbook <file>.yml

# Validate playbook syntax
ansible-playbook <file>.yml --syntax-check

# Dry run (no changes made)
ansible-playbook <file>.yml --check

# Verbose output for debugging
ansible-playbook <file>.yml -vvv

# Run only specific tagged tasks
ansible-playbook <file>.yml --tags <tagname>

# Pass a variable at runtime
ansible-playbook <file>.yml --extra-vars "var=value"
```

---

## 🛠️ Tech Covered

`Ansible` · `YAML` · `SSH` · `Linux (Amazon Linux / RHEL)` · `Apache` · `Nginx` · `MariaDB` · `PHP` · `Docker` · `Git` · `Ansible Vault` · `Ansible Roles`

---

## 📚 Reference

- [Official Ansible Documentation](https://docs.ansible.com)

---

⭐ Feel free to explore the day-by-day notes linked above — each one is a self-contained, detailed walkthrough with commands, YAML examples, and explanations.

---

**Kartik Hegadi**
InfraCorps
