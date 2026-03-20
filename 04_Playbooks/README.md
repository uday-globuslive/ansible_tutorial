# Ansible Playbooks - Complete Guide

## What is a Playbook?

A **playbook** is a YAML file that defines automation workflows. It contains one or more "plays" that map a group of hosts to tasks.

```
Playbook = Collection of Plays
Play = Hosts + Tasks
Task = Module + Arguments
```

---

## Playbook Structure

```yaml
---
# Playbook starts with three dashes
- name: Play 1 - Configure Web Servers    # Play name
  hosts: webservers                        # Target hosts/groups
  become: true                             # Use sudo
  vars:                                    # Variables
    http_port: 80
  
  tasks:                                   # List of tasks
    - name: Install nginx                  # Task name
      apt:                                 # Module
        name: nginx                        # Module arguments
        state: present

    - name: Start nginx
      service:
        name: nginx
        state: started

- name: Play 2 - Configure Database        # Second play
  hosts: dbservers
  become: true
  
  tasks:
    - name: Install PostgreSQL
      apt:
        name: postgresql
        state: present
```

---

## Your First Playbook

### Step 1: Create the Playbook

```yaml
# first_playbook.yml
---
- name: My First Playbook
  hosts: localhost
  connection: local
  
  tasks:
    - name: Print a message
      debug:
        msg: "Hello, Ansible World!"
    
    - name: Create a file
      file:
        path: /tmp/ansible_test.txt
        state: touch
        
    - name: Add content to file
      copy:
        dest: /tmp/ansible_test.txt
        content: "Created by Ansible at {{ ansible_date_time.iso8601 }}"
```

### Step 2: Run the Playbook

```bash
ansible-playbook first_playbook.yml
```

### Step 3: Understand the Output

```
PLAY [My First Playbook] ******************************************************

TASK [Gathering Facts] ********************************************************
ok: [localhost]

TASK [Print a message] ********************************************************
ok: [localhost] => {
    "msg": "Hello, Ansible World!"
}

TASK [Create a file] **********************************************************
changed: [localhost]

TASK [Add content to file] ****************************************************
changed: [localhost]

PLAY RECAP ********************************************************************
localhost                  : ok=4    changed=2    unreachable=0    failed=0
```

---

## Play-Level Keywords

```yaml
---
- name: Web Server Configuration          # Descriptive name
  hosts: webservers                        # Target hosts (required)
  become: true                             # Enable privilege escalation
  become_user: root                        # User to become
  become_method: sudo                      # Method (sudo, su, etc.)
  gather_facts: true                       # Collect system info
  connection: ssh                          # Connection type
  remote_user: admin                       # SSH user
  vars:                                    # Define variables
    http_port: 80
  vars_files:                              # Load variables from files
    - vars/main.yml
  environment:                             # Set environment variables
    http_proxy: http://proxy.example.com:8080
  serial: 2                                # Run on 2 hosts at a time
  max_fail_percentage: 30                  # Fail if >30% hosts fail
  any_errors_fatal: true                   # Stop on any error
  ignore_errors: false                     # Don't ignore errors
  ignore_unreachable: false                # Don't ignore unreachable hosts
  
  pre_tasks:                               # Run before roles
    - name: Pre-task
      debug:
        msg: "Running pre-tasks"
  
  roles:                                   # Include roles
    - common
    - webserver
  
  tasks:                                   # Main tasks
    - name: Main task
      debug:
        msg: "Running tasks"
  
  post_tasks:                              # Run after tasks
    - name: Post-task
      debug:
        msg: "Running post-tasks"
  
  handlers:                                # Event handlers
    - name: Restart nginx
      service:
        name: nginx
        state: restarted
```

---

## Task Structure

### Basic Task

```yaml
tasks:
  - name: Install nginx               # Human-readable name (recommended)
    apt:                              # Module name
      name: nginx                     # Module parameter
      state: present                  # Module parameter
```

### Task with Multiple Parameters

```yaml
tasks:
  - name: Create user account
    user:
      name: john
      uid: 1040
      group: developers
      groups: sudo,docker
      shell: /bin/bash
      home: /home/john
      createhome: yes
      state: present
```

### Task with Conditionals

```yaml
tasks:
  - name: Install nginx on Debian
    apt:
      name: nginx
      state: present
    when: ansible_os_family == "Debian"
    
  - name: Install nginx on RedHat
    yum:
      name: nginx
      state: present
    when: ansible_os_family == "RedHat"
```

### Task with Loops

```yaml
tasks:
  - name: Install multiple packages
    apt:
      name: "{{ item }}"
      state: present
    loop:
      - nginx
      - vim
      - git
      - curl
```

### Task with Register

```yaml
tasks:
  - name: Check if file exists
    stat:
      path: /etc/nginx/nginx.conf
    register: nginx_conf
    
  - name: Create config if missing
    template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    when: not nginx_conf.stat.exists
```

### Task with Handlers

```yaml
tasks:
  - name: Update nginx configuration
    template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    notify: Restart nginx           # Trigger handler

handlers:
  - name: Restart nginx
    service:
      name: nginx
      state: restarted
```

### Task with Error Handling

```yaml
tasks:
  - name: Try something risky
    command: /usr/local/bin/risky_command
    register: result
    ignore_errors: yes
    
  - name: Handle failure
    debug:
      msg: "Command failed, handling gracefully"
    when: result.failed
```

### Task with Block

```yaml
tasks:
  - name: Handle web server setup
    block:
      - name: Install nginx
        apt:
          name: nginx
          state: present
          
      - name: Start nginx
        service:
          name: nginx
          state: started
          
    rescue:
      - name: Something went wrong
        debug:
          msg: "Installation failed, cleaning up"
          
    always:
      - name: Log action
        debug:
          msg: "Task completed (success or failure)"
```

---

## Task Keywords

| Keyword | Description | Example |
|---------|-------------|---------|
| `name` | Task description | `name: Install nginx` |
| `module` | Module to execute | `apt:`, `copy:`, etc. |
| `when` | Conditional execution | `when: ansible_os_family == "Debian"` |
| `loop` | Iterate over items | `loop: [1, 2, 3]` |
| `register` | Store task result | `register: result` |
| `notify` | Trigger handler | `notify: Restart nginx` |
| `ignore_errors` | Continue on failure | `ignore_errors: yes` |
| `changed_when` | Define changed condition | `changed_when: false` |
| `failed_when` | Define failure condition | `failed_when: result.rc != 0` |
| `become` | Privilege escalation | `become: true` |
| `delegate_to` | Run on different host | `delegate_to: localhost` |
| `run_once` | Run only once | `run_once: true` |
| `tags` | Tag for selective run | `tags: [install, nginx]` |
| `timeout` | Task timeout | `timeout: 300` |
| `retries` | Retry count | `retries: 3` |
| `delay` | Delay between retries | `delay: 10` |
| `until` | Retry condition | `until: result.rc == 0` |

---

## Common Playbook Patterns

### Pattern 1: Package Installation

```yaml
---
- name: Install Required Packages
  hosts: all
  become: true
  
  vars:
    packages:
      - nginx
      - vim
      - git
      - curl
      - htop
  
  tasks:
    - name: Update package cache (Debian)
      apt:
        update_cache: yes
        cache_valid_time: 3600
      when: ansible_os_family == "Debian"
    
    - name: Install packages (Debian)
      apt:
        name: "{{ packages }}"
        state: present
      when: ansible_os_family == "Debian"
    
    - name: Install packages (RedHat)
      yum:
        name: "{{ packages }}"
        state: present
      when: ansible_os_family == "RedHat"
```

### Pattern 2: Service Configuration

```yaml
---
- name: Configure Web Server
  hosts: webservers
  become: true
  
  vars:
    nginx_port: 80
    nginx_worker_processes: auto
  
  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present
    
    - name: Configure nginx
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        owner: root
        group: root
        mode: '0644'
        validate: nginx -t -c %s
      notify: Reload nginx
    
    - name: Create virtual host
      template:
        src: templates/vhost.conf.j2
        dest: /etc/nginx/sites-available/default
      notify: Reload nginx
    
    - name: Ensure nginx is running
      service:
        name: nginx
        state: started
        enabled: yes
  
  handlers:
    - name: Reload nginx
      service:
        name: nginx
        state: reloaded
```

### Pattern 3: User Management

```yaml
---
- name: Manage Users
  hosts: all
  become: true
  
  vars:
    users:
      - name: john
        groups: [sudo, docker]
        shell: /bin/bash
        state: present
      - name: jane
        groups: [developers]
        shell: /bin/zsh
        state: present
      - name: old_user
        state: absent
  
  tasks:
    - name: Create/Remove users
      user:
        name: "{{ item.name }}"
        groups: "{{ item.groups | default([]) }}"
        shell: "{{ item.shell | default('/bin/bash') }}"
        state: "{{ item.state }}"
        remove: "{{ item.state == 'absent' }}"
      loop: "{{ users }}"
    
    - name: Add SSH keys
      authorized_key:
        user: "{{ item.name }}"
        key: "{{ lookup('file', 'files/ssh_keys/' + item.name + '.pub') }}"
        state: present
      loop: "{{ users }}"
      when: item.state == 'present'
      ignore_errors: yes
```

### Pattern 4: Application Deployment

```yaml
---
- name: Deploy Application
  hosts: appservers
  become: true
  serial: 1  # Rolling deployment
  
  vars:
    app_name: myapp
    app_version: "1.5.0"
    app_dir: /opt/{{ app_name }}
    app_user: appuser
  
  pre_tasks:
    - name: Remove from load balancer
      uri:
        url: "http://lb.example.com/api/deregister"
        method: POST
        body: '{"host": "{{ inventory_hostname }}"}'
        body_format: json
      delegate_to: localhost
  
  tasks:
    - name: Stop application
      service:
        name: "{{ app_name }}"
        state: stopped
    
    - name: Backup current version
      archive:
        path: "{{ app_dir }}"
        dest: "/backup/{{ app_name }}-{{ ansible_date_time.iso8601 }}.tar.gz"
      ignore_errors: yes
    
    - name: Download new version
      get_url:
        url: "https://releases.example.com/{{ app_name }}/{{ app_version }}/{{ app_name }}.tar.gz"
        dest: /tmp/{{ app_name }}.tar.gz
    
    - name: Extract application
      unarchive:
        src: /tmp/{{ app_name }}.tar.gz
        dest: "{{ app_dir }}"
        remote_src: yes
        owner: "{{ app_user }}"
        group: "{{ app_user }}"
    
    - name: Run database migrations
      command: "{{ app_dir }}/bin/migrate"
      run_once: true
    
    - name: Start application
      service:
        name: "{{ app_name }}"
        state: started
    
    - name: Wait for application to be ready
      uri:
        url: "http://localhost:{{ app_port }}/health"
        status_code: 200
      register: health_check
      until: health_check.status == 200
      retries: 30
      delay: 5
  
  post_tasks:
    - name: Add back to load balancer
      uri:
        url: "http://lb.example.com/api/register"
        method: POST
        body: '{"host": "{{ inventory_hostname }}"}'
        body_format: json
      delegate_to: localhost
```

### Pattern 5: Security Hardening

```yaml
---
- name: Security Hardening
  hosts: all
  become: true
  
  tasks:
    - name: Update all packages
      apt:
        upgrade: dist
        update_cache: yes
      when: ansible_os_family == "Debian"
    
    - name: Configure SSH
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: "{{ item.regexp }}"
        line: "{{ item.line }}"
        state: present
      loop:
        - { regexp: '^PermitRootLogin', line: 'PermitRootLogin no' }
        - { regexp: '^PasswordAuthentication', line: 'PasswordAuthentication no' }
        - { regexp: '^X11Forwarding', line: 'X11Forwarding no' }
        - { regexp: '^MaxAuthTries', line: 'MaxAuthTries 3' }
      notify: Restart SSH
    
    - name: Configure firewall
      ufw:
        rule: allow
        port: "{{ item }}"
        proto: tcp
      loop:
        - 22
        - 80
        - 443
    
    - name: Enable firewall
      ufw:
        state: enabled
    
    - name: Set file permissions
      file:
        path: "{{ item.path }}"
        mode: "{{ item.mode }}"
      loop:
        - { path: '/etc/passwd', mode: '0644' }
        - { path: '/etc/shadow', mode: '0600' }
        - { path: '/etc/gshadow', mode: '0600' }
  
  handlers:
    - name: Restart SSH
      service:
        name: sshd
        state: restarted
```

---

## Playbook Organization

### Single Playbook (Simple Projects)

```
project/
├── playbook.yml
├── inventory/
│   └── hosts
└── files/
    └── config.txt
```

### Multiple Playbooks (Medium Projects)

```
project/
├── site.yml           # Main playbook (imports others)
├── webservers.yml
├── dbservers.yml
├── common.yml
├── inventory/
│   └── hosts
├── group_vars/
│   └── all.yml
└── files/
```

### site.yml (Imports Other Playbooks)

```yaml
# site.yml
---
- import_playbook: common.yml
- import_playbook: webservers.yml
- import_playbook: dbservers.yml
```

---

## Running Playbooks

### Basic Execution

```bash
# Run playbook
ansible-playbook playbook.yml

# With specific inventory
ansible-playbook -i inventory/production playbook.yml

# With verbose output
ansible-playbook playbook.yml -v
ansible-playbook playbook.yml -vvv
```

### Targeting Specific Hosts

```bash
# Limit to specific hosts
ansible-playbook playbook.yml --limit webservers
ansible-playbook playbook.yml --limit "web1.example.com,web2.example.com"

# Exclude hosts
ansible-playbook playbook.yml --limit 'all:!web3.example.com'
```

### Testing & Debugging

```bash
# Syntax check
ansible-playbook playbook.yml --syntax-check

# Dry run (check mode)
ansible-playbook playbook.yml --check

# Show diff
ansible-playbook playbook.yml --check --diff

# Step through
ansible-playbook playbook.yml --step

# Start at specific task
ansible-playbook playbook.yml --start-at-task="Install nginx"
```

### Using Tags

```bash
# Run specific tags
ansible-playbook playbook.yml --tags "install,configure"

# Skip specific tags
ansible-playbook playbook.yml --skip-tags "test"

# List available tags
ansible-playbook playbook.yml --list-tags
```

---

## Next Steps

Continue to **05_Modules** to learn about Ansible's powerful built-in modules.
