# Ansible Installation Guide

## Prerequisites (What You Need First)

Before installing Ansible, ensure you have:
- **Python 3.8 or higher** - Ansible is written in Python
- **SSH client** - How Ansible connects to servers
- **A Linux/macOS machine** (for Control Node) - Ansible control node cannot be Windows directly

> **Important Notes for Beginners:**
> - You install Ansible **ONLY** on YOUR machine (the control node)
> - You do **NOT** install Ansible on the servers you want to manage
> - Managed nodes only need SSH server running and Python installed (they usually already have this!)

### Understanding the Setup

```
YOUR MACHINE (Control Node)           SERVERS TO MANAGE (Managed Nodes)
┌─────────────────────────────┐       ┌─────────────────────────────┐
│                             │       │                             │
│   Install Ansible HERE  ✓   │       │   DON'T install Ansible ✗   │
│                             │──────►│                             │
│   Runs: ansible-playbook    │  SSH  │   Just needs: SSH + Python  │
│                             │       │                             │
└─────────────────────────────┘       └─────────────────────────────┘
```

---

## Installation on Different Platforms

### 1. Ubuntu/Debian (Most Common)

```bash
# Step 1: Update your package list (like refreshing an app store)
sudo apt update

# Step 2: Install software-properties-common (allows adding new software sources)
sudo apt install software-properties-common

# Step 3: Add Ansible's official repository (where to get Ansible from)
sudo add-apt-repository --yes --update ppa:ansible/ansible

# Step 4: Install Ansible!
sudo apt install ansible

# Step 5: Verify it worked
ansible --version

# Expected output (version numbers may differ):
# ansible [core 2.15.0]
#   config file = /etc/ansible/ansible.cfg
#   python version = 3.10.6
```

**What each command does:**
- `sudo` = Run as administrator
- `apt update` = Refresh list of available packages
- `apt install` = Download and install a package
- `add-apt-repository` = Add a new source for packages
- `ppa:ansible/ansible` = Ansible's Personal Package Archive (official source)

### 2. CentOS/RHEL/Rocky Linux

```bash
# Install EPEL repository
sudo yum install epel-release

# Install Ansible
sudo yum install ansible

# For RHEL 8+/CentOS 8+
sudo dnf install ansible

# Verify installation
ansible --version
```

### 3. Amazon Linux 2

```bash
# Enable extras repository
sudo amazon-linux-extras enable ansible2

# Install Ansible
sudo yum install ansible

# Verify
ansible --version
```

### 4. macOS

```bash
# Using Homebrew
brew install ansible

# Using pip
pip3 install ansible

# Verify installation
ansible --version
```

### 5. Windows (WSL - Windows Subsystem for Linux)

```bash
# Step 1: Enable WSL (in PowerShell as Admin)
wsl --install

# Step 2: After WSL setup, open Ubuntu terminal
# Step 3: Install Ansible
sudo apt update
sudo apt install ansible

# Verify
ansible --version
```

### 6. Using Python pip (Universal)

```bash
# Install pip if not available
python3 -m ensurepip --upgrade

# Install Ansible via pip
pip3 install ansible

# Install specific version
pip3 install ansible==2.14.0

# Upgrade Ansible
pip3 install --upgrade ansible

# Verify
ansible --version
```

---

## Post-Installation Configuration

### 1. Create Ansible Configuration Directory

```bash
# Create directory structure
mkdir -p ~/ansible/{playbooks,inventory,roles,files,templates,vars}

# Create default config file
touch ~/ansible/ansible.cfg
touch ~/ansible/inventory/hosts
```

### 2. Configure ansible.cfg

```ini
# ~/ansible/ansible.cfg
[defaults]
inventory = ./inventory/hosts
remote_user = your_username
host_key_checking = False
retry_files_enabled = False
log_path = ./ansible.log

[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False

[ssh_connection]
pipelining = True
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
```

### 3. Configure Inventory File

```ini
# ~/ansible/inventory/hosts

# Local machine
[local]
localhost ansible_connection=local

# Web servers
[webservers]
web1.example.com
web2.example.com
192.168.1.10

# Database servers
[dbservers]
db1.example.com
db2.example.com

# All servers
[all:children]
webservers
dbservers
```

---

## Setting Up SSH Keys (Important!)

### 1. Generate SSH Key Pair

```bash
# Generate RSA key (recommended)
ssh-keygen -t rsa -b 4096 -C "ansible@control-node"

# Or generate ED25519 key (more secure)
ssh-keygen -t ed25519 -C "ansible@control-node"
```

### 2. Copy Public Key to Managed Nodes

```bash
# Copy to single host
ssh-copy-id user@hostname

# Copy to multiple hosts (script)
for host in web1 web2 db1 db2; do
    ssh-copy-id user@$host
done
```

### 3. Test SSH Connection

```bash
# Test SSH access
ssh user@hostname

# Test Ansible connectivity
ansible all -m ping
```

---

## Verify Installation

### 1. Check Version

```bash
# Full version info
ansible --version

# Output example:
# ansible [core 2.15.0]
#   config file = /etc/ansible/ansible.cfg
#   configured module search path = ['/home/user/.ansible/plugins/modules']
#   ansible python module location = /usr/lib/python3/dist-packages/ansible
#   ansible collection location = /home/user/.ansible/collections
#   executable location = /usr/bin/ansible
#   python version = 3.10.6
```

### 2. Test Local Connection

```bash
# Ping localhost
ansible localhost -m ping

# Expected output:
# localhost | SUCCESS => {
#     "changed": false,
#     "ping": "pong"
# }
```

### 3. Run Ad-hoc Command

```bash
# Get system info
ansible localhost -m setup | head -50

# Run shell command
ansible localhost -m shell -a "uname -a"
```

---

## Common Installation Issues & Solutions

### Issue 1: Python Not Found

```bash
# Error: Unable to find Python

# Solution: Set Python interpreter
ansible all -m ping -e 'ansible_python_interpreter=/usr/bin/python3'

# Or add to ansible.cfg
[defaults]
interpreter_python = /usr/bin/python3
```

### Issue 2: SSH Connection Refused

```bash
# Error: SSH connection refused

# Solutions:
# 1. Check SSH service
sudo systemctl status sshd
sudo systemctl start sshd

# 2. Check firewall
sudo ufw allow 22/tcp
sudo ufw reload
```

### Issue 3: Permission Denied

```bash
# Error: Permission denied (publickey)

# Solutions:
# 1. Check key permissions
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_rsa.pub

# 2. Check SSH agent
eval $(ssh-agent)
ssh-add ~/.ssh/id_rsa
```

### Issue 4: Host Key Verification Failed

```bash
# Error: Host key verification failed

# Quick fix (not recommended for production)
export ANSIBLE_HOST_KEY_CHECKING=False

# Or add to ansible.cfg
[defaults]
host_key_checking = False
```

---

## Directory Structure Best Practice

```
ansible/
├── ansible.cfg              # Main configuration
├── inventory/
│   ├── hosts                # Default inventory
│   ├── production           # Production hosts
│   └── staging              # Staging hosts
├── playbooks/
│   ├── site.yml             # Main playbook
│   ├── webservers.yml       # Web server playbook
│   └── dbservers.yml        # Database playbook
├── roles/
│   ├── common/              # Common role
│   ├── nginx/               # Nginx role
│   └── mysql/               # MySQL role
├── group_vars/
│   ├── all.yml              # Variables for all hosts
│   └── webservers.yml       # Variables for web servers
├── host_vars/
│   └── web1.example.com.yml # Host-specific variables
├── files/                   # Static files
├── templates/               # Jinja2 templates
└── vault/                   # Encrypted files
    └── secrets.yml
```

---

## Install Additional Collections

```bash
# Install from Ansible Galaxy
ansible-galaxy collection install community.general
ansible-galaxy collection install amazon.aws
ansible-galaxy collection install azure.azcollection

# List installed collections
ansible-galaxy collection list

# Install from requirements file
# requirements.yml:
# collections:
#   - name: community.general
#   - name: amazon.aws
#     version: ">=2.0.0"

ansible-galaxy install -r requirements.yml
```

---

## Quick Start Test

Create a simple test playbook:

```yaml
# test_playbook.yml
---
- name: Test Playbook
  hosts: localhost
  connection: local
  tasks:
    - name: Print hello message
      debug:
        msg: "Hello from Ansible! Installation successful!"
    
    - name: Get system info
      debug:
        msg: "OS: {{ ansible_distribution }} {{ ansible_distribution_version }}"
```

Run it:

```bash
ansible-playbook test_playbook.yml
```

---

## Next Steps

Proceed to **03_Inventory** to learn about managing hosts and groups.
