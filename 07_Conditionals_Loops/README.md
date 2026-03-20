# Conditionals and Loops - Complete Guide

## Conditionals (when)

The `when` statement controls whether a task executes based on conditions.

---

## Basic Conditionals

### Simple Conditions

```yaml
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

### Comparison Operators

```yaml
# Equals
when: my_var == "value"

# Not equals
when: my_var != "value"

# Greater/Less than
when: count > 10
when: count < 5
when: count >= 10
when: count <= 5

# In list
when: my_var in ["value1", "value2", "value3"]

# Not in list
when: my_var not in ["excluded1", "excluded2"]
```

### String Checks

```yaml
# Contains substring
when: "'error' in log_output"

# Starts with
when: hostname is match("web.*")

# Regex match
when: hostname is regex("web[0-9]+\.example\.com")

# String comparison
when: version is version('2.0', '>=')
```

### Variable Checks

```yaml
# Variable is defined
when: my_var is defined

# Variable is not defined
when: my_var is not defined

# Variable is true (boolean)
when: enable_feature

# Variable is false
when: not enable_feature

# Variable is empty
when: my_list | length == 0

# Variable has value
when: my_var | length > 0
```

### Multiple Conditions

```yaml
# AND (all must be true)
- name: Multiple conditions (AND)
  debug:
    msg: "Both conditions met"
  when:
    - ansible_os_family == "Debian"
    - ansible_distribution_major_version | int >= 20

# OR (any must be true)
- name: Multiple conditions (OR)
  debug:
    msg: "At least one condition met"
  when: ansible_os_family == "Debian" or ansible_os_family == "Ubuntu"

# Combined AND/OR
- name: Complex condition
  debug:
    msg: "Complex condition met"
  when: >
    (ansible_os_family == "Debian" and ansible_distribution_version >= "20.04")
    or
    (ansible_os_family == "RedHat" and ansible_distribution_major_version | int >= 8)
```

### Registered Variable Checks

```yaml
- name: Check if file exists
  stat:
    path: /etc/app/config.yml
  register: config_file

- name: Create config if missing
  template:
    src: config.yml.j2
    dest: /etc/app/config.yml
  when: not config_file.stat.exists

# Check command result
- name: Run command
  command: /opt/app/check_status.sh
  register: status_check
  ignore_errors: yes

- name: Handle failure
  debug:
    msg: "Status check failed"
  when: status_check.failed

- name: Handle specific output
  debug:
    msg: "Found in output"
  when: "'SUCCESS' in status_check.stdout"

- name: Check return code
  debug:
    msg: "Command succeeded"
  when: status_check.rc == 0
```

### Facts-Based Conditions

```yaml
# By OS family
- name: Debian-based tasks
  block:
    - apt:
        update_cache: yes
    - apt:
        name: nginx
  when: ansible_os_family == "Debian"

# By distribution
- name: Ubuntu 22.04 specific
  debug:
    msg: "Running on Ubuntu 22.04"
  when:
    - ansible_distribution == "Ubuntu"
    - ansible_distribution_version == "22.04"

# By architecture
- name: ARM specific
  debug:
    msg: "ARM architecture"
  when: ansible_architecture == "aarch64"

# By memory
- name: High memory server
  debug:
    msg: "Server has > 8GB RAM"
  when: ansible_memtotal_mb > 8192

# By virtualization
- name: VM specific
  debug:
    msg: "Running in VM"
  when: ansible_virtualization_type is defined
```

---

## Loops

Loops iterate over lists and execute tasks multiple times.

### Basic Loop

```yaml
- name: Install packages
  apt:
    name: "{{ item }}"
    state: present
  loop:
    - nginx
    - vim
    - git
    - curl
```

### Loop with List Variable

```yaml
vars:
  packages:
    - nginx
    - vim
    - git

tasks:
  - name: Install packages
    apt:
      name: "{{ item }}"
      state: present
    loop: "{{ packages }}"
```

### Loop over Dictionary

```yaml
vars:
  users:
    - name: john
      uid: 1001
      shell: /bin/bash
    - name: jane
      uid: 1002
      shell: /bin/zsh

tasks:
  - name: Create users
    user:
      name: "{{ item.name }}"
      uid: "{{ item.uid }}"
      shell: "{{ item.shell }}"
    loop: "{{ users }}"
```

### Loop with Index

```yaml
- name: Show index
  debug:
    msg: "Item {{ ansible_loop.index }}: {{ item }}"
  loop:
    - apple
    - banana
    - cherry

# ansible_loop variables:
# ansible_loop.index     - 1-based index
# ansible_loop.index0    - 0-based index
# ansible_loop.first     - true if first item
# ansible_loop.last      - true if last item
# ansible_loop.length    - total items
# ansible_loop.revindex  - reverse 1-based index
# ansible_loop.revindex0 - reverse 0-based index
```

### Nested Loops

```yaml
- name: Nested loop
  debug:
    msg: "{{ item[0] }} - {{ item[1] }}"
  loop: "{{ ['a', 'b'] | product(['1', '2']) | list }}"

# Output:
# a - 1
# a - 2
# b - 1
# b - 2
```

### Loop with Conditions

```yaml
- name: Install only needed packages
  apt:
    name: "{{ item.name }}"
    state: present
  loop:
    - { name: nginx, install: true }
    - { name: mysql, install: false }
    - { name: redis, install: true }
  when: item.install
```

### Loop Until (Retry)

```yaml
- name: Wait for service to be ready
  uri:
    url: http://localhost:8080/health
    status_code: 200
  register: result
  until: result.status == 200
  retries: 30
  delay: 10
```

### Loop with Register

```yaml
- name: Run commands
  command: echo "{{ item }}"
  loop:
    - one
    - two
    - three
  register: echo_results

- name: Show results
  debug:
    msg: "{{ item.stdout }}"
  loop: "{{ echo_results.results }}"
```

### Loop Control

```yaml
- name: Loop with control
  debug:
    msg: "{{ item }}"
  loop:
    - apple
    - banana
    - cherry
  loop_control:
    loop_var: fruit  # Custom loop variable name
    index_var: idx   # Variable for index
    label: "{{ fruit.name | default(fruit) }}"  # Custom label in output
    pause: 1         # Pause 1 second between items
```

### Dict2Items Loop

```yaml
vars:
  config:
    key1: value1
    key2: value2
    key3: value3

tasks:
  - name: Loop over dictionary
    debug:
      msg: "{{ item.key }} = {{ item.value }}"
    loop: "{{ config | dict2items }}"
```

### Sequence Loop

```yaml
# Generate sequence
- name: Loop with sequence
  debug:
    msg: "Number: {{ item }}"
  loop: "{{ range(1, 6) | list }}"  # 1, 2, 3, 4, 5

# With step
- name: Loop with step
  debug:
    msg: "Number: {{ item }}"
  loop: "{{ range(0, 10, 2) | list }}"  # 0, 2, 4, 6, 8
```

### File Globbing Loop

```yaml
- name: Copy config files
  copy:
    src: "{{ item }}"
    dest: /etc/app/
  loop: "{{ query('fileglob', 'files/config/*.conf') }}"
```

### Inventory Loop

```yaml
# Loop through hosts in a group
- name: List all web servers
  debug:
    msg: "Server: {{ item }}"
  loop: "{{ groups['webservers'] }}"

# With hostvars
- name: Show all web server IPs
  debug:
    msg: "{{ item }}: {{ hostvars[item]['ansible_default_ipv4']['address'] }}"
  loop: "{{ groups['webservers'] }}"
```

---

## Blocks

Blocks group tasks and apply common attributes.

### Basic Block

```yaml
- name: Web Server Setup
  block:
    - name: Install nginx
      apt:
        name: nginx
        state: present
    
    - name: Configure nginx
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
    
    - name: Start nginx
      service:
        name: nginx
        state: started
  become: true
  when: ansible_os_family == "Debian"
```

### Block with Error Handling

```yaml
- name: Deployment with error handling
  block:
    - name: Download application
      get_url:
        url: "{{ app_download_url }}"
        dest: /tmp/app.tar.gz
    
    - name: Extract application
      unarchive:
        src: /tmp/app.tar.gz
        dest: /opt/app
        remote_src: yes
    
    - name: Start application
      service:
        name: myapp
        state: started

  rescue:
    - name: Log failure
      debug:
        msg: "Deployment failed, rolling back"
    
    - name: Restore previous version
      command: /opt/scripts/rollback.sh
    
    - name: Notify team
      slack:
        token: "{{ slack_token }}"
        channel: "#deployments"
        msg: "Deployment failed on {{ inventory_hostname }}"

  always:
    - name: Cleanup temp files
      file:
        path: /tmp/app.tar.gz
        state: absent
    
    - name: Send completion notification
      debug:
        msg: "Deployment task completed (success or failure)"
```

### Nested Blocks

```yaml
- name: Outer block
  block:
    - name: First task
      debug:
        msg: "First"
    
    - name: Inner block
      block:
        - name: Inner task 1
          debug:
            msg: "Inner 1"
        
        - name: Inner task 2
          debug:
            msg: "Inner 2"
      when: inner_condition
  
  when: outer_condition
```

---

## Practical Examples

### Example 1: OS-Specific Package Installation

```yaml
---
- name: Install packages based on OS
  hosts: all
  become: true
  
  vars:
    debian_packages:
      - nginx
      - python3-pip
      - apt-transport-https
    redhat_packages:
      - nginx
      - python3-pip
      - epel-release
  
  tasks:
    - name: Install packages on Debian
      apt:
        name: "{{ item }}"
        state: present
        update_cache: yes
      loop: "{{ debian_packages }}"
      when: ansible_os_family == "Debian"
    
    - name: Install packages on RedHat
      yum:
        name: "{{ item }}"
        state: present
      loop: "{{ redhat_packages }}"
      when: ansible_os_family == "RedHat"
```

### Example 2: User Management with Conditions

```yaml
---
- name: Manage Users
  hosts: all
  become: true
  
  vars:
    users:
      - name: admin
        state: present
        groups: [sudo]
        servers: [web, db, app]
      - name: webdev
        state: present
        groups: [www-data]
        servers: [web]
      - name: dbadmin
        state: present
        groups: [postgres]
        servers: [db]
      - name: olduser
        state: absent
        servers: [all]
  
  tasks:
    - name: Manage users
      user:
        name: "{{ item.name }}"
        state: "{{ item.state }}"
        groups: "{{ item.groups | default([]) }}"
        remove: "{{ item.state == 'absent' }}"
      loop: "{{ users }}"
      when: >
        'all' in item.servers or
        server_role in item.servers
```

### Example 3: Rolling Service Restart

```yaml
---
- name: Rolling restart
  hosts: webservers
  serial: 1
  become: true
  
  tasks:
    - name: Deregister from load balancer
      uri:
        url: "http://lb.example.com/api/deregister"
        method: POST
        body: "{{ inventory_hostname }}"
      delegate_to: localhost
    
    - name: Wait for connections to drain
      wait_for:
        timeout: 30
    
    - name: Restart application
      service:
        name: myapp
        state: restarted
    
    - name: Wait for service to be healthy
      uri:
        url: "http://localhost:{{ app_port }}/health"
        status_code: 200
      register: health
      until: health.status == 200
      retries: 30
      delay: 5
    
    - name: Register with load balancer
      uri:
        url: "http://lb.example.com/api/register"
        method: POST
        body: "{{ inventory_hostname }}"
      delegate_to: localhost
```

### Example 4: Environment-Based Configuration

```yaml
---
- name: Deploy with environment config
  hosts: all
  become: true
  
  vars:
    environments:
      development:
        debug: true
        log_level: debug
        replicas: 1
      staging:
        debug: true
        log_level: info
        replicas: 2
      production:
        debug: false
        log_level: warn
        replicas: 5
  
  tasks:
    - name: Deploy configuration
      template:
        src: config.yml.j2
        dest: /etc/app/config.yml
      vars:
        config: "{{ environments[environment] }}"
    
    - name: Start replicas
      include_tasks: start_replica.yml
      loop: "{{ range(1, environments[environment].replicas + 1) | list }}"
      loop_control:
        loop_var: replica_num
```

---

## Best Practices

1. **Keep conditions simple**
   ```yaml
   # Good - clear condition
   when: ansible_os_family == "Debian"
   
   # Avoid - complex inline
   when: (a and b) or (c and d and (e or f))
   ```

2. **Use blocks for common conditions**
   ```yaml
   - block:
       - task1
       - task2
       - task3
     when: common_condition
   ```

3. **Name loop variables meaningfully**
   ```yaml
   loop_control:
     loop_var: server
     label: "{{ server.name }}"
   ```

4. **Use failed_when for custom failure conditions**
   ```yaml
   - command: /opt/app/check.sh
     register: result
     failed_when: "'ERROR' in result.stdout"
   ```

5. **Use changed_when to control change status**
   ```yaml
   - command: /opt/app/status.sh
     register: status
     changed_when: "'CHANGED' in status.stdout"
   ```

---

## Next Steps

Continue to **08_Roles** to learn about organizing your Ansible code with roles.
