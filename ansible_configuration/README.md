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
