# Sample Inventory Files

## Understanding These Examples (Beginner Guide)

These sample inventories show real-world patterns. Start with the simple examples and progress to more complex ones as you learn.

### Inventory Complexity Levels

| Level | Example | Use Case |
|-------|---------|----------|
| 1. Basic | Simple Lab | Learning, testing |
| 2. Grouped | Production | Organizing by role |
| 3. Multi-env | YAML format | Development, staging, production |
| 4. Cloud | AWS/Azure | Dynamic cloud resources |
| 5. Kubernetes | K8s cluster | Container orchestration |

---

## 1. Simple Lab Environment (Start Here!)

This is the simplest inventory - perfect for learning:

```ini
# inventory/lab
# ═══════════════════════════════════════════════════════════════
# SIMPLE LAB INVENTORY - Perfect for beginners
# ═══════════════════════════════════════════════════════════════

# Local machine - run tasks on your own computer
[local]
localhost ansible_connection=local
# │         └── Don't use SSH, run directly
# └── This is your own machine

# Web servers - maybe VirtualBox or Vagrant VMs
[webservers]
192.168.56.10 ansible_user=vagrant
192.168.56.11 ansible_user=vagrant
# │             └── SSH username
# └── IP address of the server

# Database server
[dbservers]
192.168.56.20 ansible_user=vagrant

# Variables that apply to ALL hosts in this inventory
[all:vars]
ansible_ssh_private_key_file=~/.vagrant.d/insecure_private_key
# This SSH key works for all Vagrant VMs
```

## 2. Production Environment (INI)

### Understanding This Real-World Example

This is a complete production inventory for a company called "ACME Corporation". It shows how to organize many servers by their role.

```
                        ┌─────────────────────────────────────┐
                        │         ACME PRODUCTION             │
                        └─────────────────────────────────────┘
                                         │
        ┌────────────────────────────────┼────────────────────────────────┐
        │                                │                                │
   [frontend]                       [backend]                    [infrastructure]
        │                                │                                │
  ┌─────┴─────┐           ┌──────┬───────┼───────┬──────┐        ┌───────┴───────┐
  │           │           │      │       │       │      │        │               │
[load     [web        [app   [db     [cache   [mq    [other]  [monitoring]  [logging]
balancer] server]    server] server]        queue]
```

```ini
# inventory/production
# ═══════════════════════════════════════════════════════════════
# PRODUCTION ENVIRONMENT - ACME Corporation
# ═══════════════════════════════════════════════════════════════
# Last Updated: 2024-01-15
# Owner: DevOps Team
# 
# HOW TO USE THIS FILE:
#   ansible-playbook -i inventory/production playbook.yml
#   ansible -i inventory/production all -m ping
# ═══════════════════════════════════════════════════════════════

# ┌─────────────────────────────────────────────────────────────┐
# │ LOAD BALANCERS - The "Traffic Directors"                   │
# │ These sit in front and distribute traffic to web servers   │
# └─────────────────────────────────────────────────────────────┘
[loadbalancers]
lb1.prod.acme.com ansible_host=10.0.1.10
lb2.prod.acme.com ansible_host=10.0.1.11
# │                  └── Real IP (the DNS name might be different)
# └── Hostname (how we refer to it)

[loadbalancers:vars]
haproxy_version=2.4
keepalived_enabled=true
# Variables specific to load balancers

# ┌─────────────────────────────────────────────────────────────┐
# │ WEB SERVERS - Serve web pages to users                     │
# └─────────────────────────────────────────────────────────────┘
[webservers]
web[01:04].prod.acme.com
# └── RANGE PATTERN! This expands to:
#     web01.prod.acme.com
#     web02.prod.acme.com
#     web03.prod.acme.com
#     web04.prod.acme.com
# Much shorter than listing 4 servers individually!

[webservers:vars]
nginx_port=80
app_port=8080
deploy_user=www-data

# ┌─────────────────────────────────────────────────────────────┐
# │ APPLICATION SERVERS - Run the business logic               │
# └─────────────────────────────────────────────────────────────┘
[appservers]
app[01:03].prod.acme.com
# Expands to: app01, app02, app03

[appservers:vars]
java_version=17
spring_profile=production
jvm_heap_size=4g

# ┌─────────────────────────────────────────────────────────────┐
# │ DATABASE SERVERS - Store all the data                      │
# └─────────────────────────────────────────────────────────────┘
[dbservers]
db1.prod.acme.com ansible_host=10.0.3.10 role=primary
db2.prod.acme.com ansible_host=10.0.3.11 role=replica
#                                         └── Host-specific variable
#                                             db1 is primary (main)
#                                             db2 is replica (backup)

[dbservers:vars]
postgresql_version=15
backup_enabled=true

# ┌─────────────────────────────────────────────────────────────┐
# │ CACHE SERVERS - Fast memory storage (Redis)                │
# └─────────────────────────────────────────────────────────────┘
[cache]
redis[1:3].prod.acme.com
# Expands to: redis1, redis2, redis3

[cache:vars]
redis_maxmemory=2gb
redis_cluster_enabled=true

# ┌─────────────────────────────────────────────────────────────┐
# │ MESSAGE QUEUE - For async communication (RabbitMQ)         │
# └─────────────────────────────────────────────────────────────┘
[messagequeue]
mq[1:3].prod.acme.com

# ┌─────────────────────────────────────────────────────────────┐
# │ MONITORING STACK - Watch everything                        │
# └─────────────────────────────────────────────────────────────┘
[monitoring]
prometheus.prod.acme.com    # Collects metrics
grafana.prod.acme.com       # Displays dashboards
alertmanager.prod.acme.com  # Sends alerts

# ┌─────────────────────────────────────────────────────────────┐
# │ LOGGING STACK - Collect all logs (ELK Stack)               │
# └─────────────────────────────────────────────────────────────┘
[logging]
elasticsearch[1:3].prod.acme.com  # Stores logs
logstash.prod.acme.com            # Processes logs
kibana.prod.acme.com              # Search & visualize

# ═══════════════════════════════════════════════════════════════
# GROUP HIERARCHIES (GROUPS OF GROUPS)
# ═══════════════════════════════════════════════════════════════
# The ":children" suffix means this group contains OTHER GROUPS
# Think of it like folders containing other folders

# All frontend services (user-facing)
[frontend:children]
loadbalancers
webservers

# All backend services (internal processing)
[backend:children]
appservers
dbservers
cache
messagequeue

# All infrastructure (operations)
[infrastructure:children]
monitoring
logging

# Everything in production
[production:children]
frontend
backend
infrastructure
# Now you can target ALL servers with: --limit production

# Variables that apply to ALL production servers
[production:vars]
environment=production
datacenter=dc1
ansible_user=deploy
ansible_become=true
monitoring_enabled=true
```

### Key Patterns Used

| Pattern | Example | What It Creates |
|---------|---------|-----------------|
| Range `[start:end]` | `web[01:04]` | web01, web02, web03, web04 |
| Group variables | `[group:vars]` | Settings for all hosts in group |
| Children groups | `[parent:children]` | Group that contains other groups |
| Host-specific var | `host role=primary` | Variable for just that host |

## 3. Multi-Environment (YAML) - Managing Dev, Staging, and Production

### Why YAML Format?

YAML is better for complex, hierarchical inventories:

```
INI Format                          YAML Format
├── Flat structure                  ├── Nested structure
├── Limited nesting                 ├── Unlimited nesting
├── Variables in [group:vars]       ├── Variables anywhere
└── Simple but limited              └── Powerful and flexible
```

### Understanding the Structure

```
all                          <-- Root (every inventory has this)
├── vars                     <-- Global variables
│   └── ansible_python_interpreter, ntp_server
│
└── children                 <-- Groups under "all"
    │
    ├── production           <-- Production environment
    │   ├── vars             <-- Production-specific settings
    │   └── children
    │       ├── prod_webservers
    │       └── prod_dbservers
    │
    ├── staging              <-- Staging environment
    │   └── (similar structure)
    │
    └── development          <-- Development environment
        └── (similar structure)
```

```yaml
# inventory/all_environments.yml
# ═══════════════════════════════════════════════════════════════
# MULTI-ENVIRONMENT INVENTORY
# ═══════════════════════════════════════════════════════════════
# This single file manages Dev, Staging, and Production!
#
# USAGE:
#   Deploy to production only:
#     ansible-playbook -i inventory/all_environments.yml site.yml --limit production
#
#   Deploy to staging:
#     ansible-playbook -i inventory/all_environments.yml site.yml --limit staging
#
#   Deploy everywhere:
#     ansible-playbook -i inventory/all_environments.yml site.yml
# ═══════════════════════════════════════════════════════════════
---
all:
  # ┌────────────────────────────────────────┐
  # │ GLOBAL VARIABLES (apply everywhere)   │
  # └────────────────────────────────────────┘
  vars:
    ansible_python_interpreter: /usr/bin/python3  # Use Python 3 on all servers
    ntp_server: time.company.com                   # Same time server everywhere
    
  children:
    # ════════════════════════════════════════════════════════════
    # PRODUCTION ENVIRONMENT
    # ════════════════════════════════════════════════════════════
    production:
      vars:
        environment: production
        log_level: warn           # Less logging in prod (performance)
        monitoring_enabled: true  # Always monitor production!
        
      children:
        prod_webservers:
          hosts:
            web1.prod.company.com:        # Host entry
              ansible_host: 10.0.1.10     # Actual IP address
            web2.prod.company.com:
              ansible_host: 10.0.1.11
          vars:
            http_port: 80
            worker_processes: 8           # High performance (8 workers)
            
        prod_dbservers:
          hosts:
            db1.prod.company.com:
              ansible_host: 10.0.2.10
              db_role: primary            # This is the main database
            db2.prod.company.com:
              ansible_host: 10.0.2.11
              db_role: replica            # This is the backup
          vars:
            pg_max_connections: 500       # Handle many connections
            
    # ════════════════════════════════════════════════════════════
    # STAGING ENVIRONMENT (test before production)
    # ════════════════════════════════════════════════════════════
    staging:
      vars:
        environment: staging
        log_level: info          # More logging than prod
        monitoring_enabled: true
        
      children:
        stg_webservers:
          hosts:
            web1.stg.company.com:     # Smaller setup (just 1 server)
          vars:
            http_port: 80
            worker_processes: 2       # Fewer resources needed
            
        stg_dbservers:
          hosts:
            db1.stg.company.com:      # Single database (no replica)
          vars:
            pg_max_connections: 50    # Don't need many connections
            
    # ════════════════════════════════════════════════════════════
    # DEVELOPMENT ENVIRONMENT (for developers)
    # ════════════════════════════════════════════════════════════
    development:
      vars:
        environment: development
        log_level: debug           # Maximum logging for debugging
        monitoring_enabled: false  # No monitoring in dev
        
      children:
        dev_webservers:
          hosts:
            web1.dev.company.com:
          vars:
            worker_processes: 1    # Minimal resources
            
        dev_dbservers:
          hosts:
            db1.dev.company.com:
            
    # ════════════════════════════════════════════════════════════
    # CROSS-ENVIRONMENT GROUPS (Smart way to target all of a type)
    # ════════════════════════════════════════════════════════════
    # These groups let you target ALL webservers regardless of env
    
    all_webservers:          # ansible-playbook site.yml --limit all_webservers
      children:
        prod_webservers:
        stg_webservers:
        dev_webservers:
        
    all_dbservers:           # ansible-playbook site.yml --limit all_dbservers
      children:
        prod_dbservers:
        stg_dbservers:
        dev_dbservers:
```

### Variable Inheritance in This Structure

```
all:
└── vars: ntp_server=time.company.com ──────────────────┐
                                                         │ inherited by ALL
    └── production:                                      │
        └── vars: log_level=warn ────────────────────┐  │
                                                      │  │
            └── prod_webservers:                      │  │
                └── vars: worker_processes=8 ────┐   │  │
                                                  │   │  │
                    └── web1.prod.company.com:    │   │  │
                        # This host has:          │   │  │
                        # - ntp_server         <──┼───┼──┘
                        # - log_level=warn     <──┼───┘
                        # - worker_processes=8 <──┘
```

## 4. Cloud Provider Inventory (AWS Example)

### Understanding Cloud Inventories

Cloud servers are different from traditional servers:
- They have **instance IDs** (like `i-1234567890abcdef0`)
- They're organized by **regions** (us-east-1, us-west-2, etc.)
- They have **instance types** (t3.micro, r5.large, etc.)

```
AWS Cloud Structure
═══════════════════════════════════════════════════════
                    [ AWS Account ]
                          │
        ┌─────────────────┴─────────────────┐
        │                                   │
   [us-east-1]                         [us-west-2]
   N. Virginia                          Oregon
        │                                   │
   ┌────┴────┐                        ┌────┴────┐
   │         │                        │         │
 [web]     [db]                     [web]     [db]
```

```yaml
# inventory/aws.yml
# ═══════════════════════════════════════════════════════════════
# AWS CLOUD INVENTORY
# ═══════════════════════════════════════════════════════════════
# NOTE: For truly dynamic inventory, you'd use the AWS EC2 plugin
# This is a static example showing the structure
# ═══════════════════════════════════════════════════════════════
---
all:
  children:
    aws:
      # ┌────────────────────────────────────────────────────────┐
      # │ GLOBAL AWS SETTINGS                                   │
      # └────────────────────────────────────────────────────────┘
      vars:
        ansible_user: ec2-user                              # Default user for Amazon Linux
        ansible_ssh_private_key_file: ~/.ssh/aws-key.pem    # Your AWS SSH key
        cloud_provider: aws                                  # Custom variable for reference
        
      children:
        # ══════════════════════════════════════════════════════
        # US EAST 1 (N. Virginia) - Primary Region
        # ══════════════════════════════════════════════════════
        us_east_1:
          vars:
            aws_region: us-east-1
            
          children:
            us_east_1_web:
              hosts:
                web-use1-001.aws.company.com:
                  instance_id: i-1234567890abcdef0     # AWS assigns this ID
                  instance_type: t3.medium             # 2 vCPU, 4GB RAM
                web-use1-002.aws.company.com:
                  instance_id: i-0987654321abcdef0
                  instance_type: t3.medium
                  
            us_east_1_db:
              hosts:
                db-use1-001.aws.company.com:
                  instance_type: r5.large              # Memory optimized (2 vCPU, 16GB)
                  
        # ══════════════════════════════════════════════════════
        # US WEST 2 (Oregon) - DR Region
        # ══════════════════════════════════════════════════════
        us_west_2:
          vars:
            aws_region: us-west-2
            
          children:
            us_west_2_web:
              hosts:
                web-usw2-001.aws.company.com:
                web-usw2-002.aws.company.com:
```

### Pro Tip: Dynamic Inventory

For real AWS deployments, use dynamic inventory plugins:

```ini
# aws_ec2.yml - Ansible will auto-discover EC2 instances!
plugin: amazon.aws.aws_ec2
regions:
  - us-east-1
  - us-west-2
keyed_groups:
  - key: tags.Environment  # Group by tags
    prefix: env
```

## 5. Kubernetes Cluster Inventory

### Understanding Kubernetes Node Types

```
Kubernetes Cluster Structure
═══════════════════════════════════════════════════════════════

   ┌─────────────────── MASTER NODES ───────────────────┐
   │  Control the cluster (brain)                       │
   │  ┌─────────┐   ┌─────────┐   ┌─────────┐          │
   │  │ master1 │   │ master2 │   │ master3 │          │
   │  │   🧠    │   │   🧠    │   │   🧠    │          │
   │  └─────────┘   └─────────┘   └─────────┘          │
   │  (Usually 3 or 5 for high availability)           │
   └────────────────────────────────────────────────────┘
                           │
                           ▼
   ┌─────────────────── WORKER NODES ───────────────────┐
   │  Run your applications (muscles)                   │
   │  ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ... (many)   │
   │  │ w1 │ │ w2 │ │ w3 │ │ w4 │ │ w5 │              │
   │  │ 💪 │ │ 💪 │ │ 💪 │ │ 💪 │ │ 💪 │              │
   │  └────┘ └────┘ └────┘ └────┘ └────┘              │
   └────────────────────────────────────────────────────┘
                           │
                           ▼
   ┌────────────────── INGRESS NODES ───────────────────┐
   │  Handle external traffic (doors)                   │
   │  ┌─────────┐   ┌─────────┐                        │
   │  │ ingres1 │   │ ingres2 │                        │
   │  │   🚪    │   │   🚪    │                        │
   │  └─────────┘   └─────────┘                        │
   └────────────────────────────────────────────────────┘
```

```yaml
# inventory/kubernetes.yml
# ═══════════════════════════════════════════════════════════════
# KUBERNETES CLUSTER INVENTORY
# ═══════════════════════════════════════════════════════════════
# Use this to set up a Kubernetes cluster with Ansible
# (tools like kubeadm, kubespray use similar structures)
# ═══════════════════════════════════════════════════════════════
---
all:
  children:
    k8s_cluster:
      # ┌────────────────────────────────────────────────────────┐
      # │ CLUSTER-WIDE SETTINGS                                 │
      # └────────────────────────────────────────────────────────┘
      vars:
        k8s_version: "1.28"              # Kubernetes version
        container_runtime: containerd     # What runs containers
        pod_network_cidr: "10.244.0.0/16" # IP range for pods
        
      children:
        # ══════════════════════════════════════════════════════
        # MASTER NODES (Control Plane)
        # ══════════════════════════════════════════════════════
        # These run: API Server, Scheduler, Controller Manager, etcd
        k8s_masters:
          hosts:
            k8s-master-1.company.com:
              ansible_host: 10.10.1.10
            k8s-master-2.company.com:
              ansible_host: 10.10.1.11
            k8s-master-3.company.com:
              ansible_host: 10.10.1.12
          vars:
            node_role: master
            # 3 masters = high availability (if one fails, others take over)
            
        # ══════════════════════════════════════════════════════
        # WORKER NODES (Run your applications)
        # ══════════════════════════════════════════════════════
        k8s_workers:
          hosts:
            k8s-worker-[1:10].company.com:
            # Range pattern: k8s-worker-1 through k8s-worker-10
            # That's 10 worker nodes!
          vars:
            node_role: worker
            kubelet_extra_args: "--max-pods=110"
            # Each worker can run up to 110 pods
            
        # ══════════════════════════════════════════════════════
        # INGRESS NODES (Handle external traffic)
        # ══════════════════════════════════════════════════════
        k8s_ingress:
          hosts:
            k8s-ingress-1.company.com:
            k8s-ingress-2.company.com:
          vars:
            node_role: ingress
            node_labels:
              - "node-role.kubernetes.io/ingress="
            # Ingress controllers (like NGINX) run here
```

## 6. Vagrant Development Inventory

### What is Vagrant?

Vagrant creates virtual machines on your laptop for testing. Perfect for learning!

```
Your Computer (Host)
═══════════════════════════════════════════════════════
│
├── VirtualBox/VMware (Hypervisor)
│   │
│   ├── VM: web (192.168.56.10) ──── Web server for testing
│   │
│   └── VM: db (192.168.56.20) ───── Database for testing
│
└── Ansible connects to these VMs via SSH
```

```ini
# inventory/vagrant
# ═══════════════════════════════════════════════════════════════
# VAGRANT DEVELOPMENT INVENTORY
# ═══════════════════════════════════════════════════════════════
# Perfect for learning Ansible on your laptop!
#
# Prerequisites:
#   1. Install VirtualBox: https://www.virtualbox.org/
#   2. Install Vagrant: https://www.vagrantup.com/
#   3. Create VMs with: vagrant up
# ═══════════════════════════════════════════════════════════════

# Group all Vagrant machines together
[vagrant:children]
webservers
dbservers

[webservers]
web ansible_host=192.168.56.10 ansible_port=22
#│   │                          │
#│   │                          └── SSH port (22 is default)
#│   └── IP address of the VM
#└── Nickname for this host

[dbservers]
db ansible_host=192.168.56.20 ansible_port=22

# Settings that apply to ALL Vagrant VMs
[vagrant:vars]
ansible_user=vagrant                   # Vagrant VMs use "vagrant" user
ansible_ssh_private_key_file=.vagrant/machines/default/virtualbox/private_key
#                            └── Vagrant creates this key automatically
ansible_python_interpreter=/usr/bin/python3
```

### Matching Vagrantfile

```ruby
# Vagrantfile for this inventory
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/focal64"
  
  config.vm.define "web" do |web|
    web.vm.network "private_network", ip: "192.168.56.10"
  end
  
  config.vm.define "db" do |db|
    db.vm.network "private_network", ip: "192.168.56.20"
  end
end
```

## 7. Docker Containers Inventory

### How Ansible Connects to Docker

Unlike regular servers, Docker containers don't use SSH:

```
Regular Server                      Docker Container
═══════════════════                ═══════════════════
                                   
 ┌───────────────┐                  ┌───────────────┐
 │    Server     │                  │   Container   │
 │               │                  │               │
 │  SSH Server   │                  │   No SSH!     │
 │   Port 22     │                  │               │
 └───────────────┘                  └───────────────┘
        │                                  │
        ▼                                  ▼
 ansible_connection=ssh              ansible_connection=docker
 (Uses SSH to connect)              (Uses Docker API to connect)
```

```yaml
# inventory/docker.yml
# ═══════════════════════════════════════════════════════════════
# DOCKER CONTAINERS INVENTORY
# ═══════════════════════════════════════════════════════════════
# Ansible can manage Docker containers directly!
# No SSH needed - uses Docker API instead
# ═══════════════════════════════════════════════════════════════
---
all:
  children:
    # ┌────────────────────────────────────────────────────────┐
    # │ DOCKER CONTAINERS (managed via Docker API)            │
    # └────────────────────────────────────────────────────────┘
    docker_containers:
      vars:
        ansible_connection: docker    # KEY: Use Docker, not SSH!
        
      hosts:
        nginx_container:              # Must match container name
          ansible_docker_extra_args: "-H tcp://localhost:2375"
          # Connect to Docker daemon on this address
        redis_container:
        postgres_container:
        # These container names must exist (docker ps)
        
    # ┌────────────────────────────────────────────────────────┐
    # │ DOCKER HOSTS (the servers running Docker)             │
    # └────────────────────────────────────────────────────────┘
    docker_hosts:
      vars:
        ansible_connection: ssh       # Normal SSH connection
        
      hosts:
        docker-host-1.company.com:    # Server running Docker
        docker-host-2.company.com:
```

### When to Use Docker Connection

| Use Case | Connection Type |
|----------|-----------------|
| Configure running containers | `ansible_connection: docker` |
| Install Docker on servers | `ansible_connection: ssh` |
| Build Docker images | Use `docker_image` module with SSH |
| Deploy containers | Use `docker_container` module with SSH |

---

## Usage Examples

### Common Commands Explained

```bash
# ═══════════════════════════════════════════════════════════════
# USING INVENTORIES WITH ANSIBLE
# ═══════════════════════════════════════════════════════════════

# 1. Use a specific inventory file
ansible-playbook -i inventory/production site.yml
#                 │  │                     │
#                 │  │                     └── The playbook to run
#                 │  └── Path to your inventory
#                 └── "inventory" flag

# 2. Limit to specific group or host
ansible-playbook -i inventory/production site.yml --limit webservers
#                                                   │       │
#                                                   │       └── Only run on webservers group
#                                                   └── Restrict which hosts to target

# More limit examples:
# --limit "web1,web2"           # Specific hosts
# --limit "webservers:&staging" # Webservers AND in staging (intersection)
# --limit "!dbservers"          # NOT dbservers (exclude)
# --limit "webservers:dbservers" # Webservers OR dbservers (union)

# 3. Use multiple inventories at once
ansible-playbook -i inventory/production -i inventory/staging site.yml
#                 └─────────────┬───────────────────────────────┘
#                               └── Ansible merges both inventories!

# 4. View your inventory (very useful for debugging!)
ansible-inventory -i inventory/production --graph
#                                           │
#                                           └── Show hierarchy view
# Output looks like:
# @all:
#   |--@frontend:
#   |  |--@loadbalancers:
#   |  |  |--lb1.prod.acme.com
#   |  |  |--lb2.prod.acme.com
#   |  |--@webservers:
#   |  |  |--web01.prod.acme.com
#   ...

# 5. Show all variables for a host
ansible-inventory -i inventory/production --list
#                                           │
#                                           └── JSON output with all variables

# 6. Show specific host details
ansible-inventory -i inventory/production --host db1.prod.acme.com
#                                           │     │
#                                           │     └── Specific host to inspect
#                                           └── Show host variables

# 7. Quick ping test to verify connectivity
ansible -i inventory/production all -m ping
#                                │     │
#                                │     └── Use ping module
#                                └── Target all hosts
```

### Summary Table

| Task | Command |
|------|---------|
| Run playbook | `ansible-playbook -i inventory/prod site.yml` |
| Limit to group | `--limit webservers` |
| Limit to host | `--limit web1.prod.acme.com` |
| Exclude group | `--limit '!dbservers'` |
| View graph | `ansible-inventory -i inv --graph` |
| View JSON | `ansible-inventory -i inv --list` |
| Test connectivity | `ansible -i inv all -m ping` |
