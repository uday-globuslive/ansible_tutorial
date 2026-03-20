# Ansible Inventory - Complete Guide

## What is an Inventory?

An **inventory** is a file (or collection of files) that defines the hosts and groups of hosts that Ansible manages. It's like an address book for your infrastructure.

---

## Inventory Formats

Ansible supports two formats:
1. **INI format** (simple, traditional)
2. **YAML format** (structured, powerful)

---

## INI Format Inventory

### Basic Structure

```ini
# inventory/hosts

# Ungrouped hosts
server1.example.com
192.168.1.10

# Web servers group
[webservers]
web1.example.com
web2.example.com
web3.example.com

# Database servers group
[dbservers]
db1.example.com
db2.example.com

# Application servers
[appservers]
app1.example.com
app2.example.com
```

### Hosts with Variables

```ini
[webservers]
# With specific SSH port
web1.example.com ansible_port=2222

# With specific user
web2.example.com ansible_user=admin

# With multiple variables
web3.example.com ansible_host=192.168.1.13 ansible_port=22 ansible_user=deploy

# Using IP instead of hostname
192.168.1.14 ansible_host=192.168.1.14 hostname=web4
```

### Host Ranges

```ini
[webservers]
# Numeric range: web1, web2, web3, web4, web5
web[1:5].example.com

# Alphabetic range: dba, dbb, dbc
db[a:c].example.com

# With leading zeros: web01, web02, ... web10
web[01:10].example.com
```

### Group Variables

```ini
[webservers]
web1.example.com
web2.example.com

# Variables for all webservers
[webservers:vars]
http_port=80
nginx_version=1.18.0
ansible_user=webadmin

[dbservers]
db1.example.com
db2.example.com

[dbservers:vars]
db_port=5432
backup_enabled=true
```

### Group of Groups (Children)

```ini
[webservers]
web1.example.com
web2.example.com

[dbservers]
db1.example.com

[appservers]
app1.example.com

# Parent group containing other groups
[production:children]
webservers
dbservers

# All servers
[all_servers:children]
production
appservers

[production:vars]
environment=prod
monitoring=enabled
```

---

## YAML Format Inventory

### Basic Structure

```yaml
# inventory/hosts.yml
all:
  hosts:
    mail.example.com:
  children:
    webservers:
      hosts:
        web1.example.com:
        web2.example.com:
        web3.example.com:
    dbservers:
      hosts:
        db1.example.com:
        db2.example.com:
```

### With Variables

```yaml
# inventory/hosts.yml
all:
  children:
    webservers:
      hosts:
        web1.example.com:
          ansible_port: 2222
          http_port: 80
        web2.example.com:
          ansible_port: 22
          http_port: 8080
      vars:
        nginx_version: "1.18.0"
        ansible_user: webadmin
    
    dbservers:
      hosts:
        db1.example.com:
        db2.example.com:
      vars:
        db_port: 5432
        backup_enabled: true
```

### Complex Example

```yaml
# inventory/production.yml
all:
  vars:
    ansible_user: deploy
    ansible_python_interpreter: /usr/bin/python3
  
  children:
    production:
      children:
        webservers:
          hosts:
            web1.prod.example.com:
              ansible_host: 10.0.1.10
              http_port: 80
            web2.prod.example.com:
              ansible_host: 10.0.1.11
              http_port: 80
          vars:
            nginx_workers: 4
        
        dbservers:
          hosts:
            db1.prod.example.com:
              ansible_host: 10.0.2.10
              primary: true
            db2.prod.example.com:
              ansible_host: 10.0.2.11
              primary: false
          vars:
            pg_version: "14"
            
    staging:
      children:
        webservers:
          hosts:
            web1.staging.example.com:
          vars:
            nginx_workers: 2
        dbservers:
          hosts:
            db1.staging.example.com:
```

---

## Common Ansible Variables

### Connection Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `ansible_host` | IP or hostname to connect | `192.168.1.10` |
| `ansible_port` | SSH port | `22` |
| `ansible_user` | SSH username | `admin` |
| `ansible_password` | SSH password (avoid - use keys) | `secret` |
| `ansible_ssh_private_key_file` | SSH private key path | `~/.ssh/id_rsa` |
| `ansible_connection` | Connection type | `ssh`, `local`, `docker` |
| `ansible_python_interpreter` | Python path on remote | `/usr/bin/python3` |

### Privilege Escalation

| Variable | Description | Example |
|----------|-------------|---------|
| `ansible_become` | Enable privilege escalation | `true` |
| `ansible_become_method` | Method to use | `sudo`, `su`, `doas` |
| `ansible_become_user` | User to become | `root` |
| `ansible_become_password` | Password for become | `secret` |

### Example with All Variables

```ini
[webservers]
web1.example.com ansible_host=192.168.1.10 ansible_port=22 ansible_user=admin ansible_ssh_private_key_file=~/.ssh/web_key ansible_become=true ansible_become_method=sudo ansible_python_interpreter=/usr/bin/python3
```

---

## Multiple Inventory Files

### Directory Structure

```
inventory/
├── production/
│   ├── hosts.yml
│   ├── group_vars/
│   │   ├── all.yml
│   │   ├── webservers.yml
│   │   └── dbservers.yml
│   └── host_vars/
│       └── web1.example.com.yml
├── staging/
│   ├── hosts.yml
│   └── group_vars/
│       └── all.yml
└── development/
    └── hosts.yml
```

### Usage

```bash
# Use specific inventory
ansible-playbook -i inventory/production playbook.yml

# Use multiple inventories
ansible-playbook -i inventory/production -i inventory/staging playbook.yml
```

---

## Group Variables (group_vars)

### File Structure

```
inventory/
├── hosts
└── group_vars/
    ├── all.yml           # Variables for all hosts
    ├── webservers.yml    # Variables for webservers group
    └── dbservers.yml     # Variables for dbservers group
```

### group_vars/all.yml

```yaml
# Variables applied to ALL hosts
---
# Common settings
ntp_server: time.example.com
dns_servers:
  - 8.8.8.8
  - 8.8.4.4

# SSH settings
ssh_port: 22
ssh_allow_password: false

# User settings
admin_users:
  - name: admin
    key: "ssh-rsa AAAA..."
  - name: deploy
    key: "ssh-rsa BBBB..."
```

### group_vars/webservers.yml

```yaml
# Variables for web servers
---
http_port: 80
https_port: 443

nginx:
  worker_processes: auto
  worker_connections: 1024
  keepalive_timeout: 65

ssl_certificate: /etc/ssl/certs/server.crt
ssl_certificate_key: /etc/ssl/private/server.key

firewall_allowed_ports:
  - 80
  - 443
  - 22
```

### group_vars/dbservers.yml

```yaml
# Variables for database servers
---
postgresql:
  version: "14"
  port: 5432
  max_connections: 100
  shared_buffers: "256MB"

backup:
  enabled: true
  schedule: "daily"
  retention_days: 7
  location: /backup/postgres
```

---

## Host Variables (host_vars)

### File Structure

```
inventory/
├── hosts
└── host_vars/
    ├── web1.example.com.yml
    ├── web2.example.com.yml
    └── db1.example.com.yml
```

### host_vars/web1.example.com.yml

```yaml
# Variables specific to web1
---
ansible_host: 192.168.1.10
http_port: 80

# Override group variable
nginx:
  worker_processes: 4

# Host-specific settings
server_role: primary
ssl_enabled: true
```

---

## Dynamic Inventory

### What is Dynamic Inventory?

Instead of maintaining static files, dynamic inventory scripts/plugins query external sources (AWS, Azure, GCP, etc.) in real-time.

### AWS EC2 Dynamic Inventory

```yaml
# aws_ec2.yml
plugin: amazon.aws.aws_ec2
regions:
  - us-east-1
  - us-west-2
keyed_groups:
  - key: tags.Environment
    prefix: env
  - key: instance_type
    prefix: type
filters:
  instance-state-name: running
compose:
  ansible_host: public_ip_address
```

### GCP Dynamic Inventory

```yaml
# gcp.yml
plugin: google.cloud.gcp_compute
projects:
  - my-project-id
regions:
  - us-central1
  - us-east1
keyed_groups:
  - key: labels.env
    prefix: env
filters:
  - status = RUNNING
```

### Azure Dynamic Inventory

```yaml
# azure_rm.yml
plugin: azure.azcollection.azure_rm
include_vm_resource_groups:
  - production-rg
  - staging-rg
keyed_groups:
  - key: tags.Environment
    prefix: env
conditional_groups:
  webservers: "'web' in name"
```

### Usage

```bash
# Use dynamic inventory
ansible-playbook -i aws_ec2.yml playbook.yml

# List dynamic inventory
ansible-inventory -i aws_ec2.yml --list
ansible-inventory -i aws_ec2.yml --graph
```

---

## Inventory Best Practices

### 1. Use Meaningful Group Names

```ini
# Good
[web_production]
[db_staging]
[load_balancers]

# Avoid
[group1]
[servers]
[misc]
```

### 2. Layer Variables Properly

```
# Priority (lowest to highest):
inventory/group_vars/all.yml
inventory/group_vars/<group>.yml
inventory/host_vars/<host>.yml
playbook vars
extra vars (-e)
```

### 3. Separate Environments

```bash
inventory/
├── production/
├── staging/
└── development/

# Use specific environment
ansible-playbook -i inventory/production site.yml
```

### 4. Use Vault for Secrets

```yaml
# group_vars/dbservers/vault.yml (encrypted)
vault_db_password: super_secret_password
```

### 5. Document Your Inventory

```ini
# inventory/hosts

# ================================================
# Production Environment
# ================================================

# Web Tier - Nginx servers behind load balancer
# Datacenter: DC1
# Owner: Platform Team
[webservers]
web1.prod.example.com  # Primary
web2.prod.example.com  # Secondary

# Database Tier - PostgreSQL cluster
# Datacenter: DC1
# Owner: DBA Team
[dbservers]
db1.prod.example.com   # Primary (read-write)
db2.prod.example.com   # Replica (read-only)
```

---

## Inventory Commands

```bash
# List all hosts
ansible-inventory --list

# Show graph
ansible-inventory --graph

# Show specific host
ansible-inventory --host web1.example.com

# Verify inventory
ansible-inventory --list | python -m json.tool

# Test connectivity
ansible all -m ping

# List hosts in group
ansible webservers --list-hosts
```

---

## Real-World Example

### Complete Production Inventory Structure

```
inventory/
├── production/
│   ├── hosts.yml
│   ├── group_vars/
│   │   ├── all/
│   │   │   ├── vars.yml
│   │   │   └── vault.yml      # Encrypted
│   │   ├── webservers.yml
│   │   ├── dbservers.yml
│   │   └── monitoring.yml
│   └── host_vars/
│       ├── web1.prod.example.com.yml
│       └── db1.prod.example.com.yml
└── staging/
    ├── hosts.yml
    └── group_vars/
        └── all.yml
```

### hosts.yml

```yaml
all:
  children:
    web_tier:
      children:
        load_balancers:
          hosts:
            lb1.prod.example.com:
            lb2.prod.example.com:
        webservers:
          hosts:
            web[1:4].prod.example.com:
    
    data_tier:
      children:
        dbservers:
          hosts:
            db1.prod.example.com:
              role: primary
            db2.prod.example.com:
              role: replica
        cache_servers:
          hosts:
            redis1.prod.example.com:
            redis2.prod.example.com:
    
    monitoring:
      hosts:
        prometheus.prod.example.com:
        grafana.prod.example.com:
```

---

## Next Steps

Continue to **04_Playbooks** to learn how to write automation scripts for your inventory.
