# Jinja2 Templates - Complete Guide

## What are Templates? (Beginner Explanation)

**Templates** are files with placeholders that Ansible fills in with actual values. They use the **Jinja2** templating language, allowing dynamic content generation.

### Analogy: Mad Libs for Config Files

Remember Mad Libs? "The ___adjective___ dog ___verb___ over the ___noun___."

Templates work the same way:

```
Template:                              Result:
"server_name {{ hostname }};"    →    "server_name web1.example.com;"
"listen {{ port }};"             →    "listen 80;"
```

### Why Templates Matter

**Without Templates (Separate files for each server):**
```
files/
├── nginx-web1.conf    # server_name web1.example.com;
├── nginx-web2.conf    # server_name web2.example.com;
├── nginx-web3.conf    # server_name web3.example.com;
└── ... (100 more files for 100 servers!)
```

**With Templates (ONE file for ALL servers):**
```
templates/
└── nginx.conf.j2      # server_name {{ ansible_hostname }}.example.com;
                        # Automatically becomes correct for each server!
```

### The Magic Formula

```
┌───────────────┐     ┌───────────────┐     ┌───────────────────┐
│   Template    │  +  │   Variables   │  =  │   Final Config    │
│  (.j2 file)   │     │   (values)    │     │   (actual file)   │
├───────────────┤     ├───────────────┤     ├───────────────────┤
│ listen {{ p }}│     │    p: 80      │     │   listen 80;      │
└───────────────┘     └───────────────┘     └───────────────────┘
```

```
Template + Variables = Final Configuration File
```

---

## Basic Template Syntax

### Variable Substitution

```jinja2
# Using double curly braces
server_name {{ server_name }};
listen {{ http_port }};

# Accessing dictionary values
database_host: {{ database.host }}
database_port: {{ database.port }}

# Accessing list items
first_server: {{ servers[0] }}
```

### Comments

```jinja2
{# This is a comment - it won't appear in output #}

{# 
   Multi-line comment
   explaining configuration
#}

# This is a regular comment that WILL appear in output
server_name {{ server_name }};
```

### Expressions

```jinja2
{# Mathematical expressions #}
max_connections: {{ max_conn * 2 }}
memory_limit: {{ total_memory // 4 }}MB

{# String concatenation #}
log_file: {{ log_dir }}/{{ app_name }}.log

{# Conditional expression (ternary) #}
debug: {{ 'true' if debug_mode else 'false' }}
```

---

## Control Structures

### If/Else Statements

```jinja2
{% if environment == 'production' %}
log_level = warn
debug = false
{% elif environment == 'staging' %}
log_level = info
debug = true
{% else %}
log_level = debug
debug = true
{% endif %}
```

### If with Multiple Conditions

```jinja2
{% if ssl_enabled and ssl_certificate %}
listen 443 ssl;
ssl_certificate {{ ssl_certificate }};
ssl_certificate_key {{ ssl_key }};
{% endif %}

{% if ansible_os_family == 'Debian' or ansible_os_family == 'Ubuntu' %}
apt_package_manager = true
{% endif %}
```

### For Loops

```jinja2
# Simple loop
{% for item in packages %}
- {{ item }}
{% endfor %}

# Loop with index
{% for server in servers %}
server{{ loop.index }}: {{ server }}
{% endfor %}

# Loop over dictionary
{% for key, value in settings.items() %}
{{ key }} = {{ value }}
{% endfor %}
```

### Loop Variables

```jinja2
{% for item in items %}
{{ loop.index }}     {# 1-based index #}
{{ loop.index0 }}    {# 0-based index #}
{{ loop.first }}     {# True if first iteration #}
{{ loop.last }}      {# True if last iteration #}
{{ loop.length }}    {# Total number of items #}
{{ loop.revindex }}  {# Reverse 1-based index #}
{{ loop.revindex0 }} {# Reverse 0-based index #}
{% endfor %}
```

### Loop with Conditions

```jinja2
{% for user in users if user.active %}
{{ user.name }}
{% endfor %}

# With else (if list is empty)
{% for server in servers %}
{{ server }}
{% else %}
No servers configured
{% endfor %}
```

---

## Filters

Filters transform data. Use the pipe `|` character:

### String Filters

```jinja2
{{ name | upper }}          {# JOHN #}
{{ name | lower }}          {# john #}
{{ name | capitalize }}     {# John #}
{{ name | title }}          {# John Doe #}
{{ text | trim }}           {# Remove whitespace #}
{{ text | replace("old", "new") }}
{{ text | truncate(50) }}   {# Truncate to 50 chars #}
{{ text | wordwrap(80) }}   {# Wrap at 80 chars #}
```

### Number Filters

```jinja2
{{ price | round }}         {# Round to nearest int #}
{{ price | round(2) }}      {# Round to 2 decimals #}
{{ bytes | filesizeformat }} {# 1.2 MB #}
{{ number | abs }}          {# Absolute value #}
```

### List Filters

```jinja2
{{ items | join(', ') }}    {# Comma-separated #}
{{ items | length }}        {# Number of items #}
{{ items | first }}         {# First item #}
{{ items | last }}          {# Last item #}
{{ items | sort }}          {# Sorted list #}
{{ items | unique }}        {# Remove duplicates #}
{{ items | reverse }}       {# Reverse order #}
{{ items | random }}        {# Random item #}
{{ items | min }}           {# Minimum value #}
{{ items | max }}           {# Maximum value #}
{{ items | sum }}           {# Sum of values #}
```

### Default Values

```jinja2
{{ variable | default('fallback') }}
{{ port | default(8080) }}
{{ enabled | default(true) }}

{# Default for undefined AND empty #}
{{ variable | default('fallback', true) }}
```

### Type Conversion

```jinja2
{{ '123' | int }}           {# Convert to integer #}
{{ 123 | string }}          {# Convert to string #}
{{ '3.14' | float }}        {# Convert to float #}
{{ 'true' | bool }}         {# Convert to boolean #}
{{ data | to_json }}        {# Convert to JSON #}
{{ data | to_yaml }}        {# Convert to YAML #}
{{ data | to_nice_json }}   {# Pretty JSON #}
{{ data | to_nice_yaml }}   {# Pretty YAML #}
```

### Path Filters

```jinja2
{{ path | basename }}       {# filename.txt #}
{{ path | dirname }}        {# /path/to #}
{{ path | expanduser }}     {# Expand ~ #}
{{ path | realpath }}       {# Resolve symlinks #}
{{ paths | path_join }}     {# Join paths #}
```

### Regex Filters

```jinja2
{{ text | regex_search('pattern') }}
{{ text | regex_replace('old', 'new') }}
{{ text | regex_findall('pattern') }}
```

### Ansible-Specific Filters

```jinja2
{# IP address #}
{{ ansible_default_ipv4.address | ipaddr }}

{# Hash password #}
{{ 'password' | password_hash('sha512') }}

{# Combine dictionaries #}
{{ dict1 | combine(dict2) }}

{# Select items #}
{{ users | selectattr('active', 'equalto', true) | list }}

{# Map attribute #}
{{ users | map(attribute='name') | list }}
```

---

## Practical Template Examples

### 1. Nginx Configuration

```jinja2
# /etc/nginx/nginx.conf
# Managed by Ansible - Do not edit manually!

user {{ nginx_user }};
worker_processes {{ nginx_worker_processes | default('auto') }};
pid /run/nginx.pid;

events {
    worker_connections {{ nginx_worker_connections | default(1024) }};
}

http {
    sendfile on;
    tcp_nopush on;
    keepalive_timeout {{ nginx_keepalive_timeout | default(65) }};

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Logging
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log {{ nginx_log_level | default('warn') }};

    # Gzip
{% if nginx_gzip_enabled | default(true) %}
    gzip on;
    gzip_types text/plain text/css application/json application/javascript;
{% endif %}

    # Virtual Hosts
{% for vhost in nginx_vhosts %}
    server {
        listen {{ vhost.port | default(80) }};
        server_name {{ vhost.server_name }};
        root {{ vhost.root | default('/var/www/html') }};

{% if vhost.ssl_enabled | default(false) %}
        listen 443 ssl;
        ssl_certificate {{ vhost.ssl_cert }};
        ssl_certificate_key {{ vhost.ssl_key }};
{% endif %}

{% for location in vhost.locations | default([]) %}
        location {{ location.path }} {
{% if location.proxy_pass is defined %}
            proxy_pass {{ location.proxy_pass }};
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
{% else %}
            try_files $uri $uri/ =404;
{% endif %}
        }
{% endfor %}
    }
{% endfor %}
}
```

### 2. Application Configuration

```jinja2
# config.yml
# Generated by Ansible on {{ ansible_date_time.iso8601 }}
# Host: {{ inventory_hostname }}

app:
  name: {{ app_name }}
  version: {{ app_version }}
  environment: {{ environment }}
  debug: {{ debug_mode | lower }}

server:
  host: {{ server_host | default('0.0.0.0') }}
  port: {{ server_port }}
  workers: {{ server_workers | default(ansible_processor_cores) }}

database:
  driver: {{ db_driver | default('postgresql') }}
  host: {{ db_host }}
  port: {{ db_port | default(5432) }}
  name: {{ db_name }}
  user: {{ db_user }}
  password: {{ db_password }}
  pool_size: {{ db_pool_size | default(10) }}
{% if db_ssl_enabled | default(false) %}
  ssl:
    enabled: true
    ca_cert: {{ db_ssl_ca }}
{% endif %}

cache:
{% if cache_enabled | default(true) %}
  enabled: true
  driver: {{ cache_driver | default('redis') }}
  host: {{ cache_host | default('localhost') }}
  port: {{ cache_port | default(6379) }}
{% else %}
  enabled: false
{% endif %}

logging:
  level: {{ log_level | default('info') }}
  path: {{ log_path | default('/var/log/' + app_name) }}
  format: {{ log_format | default('json') }}
{% if log_rotate_enabled | default(true) %}
  rotation:
    max_size: {{ log_max_size | default('100MB') }}
    max_files: {{ log_max_files | default(10) }}
{% endif %}

features:
{% for feature, enabled in features.items() %}
  {{ feature }}: {{ enabled | lower }}
{% endfor %}
```

### 3. Hosts File

```jinja2
# /etc/hosts
# Managed by Ansible

127.0.0.1   localhost
::1         localhost ip6-localhost ip6-loopback

# Application servers
{% for host in groups['all'] %}
{{ hostvars[host]['ansible_default_ipv4']['address'] }}   {{ host }} {{ hostvars[host]['ansible_hostname'] }}
{% endfor %}

# Custom entries
{% for entry in custom_hosts | default([]) %}
{{ entry.ip }}   {{ entry.hostnames | join(' ') }}
{% endfor %}
```

### 4. Systemd Service

```jinja2
# /etc/systemd/system/{{ service_name }}.service
[Unit]
Description={{ service_description | default(service_name) }}
Documentation={{ service_docs | default('') }}
After=network.target
{% for dep in service_after | default([]) %}
After={{ dep }}
{% endfor %}
{% for wants in service_wants | default([]) %}
Wants={{ wants }}
{% endfor %}

[Service]
Type={{ service_type | default('simple') }}
User={{ service_user | default('root') }}
Group={{ service_group | default(service_user | default('root')) }}
WorkingDirectory={{ service_working_dir | default('/opt/' + service_name) }}

{% if service_environment is defined %}
{% for key, value in service_environment.items() %}
Environment="{{ key }}={{ value }}"
{% endfor %}
{% endif %}

{% if service_env_file is defined %}
EnvironmentFile={{ service_env_file }}
{% endif %}

ExecStart={{ service_exec_start }}
{% if service_exec_stop is defined %}
ExecStop={{ service_exec_stop }}
{% endif %}

Restart={{ service_restart | default('on-failure') }}
RestartSec={{ service_restart_sec | default(5) }}

{% if service_limit_nofile is defined %}
LimitNOFILE={{ service_limit_nofile }}
{% endif %}

{% if service_memory_limit is defined %}
MemoryLimit={{ service_memory_limit }}
{% endif %}

[Install]
WantedBy=multi-user.target
```

### 5. Docker Compose

```jinja2
# docker-compose.yml
version: '3.8'

services:
{% for service in docker_services %}
  {{ service.name }}:
    image: {{ service.image }}
{% if service.build is defined %}
    build:
      context: {{ service.build.context }}
      dockerfile: {{ service.build.dockerfile | default('Dockerfile') }}
{% endif %}
{% if service.ports is defined %}
    ports:
{% for port in service.ports %}
      - "{{ port }}"
{% endfor %}
{% endif %}
{% if service.environment is defined %}
    environment:
{% for key, value in service.environment.items() %}
      {{ key }}: "{{ value }}"
{% endfor %}
{% endif %}
{% if service.volumes is defined %}
    volumes:
{% for volume in service.volumes %}
      - {{ volume }}
{% endfor %}
{% endif %}
{% if service.depends_on is defined %}
    depends_on:
{% for dep in service.depends_on %}
      - {{ dep }}
{% endfor %}
{% endif %}
{% if service.networks is defined %}
    networks:
{% for network in service.networks %}
      - {{ network }}
{% endfor %}
{% endif %}
    restart: {{ service.restart | default('unless-stopped') }}

{% endfor %}

{% if docker_networks is defined %}
networks:
{% for network in docker_networks %}
  {{ network.name }}:
    driver: {{ network.driver | default('bridge') }}
{% endfor %}
{% endif %}

{% if docker_volumes is defined %}
volumes:
{% for volume in docker_volumes %}
  {{ volume.name }}:
    driver: {{ volume.driver | default('local') }}
{% endfor %}
{% endif %}
```

### 6. HAProxy Configuration

```jinja2
# /etc/haproxy/haproxy.cfg
global
    log /dev/log local0
    log /dev/log local1 notice
    maxconn {{ haproxy_maxconn | default(4096) }}
    user haproxy
    group haproxy
    daemon

defaults
    log global
    mode http
    option httplog
    option dontlognull
    timeout connect {{ haproxy_timeout_connect | default('5000ms') }}
    timeout client {{ haproxy_timeout_client | default('50000ms') }}
    timeout server {{ haproxy_timeout_server | default('50000ms') }}
    errorfile 400 /etc/haproxy/errors/400.http
    errorfile 503 /etc/haproxy/errors/503.http

# Stats page
listen stats
    bind *:{{ haproxy_stats_port | default(8404) }}
    stats enable
    stats uri /stats
    stats auth {{ haproxy_stats_user }}:{{ haproxy_stats_password }}

{% for frontend in haproxy_frontends %}
frontend {{ frontend.name }}
    bind *:{{ frontend.port }}
{% if frontend.ssl is defined and frontend.ssl %}
    bind *:443 ssl crt {{ frontend.ssl_cert }}
    redirect scheme https if !{ ssl_fc }
{% endif %}
    default_backend {{ frontend.default_backend }}
{% if frontend.acls is defined %}
{% for acl in frontend.acls %}
    acl {{ acl.name }} {{ acl.condition }}
    use_backend {{ acl.backend }} if {{ acl.name }}
{% endfor %}
{% endif %}

{% endfor %}

{% for backend in haproxy_backends %}
backend {{ backend.name }}
    balance {{ backend.balance | default('roundrobin') }}
{% if backend.health_check is defined %}
    option {{ backend.health_check }}
{% endif %}
{% for server in backend.servers %}
    server {{ server.name }} {{ server.address }}:{{ server.port }} check{% if server.weight is defined %} weight {{ server.weight }}{% endif %}{% if server.backup is defined and server.backup %} backup{% endif %}

{% endfor %}

{% endfor %}
```

---

## Template Module Usage

### Basic Usage

```yaml
- name: Deploy configuration
  template:
    src: config.j2
    dest: /etc/myapp/config.yml
    owner: root
    group: root
    mode: '0644'
```

### With Validation

```yaml
- name: Deploy nginx config
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    validate: nginx -t -c %s
  notify: Reload nginx
```

### With Backup

```yaml
- name: Deploy config with backup
  template:
    src: config.j2
    dest: /etc/app/config.yml
    backup: yes
```

### Multiple Templates

```yaml
- name: Deploy configs
  template:
    src: "{{ item.src }}"
    dest: "{{ item.dest }}"
  loop:
    - { src: 'nginx.conf.j2', dest: '/etc/nginx/nginx.conf' }
    - { src: 'app.conf.j2', dest: '/etc/app/config.yml' }
```

---

## Template Best Practices

### 1. Add Header Comments

```jinja2
{# 
  Template: nginx.conf.j2
  Purpose: Main Nginx configuration
  Variables Required:
    - nginx_worker_processes
    - nginx_vhosts (list)
#}
# THIS FILE IS MANAGED BY ANSIBLE
# Manual changes will be overwritten
# Generated: {{ ansible_date_time.iso8601 }}
# Host: {{ inventory_hostname }}
```

### 2. Use Default Values

```jinja2
port: {{ port | default(8080) }}
workers: {{ workers | default(ansible_processor_cores) }}
```

### 3. Handle Undefined Variables

```jinja2
{% if feature_enabled is defined and feature_enabled %}
feature_flag: true
{% endif %}
```

### 4. Keep Templates Readable

```jinja2
{# Good: Clear spacing #}
{% for item in items %}
  {{ item }}
{% endfor %}

{# Bad: Cramped #}
{%for item in items%}{{item}}{%endfor%}
```

### 5. Use Whitespace Control

```jinja2
{# Control whitespace with - #}
{% for item in items -%}
{{ item }}
{%- endfor %}

{# - at start: removes preceding whitespace #}
{# - at end: removes following whitespace #}
```

---

## Next Steps

Continue to **10_Handlers** to learn about event-driven task execution.
