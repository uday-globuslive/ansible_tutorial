# Ansible Interview Preparation - Complete Supplement

This guide covers topics **NOT** in your tutorial plus **beginner-friendly explanations** of concepts.

---

## 🔰 Part 1: Concepts Explained Like You're 5

### What is Ansible, Really?

**Simple Analogy**: Imagine you're a restaurant manager with 100 restaurants. Every day you need to:
- Make sure all kitchens are clean
- Ensure the same menu is displayed
- Verify all staff are trained the same way

**Without Ansible**: You drive to each restaurant, one by one, and do everything manually. Takes forever!

**With Ansible**: You write down instructions once ("clean kitchen, display menu, train staff"), and a magic messenger delivers and executes them at all 100 restaurants simultaneously.

```
Your Laptop (Control Node) = Manager's Office
Your Instructions (Playbook) = The To-Do List
Restaurants (Managed Nodes) = Your 100 Servers
SSH = The Magic Messenger
```

### What Does "Agentless" Actually Mean?

**Other tools (Puppet, Chef)**: 
- You need to install a special program ("agent") on EVERY server you want to manage
- These agents constantly check in with a central server
- More software to maintain, more things that can break

**Ansible (Agentless)**:
- No special software needed on your servers
- Just needs SSH access (which servers already have) and Python
- Your laptop connects directly to servers when YOU run a playbook

```
PUPPET WAY:
┌────────────┐         ┌────────────┐
│  Puppet    │ ◄─────► │  Agent on  │    Agent constantly 
│  Server    │         │  Server 1  │    "calling home"
└────────────┘         └────────────┘

ANSIBLE WAY:
┌────────────┐  SSH    ┌────────────┐
│  Your      │ ─────► │  Server 1  │    You connect ONLY
│  Laptop    │        │ (no agent) │    when you need to
└────────────┘         └────────────┘
```

### What is Idempotence? (The Most Asked Interview Question!)

**Definition**: Running the same playbook multiple times produces the SAME result.

**Real-World Example**: 
Think about turning on a light switch:
- First flip: Light turns ON (change happened)
- Second flip: Light is already ON, nothing happens (no change)

```yaml
# This task is idempotent
- name: Install nginx
  apt:
    name: nginx
    state: present  # "Make sure nginx exists"
```

**Run 1**: Nginx not installed → Ansible installs it → **CHANGED**
**Run 2**: Nginx already installed → Ansible sees it's there → **OK** (no change)
**Run 3**: Same as Run 2 → **OK**

**Why it matters**: You can run playbooks safely without fear of breaking things!

### What is YAML? (The Language of Ansible)

YAML is just a way to write data that both humans and computers can read.

```yaml
# This is YAML - looks like a simple list
---
# Dashes and colons are meaningful!

# A simple key-value pair (like a labeled box)
name: John              # name = John

# A list (ordered items)
fruits:
  - apple              # First item
  - banana             # Second item
  - cherry             # Third item

# A nested structure (a box inside a box)
person:
  name: John
  age: 30
  address:
    city: New York
    zip: "10001"        # Quotes for strings with numbers
```

**YAML Rules for Beginners**:
1. **Indentation MATTERS** - Use 2 spaces (NOT tabs!)
2. **Colons create key-value pairs**: `key: value`
3. **Dashes create list items**: `- item`
4. **Start files with `---`**
5. **Comments start with `#`**

---

## 🎯 Part 2: Interview Questions NOT in Your Tutorial

### 1. Ansible Tower / AWX

**Q: What is Ansible Tower?**

**A**: Ansible Tower is Red Hat's enterprise web UI and API for Ansible.

```
Free Ansible (CLI)          vs    Ansible Tower (Enterprise)
──────────────────────            ──────────────────────────
- Run from terminal               - Web dashboard
- No built-in scheduling          - Schedule jobs
- Basic logging                   - Centralized logging
- No access control               - Role-based access control
- Manual execution                - REST API
- Free                            - Paid license
```

**AWX** = The free, open-source version of Tower (community project)

**Why interviewers ask**: Shows you understand enterprise Ansible usage.

### 2. Ansible Collections (NEW - Often Asked!)

**Q: What are Ansible Collections?**

**A**: Collections are a new way to package and distribute Ansible content.

**Before Collections** (Old way):
- Everything bundled in one massive Ansible package
- All modules came with Ansible installation
- Hard to update individual modules

**With Collections** (New way):
- Modules, roles, plugins packaged separately
- Install only what you need
- Each collection has its own version

```bash
# Installing a collection
ansible-galaxy collection install amazon.aws

# Using a module from a collection
- name: Create EC2 instance
  amazon.aws.ec2_instance:   # Format: namespace.collection.module
    name: my-instance
    state: present
```

**Popular Collections**:
| Collection | Purpose |
|------------|---------|
| `amazon.aws` | AWS resources |
| `azure.azcollection` | Azure resources |
| `community.general` | Common modules |
| `kubernetes.core` | Kubernetes management |

### 3. ansible.cfg Configuration File

**Q: What is ansible.cfg and where does Ansible look for it?**

**A**: `ansible.cfg` is the configuration file that controls Ansible's behavior.

**Search Order** (First found wins):
```
1. $ANSIBLE_CONFIG environment variable
2. ./ansible.cfg (current directory)
3. ~/.ansible.cfg (home directory)
4. /etc/ansible/ansible.cfg (system-wide)
```

**Common Settings** (Interviewers love asking!):
```ini
[defaults]
inventory = ./inventory           # Default inventory file
remote_user = deploy              # Default SSH user
private_key_file = ~/.ssh/id_rsa  # SSH key
host_key_checking = False         # Skip SSH key verification (dev only!)
retry_files_enabled = False       # Don't create .retry files
forks = 10                        # Parallel connections (default: 5)
timeout = 30                      # SSH timeout
gathering = smart                 # Fact caching
log_path = ./ansible.log          # Log file location

[privilege_escalation]
become = True                     # Use sudo by default
become_method = sudo              # How to elevate privileges
become_user = root                # User to become
become_ask_pass = False           # Don't prompt for sudo password

[ssh_connection]
pipelining = True                 # Speed up by reducing SSH operations
control_path = /tmp/ansible-%%r@%%h:%%p  # SSH socket path
```

### 4. Strategy Plugins (How Ansible Runs Tasks)

**Q: What are Ansible execution strategies?**

**A**: Strategies control HOW Ansible executes tasks across hosts.

| Strategy | Behavior | Use Case |
|----------|----------|----------|
| **linear** (default) | Runs each task on ALL hosts before moving to next task | Standard deployments |
| **free** | Each host runs all tasks independently, as fast as possible | Independent servers |
| **serial** | Runs entire playbook on X hosts at a time | Rolling updates |

**Visual Explanation**:
```
LINEAR STRATEGY (Default):
Task 1: [Host1 ───] [Host2 ───] [Host3 ───]  ← All finish Task 1
Task 2: [Host1 ───] [Host2 ───] [Host3 ───]  ← Then all do Task 2

FREE STRATEGY:
Host1: [Task1──][Task2──][Task3──]           ← Each host runs
Host2: [Task1────][Task2────]────            ← at its own pace
Host3: [Task1─][Task2─][Task3─][Task4─]      ← independently
```

```yaml
# Using strategies
- hosts: all
  strategy: free    # or 'linear' or 'serial'
  tasks:
    - name: Task 1
      ...
```

### 5. Magic Variables (Special Built-in Variables)

**Q: What are Ansible magic variables?**

**A**: Variables automatically created by Ansible that give you information about the current play.

```yaml
# Most important magic variables:

# Current host being configured
- debug: msg="{{ inventory_hostname }}"        # web1.example.com

# All groups this host belongs to
- debug: msg="{{ group_names }}"               # ['webservers', 'production']

# All hosts in a specific group
- debug: msg="{{ groups['webservers'] }}"      # ['web1', 'web2', 'web3']

# All hosts in inventory
- debug: msg="{{ groups['all'] }}"             # Every host

# Variables for another host
- debug: msg="{{ hostvars['db1']['ansible_host'] }}"

# Current play information
- debug: msg="{{ playbook_dir }}"              # /home/user/playbooks
- debug: msg="{{ role_path }}"                 # Current role's path
- debug: msg="{{ ansible_play_name }}"         # Name of current play
```

### 6. Lookup Plugins (Get Data from External Sources)

**Q: What are lookup plugins in Ansible?**

**A**: Lookups fetch data from external sources DURING playbook execution.

```yaml
# Read from a file
- debug: msg="{{ lookup('file', '/etc/hostname') }}"

# Read from environment variable
- debug: msg="{{ lookup('env', 'HOME') }}"

# Generate password
- debug: msg="{{ lookup('password', '/tmp/mypass length=12') }}"

# Read from URL
- debug: msg="{{ lookup('url', 'https://api.example.com/data') }}"

# Read from AWS parameter store
- debug: msg="{{ lookup('aws_ssm', 'my-param') }}"

# Read lines from a file
- debug: msg="{{ lookup('lines', 'cat /etc/hosts') }}"

# Read specific columns from CSV
- debug: msg="{{ lookup('csvfile', 'host1 file=data.csv delimiter=,') }}"
```

### 7. Include vs Import (VERY Common Interview Question!)

**Q: What's the difference between include and import?**

**A**:

| Aspect | **import_** | **include_** |
|--------|-------------|--------------|
| Processing | **Static** - parsed at playbook start | **Dynamic** - parsed when reached |
| Variables | Can't use variables in filenames | CAN use variables |
| Handlers | Handlers imported immediately | Handlers from included tasks work differently |
| Tags | Tags apply to imported tasks | Tags don't apply inside includes |
| Loops | CAN'T use with loops | CAN use with loops |

```yaml
# IMPORT (Static) - filename decided BEFORE play runs
- import_tasks: tasks/common.yml      # Processed at start
- import_role:
    name: webserver

# INCLUDE (Dynamic) - filename decided WHEN this line runs
- include_tasks: "tasks/{{ os_family }}.yml"   # Variable works!
- include_role:
    name: "{{ role_name }}"                    # Variable works!

# Include with loop (Import can't do this!)
- include_tasks: user_setup.yml
  loop: "{{ users }}"
  loop_control:
    loop_var: user
```

**When to use what**:
- **import_** = When you know exactly what files to include
- **include_** = When filenames depend on variables or you need loops

### 8. Async Tasks and Polling

**Q: How do you run long-running tasks in Ansible?**

**A**: Use `async` and `poll` to run tasks without waiting.

```yaml
# Problem: This task takes 10 minutes, blocks everything
- name: Long backup task
  command: /scripts/backup.sh   # Takes 600 seconds

# Solution: Run asynchronously
- name: Start long backup (don't wait)
  command: /scripts/backup.sh
  async: 1800           # Maximum runtime in seconds (30 min)
  poll: 0               # Don't wait (fire and forget)
  register: backup_job

# Later, check if it finished
- name: Check backup status
  async_status:
    jid: "{{ backup_job.ansible_job_id }}"
  register: job_result
  until: job_result.finished    # Keep checking...
  retries: 60                   # Up to 60 times
  delay: 30                     # Every 30 seconds
```

**poll values**:
- `poll: 0` = Don't wait at all (fire and forget)
- `poll: 10` = Check every 10 seconds until complete

### 9. Delegation (Run Tasks on Different Hosts)

**Q: How do you run a task on a different host than the current play target?**

**A**: Use `delegate_to`.

```yaml
- hosts: webservers
  tasks:
    # This runs on the current web server
    - name: Stop web service
      service: name=nginx state=stopped
    
    # This runs on the load balancer instead!
    - name: Remove from load balancer
      command: /remove-server.sh {{ inventory_hostname }}
      delegate_to: loadbalancer
    
    # Run on your control machine (localhost)
    - name: Send Slack notification
      uri:
        url: https://hooks.slack.com/...
        method: POST
        body: '{"text": "Deploying to {{ inventory_hostname }}"}'
      delegate_to: localhost
```

### 10. Check Mode and Diff Mode

**Q: How do you test playbooks without making changes?**

**A**: Use check mode (dry run) and diff mode.

```bash
# Check mode - shows what WOULD change (no actual changes)
ansible-playbook playbook.yml --check

# Diff mode - shows file differences
ansible-playbook playbook.yml --diff

# Both together (most useful for testing!)
ansible-playbook playbook.yml --check --diff
```

```yaml
# Force a task to ALWAYS run even in check mode
- name: Get current config
  command: cat /etc/config
  check_mode: false   # This runs even with --check

# Force a task to NEVER run in check mode
- name: Dangerous operation
  command: /dangerous-script.sh
  when: not ansible_check_mode   # Skip if in check mode
```

### 11. Tags (Selective Task Execution)

**Q: How do you run only specific parts of a playbook?**

**A**: Use tags to label and selectively run tasks.

```yaml
- hosts: webservers
  tasks:
    - name: Install packages
      apt: name=nginx state=present
      tags: 
        - packages
        - install
    
    - name: Configure nginx
      template: src=nginx.conf.j2 dest=/etc/nginx/nginx.conf
      tags:
        - config
        - nginx
    
    - name: Start nginx
      service: name=nginx state=started
      tags:
        - service
        - always     # 'always' tag = runs EVERY time
```

```bash
# Run only tasks with specific tags
ansible-playbook playbook.yml --tags "config"
ansible-playbook playbook.yml --tags "install,config"

# Skip specific tags
ansible-playbook playbook.yml --skip-tags "service"

# List all tags in playbook
ansible-playbook playbook.yml --list-tags
```

### 12. Blocks (Group Tasks Together)

**Q: How do you group tasks and handle errors together?**

**A**: Use blocks for grouping, error handling, and shared attributes.

```yaml
- hosts: webservers
  tasks:
    - name: Deployment block
      block:
        # All tasks in this block share the 'when' and 'become'
        - name: Stop service
          service: name=myapp state=stopped
        
        - name: Deploy code
          copy: src=app/ dest=/opt/app/
        
        - name: Start service
          service: name=myapp state=started
      
      rescue:
        # Runs if ANY task in 'block' fails
        - name: Rollback deployment
          command: /scripts/rollback.sh
        
        - name: Send alert
          mail:
            subject: "Deployment failed!"
            to: admin@example.com
      
      always:
        # ALWAYS runs, regardless of success/failure
        - name: Clean up temp files
          file: path=/tmp/deploy state=absent
        
        - name: Log result
          command: logger "Deployment completed"
      
      when: env == 'production'   # Applies to entire block
      become: true                # Applies to entire block
```

### 13. Ansible Facts Deep Dive

**Q: What are facts and how do you use them?**

**A**: Facts are system information gathered from managed nodes at the start of each play.

```yaml
# Facts are automatically gathered (unless disabled)
- hosts: all
  gather_facts: true   # Default, can be 'false' to skip
  
  tasks:
    # Use facts in conditions
    - name: Install on Debian only
      apt: name=nginx
      when: ansible_os_family == "Debian"
    
    # Use facts in templates
    - name: Deploy config
      template: src=config.j2 dest=/etc/config
```

**Commonly Used Facts** (Know these for interviews!):
```yaml
# System info
ansible_hostname            # "webserver1"
ansible_fqdn                # "webserver1.example.com"
ansible_distribution        # "Ubuntu"
ansible_distribution_version # "22.04"
ansible_os_family           # "Debian" or "RedHat"

# Network
ansible_default_ipv4.address # "192.168.1.10"
ansible_all_ipv4_addresses  # ["192.168.1.10", "10.0.0.5"]

# Hardware
ansible_memtotal_mb         # 16384
ansible_processor_cores     # 4
ansible_processor_count     # 2

# Storage
ansible_mounts              # List of mounted filesystems
ansible_devices             # Block devices
```

**Custom Facts** (Put files in `/etc/ansible/facts.d/`):
```bash
# On managed node: /etc/ansible/facts.d/custom.fact
[app]
version=1.5.0
environment=production
```
```yaml
# Access in playbook:
- debug: msg="{{ ansible_local.custom.app.version }}"
```

### 14. Performance Tuning

**Q: How do you speed up Ansible playbooks?**

**A**: Multiple strategies:

```ini
# ansible.cfg optimizations
[defaults]
forks = 50                    # Run on 50 hosts simultaneously (default: 5)
gathering = smart             # Cache facts
fact_caching = jsonfile
fact_caching_connection = /tmp/facts_cache
fact_caching_timeout = 86400  # Cache for 24 hours

[ssh_connection]
pipelining = True             # HUGE speed improvement
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
```

```yaml
# Skip fact gathering when not needed
- hosts: all
  gather_facts: false
  tasks:
    - debug: msg="No facts needed here"

# Use free strategy for independent tasks
- hosts: all
  strategy: free
  tasks:
    - name: Independent task
      ...
```

### 15. Testing with Molecule

**Q: How do you test Ansible roles?**

**A**: Molecule is the standard testing framework.

```bash
# Install molecule
pip install molecule molecule-docker

# Initialize testing for a role
molecule init scenario --role-name my_role --driver-name docker

# Run tests
molecule test      # Full test (create, converge, verify, destroy)
molecule converge  # Just run the playbook
molecule verify    # Run verification tests
molecule destroy   # Clean up
```

```yaml
# molecule/default/molecule.yml
---
driver:
  name: docker
platforms:
  - name: ubuntu-test
    image: ubuntu:22.04
  - name: centos-test
    image: centos:8
provisioner:
  name: ansible
verifier:
  name: ansible  # or testinfra
```

---

## 🎓 Part 3: Scenario-Based Interview Questions

### Scenario 1: Rolling Deployment

**Q: How would you deploy to 100 servers without downtime?**

**A**: Use `serial` for rolling updates:

```yaml
- hosts: webservers
  serial: 10              # Deploy to 10 at a time
  max_fail_percentage: 20  # Stop if >20% fail
  
  pre_tasks:
    - name: Remove from load balancer
      delegate_to: loadbalancer
      command: remove-server {{ inventory_hostname }}
  
  roles:
    - deploy-app
  
  post_tasks:
    - name: Wait for health check
      uri:
        url: "http://{{ inventory_hostname }}/health"
        status_code: 200
      retries: 10
      delay: 5
    
    - name: Add back to load balancer
      delegate_to: loadbalancer
      command: add-server {{ inventory_hostname }}
```

### Scenario 2: Multi-Environment Setup

**Q: How do you manage dev, staging, and production environments?**

**A**: Use inventory structure and group_vars:

```
inventory/
├── dev/
│   ├── hosts
│   └── group_vars/
│       └── all.yml
├── staging/
│   ├── hosts
│   └── group_vars/
│       └── all.yml
└── production/
    ├── hosts
    └── group_vars/
        └── all.yml
```

```bash
# Deploy to specific environment
ansible-playbook -i inventory/dev playbook.yml
ansible-playbook -i inventory/production playbook.yml
```

### Scenario 3: Secrets Management

**Q: How do you handle sensitive data in Ansible?**

**A**: Multiple approaches:

```yaml
# 1. Ansible Vault
ansible-vault create secrets.yml

# 2. Environment variables
- name: Use env var
  debug: msg="{{ lookup('env', 'DB_PASSWORD') }}"

# 3. External secrets manager
- name: Get from AWS Secrets Manager
  set_fact:
    db_password: "{{ lookup('aws_secret', 'my/db/password') }}"

# 4. HashiCorp Vault
- name: Get from HashiCorp Vault
  set_fact:
    api_key: "{{ lookup('hashi_vault', 'secret/data/api') }}"
```

### Scenario 4: Conditional OS Handling

**Q: How do you write playbooks that work on multiple OS types?**

**A**: Use facts and include files per OS:

```yaml
- hosts: all
  tasks:
    - name: Include OS-specific tasks
      include_tasks: "{{ ansible_os_family | lower }}.yml"
```

```yaml
# debian.yml
- name: Install on Debian/Ubuntu
  apt:
    name: "{{ package_name }}"
    state: present

# redhat.yml  
- name: Install on RHEL/CentOS
  yum:
    name: "{{ package_name }}"
    state: present
```

---

## 💡 Part 4: Quick Reference for Interviews

### Common Commands to Know

```bash
# Basic commands
ansible all -m ping                    # Test connectivity
ansible all -m setup                   # Gather facts
ansible all -a "uptime"                # Run ad-hoc command

# Playbook execution
ansible-playbook site.yml              # Run playbook
ansible-playbook site.yml -l web1      # Limit to host
ansible-playbook site.yml --check      # Dry run
ansible-playbook site.yml --diff       # Show changes
ansible-playbook site.yml -v           # Verbose (-vvv = more)

# Vault
ansible-vault create secrets.yml       # Create encrypted file
ansible-vault edit secrets.yml         # Edit encrypted file
ansible-vault view secrets.yml         # View encrypted file
ansible-vault encrypt_string 'secret'  # Encrypt a string

# Galaxy (roles/collections)
ansible-galaxy install geerlingguy.nginx   # Install role
ansible-galaxy collection install amazon.aws  # Install collection

# Inventory
ansible-inventory --list               # Show all inventory
ansible-inventory --graph              # Show inventory tree
```

### State Values to Remember

| Module Type | state: present | state: absent | state: latest |
|-------------|----------------|---------------|---------------|
| Package (apt/yum) | Install | Remove | Install newest |
| Service | N/A | N/A | N/A |
| File | Touch/Create | Delete | N/A |
| User | Create | Delete | N/A |

| Service Module | started | stopped | restarted | reloaded |
|----------------|---------|---------|-----------|----------|
| Effect | Start if not running | Stop if running | Always restart | Reload config |

### Interview Red Flags (Don't Say These!)

❌ "I'd disable host_key_checking in production" 
❌ "I'd use shell module for everything"
❌ "I'd hardcode passwords in playbooks"
❌ "I'd run with -vvvv in production"

✅ Say instead:
- "Use Vault for secrets"
- "Use specific modules instead of shell when possible"
- "Use check mode to verify"
- "Test with Molecule before production"

---

## 📝 Practice Questions

1. **What happens if you run `ansible-playbook` without `-i`?**
   → Uses default inventory from ansible.cfg or /etc/ansible/hosts

2. **Difference between `copy` and `template` modules?**
   → Copy: copies as-is. Template: processes Jinja2 variables first

3. **What's the `register` keyword for?**
   → Captures task output into a variable for later use

4. **How do handlers work?**
   → Run once at end of play, only if notified by a changed task

5. **What's `become: true`?**
   → Run task with elevated privileges (sudo)

6. **Default port for Ansible SSH?**
   → 22 (standard SSH)

7. **What does `changed_when: false` do?**
   → Tells Ansible the task never reports as "changed"

8. **How to skip a task?**
   → Use `when: false` or `--skip-tags`

---

Good luck with your interviews! Remember: interviewers value understanding OVER memorization. If you understand WHY Ansible works a certain way, you can answer almost any question.
