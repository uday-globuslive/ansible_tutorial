# Ansible Commands Cheatsheet

## Understanding Ansible Commands (Beginner Guide)

There are **4 main commands** you'll use daily:

| Command | Purpose | When to Use |
|---------|---------|-------------|
| `ansible` | Run single tasks on servers | Quick tests, one-off tasks |
| `ansible-playbook` | Run playbook files | Actual automation |
| `ansible-inventory` | View/debug inventory | Check which servers you're targeting |
| `ansible-galaxy` | Manage roles/collections | Install community content |

### Command Structure Explained

```
ansible <WHO> -m <WHAT> -a <HOW>
        │       │        │
        │       │        └─ Arguments (parameters for the module)
        │       └───────── Module (the action to take)
        └───────────────── Target (which servers)

Example:
ansible webservers -m apt -a "name=nginx state=present"
        └─────────┘   └─┘   └──────────────────────────┘
        WHO           WHAT           HOW
        (servers)     (module)       (parameters)
```

---

## Quick Reference Card

---

## Basic Commands

### ansible - Run Ad-hoc Commands

These are one-liner commands for quick tasks - no playbook needed!

```bash
# Syntax explained:
# ansible <host-pattern> -m <module> -a "<arguments>"
#         │               │          │
#         │               │          └── What to do (module arguments)
#         │               └───────────── Which tool to use (module)
#         └───────────────────────────── Where to run (hosts/groups)

# Examples with explanations:

# Test if servers are reachable (like ping, but for Ansible)
ansible all -m ping
# "all" = every server in inventory
# "ping" module = just checks if Ansible can connect

# Run a shell command on all web servers
ansible webservers -m shell -a "uptime"
# Shows how long each server has been running

# Install vim on database servers (requires sudo, hence -b)
ansible dbservers -m yum -a "name=vim state=present" -b
# -b means "become" (use sudo)
```

### ansible-playbook - Run Playbooks

```bash
# Syntax
ansible-playbook [options] playbook.yml

# Examples
ansible-playbook site.yml                    # Run playbook
ansible-playbook site.yml -v                 # Verbose mode
ansible-playbook site.yml --check            # Dry run
ansible-playbook site.yml --diff             # Show changes
```

### ansible-inventory - Manage Inventory

```bash
ansible-inventory --list                     # List all hosts (JSON)
ansible-inventory --graph                    # Show inventory graph
ansible-inventory --host hostname            # Show host details
```

### ansible-galaxy - Manage Roles/Collections

```bash
ansible-galaxy role init myrole              # Create role skeleton
ansible-galaxy install geerlingguy.nginx     # Install role
ansible-galaxy collection install amazon.aws  # Install collection
```

---

## Ad-hoc Command Examples

### System Information

```bash
# Get all facts
ansible all -m setup

# Get specific facts
ansible all -m setup -a "filter=ansible_distribution*"

# Get memory info
ansible all -m setup -a "filter=ansible_memory_mb"

# Get network info
ansible all -m setup -a "filter=ansible_default_ipv4"
```

### Package Management

```bash
# Install package (apt)
ansible webservers -m apt -a "name=nginx state=present" -b

# Install package (yum)
ansible webservers -m yum -a "name=httpd state=present" -b

# Remove package
ansible webservers -m apt -a "name=nginx state=absent" -b

# Update all packages
ansible all -m apt -a "upgrade=yes update_cache=yes" -b
```

### Service Management

```bash
# Start service
ansible webservers -m service -a "name=nginx state=started" -b

# Stop service
ansible webservers -m service -a "name=nginx state=stopped" -b

# Restart service
ansible webservers -m service -a "name=nginx state=restarted" -b

# Enable service on boot
ansible webservers -m service -a "name=nginx enabled=yes" -b
```

### File Operations

```bash
# Copy file to remote
ansible all -m copy -a "src=/local/file dest=/remote/file"

# Create directory
ansible all -m file -a "path=/path/to/dir state=directory mode=0755"

# Create symbolic link
ansible all -m file -a "src=/file dest=/link state=link"

# Delete file
ansible all -m file -a "path=/path/to/file state=absent"

# Change file permissions
ansible all -m file -a "path=/path/to/file mode=0644 owner=user group=group"
```

### User Management

```bash
# Create user
ansible all -m user -a "name=john state=present" -b

# Create user with home directory
ansible all -m user -a "name=john state=present create_home=yes shell=/bin/bash" -b

# Delete user
ansible all -m user -a "name=john state=absent remove=yes" -b

# Add user to group
ansible all -m user -a "name=john groups=sudo append=yes" -b
```

### Command Execution

```bash
# Run shell command
ansible all -m shell -a "df -h"

# Run command (no shell processing)
ansible all -m command -a "cat /etc/passwd"

# Run command with sudo
ansible all -m shell -a "cat /etc/shadow" -b

# Run on specific hosts
ansible "web1:web2" -m shell -a "hostname"
```

---

## Playbook Command Options

### Execution Control

```bash
# Run playbook
ansible-playbook playbook.yml

# Dry run (check mode)
ansible-playbook playbook.yml --check

# Show what would change
ansible-playbook playbook.yml --diff

# Step through tasks
ansible-playbook playbook.yml --step

# Start at specific task
ansible-playbook playbook.yml --start-at-task="Install nginx"

# Run specific tags
ansible-playbook playbook.yml --tags "install,configure"

# Skip specific tags
ansible-playbook playbook.yml --skip-tags "test"
```

### Verbosity

```bash
# Verbose output
ansible-playbook playbook.yml -v       # Basic
ansible-playbook playbook.yml -vv      # More details
ansible-playbook playbook.yml -vvv     # SSH debugging
ansible-playbook playbook.yml -vvvv    # Maximum verbosity
```

### Inventory & Hosts

```bash
# Specify inventory file
ansible-playbook playbook.yml -i inventory/production

# Limit to specific hosts
ansible-playbook playbook.yml --limit "webservers"
ansible-playbook playbook.yml --limit "web1.example.com"
ansible-playbook playbook.yml --limit "webservers:&production"
```

### Variables

```bash
# Pass extra variables
ansible-playbook playbook.yml -e "version=1.5.0"
ansible-playbook playbook.yml -e "@vars.yml"
ansible-playbook playbook.yml -e '{"users": ["john", "jane"]}'

# Prompt for variables
ansible-playbook playbook.yml --ask-vars
```

### Authentication

```bash
# Ask for SSH password
ansible-playbook playbook.yml -k

# Ask for sudo password
ansible-playbook playbook.yml -K

# Specify SSH key
ansible-playbook playbook.yml --private-key ~/.ssh/mykey

# Specify remote user
ansible-playbook playbook.yml -u admin
```

### Vault

```bash
# Ask for vault password
ansible-playbook playbook.yml --ask-vault-pass

# Use vault password file
ansible-playbook playbook.yml --vault-password-file ~/.vault_pass

# Encrypt file
ansible-vault encrypt secrets.yml

# Decrypt file
ansible-vault decrypt secrets.yml

# Edit encrypted file
ansible-vault edit secrets.yml

# View encrypted file
ansible-vault view secrets.yml

# Change vault password
ansible-vault rekey secrets.yml
```

---

## Host Pattern Examples

```bash
# All hosts
ansible all -m ping

# Specific group
ansible webservers -m ping

# Multiple groups
ansible "webservers:dbservers" -m ping

# Intersection (AND)
ansible "webservers:&production" -m ping

# Exclusion
ansible "webservers:!web3" -m ping

# Regex pattern
ansible "~web[0-9]+" -m ping

# Specific hosts
ansible "web1.example.com,web2.example.com" -m ping

# Range
ansible "web[1:5].example.com" -m ping
```

---

## Useful Debugging Commands

```bash
# Test playbook syntax
ansible-playbook playbook.yml --syntax-check

# List all tasks
ansible-playbook playbook.yml --list-tasks

# List all hosts
ansible-playbook playbook.yml --list-hosts

# List all tags
ansible-playbook playbook.yml --list-tags

# Debug single task
ansible localhost -m debug -a "msg='Hello World'"

# Check facts for host
ansible hostname -m setup

# Test connectivity
ansible all -m ping

# Check SSH connection
ansible all -m raw -a "echo 'SSH working'"
```

---

## Galaxy Commands

```bash
# Search for roles
ansible-galaxy search nginx

# Get role info
ansible-galaxy info geerlingguy.nginx

# List installed roles
ansible-galaxy list

# Install role
ansible-galaxy install geerlingguy.nginx

# Install specific version
ansible-galaxy install geerlingguy.nginx,3.0.0

# Install from requirements file
ansible-galaxy install -r requirements.yml

# Remove role
ansible-galaxy remove geerlingguy.nginx

# Initialize new role
ansible-galaxy init my_role
```

---

## Configuration Commands

```bash
# Show current config
ansible-config view

# Show all config options
ansible-config list

# Dump current settings
ansible-config dump

# Show changed settings
ansible-config dump --only-changed
```

---

## Output Formatting

```bash
# JSON output
ANSIBLE_STDOUT_CALLBACK=json ansible all -m ping

# YAML output
ANSIBLE_STDOUT_CALLBACK=yaml ansible-playbook playbook.yml

# Minimal output
ANSIBLE_STDOUT_CALLBACK=minimal ansible all -m ping

# One-line per host
ANSIBLE_STDOUT_CALLBACK=oneline ansible all -m ping
```

---

## Environment Variables

```bash
# Set ansible.cfg location
export ANSIBLE_CONFIG=/path/to/ansible.cfg

# Set inventory location
export ANSIBLE_INVENTORY=/path/to/inventory

# Disable host key checking
export ANSIBLE_HOST_KEY_CHECKING=False

# Set SSH arguments
export ANSIBLE_SSH_ARGS="-o ControlMaster=auto -o ControlPersist=60s"

# Set Python interpreter
export ANSIBLE_PYTHON_INTERPRETER=/usr/bin/python3

# Set log file
export ANSIBLE_LOG_PATH=/var/log/ansible.log
```

---

## Quick Troubleshooting

```bash
# Check SSH connectivity
ssh -v user@hostname

# Check Ansible can reach hosts
ansible all -m ping -vvv

# Check sudo access
ansible all -m shell -a "whoami" -b

# Check Python on remote
ansible all -m raw -a "which python python3"

# Show gathered facts
ansible hostname -m setup
```

---

## Common Command Combinations

```bash
# Deploy and verify
ansible-playbook deploy.yml && ansible all -m uri -a "url=http://localhost:80"

# Update and reboot if needed
ansible webservers -m apt -a "upgrade=yes" -b && \
ansible webservers -m reboot -b --when "ansible_reboot_pending"

# Backup before change
ansible dbservers -m shell -a "pg_dump mydb > backup.sql" -b && \
ansible-playbook update_db.yml
```

---

*Bookmark this cheatsheet for quick reference!*
