# Ansible Variables - Complete Guide

## What are Variables? (Beginner Explanation)

**Variables** in Ansible store values that can be reused throughout your playbooks. They make your automation flexible and maintainable.

### Real-World Analogy

Think of variables like **labeled storage boxes**:

```
┌────────────────────────┐    ┌────────────────────────┐
│    Box: http_port       │    │    Box: app_name        │
│    Contents: 80         │    │    Contents: "myapp"    │
└────────────────────────┘    └────────────────────────┘
```

- **Label** (variable name) = `http_port`
- **Contents** (value) = `80`

Anytime you need the port number, you grab the `http_port` box instead of typing `80` everywhere!

### Why Variables Matter

**Without Variables (Hard-coded values):**
```yaml
# ❌ BAD - If port changes, you edit 20 places!
- name: Config file 1
  template:
    src: app.conf.j2
  vars:
    port: 8080

- name: Config file 2
  template:
    src: nginx.conf.j2
  vars:
    port: 8080  # Same value repeated!
```

**With Variables (Centralized value):**
```yaml
# ✅ GOOD - Change once, affects everywhere!
vars:
  http_port: 8080

tasks:
  - name: Config 1
    template: src=app.conf.j2   # Uses http_port
  - name: Config 2
    template: src=nginx.conf.j2  # Uses same http_port
```

```yaml
# Variables allow you to write once, use everywhere
http_port: 80
app_name: myapp
env: production
```

---

## Variable Naming Rules

```
✅ Valid:
- my_variable
- myVariable
- my_var_1
- _private_var

❌ Invalid:
- 123variable (starts with number)
- my-variable (contains hyphen)
- my.variable (contains dot)
- my variable (contains space)
```

---

## Ways to Define Variables

### 1. In Playbook (vars)

```yaml
---
- name: Example Playbook
  hosts: all
  vars:
    http_port: 80
    https_port: 443
    app_name: myapp
    max_connections: 100
  
  tasks:
    - name: Print variables
      debug:
        msg: "App: {{ app_name }}, Port: {{ http_port }}"
```

### 2. In Separate File (vars_files)

```yaml
# vars/main.yml
---
http_port: 80
https_port: 443
app_settings:
  name: myapp
  version: "1.5.0"
  debug: false

# playbook.yml
---
- name: Example Playbook
  hosts: all
  vars_files:
    - vars/main.yml
    - vars/secrets.yml
  
  tasks:
    - name: Use variables
      debug:
        msg: "{{ app_settings.name }} v{{ app_settings.version }}"
```

### 3. In Inventory

```ini
# inventory/hosts

[webservers]
web1.example.com http_port=80
web2.example.com http_port=8080

[webservers:vars]
nginx_workers=4
log_level=info
```

### 4. In group_vars

```yaml
# group_vars/webservers.yml
---
http_port: 80
https_port: 443
nginx:
  worker_processes: auto
  worker_connections: 1024

# group_vars/all.yml (applies to all hosts)
---
ntp_server: time.example.com
dns_servers:
  - 8.8.8.8
  - 8.8.4.4
```

### 5. In host_vars

```yaml
# host_vars/web1.example.com.yml
---
http_port: 8080
custom_setting: "web1-specific"
```

### 6. Command Line (-e)

```bash
# Single variable
ansible-playbook playbook.yml -e "version=1.5.0"

# Multiple variables
ansible-playbook playbook.yml -e "version=1.5.0 env=production"

# From JSON
ansible-playbook playbook.yml -e '{"version": "1.5.0", "debug": true}'

# From file
ansible-playbook playbook.yml -e "@vars.yml"
```

### 7. Registered Variables

```yaml
- name: Run command
  command: cat /etc/hostname
  register: hostname_result

- name: Display hostname
  debug:
    msg: "Hostname is {{ hostname_result.stdout }}"
```

### 8. Set Fact (Dynamic)

```yaml
- name: Set custom fact
  set_fact:
    my_custom_var: "calculated_value"
    combined_value: "{{ var1 }}_{{ var2 }}"

- name: Use custom fact
  debug:
    var: my_custom_var
```

---

## Variable Precedence (Lowest to Highest)

```
1.  role defaults (roles/myrole/defaults/main.yml)
2.  inventory file or script group vars
3.  inventory group_vars/all
4.  playbook group_vars/all
5.  inventory group_vars/*
6.  playbook group_vars/*
7.  inventory file or script host vars
8.  inventory host_vars/*
9.  playbook host_vars/*
10. host facts / cached set_facts
11. play vars
12. play vars_prompt
13. play vars_files
14. role vars (roles/myrole/vars/main.yml)
15. block vars (only for tasks in block)
16. task vars (only for the task)
17. include_vars
18. set_facts / registered vars
19. role params
20. include params
21. extra vars (-e) ← ALWAYS WINS!
```

**Rule of Thumb**: More specific = higher precedence

---

## Variable Data Types

### Strings

```yaml
# Simple string
name: "John Doe"
greeting: Hello World

# Multi-line strings
description: |
  This is a multi-line
  description that keeps
  line breaks.

# Folded string (joins lines)
oneliner: >
  This will all be on
  one single line when
  used in the playbook.
```

### Numbers

```yaml
port: 80
max_connections: 1000
timeout: 30.5
hex_value: 0xFF
octal_value: 0o644
```

### Booleans

```yaml
# All valid boolean values
enabled: true
debug: false
active: yes
disabled: no
on: True
off: False
```

### Lists

```yaml
# List of strings
packages:
  - nginx
  - vim
  - git

# Inline list
users: ['john', 'jane', 'bob']

# List of complex items
servers:
  - name: web1
    ip: 192.168.1.10
  - name: web2
    ip: 192.168.1.11
```

### Dictionaries

```yaml
# Dictionary
user:
  name: john
  uid: 1000
  shell: /bin/bash

# Inline dictionary
person: {name: john, age: 30, city: NYC}

# Nested dictionaries
database:
  production:
    host: db.prod.example.com
    port: 5432
  staging:
    host: db.staging.example.com
    port: 5432
```

---

## Accessing Variables

### Simple Variables

```yaml
- debug:
    msg: "Port is {{ http_port }}"
```

### List Elements

```yaml
vars:
  users:
    - john
    - jane
    - bob

tasks:
  - debug:
      msg: "First user: {{ users[0] }}"  # john
  - debug:
      msg: "Last user: {{ users[-1] }}"  # bob
```

### Dictionary Elements

```yaml
vars:
  database:
    host: localhost
    port: 5432
    name: mydb

tasks:
  # Dot notation
  - debug:
      msg: "Host: {{ database.host }}"
  
  # Bracket notation
  - debug:
      msg: "Port: {{ database['port'] }}"
```

### Nested Access

```yaml
vars:
  servers:
    web:
      primary:
        ip: 192.168.1.10
        port: 80

tasks:
  - debug:
      msg: "{{ servers.web.primary.ip }}"
  - debug:
      msg: "{{ servers['web']['primary']['port'] }}"
```

---

## Ansible Facts

Facts are system information automatically gathered from managed hosts.

### Viewing Facts

```bash
# Get all facts
ansible hostname -m setup

# Filter facts
ansible hostname -m setup -a "filter=ansible_distribution*"
```

### Common Facts

```yaml
# Operating System
ansible_distribution        # Ubuntu, CentOS, etc.
ansible_distribution_version # 20.04, 8, etc.
ansible_os_family          # Debian, RedHat, etc.

# Hardware
ansible_processor_count    # Number of CPUs
ansible_memtotal_mb       # Total RAM in MB
ansible_hostname          # Hostname
ansible_fqdn              # Fully Qualified Domain Name

# Network
ansible_default_ipv4.address    # Primary IP
ansible_all_ipv4_addresses      # All IPv4 addresses
ansible_interfaces              # Network interfaces

# Date/Time
ansible_date_time.iso8601      # Current timestamp
ansible_date_time.date         # Current date
```

### Using Facts

```yaml
- name: Use facts
  debug:
    msg: |
      Host: {{ ansible_hostname }}
      OS: {{ ansible_distribution }} {{ ansible_distribution_version }}
      IP: {{ ansible_default_ipv4.address }}
      Memory: {{ ansible_memtotal_mb }} MB
```

### Disabling Fact Gathering

```yaml
- name: Playbook without facts
  hosts: all
  gather_facts: no  # Faster, but no facts available
  
  tasks:
    - name: Quick task
      command: echo "Hello"
```

### Custom Facts

Create a file on managed hosts: `/etc/ansible/facts.d/custom.fact`

```ini
[general]
app_name=myapp
version=1.5.0

[database]
host=localhost
port=5432
```

Access as: `ansible_local.custom.general.app_name`

---

## Variable Filters

Filters transform variable values using Jinja2.

### String Filters

```yaml
vars:
  name: "john doe"

tasks:
  - debug:
      msg: "{{ name | upper }}"        # JOHN DOE
  - debug:
      msg: "{{ name | lower }}"        # john doe
  - debug:
      msg: "{{ name | capitalize }}"   # John doe
  - debug:
      msg: "{{ name | title }}"        # John Doe
  - debug:
      msg: "{{ name | replace('john', 'jane') }}"  # jane doe
```

### Default Values

```yaml
# Use default if undefined
- debug:
    msg: "{{ my_var | default('default_value') }}"

# Default for empty string
- debug:
    msg: "{{ my_var | default('fallback', true) }}"
```

### List Filters

```yaml
vars:
  numbers: [3, 1, 4, 1, 5, 9]

tasks:
  - debug:
      msg: "{{ numbers | sort }}"       # [1, 1, 3, 4, 5, 9]
  - debug:
      msg: "{{ numbers | unique }}"     # [3, 1, 4, 5, 9]
  - debug:
      msg: "{{ numbers | length }}"     # 6
  - debug:
      msg: "{{ numbers | first }}"      # 3
  - debug:
      msg: "{{ numbers | last }}"       # 9
  - debug:
      msg: "{{ numbers | min }}"        # 1
  - debug:
      msg: "{{ numbers | max }}"        # 9
  - debug:
      msg: "{{ numbers | sum }}"        # 23
  - debug:
      msg: "{{ [1,2] | union([2,3]) }}" # [1, 2, 3]
```

### Dictionary Filters

```yaml
vars:
  user:
    name: john
    age: 30

tasks:
  - debug:
      msg: "{{ user | dict2items }}"   # [{key: name, value: john}, ...]
  - debug:
      msg: "{{ user.keys() | list }}"  # [name, age]
  - debug:
      msg: "{{ user.values() | list }}" # [john, 30]
```

### JSON/YAML Filters

```yaml
- debug:
    msg: "{{ my_dict | to_json }}"
- debug:
    msg: "{{ my_dict | to_yaml }}"
- debug:
    msg: "{{ my_dict | to_nice_json }}"
- debug:
    msg: "{{ json_string | from_json }}"
```

### Path Filters

```yaml
vars:
  path: /etc/nginx/nginx.conf

tasks:
  - debug:
      msg: "{{ path | basename }}"      # nginx.conf
  - debug:
      msg: "{{ path | dirname }}"       # /etc/nginx
  - debug:
      msg: "{{ path | splitext }}"      # [/etc/nginx/nginx, .conf]
```

### Type Conversion

```yaml
- debug:
    msg: "{{ '42' | int }}"             # 42 (integer)
- debug:
    msg: "{{ 42 | string }}"            # "42" (string)
- debug:
    msg: "{{ '3.14' | float }}"         # 3.14 (float)
- debug:
    msg: "{{ 'true' | bool }}"          # true (boolean)
```

### Regex Filters

```yaml
vars:
  text: "Hello World 123"

tasks:
  - debug:
      msg: "{{ text | regex_search('[0-9]+') }}"  # 123
  - debug:
      msg: "{{ text | regex_replace('[0-9]+', 'XXX') }}"  # Hello World XXX
```

---

## Lookups

Lookups fetch data from external sources.

### File Lookup

```yaml
- debug:
    msg: "{{ lookup('file', '/etc/hostname') }}"

# Read SSH key
- authorized_key:
    user: john
    key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
```

### Environment Lookup

```yaml
- debug:
    msg: "Home: {{ lookup('env', 'HOME') }}"
```

### Password Lookup

```yaml
# Generate random password
- user:
    name: john
    password: "{{ lookup('password', '/dev/null length=16') | password_hash('sha512') }}"
```

### Template Lookup

```yaml
- copy:
    content: "{{ lookup('template', 'mytemplate.j2') }}"
    dest: /etc/app.conf
```

### URL Lookup

```yaml
- debug:
    msg: "{{ lookup('url', 'https://api.example.com/data') }}"
```

---

## Special Variables

Ansible provides built-in special variables:

```yaml
# Host information
inventory_hostname         # Current host name from inventory
inventory_hostname_short   # Short hostname
ansible_host              # Actual host to connect to

# Group information
groups                    # All groups and their hosts
group_names              # Groups current host belongs to

# Play information
play_hosts               # All hosts in current play
ansible_play_hosts      # Same as play_hosts
ansible_play_name       # Name of current play

# Hostvars (access other hosts' variables)
hostvars['other_host']['ansible_default_ipv4']['address']

# Magic variables
ansible_check_mode       # True if running in check mode
ansible_version          # Ansible version info
```

### Using hostvars

```yaml
# Access variables from another host
- name: Get DB server IP
  debug:
    msg: "DB IP: {{ hostvars['db1.example.com']['ansible_default_ipv4']['address'] }}"

# Loop through all hosts in a group
- name: List all web servers
  debug:
    msg: "{{ hostvars[item]['ansible_default_ipv4']['address'] }}"
  loop: "{{ groups['webservers'] }}"
```

---

## Prompting for Variables

```yaml
- name: Playbook with prompts
  hosts: all
  vars_prompt:
    - name: username
      prompt: "Enter username"
      private: no
    
    - name: password
      prompt: "Enter password"
      private: yes
      encrypt: sha512_crypt
      confirm: yes
    
    - name: version
      prompt: "Enter version"
      default: "1.0.0"
  
  tasks:
    - debug:
        msg: "Deploying {{ version }} for {{ username }}"
```

---

## Best Practices

### 1. Use Descriptive Names

```yaml
# Bad
p: 80
n: nginx

# Good
http_port: 80
webserver_package: nginx
```

### 2. Group Related Variables

```yaml
# Good organization
nginx:
  port: 80
  workers: auto
  log_path: /var/log/nginx

database:
  host: localhost
  port: 5432
  name: myapp
```

### 3. Use Default Values

```yaml
- name: Install package
  apt:
    name: "{{ package_name | default('nginx') }}"
```

### 4. Document Variables

```yaml
# vars/main.yml
---
# Application Settings
# --------------------

# HTTP port for the web server (default: 80)
http_port: 80

# Application name used in paths and configs
app_name: myapp

# Environment: development, staging, production
environment: production
```

### 5. Separate Secrets

```yaml
# Keep secrets in separate encrypted file
- vars_files:
    - vars/main.yml
    - vars/secrets.yml  # Encrypted with ansible-vault
```

---

## Next Steps

Continue to **07_Conditionals_Loops** to learn about controlling task execution flow.
