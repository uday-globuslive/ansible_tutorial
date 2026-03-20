# Ansible Roles - Complete Guide

## What are Roles? (Beginner Explanation)

**Roles** are a way to organize playbooks into reusable, self-contained units. They encapsulate tasks, variables, files, templates, and handlers in a standardized directory structure.

### Analogy: LEGO Instruction Sets

Think of roles like **LEGO instruction booklets**:
- Each booklet (role) builds one complete thing (e.g., "Spaceship", "Castle")
- You can combine booklets to build bigger things
- You can share booklets with friends
- Anyone following the same booklet gets the same result

```
Playbook WITHOUT Roles            Playbook WITH Roles
────────────────────            ───────────────────

┌────────────────────┐          ┌────────────────────┐
│ 500 lines of      │          │ roles:            │
│ tasks all in      │          │   - common        │
│ one huge file     │          │   - nginx         │
│                   │          │   - postgresql    │
│ Hard to read!     │          │                   │
│ Hard to reuse!    │          │ Clean & Reusable! │
└────────────────────┘          └────────────────────┘
```

### Why Roles Matter

| Problem Without Roles | Solution With Roles |
|----------------------|--------------------|
| 500-line playbooks | 20-line playbook using 5 roles |
| Copy-paste code between projects | Import the same role |
| Hard to test | Test each role independently |
| Hard to share | Upload to Ansible Galaxy |
| Everyone writes differently | Standard structure everyone knows |

```
Think of roles as "packages" of automation that can be:
- Reused across projects
- Shared with others
- Maintained independently
```

---

## Role Directory Structure

```
roles/
└── webserver/              # Role name
    ├── defaults/           # Default variables (lowest precedence)
    │   └── main.yml
    ├── vars/               # Role variables (higher precedence)
    │   └── main.yml
    ├── tasks/              # Tasks to execute
    │   └── main.yml
    ├── handlers/           # Handlers
    │   └── main.yml
    ├── templates/          # Jinja2 templates
    │   └── nginx.conf.j2
    ├── files/              # Static files to copy
    │   └── index.html
    ├── meta/               # Role dependencies and metadata
    │   └── main.yml
    ├── library/            # Custom modules (optional)
    ├── module_utils/       # Module utilities (optional)
    ├── lookup_plugins/     # Custom lookups (optional)
    └── README.md           # Documentation
```

---

## Creating a Role

### Method 1: Manually

```bash
mkdir -p roles/webserver/{tasks,handlers,templates,files,vars,defaults,meta}
touch roles/webserver/tasks/main.yml
touch roles/webserver/handlers/main.yml
touch roles/webserver/defaults/main.yml
touch roles/webserver/vars/main.yml
touch roles/webserver/meta/main.yml
touch roles/webserver/README.md
```

### Method 2: Using ansible-galaxy

```bash
# Create role skeleton
ansible-galaxy init roles/webserver

# Output:
# - Role webserver was created successfully
```

---

## Complete Role Example: nginx

### defaults/main.yml

```yaml
# roles/nginx/defaults/main.yml
---
# Default variables (can be overridden)
nginx_port: 80
nginx_server_name: localhost
nginx_worker_processes: auto
nginx_worker_connections: 1024
nginx_keepalive_timeout: 65
nginx_root: /var/www/html

nginx_remove_default_site: true
nginx_start_service: true
nginx_enable_service: true

# SSL defaults
nginx_ssl_enabled: false
nginx_ssl_certificate: ""
nginx_ssl_certificate_key: ""
```

### vars/main.yml

```yaml
# roles/nginx/vars/main.yml
---
# Variables that shouldn't be overridden
nginx_package_name: nginx
nginx_service_name: nginx
nginx_config_dir: /etc/nginx
nginx_sites_available: /etc/nginx/sites-available
nginx_sites_enabled: /etc/nginx/sites-enabled
nginx_log_dir: /var/log/nginx
```

### tasks/main.yml

```yaml
# roles/nginx/tasks/main.yml
---
- name: Include OS-specific variables
  include_vars: "{{ ansible_os_family | lower }}.yml"
  
- name: Install nginx
  include_tasks: install.yml
  
- name: Configure nginx
  include_tasks: configure.yml
  
- name: Manage nginx service
  include_tasks: service.yml
```

### tasks/install.yml

```yaml
# roles/nginx/tasks/install.yml
---
- name: Install nginx package
  package:
    name: "{{ nginx_package_name }}"
    state: present

- name: Ensure nginx directories exist
  file:
    path: "{{ item }}"
    state: directory
    owner: root
    group: root
    mode: '0755'
  loop:
    - "{{ nginx_config_dir }}"
    - "{{ nginx_sites_available }}"
    - "{{ nginx_sites_enabled }}"
    - "{{ nginx_root }}"
    - "{{ nginx_log_dir }}"
```

### tasks/configure.yml

```yaml
# roles/nginx/tasks/configure.yml
---
- name: Deploy nginx.conf
  template:
    src: nginx.conf.j2
    dest: "{{ nginx_config_dir }}/nginx.conf"
    owner: root
    group: root
    mode: '0644'
    validate: nginx -t -c %s
  notify: Reload nginx

- name: Deploy default site configuration
  template:
    src: default-site.conf.j2
    dest: "{{ nginx_sites_available }}/default"
    owner: root
    group: root
    mode: '0644'
  notify: Reload nginx

- name: Enable default site
  file:
    src: "{{ nginx_sites_available }}/default"
    dest: "{{ nginx_sites_enabled }}/default"
    state: link
  notify: Reload nginx

- name: Remove default nginx site if requested
  file:
    path: "{{ nginx_sites_enabled }}/default"
    state: absent
  when: nginx_remove_default_site
  notify: Reload nginx

- name: Deploy index.html
  copy:
    src: index.html
    dest: "{{ nginx_root }}/index.html"
    owner: www-data
    group: www-data
    mode: '0644'
```

### tasks/service.yml

```yaml
# roles/nginx/tasks/service.yml
---
- name: Start nginx service
  service:
    name: "{{ nginx_service_name }}"
    state: started
  when: nginx_start_service

- name: Enable nginx service
  service:
    name: "{{ nginx_service_name }}"
    enabled: yes
  when: nginx_enable_service
```

### handlers/main.yml

```yaml
# roles/nginx/handlers/main.yml
---
- name: Restart nginx
  service:
    name: "{{ nginx_service_name }}"
    state: restarted

- name: Reload nginx
  service:
    name: "{{ nginx_service_name }}"
    state: reloaded
```

### templates/nginx.conf.j2

```jinja2
# roles/nginx/templates/nginx.conf.j2
user www-data;
worker_processes {{ nginx_worker_processes }};
pid /run/nginx.pid;

events {
    worker_connections {{ nginx_worker_connections }};
}

http {
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout {{ nginx_keepalive_timeout }};
    types_hash_max_size 2048;

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Logging
    access_log {{ nginx_log_dir }}/access.log;
    error_log {{ nginx_log_dir }}/error.log;

    # Gzip
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml application/json application/javascript;

    # Virtual Hosts
    include {{ nginx_sites_enabled }}/*;
}
```

### templates/default-site.conf.j2

```jinja2
# roles/nginx/templates/default-site.conf.j2
server {
    listen {{ nginx_port }};
    server_name {{ nginx_server_name }};

    root {{ nginx_root }};
    index index.html index.htm;

{% if nginx_ssl_enabled %}
    listen 443 ssl;
    ssl_certificate {{ nginx_ssl_certificate }};
    ssl_certificate_key {{ nginx_ssl_certificate_key }};
    ssl_protocols TLSv1.2 TLSv1.3;
{% endif %}

    location / {
        try_files $uri $uri/ =404;
    }

    # Health check endpoint
    location /health {
        access_log off;
        return 200 "healthy\n";
    }
}
```

### meta/main.yml

```yaml
# roles/nginx/meta/main.yml
---
galaxy_info:
  author: Your Name
  description: Install and configure Nginx web server
  company: Your Company
  license: MIT
  min_ansible_version: "2.9"
  
  platforms:
    - name: Ubuntu
      versions:
        - focal
        - jammy
    - name: Debian
      versions:
        - bullseye
        - bookworm
    - name: EL  # Enterprise Linux (RHEL/CentOS)
      versions:
        - "8"
        - "9"
  
  galaxy_tags:
    - web
    - nginx
    - webserver

dependencies:
  # Role dependencies (installed first)
  # - role: common
  # - role: firewall
  #   vars:
  #     firewall_allowed_tcp_ports:
  #       - 80
  #       - 443
  []
```

### README.md

```markdown
# Nginx Role

Installs and configures Nginx web server.

## Requirements

- Ansible 2.9+
- Supported OS: Ubuntu 20.04+, Debian 11+, RHEL/CentOS 8+

## Role Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `nginx_port` | 80 | HTTP listen port |
| `nginx_server_name` | localhost | Server name |
| `nginx_worker_processes` | auto | Worker processes |
| `nginx_ssl_enabled` | false | Enable SSL |

## Example Playbook

```yaml
- hosts: webservers
  roles:
    - role: nginx
      vars:
        nginx_port: 8080
        nginx_server_name: www.example.com
```

## License

MIT
```

---

## Using Roles

### Method 1: In Playbook (roles section)

```yaml
---
- name: Configure Web Servers
  hosts: webservers
  become: true
  
  roles:
    - nginx
    - php
    - mysql
```

### Method 2: With Variables

```yaml
---
- name: Configure Web Servers
  hosts: webservers
  become: true
  
  roles:
    - role: nginx
      vars:
        nginx_port: 8080
    
    - role: mysql
      vars:
        mysql_root_password: "{{ vault_mysql_password }}"
```

### Method 3: With Tags

```yaml
---
- name: Configure Web Servers
  hosts: webservers
  become: true
  
  roles:
    - role: nginx
      tags: [webserver, nginx]
    
    - role: mysql
      tags: [database, mysql]
```

### Method 4: Conditional Roles

```yaml
---
- name: Configure Servers
  hosts: all
  become: true
  
  roles:
    - role: nginx
      when: "'webservers' in group_names"
    
    - role: mysql
      when: "'dbservers' in group_names"
```

### Method 5: Include/Import Role

```yaml
---
- name: Configure Server
  hosts: all
  become: true
  
  tasks:
    - name: Setup base system
      include_role:
        name: common
    
    - name: Setup web server
      import_role:
        name: nginx
      when: server_type == "web"
```

---

## Role Dependencies

### Defining Dependencies

```yaml
# roles/webapp/meta/main.yml
---
dependencies:
  - role: common
  
  - role: nginx
    vars:
      nginx_port: 8080
  
  - role: nodejs
    vars:
      nodejs_version: "18"
```

### Conditional Dependencies

```yaml
# roles/webapp/meta/main.yml
---
dependencies:
  - role: nginx
    when: webserver == "nginx"
  
  - role: apache
    when: webserver == "apache"
```

---

## Ansible Galaxy

### Installing Roles

```bash
# Install from Galaxy
ansible-galaxy install geerlingguy.nginx

# Install specific version
ansible-galaxy install geerlingguy.nginx,3.1.0

# Install from GitHub
ansible-galaxy install git+https://github.com/user/role.git

# Install from requirements file
ansible-galaxy install -r requirements.yml
```

### requirements.yml

```yaml
# requirements.yml
---
roles:
  - name: geerlingguy.nginx
    version: "3.1.0"
  
  - name: geerlingguy.mysql
    version: "4.0.0"
  
  - name: my_custom_role
    src: git+https://github.com/company/ansible-role-custom.git
    version: main
  
  - name: private_role
    src: git@github.com:company/private-role.git
    scm: git
    version: v1.0.0

collections:
  - name: community.general
  - name: amazon.aws
    version: ">=2.0.0"
```

### Installing from requirements.yml

```bash
ansible-galaxy install -r requirements.yml
ansible-galaxy collection install -r requirements.yml
```

### Publishing Roles

```bash
# Login to Galaxy
ansible-galaxy login

# Import role from GitHub
ansible-galaxy import github_user role_name

# Or push directly
ansible-galaxy role import github_org role_name
```

---

## Role Best Practices

### 1. Keep Roles Focused

```yaml
# Good: Single responsibility
roles/
├── nginx/
├── php/
└── mysql/

# Avoid: Do-everything role
roles/
└── lamp_stack/  # Too monolithic
```

### 2. Use Defaults for Flexibility

```yaml
# defaults/main.yml
---
# All variables should have sensible defaults
app_port: 8080
app_user: appuser
app_log_level: info
```

### 3. Document Variables

```yaml
# defaults/main.yml
---
# Port the application listens on
# Type: integer
# Required: no
app_port: 8080

# User to run the application as
# Type: string
# Required: no
app_user: appuser
```

### 4. Use OS-Specific Variables

```yaml
# vars/debian.yml
package_name: nginx

# vars/redhat.yml
package_name: nginx

# tasks/main.yml
- name: Include OS-specific vars
  include_vars: "{{ ansible_os_family | lower }}.yml"
```

### 5. Use Handlers Properly

```yaml
# handlers/main.yml
---
- name: Restart nginx
  service:
    name: nginx
    state: restarted
  listen: "restart web server"

- name: Reload nginx
  service:
    name: nginx
    state: reloaded
  listen: "reload web config"
```

### 6. Test Your Roles

```yaml
# molecule/default/molecule.yml (for testing)
---
dependency:
  name: galaxy
driver:
  name: docker
platforms:
  - name: ubuntu-22
    image: ubuntu:22.04
  - name: centos-8
    image: centos:8
provisioner:
  name: ansible
verifier:
  name: ansible
```

---

## Project Structure with Roles

```
ansible-project/
├── ansible.cfg
├── inventory/
│   ├── production/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   └── staging/
│       ├── hosts.yml
│       └── group_vars/
├── playbooks/
│   ├── site.yml
│   ├── webservers.yml
│   └── dbservers.yml
├── roles/
│   ├── common/
│   ├── nginx/
│   ├── mysql/
│   └── app/
├── requirements.yml
└── README.md
```

### site.yml (Main Playbook)

```yaml
# playbooks/site.yml
---
- name: Apply common configuration
  hosts: all
  become: true
  roles:
    - common

- import_playbook: webservers.yml
- import_playbook: dbservers.yml
```

### webservers.yml

```yaml
# playbooks/webservers.yml
---
- name: Configure Web Servers
  hosts: webservers
  become: true
  
  roles:
    - nginx
    - php
    - app
```

---

## Next Steps

Continue to **09_Templates** to learn about Jinja2 templating in Ansible.
