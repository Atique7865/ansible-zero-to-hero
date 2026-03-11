# Ansible Real-Time Setup Guide

A step-by-step guide to set up Ansible with a master node and worker nodes using a dedicated `devops` user with SSH key-based authentication.

---

## Architecture Overview

```
+------------------+        SSH (Key-Based)       +------------------+
|   Master Node    | ---------------------------> |   Worker Node 1  |
|  (Control Node)  |                              +------------------+
|  user: devops    | ---------------------------> +------------------+
+------------------+                              |   Worker Node 2  |
                                                  +------------------+
```

---

## Prerequisites


- 1 Master Node (Control Node) — Linux (Ubuntu/CentOS/RHEL)
- 1 or more Worker Nodes (Managed Nodes) — Linux
- Root or sudo access on all nodes
- All nodes should be able to reach each other over the network

---

## Step 1: Create `devops` User on Master Node

Run the following commands on the **Master Node** as root or with sudo:

```bash
# Create the devops user
sudo useradd -m -s /bin/bash devops

# Set a password for the devops user
sudo passwd devops

# Grant sudo privileges to devops user
sudo usermod -aG sudo devops        # For Ubuntu/Debian
# OR
sudo usermod -aG wheel devops       # For CentOS/RHEL/Fedora
```

### Enable passwordless sudo for devops (Recommended for Ansible)

```bash
# Open the sudoers file safely
sudo visudo
```

Add the following line at the end of the file:

```
devops  ALL=(ALL) NOPASSWD:ALL
```

> **Tip:** Alternatively, create a dedicated sudoers drop-in file:
> ```bash
> echo "devops ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/devops
> sudo chmod 440 /etc/sudoers.d/devops
> ```

---

## Step 2: Create `devops` User on Worker Node(s)

Repeat the **same steps** on **each Worker Node**:

```bash
# Create the devops user
sudo useradd -m -s /bin/bash devops

# Set a password
sudo passwd devops

# Grant sudo privileges
sudo usermod -aG sudo devops        # Ubuntu/Debian
# OR
sudo usermod -aG wheel devops       # CentOS/RHEL/Fedora

# Enable passwordless sudo
echo "devops ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/devops
sudo chmod 440 /etc/sudoers.d/devops
```

---

## Step 3: Generate SSH Key on Master Node

Switch to the `devops` user on the **Master Node** and generate an SSH key pair:

```bash
# Switch to devops user
su - devops

# Generate SSH key pair (press Enter for all prompts to use defaults)
ssh-keygen -t rsa -b 4096 -C "devops@master"
```

**Expected output:**

```
Generating public/private rsa key pair.
Enter file in which to save the key (/home/devops/.ssh/id_rsa):   <-- Press Enter
Enter passphrase (empty for no passphrase):                        <-- Press Enter
Enter same passphrase again:                                       <-- Press Enter
Your identification has been saved in /home/devops/.ssh/id_rsa
Your public key has been saved in /home/devops/.ssh/id_rsa.pub
```

### Verify the keys were created

```bash
ls -la /home/devops/.ssh/
# You should see:
#   id_rsa        (private key — keep this secret!)
#   id_rsa.pub    (public key — this gets copied to workers)
```

---

## Step 4: Copy Public Key to Worker Node(s)

Still as the `devops` user on the **Master Node**, use `ssh-copy-id` to push the public key to each worker:

```bash
# Copy public key to Worker Node 1
ssh-copy-id devops@<worker-node-1-IP>

# Copy public key to Worker Node 2
ssh-copy-id devops@<worker-node-2-IP>
```

**Example:**

```bash
ssh-copy-id devops@192.168.1.101
ssh-copy-id devops@192.168.1.102
```

You will be prompted for the `devops` user's password on the worker node (this is a one-time step).

### Verify passwordless SSH login

```bash
# Test SSH connection — should NOT ask for a password
ssh devops@192.168.1.101
ssh devops@192.168.1.102
```

---

## Step 5: Install Ansible on Master Node

```bash
# Switch back to a sudo-capable shell or use devops with sudo
sudo apt update && sudo apt install -y ansible       # Ubuntu/Debian
# OR
sudo yum install -y epel-release && sudo yum install -y ansible   # CentOS/RHEL
```

Verify installation:

```bash
ansible --version
```

---

## Step 6: Configure Ansible Inventory

Create an inventory file on the **Master Node**:

```bash
sudo mkdir -p /etc/ansible
sudo nano /etc/ansible/hosts
```

Add your worker nodes:

```ini
[workers]
worker1 ansible_host=192.168.1.101 ansible_user=devops
worker2 ansible_host=192.168.1.102 ansible_user=devops

[workers:vars]
ansible_ssh_private_key_file=/home/devops/.ssh/id_rsa
ansible_become=yes
ansible_become_method=sudo
```

---

## Step 7: Test Ansible Connectivity

```bash
# Run a ping test to all worker nodes
ansible -i /etc/ansible/hosts workers -m ping
```

**Expected output:**

```
worker1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
worker2 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

---

## Quick Reference — All Commands at a Glance

### Master Node Setup
```bash
sudo useradd -m -s /bin/bash devops
sudo passwd devops
sudo usermod -aG sudo devops
echo "devops ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/devops
sudo chmod 440 /etc/sudoers.d/devops
su - devops
ssh-keygen -t rsa -b 4096 -C "devops@master"
```

### Worker Node Setup (repeat per worker)
```bash
sudo useradd -m -s /bin/bash devops
sudo passwd devops
sudo usermod -aG sudo devops
echo "devops ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/devops
sudo chmod 440 /etc/sudoers.d/devops
```

### Copy SSH Key from Master to Workers
```bash
ssh-copy-id devops@<worker-IP>
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `Permission denied (publickey)` | Ensure `ssh-copy-id` completed successfully; check `/home/devops/.ssh/authorized_keys` on worker |
| `sudo: requires a password` | Re-check `/etc/sudoers.d/devops` file on the worker node |
| Ansible ping fails | Verify SSH connectivity manually with `ssh devops@<worker-IP>` |
| `Host key verification failed` | Run `ssh-keyscan <worker-IP> >> ~/.ssh/known_hosts` on master |

---

## Directory Structure

```
/home/devops/
└── .ssh/
    ├── id_rsa            # Private key (Master only)
    ├── id_rsa.pub        # Public key (Master only)
    └── authorized_keys   # Public key stored here on Workers
```

---

> **Security Note:** Never share or expose `id_rsa` (private key). Only the public key (`id_rsa.pub`) is copied to worker nodes.

---

## Step 8: Create a Playbook to Install and Configure Nginx

Create a file named `install_nginx.yml` on the **Master Node**:

```bash
nano install_nginx.yml
```

Add the following content:

```yaml
---
- name: Install and configure Nginx
  hosts: workers
  become: yes

  tasks:
    - name: Update apt cache (Ubuntu/Debian)
      apt:
        update_cache: yes
      when: ansible_os_family == "Debian"

    - name: Install Nginx
      package:
        name: nginx
        state: present

    - name: Start and enable Nginx service
      service:
        name: nginx
        state: started
        enabled: yes

    - name: Ensure Nginx is running
      uri:
        url: http://localhost
        status_code: 200
      register: result
      ignore_errors: yes
```

---

## Step 9: Run the Nginx Playbook

```bash
# Dry-run first (no changes made — recommended before first run)
ansible-playbook -i /etc/ansible/hosts install_nginx.yml --check

# Run the playbook
ansible-playbook -i /etc/ansible/hosts install_nginx.yml

# Run with verbose output to see task details
ansible-playbook -i /etc/ansible/hosts install_nginx.yml -v
```

**Expected output:**

```
PLAY [Install and configure Nginx] *********************************************

TASK [Update apt cache] ********************************************************
ok: [worker1]
ok: [worker2]

TASK [Install Nginx] ***********************************************************
changed: [worker1]
changed: [worker2]

TASK [Start and enable Nginx service] ******************************************
changed: [worker1]
changed: [worker2]

TASK [Ensure Nginx is running] *************************************************
ok: [worker1]
ok: [worker2]

PLAY RECAP *********************************************************************
worker1  : ok=4  changed=2  unreachable=0  failed=0
worker2  : ok=4  changed=2  unreachable=0  failed=0
```

### Useful Playbook Flags

| Flag | Purpose |
|------|---------|
| `-i` | Specify inventory file |
| `-v / -vv` | Verbose / extra verbose output |
| `--check` | Dry-run (no changes made) |
| `--limit worker1` | Run on a specific host only |
| `--tags install` | Run only tasks with a specific tag |

---

## Real-Life Ansible Shell Commands (Ad-Hoc)

Ad-hoc commands let you run one-off tasks on remote hosts without writing a playbook.  
**Syntax:** `ansible <host-pattern> -i <inventory> -m <module> -a "<args>"`

---

### 🔍 System Information

```bash
# Check uptime on all workers
ansible workers -i /etc/ansible/hosts -m command -a "uptime"

# Get OS details
ansible workers -i /etc/ansible/hosts -m command -a "cat /etc/os-release"

# Check free disk space
ansible workers -i /etc/ansible/hosts -m command -a "df -h"

# Check memory usage
ansible workers -i /etc/ansible/hosts -m command -a "free -m"

# List all running processes
ansible workers -i /etc/ansible/hosts -m command -a "ps aux"

# Check hostname of each worker
ansible workers -i /etc/ansible/hosts -m command -a "hostname"

# Gather full system facts (CPU, memory, IPs, OS, etc.)
ansible workers -i /etc/ansible/hosts -m setup

# Filter facts — only get IP addresses
ansible workers -i /etc/ansible/hosts -m setup -a "filter=ansible_default_ipv4"
```

---

### 📦 Package Management

```bash
# Install a package (apt — Ubuntu/Debian)
ansible workers -i /etc/ansible/hosts -m apt -a "name=curl state=present" --become

# Install a package (yum — CentOS/RHEL)
ansible workers -i /etc/ansible/hosts -m yum -a "name=curl state=present" --become

# Remove a package
ansible workers -i /etc/ansible/hosts -m apt -a "name=curl state=absent" --become

# Update all packages (Ubuntu/Debian)
ansible workers -i /etc/ansible/hosts -m apt -a "upgrade=dist update_cache=yes" --become

# Check if a package is installed
ansible workers -i /etc/ansible/hosts -m command -a "dpkg -l | grep nginx"
```

---

### 🔧 Service Management

```bash
# Start a service
ansible workers -i /etc/ansible/hosts -m service -a "name=nginx state=started" --become

# Stop a service
ansible workers -i /etc/ansible/hosts -m service -a "name=nginx state=stopped" --become

# Restart a service
ansible workers -i /etc/ansible/hosts -m service -a "name=nginx state=restarted" --become

# Enable a service to start on boot
ansible workers -i /etc/ansible/hosts -m service -a "name=nginx enabled=yes" --become

# Check service status
ansible workers -i /etc/ansible/hosts -m command -a "systemctl status nginx" --become
```

---

### 📁 File & Directory Operations

```bash
# Create a directory on all workers
ansible workers -i /etc/ansible/hosts -m file -a "path=/opt/myapp state=directory mode=0755" --become

# Create an empty file
ansible workers -i /etc/ansible/hosts -m file -a "path=/tmp/test.txt state=touch"

# Delete a file
ansible workers -i /etc/ansible/hosts -m file -a "path=/tmp/test.txt state=absent"

# Copy a local file to all workers
ansible workers -i /etc/ansible/hosts -m copy -a "src=/home/devops/app.conf dest=/etc/app.conf mode=0644" --become

# Fetch a file FROM workers to master
ansible workers -i /etc/ansible/hosts -m fetch -a "src=/var/log/nginx/error.log dest=/tmp/logs/ flat=no"

# Write content directly into a file on workers
ansible workers -i /etc/ansible/hosts -m copy -a "content='Hello from Ansible' dest=/tmp/hello.txt"
```

---

### 👤 User Management

```bash
# Create a new user
ansible workers -i /etc/ansible/hosts -m user -a "name=john state=present shell=/bin/bash" --become

# Delete a user
ansible workers -i /etc/ansible/hosts -m user -a "name=john state=absent remove=yes" --become

# Add a user to a group
ansible workers -i /etc/ansible/hosts -m user -a "name=john groups=sudo append=yes" --become

# Check existing users
ansible workers -i /etc/ansible/hosts -m command -a "cat /etc/passwd"
```

---

### 🔒 Security & Firewall

```bash
# Open port 80 with UFW (Ubuntu)
ansible workers -i /etc/ansible/hosts -m ufw -a "rule=allow port=80 proto=tcp" --become

# Check UFW status
ansible workers -i /etc/ansible/hosts -m command -a "ufw status" --become

# Set file permissions
ansible workers -i /etc/ansible/hosts -m file -a "path=/etc/app.conf mode=0600 owner=root group=root" --become

# Check open ports
ansible workers -i /etc/ansible/hosts -m command -a "ss -tlnp" --become
```

---

### 🐚 Shell Commands (for pipelines, redirects, env vars)

> Use `-m shell` when your command includes pipes `|`, redirects `>`, or environment variables `$`.

```bash
# Run a shell command with a pipe
ansible workers -i /etc/ansible/hosts -m shell -a "ps aux | grep nginx"

# Check disk usage of a specific directory
ansible workers -i /etc/ansible/hosts -m shell -a "du -sh /var/log/*"

# Tail the last 20 lines of a log file
ansible workers -i /etc/ansible/hosts -m shell -a "tail -n 20 /var/log/syslog" --become

# Run a script stored on the master node
ansible workers -i /etc/ansible/hosts -m script -a "/home/devops/scripts/setup.sh" --become

# Export an env variable and run a command
ansible workers -i /etc/ansible/hosts -m shell -a "export APP_ENV=production && echo \$APP_ENV"

# Reboot all worker nodes
ansible workers -i /etc/ansible/hosts -m reboot --become

# Check which user Ansible is connecting as
ansible workers -i /etc/ansible/hosts -m command -a "whoami"
```

---

### 📊 Quick Ad-Hoc Reference

| Task | Command |
|------|---------|
| Ping all hosts | `ansible all -i hosts -m ping` |
| Run uptime | `ansible all -i hosts -m command -a "uptime"` |
| Install nginx | `ansible workers -i hosts -m apt -a "name=nginx state=present" --become` |
| Restart nginx | `ansible workers -i hosts -m service -a "name=nginx state=restarted" --become` |
| Copy a file | `ansible workers -i hosts -m copy -a "src=app.conf dest=/etc/app.conf" --become` |
| Create a user | `ansible workers -i hosts -m user -a "name=john state=present" --become` |
| Get system facts | `ansible workers -i hosts -m setup` |
| Reboot nodes | `ansible workers -i hosts -m reboot --become` |
| Run shell script | `ansible workers -i hosts -m script -a "/path/to/script.sh" --become` |
| Check disk space | `ansible workers -i hosts -m command -a "df -h"` |

---

## Real-Life Ansible Playbook Examples — `vars` & `when` Conditions

---

### 🖥️ Example 1: Install a Web Server Based on OS Family

Uses `vars` to define the package name and `when` to branch by OS.

```yaml
---
- name: Install Web Server (OS-aware)
  hosts: workers
  become: yes
  vars:
    web_server_debian: nginx
    web_server_redhat: httpd

  tasks:
    - name: Install Nginx on Debian/Ubuntu
      apt:
        name: "{{ web_server_debian }}"
        state: present
        update_cache: yes
      when: ansible_os_family == "Debian"

    - name: Install Apache (httpd) on CentOS/RHEL
      yum:
        name: "{{ web_server_redhat }}"
        state: present
      when: ansible_os_family == "RedHat"

    - name: Start Nginx on Debian/Ubuntu
      service:
        name: "{{ web_server_debian }}"
        state: started
        enabled: yes
      when: ansible_os_family == "Debian"

    - name: Start Apache on CentOS/RHEL
      service:
        name: "{{ web_server_redhat }}"
        state: started
        enabled: yes
      when: ansible_os_family == "RedHat"
```

---

### 👤 Example 2: Create a Deploy User Only in Production

Uses `vars` to define environment and `when` to conditionally create the user.

```yaml
---
- name: Setup Deploy User
  hosts: workers
  become: yes
  vars:
    environment: production
    deploy_user: deployer
    deploy_home: /home/deployer

  tasks:
    - name: Create deploy user (production only)
      user:
        name: "{{ deploy_user }}"
        home: "{{ deploy_home }}"
        shell: /bin/bash
        state: present
      when: environment == "production"

    - name: Add deploy user to sudo group (production only)
      user:
        name: "{{ deploy_user }}"
        groups: sudo
        append: yes
      when: environment == "production"

    - name: Skip deploy setup — not production
      debug:
        msg: "Skipping deploy user setup in {{ environment }} environment"
      when: environment != "production"
```

---

### 💾 Example 3: Alert and Extend Disk Space if Usage is High

Uses `register` to capture disk usage, then `when` to act only if a threshold is exceeded.

```yaml
---
- name: Monitor and Alert Disk Usage
  hosts: workers
  become: yes
  vars:
    disk_threshold: 80
    alert_email: ops-team@company.com

  tasks:
    - name: Get disk usage percentage on /
      shell: df / | awk 'NR==2 {print $5}' | tr -d '%'
      register: disk_usage
      changed_when: false

    - name: Print current disk usage
      debug:
        msg: "Current disk usage on {{ inventory_hostname }}: {{ disk_usage.stdout }}%"

    - name: Warn if disk usage is above threshold
      debug:
        msg: "⚠️  WARNING: Disk usage {{ disk_usage.stdout }}% exceeds {{ disk_threshold }}% on {{ inventory_hostname }}!"
      when: disk_usage.stdout | int > disk_threshold

    - name: Clean apt cache if disk is above threshold (Debian)
      apt:
        autoclean: yes
        autoremove: yes
      when:
        - disk_usage.stdout | int > disk_threshold
        - ansible_os_family == "Debian"
```

---

### 🔒 Example 4: Open Firewall Ports Based on App Role

Uses `vars` to define port list and `when` to apply rules only for specific server roles.

```yaml
---
- name: Configure Firewall by Server Role
  hosts: workers
  become: yes
  vars:
    server_role: webserver        # Options: webserver, dbserver, cacheserver
    web_ports: [80, 443]
    db_port: 5432
    cache_port: 6379

  tasks:
    - name: Open HTTP/HTTPS ports (web servers only)
      ufw:
        rule: allow
        port: "{{ item }}"
        proto: tcp
      loop: "{{ web_ports }}"
      when: server_role == "webserver"

    - name: Open PostgreSQL port (DB servers only)
      ufw:
        rule: allow
        port: "{{ db_port }}"
        proto: tcp
      when: server_role == "dbserver"

    - name: Open Redis port (cache servers only)
      ufw:
        rule: allow
        port: "{{ cache_port }}"
        proto: tcp
      when: server_role == "cacheserver"

    - name: Enable UFW
      ufw:
        state: enabled
```

---

### 🚀 Example 5: Deploy App and Run DB Migration Only on First Deploy

Uses `vars` + `stat` module + `when` to detect first-time deploys.

```yaml
---
- name: Deploy Application
  hosts: workers
  become: yes
  vars:
    app_dir: /opt/myapp
    app_repo: https://github.com/myorg/myapp.git
    app_version: v2.1.0
    run_migrations: true

  tasks:
    - name: Check if app directory already exists
      stat:
        path: "{{ app_dir }}"
      register: app_dir_stat

    - name: Clone app repo (first deploy only)
      git:
        repo: "{{ app_repo }}"
        dest: "{{ app_dir }}"
        version: "{{ app_version }}"
      when: not app_dir_stat.stat.exists

    - name: Pull latest code (subsequent deploys)
      git:
        repo: "{{ app_repo }}"
        dest: "{{ app_dir }}"
        version: "{{ app_version }}"
        update: yes
      when: app_dir_stat.stat.exists

    - name: Run database migrations (only if enabled)
      command: python manage.py migrate
      args:
        chdir: "{{ app_dir }}"
      when:
        - run_migrations | bool
        - not app_dir_stat.stat.exists   # Only on first deploy

    - name: Restart application service
      service:
        name: myapp
        state: restarted
```

---

### 📋 Vars & Conditions Quick Reference

| Feature | Syntax | Use Case |
|---------|--------|----------|
| Define variable | `vars: myvar: value` | Reusable config values |
| Use variable | `"{{ myvar }}"` | Reference a var in a task |
| OS condition | `when: ansible_os_family == "Debian"` | Branch by OS type |
| Variable condition | `when: environment == "production"` | Branch by custom var |
| Register output | `register: result` | Capture task output |
| Condition on output | `when: result.stdout \| int > 80` | Act based on task output |
| Boolean check | `when: run_migrations \| bool` | Check boolean vars |
| Multiple conditions | `when:` + list of conditions | All must be true (AND logic) |
| Negation | `when: not app_exists` | Invert a condition |
