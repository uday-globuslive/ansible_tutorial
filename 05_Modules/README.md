# Ansible Modules - Complete Guide

## What are Modules?

**Modules** are the units of work in Ansible. Each task in a playbook uses one module that performs a specific action on managed nodes.

```
Ansible has 3000+ built-in modules covering:
- System management
- Cloud services
- Networking
- Databases
- Files and templates
- Security
- And much more...
```

---

## Module Categories

| Category | Purpose | Examples |
|----------|---------|----------|
| **System** | Package, service, user management | `apt`, `yum`, `service`, `user` |
| **Files** | File operations | `file`, `copy`, `template`, `lineinfile` |
| **Commands** | Execute commands | `command`, `shell`, `raw`, `script` |
| **Cloud** | Cloud provider management | `aws_ec2`, `azure_rm`, `gcp_compute` |
| **Database** | Database operations | `mysql_db`, `postgresql_db`, `mongodb` |
| **Network** | Network device configuration | `ios_config`, `junos_config`, `nxos` |
| **Security** | Security management | `firewalld`, `ufw`, `iptables` |

---

## Essential Modules (Must Know!)

### 1. Package Management

#### apt (Debian/Ubuntu)

```yaml
# Install a package
- name: Install nginx
  apt:
    name: nginx
    state: present
    update_cache: yes

# Install multiple packages
- name: Install packages
  apt:
    name:
      - nginx
      - vim
      - git
    state: present

# Install specific version
- name: Install specific version
  apt:
    name: nginx=1.18.0-0ubuntu1
    state: present

# Remove a package
- name: Remove nginx
  apt:
    name: nginx
    state: absent

# Upgrade all packages
- name: Upgrade all packages
  apt:
    upgrade: dist
    update_cache: yes

# Autoremove unused packages
- name: Autoremove
  apt:
    autoremove: yes
```

#### yum/dnf (RHEL/CentOS)

```yaml
# Install a package
- name: Install httpd
  yum:
    name: httpd
    state: present

# Using dnf (RHEL 8+)
- name: Install nginx
  dnf:
    name: nginx
    state: latest

# Install from URL
- name: Install package from URL
  yum:
    name: https://example.com/package.rpm
    state: present

# Enable a repo and install
- name: Install from specific repo
  yum:
    name: nginx
    enablerepo: nginx-stable
    state: present
```

#### package (Generic)

```yaml
# Works on any OS
- name: Install package (auto-detect manager)
  package:
    name: vim
    state: present
```

### 2. Service Management

```yaml
# Start a service
- name: Start nginx
  service:
    name: nginx
    state: started

# Stop a service
- name: Stop nginx
  service:
    name: nginx
    state: stopped

# Restart a service
- name: Restart nginx
  service:
    name: nginx
    state: restarted

# Reload configuration
- name: Reload nginx
  service:
    name: nginx
    state: reloaded

# Enable service on boot
- name: Enable nginx
  service:
    name: nginx
    enabled: yes

# Using systemd module (more features)
- name: Start and enable nginx
  systemd:
    name: nginx
    state: started
    enabled: yes
    daemon_reload: yes  # If unit file changed
```

### 3. File Operations

#### file Module

```yaml
# Create a file
- name: Create file
  file:
    path: /tmp/myfile.txt
    state: touch
    mode: '0644'

# Create a directory
- name: Create directory
  file:
    path: /opt/myapp
    state: directory
    owner: appuser
    group: appgroup
    mode: '0755'

# Create symbolic link
- name: Create symlink
  file:
    src: /opt/myapp/current
    dest: /var/www/html
    state: link

# Remove file/directory
- name: Remove file
  file:
    path: /tmp/unwanted.txt
    state: absent

# Change permissions
- name: Set permissions
  file:
    path: /etc/app/config.yml
    owner: root
    group: app
    mode: '0640'

# Create directory recursively
- name: Create nested directories
  file:
    path: /opt/myapp/logs/archive
    state: directory
    recurse: yes
```

#### copy Module

```yaml
# Copy file from control node
- name: Copy configuration
  copy:
    src: files/nginx.conf
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: '0644'
    backup: yes  # Create backup before overwrite

# Copy with inline content
- name: Create file with content
  copy:
    content: |
      # My configuration file
      setting1=value1
      setting2=value2
    dest: /etc/myapp/config.ini

# Copy entire directory
- name: Copy directory
  copy:
    src: files/webapp/
    dest: /var/www/html/
    owner: www-data
    group: www-data

# Validate before copy
- name: Copy and validate
  copy:
    src: files/nginx.conf
    dest: /etc/nginx/nginx.conf
    validate: nginx -t -c %s
```

#### template Module

```yaml
# Deploy Jinja2 template
- name: Deploy nginx config
  template:
    src: templates/nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: '0644'
    validate: nginx -t -c %s
  notify: Restart nginx

# Template with custom delimiters
- name: Deploy template
  template:
    src: custom.j2
    dest: /etc/app/config
    variable_start_string: '<<'
    variable_end_string: '>>'
```

#### lineinfile Module

```yaml
# Add a line to file
- name: Add line to hosts
  lineinfile:
    path: /etc/hosts
    line: '192.168.1.10 myserver.local'
    state: present

# Modify existing line
- name: Change SSH port
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^Port'
    line: 'Port 2222'

# Add line after match
- name: Add config after section
  lineinfile:
    path: /etc/app.conf
    insertafter: '^\[database\]'
    line: 'connection_pool=10'

# Remove a line
- name: Remove line
  lineinfile:
    path: /etc/hosts
    regexp: 'oldserver\.local'
    state: absent

# Ensure line at end of file
- name: Add to end of file
  lineinfile:
    path: /etc/environment
    line: 'JAVA_HOME=/usr/lib/jvm/java-11'
    insertafter: EOF
```

#### blockinfile Module

```yaml
# Add a block of text
- name: Add configuration block
  blockinfile:
    path: /etc/app.conf
    block: |
      # Custom settings
      setting1=value1
      setting2=value2
      setting3=value3
    marker: "# {mark} ANSIBLE MANAGED BLOCK"

# Insert after specific section
- name: Add SSL configuration
  blockinfile:
    path: /etc/nginx/nginx.conf
    insertafter: "server {"
    block: |
        ssl_certificate /etc/ssl/certs/server.crt;
        ssl_certificate_key /etc/ssl/private/server.key;
        ssl_protocols TLSv1.2 TLSv1.3;
```

### 4. User and Group Management

```yaml
# Create user
- name: Create user
  user:
    name: john
    uid: 1050
    group: developers
    groups: sudo,docker
    shell: /bin/bash
    home: /home/john
    create_home: yes
    comment: "John Doe"
    password: "{{ 'password123' | password_hash('sha512') }}"

# Create system user (no home, no login)
- name: Create app user
  user:
    name: myapp
    system: yes
    shell: /sbin/nologin
    create_home: no

# Remove user
- name: Remove user
  user:
    name: olduser
    state: absent
    remove: yes  # Remove home directory

# Create group
- name: Create group
  group:
    name: developers
    gid: 3000
    state: present

# Add SSH key
- name: Add SSH key
  authorized_key:
    user: john
    key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
    state: present
    exclusive: no
```

### 5. Command Execution

#### command Module

```yaml
# Run simple command
- name: Check disk space
  command: df -h
  register: disk_space

# With arguments
- name: Run with args
  command:
    cmd: ls -la /var/log
    chdir: /tmp
  register: result

# Avoid running if file exists
- name: Initialize app
  command: /opt/app/init.sh
  args:
    creates: /opt/app/.initialized
```

#### shell Module

```yaml
# Command with shell features
- name: Run with pipes
  shell: cat /var/log/syslog | grep error | wc -l
  register: error_count

# With environment variables
- name: Run with env
  shell: echo $MY_VAR
  environment:
    MY_VAR: "hello"

# Multi-line command
- name: Complex command
  shell: |
    cd /opt/app
    ./build.sh
    ./test.sh
    ./deploy.sh
```

#### raw Module (No Python Required)

```yaml
# For hosts without Python
- name: Install Python
  raw: apt-get install -y python3
  become: yes
```

#### script Module

```yaml
# Run local script on remote
- name: Run script
  script: 
    cmd: scripts/setup.sh
    creates: /opt/app/.setup_complete
```

### 6. Archive Operations

```yaml
# Create archive
- name: Create backup archive
  archive:
    path: /var/www/html
    dest: /backup/html_backup.tar.gz
    format: gz

# Extract archive
- name: Extract archive
  unarchive:
    src: /tmp/app.tar.gz
    dest: /opt/app
    remote_src: yes  # Archive is on remote host

# Download and extract
- name: Download and extract
  unarchive:
    src: https://example.com/app.tar.gz
    dest: /opt/app
    remote_src: yes
```

### 7. Network Operations

#### get_url Module

```yaml
# Download file
- name: Download file
  get_url:
    url: https://example.com/file.tar.gz
    dest: /tmp/file.tar.gz
    checksum: sha256:abc123...
    mode: '0644'
    timeout: 60
```

#### uri Module

```yaml
# HTTP GET request
- name: Check website
  uri:
    url: http://localhost:8080/health
    return_content: yes
  register: health_check

# HTTP POST request
- name: API call
  uri:
    url: http://api.example.com/data
    method: POST
    body_format: json
    body:
      key: value
    headers:
      Authorization: "Bearer {{ token }}"
    status_code: [200, 201]
```

### 8. Git Operations

```yaml
# Clone repository
- name: Clone repo
  git:
    repo: https://github.com/user/repo.git
    dest: /opt/app
    version: main
    force: yes

# Clone with SSH key
- name: Clone private repo
  git:
    repo: git@github.com:company/private-repo.git
    dest: /opt/app
    version: v1.5.0
    key_file: /home/deploy/.ssh/id_rsa
    accept_hostkey: yes
```

### 9. Cron Jobs

```yaml
# Create cron job
- name: Create backup cron
  cron:
    name: "Daily backup"
    minute: "0"
    hour: "2"
    job: "/opt/scripts/backup.sh"
    user: root

# Create cron with special time
- name: Weekly cleanup
  cron:
    name: "Weekly cleanup"
    special_time: weekly
    job: "/opt/scripts/cleanup.sh"

# Remove cron job
- name: Remove cron
  cron:
    name: "Old job"
    state: absent
```

### 10. Firewall Management

#### ufw (Ubuntu)

```yaml
# Allow SSH
- name: Allow SSH
  ufw:
    rule: allow
    port: '22'
    proto: tcp

# Allow from specific network
- name: Allow from network
  ufw:
    rule: allow
    src: '192.168.1.0/24'

# Enable firewall
- name: Enable UFW
  ufw:
    state: enabled
    policy: deny
```

#### firewalld (RHEL/CentOS)

```yaml
# Open port
- name: Open HTTP port
  firewalld:
    port: 80/tcp
    permanent: yes
    state: enabled
    immediate: yes

# Add service
- name: Allow HTTPS
  firewalld:
    service: https
    permanent: yes
    state: enabled
```

### 11. Debug and Information

```yaml
# Print message
- name: Print message
  debug:
    msg: "The value is {{ my_variable }}"

# Print variable
- name: Show variable
  debug:
    var: my_variable

# Print with verbosity
- name: Debug (verbose only)
  debug:
    msg: "Detailed info"
    verbosity: 2  # Only shows with -vv

# Fail with message
- name: Fail if condition
  fail:
    msg: "Required variable not set!"
  when: required_var is not defined

# Assert conditions
- name: Validate
  assert:
    that:
      - port > 0
      - port < 65536
    fail_msg: "Invalid port number"
```

### 12. Wait and Check

```yaml
# Wait for port
- name: Wait for service
  wait_for:
    port: 8080
    host: localhost
    state: started
    timeout: 300

# Wait for file
- name: Wait for file
  wait_for:
    path: /tmp/ready.txt
    state: present
    timeout: 60

# Pause execution
- name: Pause for user
  pause:
    prompt: "Press Enter to continue"

# Pause for time
- name: Wait 30 seconds
  pause:
    seconds: 30
```

---

## Module Return Values

Most modules return data that can be captured with `register`:

```yaml
- name: Run command
  command: whoami
  register: result

# Common return values:
# result.stdout        - Standard output
# result.stderr        - Standard error
# result.rc            - Return code
# result.changed       - Whether task made changes
# result.failed        - Whether task failed

- name: Use result
  debug:
    msg: "Command ran as {{ result.stdout }}"
```

---

## Finding Modules

```bash
# List all modules
ansible-doc -l

# Search modules
ansible-doc -l | grep -i docker

# Get module documentation
ansible-doc apt
ansible-doc copy
ansible-doc user

# Show module examples
ansible-doc -s apt
```

---

## Module Best Practices

1. **Use specific modules over command/shell**
   ```yaml
   # Bad
   - shell: apt-get install nginx
   
   # Good
   - apt:
       name: nginx
       state: present
   ```

2. **Check module documentation**
   ```bash
   ansible-doc <module_name>
   ```

3. **Use register to capture output**
   ```yaml
   - command: ls /opt
     register: dir_contents
   ```

4. **Handle errors gracefully**
   ```yaml
   - name: Risky task
     command: /opt/script.sh
     ignore_errors: yes
     register: result
   ```

---

## Next Steps

Continue to **06_Variables** to learn about managing variables and facts in Ansible.
