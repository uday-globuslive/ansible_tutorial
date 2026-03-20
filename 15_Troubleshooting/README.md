# Ansible Troubleshooting Guide

## Troubleshooting Mindset (Before We Start)

When Ansible fails, don't panic! Follow this process:

```
┌────────────────────────────────────────────────────┐
│  ANSIBLE TROUBLESHOOTING FLOWCHART              │
├────────────────────────────────────────────────────┤
│                                                  │
│ 1️⃣ Read the error message carefully              │
│    (It usually tells you what's wrong!)         │
│                    ▼                             │
│ 2️⃣ Add verbosity: -v, -vv, -vvv, or -vvvv       │
│    (Each level shows more details)              │
│                    ▼                             │
│ 3️⃣ Test the connection separately:               │
│    ansible hostname -m ping                     │
│                    ▼                             │
│ 4️⃣ Check the specific task:                      │
│    --start-at-task="Task Name"                  │
│                    ▼                             │
│ 5️⃣ Search the error online or check this guide   │
│                                                  │
└────────────────────────────────────────────────────┘
```

### Understanding Verbosity Levels

| Flag | What It Shows | When To Use |
|------|---------------|-------------|
| `-v` | Task results | First attempt |
| `-vv` | Task input parameters | Check what was sent |
| `-vvv` | SSH connection details | Connection problems |
| `-vvvv` | Everything including SSH commands | Deep debugging |

---

## Common Errors and Solutions

---

## Connection Issues

### Error: SSH Connection Refused

```
UNREACHABLE! => {"changed": false, "msg": "Failed to connect to the host via ssh"}
```

**Solutions:**

```bash
# 1. Check if SSH is running on target
ssh user@hostname

# 2. Check if host is reachable
ping hostname

# 3. Check SSH port
nc -zv hostname 22

# 4. Verify SSH key
ssh-add -l
ssh-add ~/.ssh/id_rsa

# 5. Test with verbose mode
ansible hostname -m ping -vvvv

# 6. Check inventory configuration
ansible-inventory --host hostname
```

### Error: Permission Denied (publickey)

```
UNREACHABLE! => {"msg": "Permission denied (publickey)"}
```

**Solutions:**

```bash
# 1. Copy SSH key to remote host
ssh-copy-id user@hostname

# 2. Check key permissions
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_rsa.pub

# 3. Specify key in inventory
# hosts file
web1 ansible_ssh_private_key_file=~/.ssh/custom_key

# 4. Or in ansible.cfg
# [defaults]
# private_key_file = ~/.ssh/custom_key
```

### Error: Host Key Verification Failed

```
UNREACHABLE! => {"msg": "Host key verification failed"}
```

**Solutions:**

```bash
# 1. Add host key manually
ssh-keyscan -H hostname >> ~/.ssh/known_hosts

# 2. Or disable host key checking (dev only)
# ansible.cfg
[defaults]
host_key_checking = False

# 3. Environment variable
export ANSIBLE_HOST_KEY_CHECKING=False
```

---

## Authentication Issues

### Error: Incorrect sudo password

```
FAILED! => {"msg": "Incorrect sudo password"}
```

**Solutions:**

```bash
# 1. Provide sudo password
ansible-playbook playbook.yml -K

# 2. Configure passwordless sudo on target
# /etc/sudoers.d/ansible
deploy ALL=(ALL) NOPASSWD: ALL

# 3. Set in inventory
# inventory/hosts
[all:vars]
ansible_become_pass="{{ vault_sudo_password }}"
```

### Error: Missing sudo password

```
FAILED! => {"msg": "Missing sudo password"}
```

**Solutions:**

```yaml
# 1. In playbook
- hosts: all
  become: true
  become_method: sudo
  
# 2. In ansible.cfg
[privilege_escalation]
become = True
become_method = sudo
become_ask_pass = True
```

---

## Python Issues

### Error: Python interpreter not found

```
FAILED! => {"msg": "/usr/bin/python: not found"}
```

**Solutions:**

```bash
# 1. Set Python interpreter
ansible all -m ping -e 'ansible_python_interpreter=/usr/bin/python3'

# 2. In inventory
[all:vars]
ansible_python_interpreter=/usr/bin/python3

# 3. In ansible.cfg
[defaults]
interpreter_python = /usr/bin/python3

# 4. Auto-discovery
[defaults]
interpreter_python = auto
```

### Error: Module requires Python library

```
FAILED! => {"msg": "The PyMySQL (Python 2.7 and Python 3.X) module is required"}
```

**Solutions:**

```yaml
# 1. Install the library on target
- name: Install PyMySQL
  pip:
    name: PyMySQL
    state: present

# 2. Or use package manager
- name: Install python3-mysqldb
  apt:
    name: python3-mysqldb
    state: present
```

---

## Module Errors

### Error: Module failure

```
FAILED! => {"changed": false, "msg": "No package matching 'nonexistent' found"}
```

**Solutions:**

```yaml
# 1. Use ignore_errors
- name: Install package
  apt:
    name: package-name
  ignore_errors: yes

# 2. Use failed_when for custom failure
- name: Run command
  command: /opt/script.sh
  register: result
  failed_when: 
    - result.rc != 0
    - "'expected_error' not in result.stderr"

# 3. Use block/rescue
- block:
    - name: Attempt installation
      apt:
        name: package
  rescue:
    - name: Fallback action
      debug:
        msg: "Package not found, using alternative"
```

### Error: No module named 'ansible.module_utils'

```bash
# Solution: Check Ansible installation
pip install --upgrade ansible

# Or reinstall
pip uninstall ansible ansible-core
pip install ansible
```

---

## Variable Issues

### Error: 'variable' is undefined

```
FAILED! => {"msg": "'my_variable' is undefined"}
```

**Solutions:**

```yaml
# 1. Use default filter
"{{ my_variable | default('default_value') }}"

# 2. Check if defined
- name: Task with condition
  debug:
    var: my_variable
  when: my_variable is defined

# 3. Use vars_prompt for required variables
vars_prompt:
  - name: my_variable
    prompt: "Enter value"
    private: no
```

### Error: Invalid YAML syntax

```
ERROR! Syntax Error while loading YAML
```

**Solutions:**

```bash
# 1. Validate YAML syntax
ansible-playbook playbook.yml --syntax-check

# 2. Use yamllint
pip install yamllint
yamllint playbook.yml

# 3. Common YAML issues:
# - Wrong indentation (use 2 spaces, not tabs)
# - Missing colons
# - Missing quotes around special characters
# - Incorrect list format
```

### Debugging Variables

```yaml
# Print all variables
- debug:
    var: vars

# Print specific variable
- debug:
    var: my_variable

# Print with verbosity level
- debug:
    msg: "Variable value: {{ my_variable }}"
    verbosity: 2  # Only shows with -vv
```

---

## Playbook Issues

### Error: Task execution slow

**Solutions:**

```yaml
# 1. Disable fact gathering if not needed
- hosts: all
  gather_facts: no

# 2. Gather only needed facts
- setup:
    gather_subset:
      - min
      - network

# 3. Use pipelining
# ansible.cfg
[ssh_connection]
pipelining = True

# 4. Increase forks
# ansible.cfg
[defaults]
forks = 50
```

### Error: Task changed when it shouldn't

```yaml
# Use changed_when
- name: Check something
  command: /opt/check.sh
  register: result
  changed_when: false

# Or
- name: Conditional changed
  command: /opt/update.sh
  register: result
  changed_when: "'updated' in result.stdout"
```

---

## Vault Issues

### Error: Decryption failed

```
ERROR! Decryption failed
```

**Solutions:**

```bash
# 1. Check vault password
ansible-vault view encrypted.yml

# 2. Rekey with correct password
ansible-vault rekey encrypted.yml

# 3. Verify file is actually encrypted
head -1 encrypted.yml
# Should show: $ANSIBLE_VAULT;1.1;AES256

# 4. Check vault password file permissions
chmod 600 ~/.vault_pass
```

### Error: No vault secrets found

```bash
# Provide vault password
ansible-playbook playbook.yml --ask-vault-pass

# Or use password file
ansible-playbook playbook.yml --vault-password-file ~/.vault_pass

# Set in ansible.cfg
[defaults]
vault_password_file = ~/.vault_pass
```

---

## Debugging Techniques

### Verbose Mode

```bash
# Increasing verbosity
ansible-playbook playbook.yml -v    # Basic
ansible-playbook playbook.yml -vv   # More detailed
ansible-playbook playbook.yml -vvv  # SSH debugging
ansible-playbook playbook.yml -vvvv # Maximum verbosity
```

### Debug Module

```yaml
- name: Debug variables
  debug:
    msg: |
      Variable1: {{ var1 }}
      Variable2: {{ var2 }}
      Facts: {{ ansible_os_family }}

- name: Show all variables
  debug:
    var: hostvars[inventory_hostname]
```

### Strategy: Debug

```yaml
# Enable interactive debugging
- hosts: all
  strategy: debug
  
  tasks:
    - name: Failing task
      command: /bin/false
      
# Debugger commands:
# p task.args - Print task arguments
# p result - Print result
# p vars - Print all variables
# c - Continue to next task
# r - Re-run task
# q - Quit
```

### Debugger Keyword

```yaml
- name: Task with debugger
  command: /opt/script.sh
  debugger: on_failed  # Options: always, never, on_failed, on_unreachable, on_skipped
```

---

## Performance Troubleshooting

### Slow Playbook Execution

```bash
# Profile playbook
time ansible-playbook playbook.yml

# Use callback plugin for timing
# ansible.cfg
[defaults]
callback_whitelist = timer, profile_tasks
```

### Memory Issues

```bash
# Increase memory
export ANSIBLE_MEMORY_PROFILING=True

# Limit inventory processing
ansible-playbook playbook.yml --limit "web1:web2"

# Use smaller batches
- hosts: all
  serial: 10
```

---

## Common Mistakes Checklist

```
✅ YAML indentation (2 spaces, no tabs)
✅ Correct module syntax
✅ Variables quoted when needed: "{{ var }}"
✅ Inventory hosts reachable
✅ SSH keys configured
✅ Python available on targets
✅ Vault password provided
✅ Correct become configuration
✅ Required collections installed
✅ File paths are absolute
```

---

## Useful Debug Commands

```bash
# Check playbook syntax
ansible-playbook playbook.yml --syntax-check

# List tasks
ansible-playbook playbook.yml --list-tasks

# List hosts
ansible-playbook playbook.yml --list-hosts

# Dry run
ansible-playbook playbook.yml --check

# Show diff
ansible-playbook playbook.yml --check --diff

# Step through
ansible-playbook playbook.yml --step

# Start at task
ansible-playbook playbook.yml --start-at-task="Task name"

# Test connectivity
ansible all -m ping

# Get facts
ansible hostname -m setup
ansible hostname -m setup -a "filter=ansible_*_ipv4"

# Check inventory
ansible-inventory --list
ansible-inventory --graph
ansible-inventory --host hostname
```

---

## Getting Help

```bash
# Module documentation
ansible-doc module_name
ansible-doc -l              # List all modules
ansible-doc -s module_name  # Show examples

# Ansible version
ansible --version

# Configuration
ansible-config view
ansible-config dump --only-changed
```

---

*When in doubt: increase verbosity (-vvvv), check syntax, and verify connectivity!*
