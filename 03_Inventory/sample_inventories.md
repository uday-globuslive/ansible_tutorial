# Sample Inventory Files

## 1. Simple Lab Environment

```ini
# inventory/lab
[local]
localhost ansible_connection=local

[webservers]
192.168.56.10 ansible_user=vagrant
192.168.56.11 ansible_user=vagrant

[dbservers]
192.168.56.20 ansible_user=vagrant

[all:vars]
ansible_ssh_private_key_file=~/.vagrant.d/insecure_private_key
```

## 2. Production Environment (INI)

```ini
# inventory/production

# =============================================
# PRODUCTION ENVIRONMENT - ACME Corporation
# Last Updated: 2024-01-15
# Owner: DevOps Team
# =============================================

# Load Balancers (HAProxy)
[loadbalancers]
lb1.prod.acme.com ansible_host=10.0.1.10
lb2.prod.acme.com ansible_host=10.0.1.11

[loadbalancers:vars]
haproxy_version=2.4
keepalived_enabled=true

# Web Servers (Nginx + Application)
[webservers]
web[01:04].prod.acme.com

[webservers:vars]
nginx_port=80
app_port=8080
deploy_user=www-data

# Application Servers (Java/Spring)
[appservers]
app[01:03].prod.acme.com

[appservers:vars]
java_version=17
spring_profile=production
jvm_heap_size=4g

# Database Servers (PostgreSQL)
[dbservers]
db1.prod.acme.com ansible_host=10.0.3.10 role=primary
db2.prod.acme.com ansible_host=10.0.3.11 role=replica

[dbservers:vars]
postgresql_version=15
backup_enabled=true

# Cache Servers (Redis)
[cache]
redis[1:3].prod.acme.com

[cache:vars]
redis_maxmemory=2gb
redis_cluster_enabled=true

# Message Queue (RabbitMQ)
[messagequeue]
mq[1:3].prod.acme.com

# Monitoring Stack
[monitoring]
prometheus.prod.acme.com
grafana.prod.acme.com
alertmanager.prod.acme.com

# Logging Stack
[logging]
elasticsearch[1:3].prod.acme.com
logstash.prod.acme.com
kibana.prod.acme.com

# =====  GROUP HIERARCHIES =====

# All frontend services
[frontend:children]
loadbalancers
webservers

# All backend services
[backend:children]
appservers
dbservers
cache
messagequeue

# All infrastructure
[infrastructure:children]
monitoring
logging

# All production servers
[production:children]
frontend
backend
infrastructure

# Global production variables
[production:vars]
environment=production
datacenter=dc1
ansible_user=deploy
ansible_become=true
monitoring_enabled=true
```

## 3. Multi-Environment (YAML)

```yaml
# inventory/all_environments.yml
---
all:
  vars:
    ansible_python_interpreter: /usr/bin/python3
    ntp_server: time.company.com
    
  children:
    # ================== PRODUCTION ==================
    production:
      vars:
        environment: production
        log_level: warn
        monitoring_enabled: true
        
      children:
        prod_webservers:
          hosts:
            web1.prod.company.com:
              ansible_host: 10.0.1.10
            web2.prod.company.com:
              ansible_host: 10.0.1.11
          vars:
            http_port: 80
            worker_processes: 8
            
        prod_dbservers:
          hosts:
            db1.prod.company.com:
              ansible_host: 10.0.2.10
              db_role: primary
            db2.prod.company.com:
              ansible_host: 10.0.2.11
              db_role: replica
          vars:
            pg_max_connections: 500
            
    # ================== STAGING ==================
    staging:
      vars:
        environment: staging
        log_level: info
        monitoring_enabled: true
        
      children:
        stg_webservers:
          hosts:
            web1.stg.company.com:
          vars:
            http_port: 80
            worker_processes: 2
            
        stg_dbservers:
          hosts:
            db1.stg.company.com:
          vars:
            pg_max_connections: 50
            
    # ================== DEVELOPMENT ==================
    development:
      vars:
        environment: development
        log_level: debug
        monitoring_enabled: false
        
      children:
        dev_webservers:
          hosts:
            web1.dev.company.com:
          vars:
            worker_processes: 1
            
        dev_dbservers:
          hosts:
            db1.dev.company.com:
            
    # ================== CROSS-ENVIRONMENT GROUPS ==================
    all_webservers:
      children:
        prod_webservers:
        stg_webservers:
        dev_webservers:
        
    all_dbservers:
      children:
        prod_dbservers:
        stg_dbservers:
        dev_dbservers:
```

## 4. Cloud Provider Inventory

```yaml
# inventory/aws.yml
---
all:
  children:
    aws:
      vars:
        ansible_user: ec2-user
        ansible_ssh_private_key_file: ~/.ssh/aws-key.pem
        cloud_provider: aws
        
      children:
        us_east_1:
          vars:
            aws_region: us-east-1
            
          children:
            us_east_1_web:
              hosts:
                web-use1-001.aws.company.com:
                  instance_id: i-1234567890abcdef0
                  instance_type: t3.medium
                web-use1-002.aws.company.com:
                  instance_id: i-0987654321abcdef0
                  instance_type: t3.medium
                  
            us_east_1_db:
              hosts:
                db-use1-001.aws.company.com:
                  instance_type: r5.large
                  
        us_west_2:
          vars:
            aws_region: us-west-2
            
          children:
            us_west_2_web:
              hosts:
                web-usw2-001.aws.company.com:
                web-usw2-002.aws.company.com:
```

## 5. Kubernetes Cluster Inventory

```yaml
# inventory/kubernetes.yml
---
all:
  children:
    k8s_cluster:
      vars:
        k8s_version: "1.28"
        container_runtime: containerd
        pod_network_cidr: "10.244.0.0/16"
        
      children:
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
            
        k8s_workers:
          hosts:
            k8s-worker-[1:10].company.com:
          vars:
            node_role: worker
            kubelet_extra_args: "--max-pods=110"
            
        k8s_ingress:
          hosts:
            k8s-ingress-1.company.com:
            k8s-ingress-2.company.com:
          vars:
            node_role: ingress
            node_labels:
              - "node-role.kubernetes.io/ingress="
```

## 6. Vagrant Development Inventory

```ini
# inventory/vagrant

[vagrant:children]
webservers
dbservers

[webservers]
web ansible_host=192.168.56.10 ansible_port=22

[dbservers]
db ansible_host=192.168.56.20 ansible_port=22

[vagrant:vars]
ansible_user=vagrant
ansible_ssh_private_key_file=.vagrant/machines/default/virtualbox/private_key
ansible_python_interpreter=/usr/bin/python3
```

## 7. Docker Containers Inventory

```yaml
# inventory/docker.yml
---
all:
  children:
    docker_containers:
      vars:
        ansible_connection: docker
        
      hosts:
        nginx_container:
          ansible_docker_extra_args: "-H tcp://localhost:2375"
        redis_container:
        postgres_container:
        
    docker_hosts:
      vars:
        ansible_connection: ssh
        
      hosts:
        docker-host-1.company.com:
        docker-host-2.company.com:
```

## Usage Examples

```bash
# Use specific inventory
ansible-playbook -i inventory/production site.yml

# Limit to specific group
ansible-playbook -i inventory/production site.yml --limit webservers

# Use multiple inventories
ansible-playbook -i inventory/production -i inventory/staging site.yml

# List hosts in inventory
ansible-inventory -i inventory/production --graph
ansible-inventory -i inventory/production --list
```
