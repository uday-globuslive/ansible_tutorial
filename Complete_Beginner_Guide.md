# Ansible Complete Beginner Guide - All Topics Elaborated

This guide explains EVERY Ansible concept as if you're brand new to automation. Each section builds on the previous one.

---

# 📘 PART 1: UNDERSTANDING THE BASICS

## Chapter 1: What is Ansible and Why Should You Care?

### The Problem Ansible Solves

Imagine you're a system administrator managing 100 servers. Every time there's a security update, you need to:
1. SSH into server 1, run updates, exit
2. SSH into server 2, run updates, exit
3. SSH into server 3... (repeat 97 more times)

**This is painful because:**
- It takes HOURS or DAYS
- You might make mistakes (typos, forgetting a server)
- There's no record of what you did
- Different servers end up configured differently

### Enter Ansible

Ansible lets you write your commands ONCE and run them on ALL servers simultaneously.

```
WITHOUT ANSIBLE:                    WITH ANSIBLE:
┌──────────────────┐               ┌──────────────────┐
│ You → Server 1   │               │                  │
│ You → Server 2   │               │ You → Ansible    │──→ All 100
│ You → Server 3   │               │     (once!)      │    servers
│ ...              │               │                  │    at once!
│ You → Server 100 │               └──────────────────┘
└──────────────────┘
   Takes: 8 hours                     Takes: 5 minutes
```

### Real-World Example

**Task**: Install nginx on 50 web servers

**Without Ansible** (Manual):
```bash
# You type this 50 times:
ssh admin@server1.example.com
sudo apt update
sudo apt install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx
exit
# ... repeat for server2, server3, etc.
```

**With Ansible** (Automated):
```yaml
# Write once, run everywhere
- name: Install nginx on all web servers
  hosts: webservers  # All 50 servers
  become: true       # Use sudo
  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present
    - name: Start and enable nginx
      service:
        name: nginx
        state: started
        enabled: true
```

Run ONE command: `ansible-playbook install_nginx.yml`
**Done!** All 50 servers configured in minutes.

---

## Chapter 2: How Ansible Works - The Architecture Explained

### The Players

**1. Control Node (Your Machine)**
```
┌─────────────────────────────────────────────────────┐
│                    CONTROL NODE                      │
│        (Your laptop/workstation/jump server)        │
│                                                      │
│  What lives here:                                   │
│  • Ansible software (installed only HERE)           │
│  • Your playbooks (YAML files)                     │
│  • Inventory file (list of servers)                │
│  • SSH keys (for authentication)                   │
│                                                      │
└─────────────────────────────────────────────────────┘
```

**2. Managed Nodes (Your Servers)**
```
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│  MANAGED NODE   │  │  MANAGED NODE   │  │  MANAGED NODE   │
│    Server 1     │  │    Server 2     │  │    Server 3     │
├─────────────────┤  ├─────────────────┤  ├─────────────────┤
│ Requirements:   │  │ Requirements:   │  │ Requirements:   │
│ • SSH access    │  │ • SSH access    │  │ • SSH access    │
│ • Python        │  │ • Python        │  │ • Python        │
│ • Network       │  │ • Network       │  │ • Network       │
│                 │  │                 │  │                 │
│ NO Ansible      │  │ NO Ansible      │  │ NO Ansible      │
│ installed!      │  │ installed!      │  │ installed!      │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

### Why "Agentless" Matters

**Agent-Based Tools (Puppet, Chef, SaltStack):**
```
Step 1: Install agent on EVERY server
Step 2: Configure agent to talk to master server
Step 3: Agent constantly "phones home" for instructions
Step 4: More software = more things that can break
```

**Ansible (Agentless):**
```
Step 1: Install Ansible on YOUR machine
Step 2: Make sure servers have SSH access
Step 3: That's it. No agents, no master servers, no complexity.
```

### The Communication Flow

When you run a playbook, here's what happens:

```
Step 1: Ansible reads your playbook
        ┌─────────────────┐
        │ deploy_app.yml  │
        └────────┬────────┘
                 │
                 ▼
Step 2: Ansible reads your inventory
        ┌─────────────────┐
        │ Which servers?  │
        │ web1, web2, db1 │
        └────────┬────────┘
                 │
                 ▼
Step 3: Ansible connects via SSH
        ┌─────────────────────────────────────┐
        │  SSH: "Hi servers, it's me!"        │
        │  ─────────────────────────────────  │
        │  ──→ web1.example.com               │
        │  ──→ web2.example.com               │
        │  ──→ db1.example.com                │
        └────────┬────────────────────────────┘
                 │
                 ▼
Step 4: Ansible sends Python code to servers
        ┌─────────────────────────────────────┐
        │  "Here's a tiny Python script       │
        │   that installs nginx"              │
        └────────┬────────────────────────────┘
                 │
                 ▼
Step 5: Servers execute code, report back
        ┌─────────────────────────────────────┐
        │  web1: "Done! nginx installed"      │
        │  web2: "Done! nginx installed"      │
        │  db1:  "Done! nginx installed"      │
        └────────┬────────────────────────────┘
                 │
                 ▼
Step 6: Ansible shows you the results
        PLAY RECAP ****************************
        web1 : ok=2  changed=1  failed=0
        web2 : ok=2  changed=1  failed=0
        db1  : ok=2  changed=1  failed=0
```

---

## Chapter 3: Key Concepts - The Vocabulary

### 3.1 What is an Inventory?

**Simple Definition**: A list of servers you want to manage.

**Analogy**: Think of it like your phone's contact list:
- Contact List = Inventory
- Each contact = A server
- Contact groups (Family, Work, Friends) = Server groups

**Basic Inventory File:**
```ini
# This is a simple inventory file
# Location: /etc/ansible/hosts or ./inventory

# Standalone server (no group)
backup.example.com

# Group of web servers
[webservers]
web1.example.com
web2.example.com
web3.example.com

# Group of database servers
[dbservers]
db1.example.com
db2.example.com

# You can create groups of groups!
[production:children]
webservers
dbservers
```

**What This Means:**
- `webservers` group has 3 servers
- `dbservers` group has 2 servers
- `production` group contains ALL servers from webservers AND dbservers

**Using the Inventory:**
```bash
# Run a command on ALL web servers
ansible webservers -m ping

# Run a command on ALL database servers
ansible dbservers -m ping

# Run on ALL production servers (web + db)
ansible production -m ping

# Run on EVERYTHING
ansible all -m ping
```

### 3.2 What is a Playbook?

**Simple Definition**: A file containing instructions for Ansible to execute.

**Analogy**: A playbook is like a recipe book:
- Recipe Book = Playbook
- Each Recipe = A "Play"
- Each cooking step = A "Task"

**Visual Breakdown:**
```
PLAYBOOK (the whole file)
│
├── PLAY 1: "Configure Web Servers"
│   │
│   ├── TASK 1: Install nginx
│   ├── TASK 2: Copy config file
│   └── TASK 3: Start nginx
│
└── PLAY 2: "Configure Database"
    │
    ├── TASK 1: Install PostgreSQL
    ├── TASK 2: Create database
    └── TASK 3: Create user
```

**Actual Playbook Example:**
```yaml
---
# This is a playbook with 2 plays

- name: Configure Web Servers         # ← PLAY 1
  hosts: webservers                   # Target these servers
  become: true                        # Use sudo
  
  tasks:                              # List of things to do
    - name: Install nginx             # ← TASK 1
      apt:
        name: nginx
        state: present
    
    - name: Copy nginx config         # ← TASK 2
      copy:
        src: files/nginx.conf
        dest: /etc/nginx/nginx.conf
    
    - name: Start nginx               # ← TASK 3
      service:
        name: nginx
        state: started

- name: Configure Database             # ← PLAY 2
  hosts: dbservers                     # Different servers
  become: true
  
  tasks:
    - name: Install PostgreSQL         # ← TASK 1
      apt:
        name: postgresql
        state: present
```

### 3.3 What is a Module?

**Simple Definition**: A module is a pre-built tool that performs a specific action.

**Analogy**: Modules are like apps on your phone:
- Want to send a message? Use the Messaging app
- Want to install software? Use the `apt` module
- Want to copy a file? Use the `copy` module
- Want to manage services? Use the `service` module

**Ansible has 3000+ modules for:**
| Category | What it does | Example Modules |
|----------|--------------|-----------------|
| **Package Management** | Install/remove software | `apt`, `yum`, `dnf`, `pip` |
| **File Operations** | Create, copy, modify files | `file`, `copy`, `template`, `lineinfile` |
| **Service Management** | Start, stop, restart services | `service`, `systemd` |
| **User Management** | Create users and groups | `user`, `group` |
| **Cloud** | Manage cloud resources | `ec2`, `azure_rm`, `gcp_compute` |
| **Command Execution** | Run shell commands | `command`, `shell`, `raw` |

**Module Usage Examples:**
```yaml
tasks:
  # The "apt" module installs packages on Debian/Ubuntu
  - name: Install nginx
    apt:                    # ← Module name
      name: nginx           # ← Module parameter
      state: present        # ← Module parameter

  # The "copy" module copies files to servers
  - name: Copy configuration
    copy:                   # ← Different module
      src: local_file.txt
      dest: /remote/path/file.txt

  # The "service" module manages system services
  - name: Start nginx
    service:                # ← Another module
      name: nginx
      state: started
```

### 3.4 What is Idempotence?

**Simple Definition**: Running the same thing multiple times gives the same result.

**Why This Matters - A Story:**

**Scenario WITHOUT Idempotence:**
```bash
# You want to add a line to a config file
echo "max_connections=100" >> /etc/app.conf

# Run 1: File has 1 line ✓
# Run 2: File has 2 identical lines ✗
# Run 3: File has 3 identical lines ✗✗
# Your config is now BROKEN!
```

**Scenario WITH Idempotence (Ansible's way):**
```yaml
- name: Set max connections
  lineinfile:
    path: /etc/app.conf
    line: "max_connections=100"
    state: present

# Run 1: Line doesn't exist → adds it → CHANGED
# Run 2: Line exists → does nothing → OK
# Run 3: Line exists → does nothing → OK
# Config is ALWAYS correct!
```

**Visual Representation:**
```
NON-IDEMPOTENT (Dangerous):
Run 1: [●○○] → [●●○]  (Made a change)
Run 2: [●●○] → [●●●]  (Made ANOTHER change - BAD!)
Run 3: [●●●] → [●●●●] (KEEPS changing!)

IDEMPOTENT (Safe - Ansible's way):
Run 1: [○○○] → [●○○]  (Made a change)
Run 2: [●○○] → [●○○]  (Already done, no change)
Run 3: [●○○] → [●○○]  (Still no change - GOOD!)
```

---

# 📘 PART 2: INVENTORY DEEP DIVE

## Chapter 4: Mastering Inventory Files

### 4.1 INI Format (Traditional)

```ini
# inventory/hosts

# ═══════════════════════════════════════════════════
# SECTION 1: Standalone Hosts (no group)
# ═══════════════════════════════════════════════════
jumpserver.example.com
monitoring.example.com

# ═══════════════════════════════════════════════════
# SECTION 2: Grouped Hosts
# ═══════════════════════════════════════════════════

# Web server group
[webservers]
web1.example.com
web2.example.com
web3.example.com

# Database server group
[dbservers]
db-master.example.com
db-replica.example.com

# ═══════════════════════════════════════════════════
# SECTION 3: Hosts with Variables
# ═══════════════════════════════════════════════════

[appservers]
# Format: hostname variable1=value variable2=value
app1.example.com ansible_user=admin http_port=8080
app2.example.com ansible_user=deploy http_port=8081
app3.example.com ansible_user=deploy http_port=8082 backup_enabled=true

# ═══════════════════════════════════════════════════
# SECTION 4: Host Ranges (save typing!)
# ═══════════════════════════════════════════════════

[frontend]
# This expands to: front01, front02, front03, front04, front05
front[01:05].example.com

# Letter ranges: staging-a, staging-b, staging-c
staging-[a:c].example.com

# ═══════════════════════════════════════════════════
# SECTION 5: Group Variables
# ═══════════════════════════════════════════════════

[webservers:vars]
# These variables apply to ALL hosts in [webservers]
http_port=80
https_port=443
nginx_user=www-data
log_dir=/var/log/nginx

[dbservers:vars]
# These variables apply to ALL hosts in [dbservers]
db_port=5432
backup_time="02:00"

# ═══════════════════════════════════════════════════
# SECTION 6: Groups of Groups (Children)
# ═══════════════════════════════════════════════════

[production:children]
webservers
dbservers
appservers

[development:children]
frontend

# ═══════════════════════════════════════════════════
# SECTION 7: All Hosts Variables
# ═══════════════════════════════════════════════════

[all:vars]
# These apply to EVERY host in the inventory
ansible_python_interpreter=/usr/bin/python3
ansible_user=deploy
timezone=UTC
```

### 4.2 YAML Format (Modern)

```yaml
# inventory/hosts.yml
---
all:
  vars:
    # Variables for ALL hosts
    ansible_python_interpreter: /usr/bin/python3
    timezone: UTC
  
  hosts:
    # Standalone hosts
    jumpserver.example.com:
    monitoring.example.com:
  
  children:
    # Web servers group
    webservers:
      vars:
        http_port: 80
        https_port: 443
      hosts:
        web1.example.com:
        web2.example.com:
        web3.example.com:
    
    # Database servers group
    dbservers:
      vars:
        db_port: 5432
      hosts:
        db-master.example.com:
          role: master        # Host-specific variable
        db-replica.example.com:
          role: replica
    
    # Production group (contains other groups)
    production:
      children:
        webservers:
        dbservers:
```

### 4.3 Special Ansible Variables for Hosts

These variables tell Ansible HOW to connect to each host:

```ini
[servers]
# Basic connection
server1.example.com ansible_host=192.168.1.10

# Different SSH port
server2.example.com ansible_port=2222

# Different username
server3.example.com ansible_user=admin

# Specific SSH key
server4.example.com ansible_ssh_private_key_file=~/.ssh/server4_key

# Different Python interpreter
server5.example.com ansible_python_interpreter=/usr/bin/python3.9

# Windows server (use WinRM instead of SSH)
winserver.example.com ansible_connection=winrm ansible_winrm_transport=basic

# Local connection (run on control node itself)
localhost ansible_connection=local
```

**Complete Reference Table:**

| Variable | What it does | Example |
|----------|--------------|---------|
| `ansible_host` | IP or hostname to connect to | `192.168.1.10` |
| `ansible_port` | SSH port | `2222` |
| `ansible_user` | SSH username | `admin` |
| `ansible_ssh_private_key_file` | Path to SSH key | `~/.ssh/mykey` |
| `ansible_password` | SSH password (avoid!) | `secret123` |
| `ansible_become` | Use sudo | `true` |
| `ansible_become_user` | User to become | `root` |
| `ansible_become_pass` | Sudo password | `secret123` |
| `ansible_python_interpreter` | Path to Python | `/usr/bin/python3` |
| `ansible_connection` | Connection type | `ssh`, `local`, `winrm` |

---

# 📘 PART 3: PLAYBOOKS IN DEPTH

## Chapter 5: Anatomy of a Playbook

### 5.1 Complete Playbook Structure

```yaml
---
# ═══════════════════════════════════════════════════════════
# PLAY 1: Configure All Servers (Common Tasks)
# ═══════════════════════════════════════════════════════════
- name: Configure all servers with common settings
  hosts: all                    # Run on ALL servers
  become: true                  # Use sudo
  gather_facts: true            # Collect server info
  
  # Variables specific to this play
  vars:
    ntp_server: time.google.com
    admin_email: admin@company.com
  
  # Variables loaded from files
  vars_files:
    - vars/common.yml
    - vars/secrets.yml          # Vault encrypted
  
  # Pre-tasks run BEFORE roles and tasks
  pre_tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
        cache_valid_time: 3600
  
  # Roles to apply (reusable collections of tasks)
  roles:
    - common
    - security
  
  # Main tasks
  tasks:
    - name: Set timezone
      timezone:
        name: "{{ timezone }}"
    
    - name: Install basic packages
      apt:
        name: "{{ item }}"
        state: present
      loop:
        - vim
        - htop
        - curl
        - wget
  
  # Post-tasks run AFTER tasks
  post_tasks:
    - name: Send completion notification
      debug:
        msg: "Server {{ inventory_hostname }} configured successfully"
  
  # Handlers run ONLY when notified
  handlers:
    - name: Restart sshd
      service:
        name: sshd
        state: restarted

# ═══════════════════════════════════════════════════════════
# PLAY 2: Configure Web Servers Specifically
# ═══════════════════════════════════════════════════════════
- name: Configure web servers
  hosts: webservers
  become: true
  serial: 2                     # Run on 2 servers at a time
  
  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present
      notify: Restart nginx
    
    - name: Deploy website
      copy:
        src: files/index.html
        dest: /var/www/html/
  
  handlers:
    - name: Restart nginx
      service:
        name: nginx
        state: restarted
```

### 5.2 Play Keywords Explained

```yaml
- name: Play name (descriptive)
  
  # ─── TARGETING ───
  hosts: webservers         # Which hosts to run on
                            # Options: hostname, group, 'all', patterns
  
  # ─── CONNECTION ───
  connection: ssh           # How to connect (ssh, local, winrm)
  remote_user: deploy       # SSH as this user
  
  # ─── PRIVILEGE ESCALATION ───
  become: true              # Use sudo
  become_user: root         # Become this user
  become_method: sudo       # How to escalate (sudo, su, pbrun)
  
  # ─── FACT GATHERING ───
  gather_facts: true        # Collect system info (default: true)
  
  # ─── EXECUTION CONTROL ───
  serial: 5                 # Run on 5 hosts at a time
  serial: "25%"             # Or percentage
  max_fail_percentage: 30   # Stop if 30% of hosts fail
  any_errors_fatal: true    # Stop on ANY error
  
  # ─── VARIABLES ───
  vars: {}                  # Define variables
  vars_files: []            # Load from files
  vars_prompt: []           # Ask user for input
  
  # ─── ENVIRONMENT ───
  environment:              # Set environment variables
    PATH: "/custom/path:{{ ansible_env.PATH }}"
```

### 5.3 Task Keywords Explained

```yaml
tasks:
  - name: Task description (always include this!)
    
    # ─── THE MODULE (required) ───
    apt:                          # Module name
      name: nginx                 # Module parameters
      state: present
    
    # ─── CONDITIONALS ───
    when: ansible_os_family == "Debian"   # Only run if condition is true
    
    # ─── PRIVILEGE ───
    become: true                  # Use sudo for this task
    become_user: postgres         # Become specific user
    
    # ─── CAPTURE OUTPUT ───
    register: task_result         # Store output in variable
    
    # ─── ERROR HANDLING ───
    ignore_errors: yes            # Continue even if task fails
    failed_when: "'error' in task_result.stderr"  # Custom failure condition
    changed_when: false           # Never report as "changed"
    
    # ─── NOTIFICATIONS ───
    notify: Restart nginx         # Trigger a handler
    notify:                       # Or multiple handlers
      - Restart nginx
      - Clear cache
    
    # ─── LOOPING ───
    loop:                         # Run task once per item
      - nginx
      - vim
      - git
    
    # ─── DELEGATION ───
    delegate_to: localhost        # Run on different host
    run_once: true                # Run only once (not on each host)
    
    # ─── ASYNC ───
    async: 300                    # Run asynchronously (max 300 seconds)
    poll: 10                      # Check every 10 seconds (0 = fire & forget)
    
    # ─── TAGS ───
    tags:                         # For selective execution
      - packages
      - install
```

---

# 📘 PART 4: MODULES - YOUR TOOLBOX

## Chapter 6: Essential Modules Explained

### 6.1 Package Management Modules

**The `apt` Module (Debian/Ubuntu):**
```yaml
tasks:
  # Install a single package
  - name: Install nginx
    apt:
      name: nginx
      state: present          # Install if not present
    # What happens:
    # 1. Ansible checks if nginx is installed
    # 2. If not installed → runs "apt install nginx"
    # 3. If already installed → does nothing (idempotent!)
  
  # Install multiple packages
  - name: Install web stack
    apt:
      name:
        - nginx
        - php
        - php-fpm
        - php-mysql
      state: present
      update_cache: yes       # Run "apt update" first
  
  # Install specific version
  - name: Install specific nginx version
    apt:
      name: nginx=1.18.0-0ubuntu1
      state: present
  
  # Remove a package
  - name: Remove apache (we're using nginx)
    apt:
      name: apache2
      state: absent           # Remove if present
      purge: yes              # Also remove config files
  
  # Upgrade all packages
  - name: Upgrade all packages
    apt:
      upgrade: dist           # Full distribution upgrade
      update_cache: yes
```

**The `yum`/`dnf` Module (RHEL/CentOS):**
```yaml
tasks:
  # Install package
  - name: Install httpd
    yum:
      name: httpd
      state: present
  
  # Same with dnf (RHEL 8+)
  - name: Install httpd
    dnf:
      name: httpd
      state: present
  
  # Install from URL
  - name: Install EPEL release
    yum:
      name: https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
      state: present
```

### 6.2 File Management Modules

**The `file` Module:**
```yaml
tasks:
  # Create a directory
  - name: Create application directory
    file:
      path: /opt/myapp
      state: directory        # Create directory
      owner: appuser          # Set owner
      group: appgroup         # Set group
      mode: '0755'            # Set permissions
    # Result: /opt/myapp directory created with specified permissions
  
  # Create an empty file
  - name: Create log file
    file:
      path: /var/log/myapp.log
      state: touch            # Create empty file
      owner: appuser
      mode: '0644'
  
  # Create a symbolic link
  - name: Link current release
    file:
      src: /opt/myapp/releases/v1.2.3    # Target
      dest: /opt/myapp/current            # Link name
      state: link
    # Result: /opt/myapp/current → /opt/myapp/releases/v1.2.3
  
  # Delete a file or directory
  - name: Remove old logs
    file:
      path: /var/log/old_logs
      state: absent           # Delete
```

**The `copy` Module:**
```yaml
tasks:
  # Copy a file from control node to managed node
  - name: Copy nginx config
    copy:
      src: files/nginx.conf           # Local file (on YOUR machine)
      dest: /etc/nginx/nginx.conf     # Remote path (on server)
      owner: root
      group: root
      mode: '0644'
      backup: yes               # Create backup before overwriting
    # src can be:
    # - Relative path: files/nginx.conf (relative to playbook)
    # - Absolute path: /home/me/files/nginx.conf
  
  # Create a file with inline content
  - name: Create config file
    copy:
      content: |
        server {
            listen 80;
            server_name example.com;
        }
      dest: /etc/nginx/sites-available/example.conf
```

**The `template` Module:**
```yaml
tasks:
  # Deploy a template (with variables)
  - name: Deploy nginx config from template
    template:
      src: templates/nginx.conf.j2    # Jinja2 template
      dest: /etc/nginx/nginx.conf
      owner: root
      mode: '0644'
    # The .j2 file can contain variables like {{ server_name }}
    # Ansible replaces them with actual values
```

### 6.3 Service Management Modules

```yaml
tasks:
  # Start a service
  - name: Start nginx
    service:
      name: nginx
      state: started          # Start if not running
    # state options:
    # - started: Start if not running (does nothing if already running)
    # - stopped: Stop if running
    # - restarted: Always restart
    # - reloaded: Reload configuration (graceful)
  
  # Start AND enable at boot
  - name: Enable nginx
    service:
      name: nginx
      state: started
      enabled: yes            # Start on boot
  
  # Using systemd module (more features)
  - name: Manage nginx with systemd
    systemd:
      name: nginx
      state: started
      enabled: yes
      daemon_reload: yes      # Reload systemd if unit file changed
```

### 6.4 Command Execution Modules

**When to use which:**
| Module | Use When | Example |
|--------|----------|---------|
| `command` | Simple commands, no shell features needed | `ls -la /tmp` |
| `shell` | Need pipes, redirects, environment vars | `cat /etc/hosts \| grep web` |
| `raw` | No Python on remote host | `apt install python` |
| `script` | Run a local script on remote host | Complex bash script |

```yaml
tasks:
  # COMMAND module (safer, no shell features)
  - name: Check disk space
    command: df -h /
    register: disk_space
  # ✓ Works: "df -h /"
  # ✗ Fails: "df -h / | grep root" (no pipes!)
  
  # SHELL module (full shell features)
  - name: Find large files
    shell: find /var -size +100M 2>/dev/null | head -10
    register: large_files
  # ✓ Works with pipes, redirects, etc.
  
  # SHELL with complex commands
  - name: Run complex shell command
    shell: |
      cd /opt/myapp
      git pull
      npm install
      npm run build
    args:
      chdir: /opt/myapp       # Set working directory
      executable: /bin/bash   # Use specific shell
  
  # SCRIPT module (run local script remotely)
  - name: Run setup script
    script: scripts/setup.sh arg1 arg2
    # The script exists on YOUR machine
    # Ansible copies it to remote, executes, then removes it
  
  # RAW module (no Python required)
  - name: Install Python on bare server
    raw: apt update && apt install -y python3
    # Use this for bootstrapping servers without Python
```

### 6.5 User and Group Management

```yaml
tasks:
  # Create a user
  - name: Create application user
    user:
      name: appuser
      comment: "Application User"
      uid: 1500
      group: appgroup
      groups: sudo,docker        # Additional groups
      shell: /bin/bash
      home: /home/appuser
      create_home: yes
      state: present
  
  # Create a user with SSH key
  - name: Create deploy user
    user:
      name: deploy
      generate_ssh_key: yes      # Generate SSH keypair
      ssh_key_bits: 4096
      ssh_key_comment: "deploy@{{ inventory_hostname }}"
  
  # Remove a user
  - name: Remove old user
    user:
      name: olduser
      state: absent
      remove: yes                # Also remove home directory
  
  # Create a group
  - name: Create application group
    group:
      name: appgroup
      gid: 1500
      state: present
```

---

# 📘 PART 5: VARIABLES IN DEPTH

## Chapter 7: Understanding Variables

### 7.1 What Are Variables?

Think of variables as "labeled boxes" that hold values:

```
┌────────────────────────┐
│ Box Label: http_port   │
│ Contents: 80           │
└────────────────────────┘

┌────────────────────────┐
│ Box Label: app_name    │
│ Contents: "myapp"      │
└────────────────────────┘

┌────────────────────────┐
│ Box Label: servers     │
│ Contents: ["web1",     │
│            "web2",     │
│            "web3"]     │
└────────────────────────┘
```

### 7.2 Variable Types

```yaml
# ═══ STRINGS ═══
app_name: "myapp"
environment: production
server_name: web1.example.com

# ═══ NUMBERS ═══
http_port: 80
max_connections: 100
timeout: 30.5               # Float

# ═══ BOOLEANS ═══
debug_mode: true            # or yes, on
ssl_enabled: false          # or no, off

# ═══ LISTS (Arrays) ═══
packages:
  - nginx
  - vim
  - git
# Or inline: packages: ['nginx', 'vim', 'git']

allowed_ips:
  - 192.168.1.0/24
  - 10.0.0.0/8

# ═══ DICTIONARIES (Maps) ═══
database:
  host: db.example.com
  port: 5432
  name: mydb
  user: dbuser

# Nested dictionaries
app_config:
  server:
    host: 0.0.0.0
    port: 8080
  logging:
    level: info
    path: /var/log/app
```

### 7.3 Using Variables in Playbooks

```yaml
# Reference variables with {{ }}
- hosts: webservers
  vars:
    app_name: myapp
    http_port: 80
    database:
      host: db.example.com
      port: 5432
    servers:
      - web1
      - web2
      - web3
  
  tasks:
    # Simple variable
    - debug:
        msg: "Deploying {{ app_name }} on port {{ http_port }}"
    
    # Dictionary access (dot notation)
    - debug:
        msg: "Database: {{ database.host }}:{{ database.port }}"
    
    # Dictionary access (bracket notation)
    - debug:
        msg: "Database: {{ database['host'] }}"
    
    # List access
    - debug:
        msg: "First server: {{ servers[0] }}"
    
    # List length
    - debug:
        msg: "Total servers: {{ servers | length }}"
```

### 7.4 Where Variables Can Be Defined

```
Priority (Lowest to Highest):
═══════════════════════════════════════════════════════════

1. Role defaults (roles/myrole/defaults/main.yml)
   └── Meant to be overridden, lowest priority

2. Inventory file variables
   └── [webservers:vars]

3. Inventory group_vars/all
   └── group_vars/all.yml

4. Inventory group_vars/*
   └── group_vars/webservers.yml

5. Inventory host_vars/*
   └── host_vars/web1.yml

6. Playbook vars
   └── vars: in playbook

7. Playbook vars_files
   └── vars_files: - vars.yml

8. Role vars (roles/myrole/vars/main.yml)
   └── NOT meant to be overridden

9. Task vars
   └── vars: in task

10. Extra vars (-e or --extra-vars)
    └── ALWAYS WINS - highest priority!

═══════════════════════════════════════════════════════════
```

**Practical Example:**
```
inventory/
├── hosts.yml
├── group_vars/
│   ├── all.yml           # Applied to ALL hosts
│   └── webservers.yml    # Applied to webservers group
└── host_vars/
    └── web1.example.com.yml  # Applied to this specific host

playbook.yml
├── vars:                  # In the playbook itself
│   └── http_port: 80
└── vars_files:
    └── secrets.yml        # Loaded from file
```

### 7.5 Facts - Automatic Variables

Facts are information Ansible gathers about each host:

```yaml
# Facts are automatically available after gather_facts
- hosts: all
  tasks:
    - debug:
        msg: |
          Hostname: {{ ansible_hostname }}
          FQDN: {{ ansible_fqdn }}
          OS: {{ ansible_distribution }} {{ ansible_distribution_version }}
          IP: {{ ansible_default_ipv4.address }}
          Memory: {{ ansible_memtotal_mb }} MB
          CPU Cores: {{ ansible_processor_cores }}
```

**View all facts for a host:**
```bash
ansible web1.example.com -m setup
```

**Common facts you'll use:**
| Fact | Example Value | Use Case |
|------|---------------|----------|
| `ansible_hostname` | `web1` | Naming files |
| `ansible_fqdn` | `web1.example.com` | SSL certs |
| `ansible_distribution` | `Ubuntu` | OS-specific tasks |
| `ansible_distribution_version` | `22.04` | Version checks |
| `ansible_os_family` | `Debian` | Package managers |
| `ansible_default_ipv4.address` | `192.168.1.10` | Network config |
| `ansible_memtotal_mb` | `16384` | Sizing decisions |
| `ansible_processor_cores` | `4` | Worker configs |

### 7.6 Registered Variables

Capture task output for later use:

```yaml
tasks:
  # Run a command and capture output
  - name: Get current date
    command: date +%Y%m%d
    register: current_date
  # 'current_date' now holds the command output
  
  - name: Show captured date
    debug:
      msg: "Today is {{ current_date.stdout }}"
  
  # Registered variable structure:
  # current_date:
  #   stdout: "20240115"
  #   stdout_lines: ["20240115"]
  #   stderr: ""
  #   rc: 0  (return code)
  #   changed: true
  #   failed: false
  
  # Use in conditions
  - name: Check if file exists
    stat:
      path: /etc/app/config.yml
    register: config_file
  
  - name: Create config if missing
    template:
      src: config.yml.j2
      dest: /etc/app/config.yml
    when: not config_file.stat.exists
```

---

# 📘 PART 6: CONDITIONALS AND LOOPS

## Chapter 8: Making Decisions with `when`

### 8.1 Basic Conditionals

```yaml
tasks:
  # Simple equality
  - name: Install apt packages on Debian
    apt:
      name: nginx
    when: ansible_os_family == "Debian"
  
  - name: Install yum packages on RedHat
    yum:
      name: nginx
    when: ansible_os_family == "RedHat"
  
  # Not equal
  - name: Run on all except production
    debug:
      msg: "Not production"
    when: environment != "production"
  
  # Numeric comparisons
  - name: Warn if low memory
    debug:
      msg: "Low memory warning!"
    when: ansible_memtotal_mb < 1024
  
  # Check if variable is defined
  - name: Use custom port if defined
    debug:
      msg: "Using port {{ custom_port }}"
    when: custom_port is defined
  
  # Check if variable is undefined
  - name: Use default port
    debug:
      msg: "Using default port 80"
    when: custom_port is not defined
```

### 8.2 Multiple Conditions

```yaml
tasks:
  # AND - all conditions must be true (list format)
  - name: Install on Ubuntu 22.04 only
    apt:
      name: special-package
    when:
      - ansible_distribution == "Ubuntu"
      - ansible_distribution_version == "22.04"
  
  # OR - any condition can be true
  - name: Install on Debian OR Ubuntu
    apt:
      name: nginx
    when: ansible_distribution == "Debian" or ansible_distribution == "Ubuntu"
  
  # Complex conditions with parentheses
  - name: Complex condition
    debug:
      msg: "Matches!"
    when: >
      (environment == "production" and datacenter == "us-east")
      or
      (environment == "staging" and datacenter == "us-west")
```

### 8.3 Conditionals with Registered Variables

```yaml
tasks:
  - name: Check if service exists
    command: systemctl status myapp
    register: service_check
    ignore_errors: yes        # Don't fail if service doesn't exist
  
  - name: Install if service missing
    apt:
      name: myapp
    when: service_check.rc != 0   # rc = return code
  
  - name: Check file content
    command: cat /etc/myapp.conf
    register: config_content
  
  - name: Add config if missing
    lineinfile:
      path: /etc/myapp.conf
      line: "enable_feature=true"
    when: "'enable_feature' not in config_content.stdout"
```

## Chapter 9: Loops - Doing Things Repeatedly

### 9.1 Basic Loops

```yaml
tasks:
  # Simple list loop
  - name: Install multiple packages
    apt:
      name: "{{ item }}"
      state: present
    loop:
      - nginx
      - vim
      - git
      - curl
    # Runs 4 times, "item" changes each time
  
  # Better: pass entire list to apt (more efficient)
  - name: Install packages (optimized)
    apt:
      name: "{{ packages }}"
      state: present
    vars:
      packages:
        - nginx
        - vim
        - git
```

### 9.2 Loops with Dictionaries

```yaml
tasks:
  # Loop over list of dictionaries
  - name: Create users
    user:
      name: "{{ item.name }}"
      uid: "{{ item.uid }}"
      group: "{{ item.group }}"
    loop:
      - { name: 'alice', uid: 1001, group: 'developers' }
      - { name: 'bob', uid: 1002, group: 'developers' }
      - { name: 'charlie', uid: 1003, group: 'admin' }
  
  # Same thing, cleaner format
  - name: Create users (cleaner)
    user:
      name: "{{ item.name }}"
      uid: "{{ item.uid }}"
      group: "{{ item.group }}"
    loop:
      - name: alice
        uid: 1001
        group: developers
      - name: bob
        uid: 1002
        group: developers
```

### 9.3 Loop Control

```yaml
tasks:
  # Custom loop variable name
  - name: Create users with custom var
    user:
      name: "{{ user.name }}"
    loop: "{{ users }}"
    loop_control:
      loop_var: user          # Use 'user' instead of 'item'
  
  # Show loop index
  - name: Install with index
    debug:
      msg: "Installing package {{ index }}: {{ pkg }}"
    loop:
      - nginx
      - vim
    loop_control:
      index_var: index        # 0, 1, 2, ...
      loop_var: pkg
  
  # Pause between iterations
  - name: Deploy with delay
    include_tasks: deploy_server.yml
    loop: "{{ servers }}"
    loop_control:
      pause: 5                # 5 second pause between each
```

### 9.4 Special Loops

```yaml
tasks:
  # Loop over dictionary
  - name: Set sysctl values
    sysctl:
      name: "{{ item.key }}"
      value: "{{ item.value }}"
    loop: "{{ sysctl_config | dict2items }}"
    vars:
      sysctl_config:
        net.ipv4.ip_forward: 1
        net.core.somaxconn: 65535
  
  # Loop n times
  - name: Create numbered directories
    file:
      path: "/data/partition{{ item }}"
      state: directory
    loop: "{{ range(1, 6) | list }}"   # 1, 2, 3, 4, 5
  
  # Nested loops
  - name: Create user directories
    file:
      path: "/home/{{ item.0 }}/{{ item.1 }}"
      state: directory
    loop: "{{ users | product(directories) | list }}"
    vars:
      users: ['alice', 'bob']
      directories: ['documents', 'downloads']
    # Creates: /home/alice/documents, /home/alice/downloads,
    #          /home/bob/documents, /home/bob/downloads
```

---

# 📘 PART 7: TEMPLATES (JINJA2)

## Chapter 10: Dynamic Configuration with Templates

### 10.1 Why Templates?

**Problem:** You need different config for each server:
```
# web1 needs:
server_name web1.example.com;
listen 192.168.1.10:80;

# web2 needs:
server_name web2.example.com;
listen 192.168.1.11:80;
```

**Without templates:** Create separate config files for each server (nightmare!)

**With templates:** One template, Ansible fills in the blanks:
```nginx
# templates/nginx.conf.j2
server_name {{ ansible_hostname }}.example.com;
listen {{ ansible_default_ipv4.address }}:{{ http_port }};
```

### 10.2 Template Syntax

```jinja2
{# templates/app.conf.j2 #}

{# COMMENTS - won't appear in output #}
{# This is a template for the application config #}

{# VARIABLES - replaced with values #}
app_name = {{ app_name }}
environment = {{ environment }}
debug = {{ debug | lower }}

{# CONDITIONALS #}
{% if environment == 'production' %}
log_level = warn
ssl_enabled = true
{% elif environment == 'staging' %}
log_level = info
ssl_enabled = true
{% else %}
log_level = debug
ssl_enabled = false
{% endif %}

{# LOOPS #}
# Backend servers
{% for server in backend_servers %}
upstream_{{ loop.index }} = {{ server }}
{% endfor %}

{# WITH LIST #}
allowed_ips = {% for ip in allowed_ips %}{{ ip }}{% if not loop.last %}, {% endif %}{% endfor %}

{# Output: allowed_ips = 192.168.1.1, 192.168.1.2, 192.168.1.3 #}
```

### 10.3 Filters - Transform Data

```jinja2
{# STRING FILTERS #}
hostname = {{ server_name | upper }}           {# MYSERVER #}
lowercase = {{ server_name | lower }}          {# myserver #}
path = {{ base_path | trim }}                  {# Remove whitespace #}

{# DEFAULT VALUES #}
port = {{ custom_port | default(8080) }}       {# Use 8080 if undefined #}
name = {{ app_name | default('myapp', true) }} {# true = also for empty #}

{# MATH #}
workers = {{ ansible_processor_cores * 2 }}
memory = {{ ansible_memtotal_mb // 1024 }}GB   {# Integer division #}

{# LIST OPERATIONS #}
servers = {{ server_list | join(', ') }}       {# item1, item2, item3 #}
first = {{ servers | first }}                  {# First item #}
count = {{ servers | length }}                 {# Number of items #}

{# JSON/YAML #}
config = {{ settings | to_json }}              {# As JSON #}
config = {{ settings | to_nice_json }}         {# Pretty JSON #}
config = {{ settings | to_yaml }}              {# As YAML #}

{# FILE OPERATIONS #}
{% set content = lookup('file', '/path/to/file') %}
{{ content | b64encode }}                      {# Base64 encode #}

{# REGEX #}
{{ hostname | regex_replace('^web', 'app') }}  {# Replace pattern #}
{{ text | regex_search('version: (\d+)') }}    {# Find pattern #}
```

### 10.4 Complete Template Example

```jinja2
{# templates/nginx-site.conf.j2 #}
# Nginx configuration for {{ app_name }}
# Generated by Ansible on {{ ansible_date_time.date }}
# DO NOT EDIT MANUALLY

{% if ssl_enabled %}
server {
    listen 80;
    server_name {{ domain_name }};
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name {{ domain_name }};
    
    ssl_certificate {{ ssl_cert_path }};
    ssl_certificate_key {{ ssl_key_path }};
    ssl_protocols TLSv1.2 TLSv1.3;
{% else %}
server {
    listen 80;
    server_name {{ domain_name }};
{% endif %}
    
    root {{ document_root | default('/var/www/html') }};
    index index.html index.php;
    
    # Logging
    access_log /var/log/nginx/{{ app_name }}-access.log;
    error_log /var/log/nginx/{{ app_name }}-error.log {{ log_level | default('warn') }};
    
    # PHP handling
{% if php_enabled | default(false) %}
    location ~ \.php$ {
        fastcgi_pass unix:/run/php/php{{ php_version }}-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
{% endif %}
    
    # Upstream backends
{% if backend_servers is defined and backend_servers | length > 0 %}
    upstream backend {
{% for server in backend_servers %}
        server {{ server.host }}:{{ server.port | default(8080) }} weight={{ server.weight | default(1) }};
{% endfor %}
    }
    
    location /api {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
{% endif %}
}
```

**Using the template:**
```yaml
- name: Deploy nginx site config
  template:
    src: nginx-site.conf.j2
    dest: /etc/nginx/sites-available/{{ app_name }}
    owner: root
    group: root
    mode: '0644'
  vars:
    app_name: myapp
    domain_name: myapp.example.com
    ssl_enabled: true
    ssl_cert_path: /etc/ssl/certs/myapp.crt
    ssl_key_path: /etc/ssl/private/myapp.key
    php_enabled: true
    php_version: "8.1"
    backend_servers:
      - { host: "10.0.1.1", port: 8080, weight: 3 }
      - { host: "10.0.1.2", port: 8080, weight: 2 }
  notify: Reload nginx
```

---

# 📘 PART 8: ROLES - REUSABLE AUTOMATION

## Chapter 11: Understanding Roles

### 11.1 Why Roles?

**Problem:** Your playbook is getting HUGE and hard to manage:
```yaml
# One massive playbook.yml - 500 lines!
- hosts: all
  tasks:
    # 50 tasks for base configuration
    # 30 tasks for nginx
    # 40 tasks for database
    # 25 tasks for monitoring
    # ... hundreds more
```

**Solution:** Break into reusable "roles":
```yaml
# Clean and simple!
- hosts: all
  roles:
    - common        # Base configuration
    - nginx         # Web server setup
    - postgresql    # Database setup
    - monitoring    # Monitoring agents
```

### 11.2 Role Directory Structure

```
roles/
└── nginx/                    # Role name
    ├── README.md             # Documentation
    │
    ├── defaults/
    │   └── main.yml          # Default variables (lowest priority)
    │                         # Users SHOULD override these
    │
    ├── vars/
    │   └── main.yml          # Role variables (higher priority)
    │                         # Users should NOT override these
    │
    ├── tasks/
    │   ├── main.yml          # Main tasks (entry point)
    │   ├── install.yml       # Installation tasks
    │   └── configure.yml     # Configuration tasks
    │
    ├── handlers/
    │   └── main.yml          # Event handlers
    │
    ├── templates/
    │   └── nginx.conf.j2     # Jinja2 templates
    │
    ├── files/
    │   ├── index.html        # Static files to copy
    │   └── ssl.crt
    │
    └── meta/
        └── main.yml          # Role metadata & dependencies
```

### 11.3 Creating a Role - Step by Step

```bash
# Create role skeleton
ansible-galaxy init roles/nginx

# Or create manually:
mkdir -p roles/nginx/{tasks,handlers,templates,files,vars,defaults,meta}
```

**roles/nginx/defaults/main.yml:**
```yaml
---
# These are DEFAULTS - users can override them
nginx_port: 80
nginx_worker_processes: auto
nginx_worker_connections: 1024
nginx_server_name: localhost
nginx_root: /var/www/html
nginx_enabled: true
```

**roles/nginx/vars/main.yml:**
```yaml
---
# These should NOT be overridden
nginx_package: nginx
nginx_service: nginx
nginx_config_path: /etc/nginx
nginx_user: www-data
```

**roles/nginx/tasks/main.yml:**
```yaml
---
# Main entry point - include other task files
- name: Include installation tasks
  include_tasks: install.yml

- name: Include configuration tasks
  include_tasks: configure.yml
  when: nginx_enabled

- name: Include service tasks
  include_tasks: service.yml
```

**roles/nginx/tasks/install.yml:**
```yaml
---
- name: Install nginx package
  apt:
    name: "{{ nginx_package }}"
    state: present
    update_cache: yes

- name: Ensure nginx directories exist
  file:
    path: "{{ item }}"
    state: directory
    owner: "{{ nginx_user }}"
  loop:
    - "{{ nginx_root }}"
    - /etc/nginx/sites-available
    - /etc/nginx/sites-enabled
```

**roles/nginx/tasks/configure.yml:**
```yaml
---
- name: Deploy nginx.conf
  template:
    src: nginx.conf.j2
    dest: "{{ nginx_config_path }}/nginx.conf"
  notify: Reload nginx

- name: Deploy default site
  template:
    src: default-site.conf.j2
    dest: /etc/nginx/sites-available/default
  notify: Reload nginx

- name: Enable default site
  file:
    src: /etc/nginx/sites-available/default
    dest: /etc/nginx/sites-enabled/default
    state: link
  notify: Reload nginx
```

**roles/nginx/handlers/main.yml:**
```yaml
---
- name: Reload nginx
  service:
    name: "{{ nginx_service }}"
    state: reloaded

- name: Restart nginx
  service:
    name: "{{ nginx_service }}"
    state: restarted
```

**roles/nginx/meta/main.yml:**
```yaml
---
galaxy_info:
  author: Your Name
  description: Install and configure nginx
  license: MIT
  min_ansible_version: 2.9
  platforms:
    - name: Ubuntu
      versions:
        - focal
        - jammy

# Dependencies - roles that must run first
dependencies:
  - role: common
  - role: firewall
    vars:
      firewall_open_ports:
        - 80
        - 443
```

### 11.4 Using Roles

```yaml
# playbook.yml
---
- hosts: webservers
  become: true
  
  # Simple usage
  roles:
    - common
    - nginx
    - php
  
  # With variables
  roles:
    - common
    - role: nginx
      vars:
        nginx_port: 8080
        nginx_server_name: myapp.example.com
    - role: php
      vars:
        php_version: "8.1"
  
  # Conditional roles
  roles:
    - common
    - role: nginx
      when: "'webserver' in group_names"
    - role: postgresql
      when: "'dbserver' in group_names"
```

---

# 📘 PART 9: HANDLERS - EVENT-DRIVEN TASKS

## Chapter 12: Understanding Handlers

### 12.1 What Are Handlers?

Handlers are tasks that run ONLY when triggered and run ONCE at the end of a play.

**Scenario:** You update 5 config files for nginx. Without handlers:
```
Task 1: Update nginx.conf → Restart nginx
Task 2: Update site.conf → Restart nginx
Task 3: Update ssl.conf → Restart nginx
Task 4: Update logging.conf → Restart nginx
Task 5: Update proxy.conf → Restart nginx
= NGINX RESTARTS 5 TIMES! (Bad for users)
```

**With handlers:**
```
Task 1: Update nginx.conf → notify: Restart nginx
Task 2: Update site.conf → notify: Restart nginx
Task 3: Update ssl.conf → notify: Restart nginx
Task 4: Update logging.conf → notify: Restart nginx
Task 5: Update proxy.conf → notify: Restart nginx
... (at end of play)
Handler: Restart nginx → RUNS ONCE!
```

### 12.2 Handler Examples

```yaml
---
- hosts: webservers
  become: true
  
  tasks:
    - name: Update nginx.conf
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: Restart nginx        # Trigger handler
    
    - name: Update site config
      template:
        src: site.conf.j2
        dest: /etc/nginx/sites-available/mysite
      notify:
        - Reload nginx             # Multiple handlers
        - Clear nginx cache
    
    - name: Update SSL certificate
      copy:
        src: ssl.crt
        dest: /etc/nginx/ssl/
      notify: Restart nginx
  
  handlers:
    - name: Restart nginx
      service:
        name: nginx
        state: restarted
    
    - name: Reload nginx
      service:
        name: nginx
        state: reloaded
    
    - name: Clear nginx cache
      file:
        path: /var/cache/nginx
        state: absent
```

### 12.3 Important Handler Behaviors

```yaml
# 1. Handlers run in ORDER DEFINED, not notification order
handlers:
  - name: Handler A          # Runs first (defined first)
    debug: msg="A"
  - name: Handler B          # Runs second
    debug: msg="B"

# 2. Handlers only run if CHANGED
tasks:
  - name: This task doesn't change anything
    apt:
      name: nginx
      state: present         # Already installed
    notify: Restart nginx    # Handler WON'T run

# 3. Force handlers to run mid-play
tasks:
  - name: Deploy config
    template:
      src: config.j2
      dest: /etc/app/config
    notify: Restart app
  
  - meta: flush_handlers     # Run handlers NOW!
  
  - name: Test the app       # App is already restarted
    uri:
      url: http://localhost/health

# 4. Listen directive (multiple handlers for one event)
tasks:
  - name: Deploy application
    copy: ...
    notify: "deploy complete"    # Generic event name

handlers:
  - name: Restart app
    service: name=app state=restarted
    listen: "deploy complete"    # All three run!
    
  - name: Clear cache
    command: /usr/bin/clear-cache
    listen: "deploy complete"
    
  - name: Send notification
    mail: ...
    listen: "deploy complete"
```

---

# 📘 PART 10: ERROR HANDLING

## Chapter 13: Handling Failures Gracefully

### 13.1 Basic Error Handling

```yaml
tasks:
  # Ignore errors and continue
  - name: Try to remove old package
    apt:
      name: old-package
      state: absent
    ignore_errors: yes        # Continue even if fails
  
  # Custom failure condition
  - name: Run health check
    command: /usr/bin/health-check
    register: health
    failed_when: health.rc != 0 and health.rc != 2
    # Fail only if return code is not 0 or 2
  
  # Custom changed condition
  - name: Check status
    command: /usr/bin/check-status
    register: status
    changed_when: "'modified' in status.stdout"
    # Only report "changed" if output contains "modified"
  
  # Never report as failed
  - name: Optional task
    command: /usr/bin/optional-tool
    failed_when: false        # Never fails
```

### 13.2 Blocks for Error Handling

```yaml
tasks:
  - name: Deployment with error handling
    block:
      # Try these tasks
      - name: Stop application
        service:
          name: myapp
          state: stopped
      
      - name: Deploy new version
        copy:
          src: app-v2.tar.gz
          dest: /opt/app/
      
      - name: Start application
        service:
          name: myapp
          state: started
    
    rescue:
      # If ANY task in block fails, run these
      - name: Log failure
        command: logger "Deployment failed, rolling back"
      
      - name: Restore previous version
        copy:
          src: /opt/app/backup/
          dest: /opt/app/current/
      
      - name: Start old version
        service:
          name: myapp
          state: started
      
      - name: Alert team
        mail:
          to: ops@company.com
          subject: "Deployment Failed"
    
    always:
      # ALWAYS run these, regardless of success/failure
      - name: Cleanup temp files
        file:
          path: /tmp/deploy/
          state: absent
      
      - name: Log completion
        command: logger "Deployment process completed"
```

### 13.3 Retrying Tasks

```yaml
tasks:
  # Retry until successful
  - name: Wait for application to start
    uri:
      url: http://localhost:8080/health
      status_code: 200
    register: result
    until: result.status == 200     # Keep trying until true
    retries: 30                     # Maximum attempts
    delay: 10                       # Seconds between attempts
    # Total max wait: 30 * 10 = 300 seconds (5 minutes)
  
  # Retry with different condition
  - name: Wait for file to appear
    stat:
      path: /tmp/ready.flag
    register: flag
    until: flag.stat.exists
    retries: 60
    delay: 5
```

---

# 📘 PART 11: ANSIBLE VAULT - SECRETS MANAGEMENT

## Chapter 14: Keeping Secrets Safe

### 14.1 Why Vault?

You need passwords, API keys, and certificates in your playbooks. But you CANNOT put them in plain text:

```yaml
# NEVER DO THIS!
database_password: supersecretpassword123
aws_secret_key: AKIAIOSFODNN7EXAMPLE
```

If you commit this to Git, everyone sees your secrets!

### 14.2 How Vault Works

```
┌─────────────────────────────────────┐
│     secrets.yml (PLAIN TEXT)       │
│                                     │
│ database_password: secret123        │
│ api_key: abc123xyz                  │
└─────────────────────────────────────┘
                 │
                 ▼ ansible-vault encrypt
┌─────────────────────────────────────┐
│     secrets.yml (ENCRYPTED)        │
│                                     │
│ $ANSIBLE_VAULT;1.1;AES256          │
│ 38613033346465636234663136383366... │
│ 62343639626234383134623432326162... │
└─────────────────────────────────────┘
                 │
                 ▼ Safe to commit to Git!
```

### 14.3 Vault Commands

```bash
# ══════════ CREATE / ENCRYPT ══════════

# Create new encrypted file (prompts for password)
ansible-vault create secrets.yml

# Encrypt existing file
ansible-vault encrypt existing_file.yml

# Encrypt string (for use in playbooks)
ansible-vault encrypt_string 'my_secret_password' --name 'db_password'
# Output:
# db_password: !vault |
#   $ANSIBLE_VAULT;1.1;AES256
#   38613033...


# ══════════ VIEW / EDIT ══════════

# View encrypted file (shows contents, stays encrypted)
ansible-vault view secrets.yml

# Edit encrypted file (decrypts to editor, re-encrypts on save)
ansible-vault edit secrets.yml


# ══════════ DECRYPT ══════════

# Permanently decrypt file
ansible-vault decrypt secrets.yml


# ══════════ CHANGE PASSWORD ══════════

# Change vault password
ansible-vault rekey secrets.yml
```

### 14.4 Using Vault in Playbooks

**Method 1: Separate vault file**
```yaml
# group_vars/production/vault.yml (encrypted)
vault_db_password: "supersecret"
vault_api_key: "abc123"

# group_vars/production/vars.yml (not encrypted)
db_password: "{{ vault_db_password }}"
api_key: "{{ vault_api_key }}"
```

**Method 2: Inline encrypted values**
```yaml
# vars.yml
db_user: myapp                    # Normal variable
db_password: !vault |             # Encrypted value
  $ANSIBLE_VAULT;1.1;AES256
  38613033346465636234663136383366
  62343639626234383134623432326162
```

**Running playbooks with vault:**
```bash
# Prompt for password
ansible-playbook site.yml --ask-vault-pass

# Use password file
echo "mypassword" > ~/.vault_pass
chmod 600 ~/.vault_pass
ansible-playbook site.yml --vault-password-file ~/.vault_pass

# Or in ansible.cfg
# [defaults]
# vault_password_file = ~/.vault_pass
```

---

# 📘 PART 12: BEST PRACTICES

## Chapter 15: Writing Clean, Maintainable Ansible

### 15.1 Project Structure

```
ansible-project/
├── ansible.cfg               # Configuration
├── requirements.yml          # Role/collection deps
│
├── inventories/
│   ├── production/
│   │   ├── hosts.yml
│   │   ├── group_vars/
│   │   │   ├── all.yml
│   │   │   └── webservers.yml
│   │   └── host_vars/
│   └── staging/
│       └── ...
│
├── playbooks/
│   ├── site.yml             # Main playbook
│   ├── webservers.yml
│   └── databases.yml
│
├── roles/
│   ├── common/
│   ├── nginx/
│   └── postgresql/
│
└── files/                    # Global static files
```

### 15.2 Naming Conventions

```yaml
# ✅ GOOD variable names
http_port: 80
database_connection_timeout: 30
enable_ssl_redirect: true
app_deployment_path: /opt/myapp

# ❌ BAD variable names
p: 80                          # Too short
DatabasePort: 3306             # CamelCase
enable-ssl: true               # Hyphens not allowed
app path: /opt/myapp           # Spaces not allowed

# ✅ GOOD task names
- name: Install nginx web server package
- name: Create application user with restricted shell
- name: Deploy nginx configuration from template

# ❌ BAD task names
- name: Install                # Too vague
- name: Do stuff               # Not descriptive
- apt: name=nginx              # Missing name entirely
```

### 15.3 Security Best Practices

```yaml
# ✅ Use vault for secrets
db_password: "{{ vault_db_password }}"

# ❌ Never hardcode secrets
db_password: "plaintext_password"

# ✅ Use no_log for sensitive tasks
- name: Set database password
  mysql_user:
    name: admin
    password: "{{ db_password }}"
  no_log: true                 # Hides output in logs

# ✅ Limit permissions
- name: Deploy config file
  template:
    src: config.j2
    dest: /etc/app/config.yml
    owner: root
    group: app
    mode: '0640'               # Owner rw, group r, others none
```

### 15.4 Performance Tips

```yaml
# 1. Disable facts when not needed
- hosts: all
  gather_facts: false
  tasks: [...]

# 2. Use free strategy for independent tasks
- hosts: all
  strategy: free               # Each host proceeds independently

# 3. Use async for long-running tasks
- name: Run long backup
  command: /backup.sh
  async: 3600                  # Max 1 hour
  poll: 0                      # Don't wait

# 4. Combine package installations
# ❌ Slow (runs apt 3 times)
- apt: name=nginx
- apt: name=vim
- apt: name=git

# ✅ Fast (runs apt once)
- apt:
    name:
      - nginx
      - vim
      - git
```

---

# 📘 APPENDIX: QUICK REFERENCE

## Essential Commands

```bash
# ═══ AD-HOC COMMANDS ═══
ansible all -m ping                    # Test connectivity
ansible webservers -m shell -a "uptime"  # Run command
ansible all -m setup                   # Gather facts

# ═══ PLAYBOOK EXECUTION ═══
ansible-playbook site.yml              # Run playbook
ansible-playbook site.yml -l web1      # Limit to host
ansible-playbook site.yml --check      # Dry run
ansible-playbook site.yml --diff       # Show changes
ansible-playbook site.yml -v           # Verbose (-vvv = more)
ansible-playbook site.yml --tags "install"  # Only tagged tasks
ansible-playbook site.yml --skip-tags "test"  # Skip tagged tasks

# ═══ VAULT ═══
ansible-vault create secrets.yml       # Create encrypted file
ansible-vault edit secrets.yml         # Edit encrypted file
ansible-vault view secrets.yml         # View contents
ansible-vault encrypt file.yml         # Encrypt existing
ansible-vault decrypt file.yml         # Decrypt file

# ═══ ROLES & COLLECTIONS ═══
ansible-galaxy init roles/myrole       # Create role skeleton
ansible-galaxy install geerlingguy.nginx  # Install role
ansible-galaxy collection install amazon.aws  # Install collection

# ═══ INVENTORY ═══
ansible-inventory --list              # Show inventory as JSON
ansible-inventory --graph             # Show inventory tree
ansible-inventory --host web1         # Show host variables
```

## Module State Values

| Module | `present` | `absent` | `latest` | `started` | `stopped` |
|--------|-----------|----------|----------|-----------|-----------|
| apt/yum | Install | Remove | Latest version | N/A | N/A |
| service | N/A | N/A | N/A | Start | Stop |
| file | Create | Delete | N/A | N/A | N/A |
| user | Create | Delete | N/A | N/A | N/A |

## Common Facts

```yaml
ansible_hostname          # web1
ansible_fqdn              # web1.example.com
ansible_distribution      # Ubuntu
ansible_distribution_version  # 22.04
ansible_os_family         # Debian
ansible_default_ipv4.address  # 192.168.1.10
ansible_memtotal_mb       # 16384
ansible_processor_cores   # 4
```

---

**Congratulations!** You now have a comprehensive understanding of Ansible. Practice by building real projects, and refer back to this guide whenever needed.

