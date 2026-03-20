# Advanced Ansible Topics

## When You Need Advanced Topics (Beginner Explanation)

You'll reach for advanced features when:
- Built-in modules don't do what you need
- You want to share automation across teams
- You need to transform data in special ways
- You want custom logging or notifications

### Overview of Advanced Topics

| Topic | When To Use | Difficulty |
|-------|-------------|------------|
| Custom Modules | Built-in modules don't exist for your need | Hard |
| Custom Filters | Need to transform data in unique ways | Medium |
| Callback Plugins | Want custom logging/notifications | Medium |
| Dynamic Inventory | Hosts change frequently (cloud, containers) | Medium |
| Ansible Collections | Packaging modules/roles for distribution | Medium |
| Testing with Molecule | Ensure your roles work correctly | Medium |

---

## Custom Modules (Explained)

### What is a Custom Module?

When Ansible's 3000+ modules still don't do what you need, you write your own!

**Example scenarios:**
- Interacting with your company's proprietary API
- Managing a custom in-house application
- Complex business logic that's hard to express in YAML

### How Custom Modules Work

```
┌─────────────────────┐       ┌─────────────────────┐
│ Your Playbook       │       │ library/             │
│                     │       │   my_module.py       │
│ - my_custom_module: │──────▶│                     │
│     name: thing     │       │ Python code that     │
│     state: present  │       │ does the actual work │
└─────────────────────┘       └─────────────────────┘
```

### Creating a Simple Custom Module

```python
#!/usr/bin/python
# library/my_custom_module.py

from ansible.module_utils.basic import AnsibleModule

def main():
    module = AnsibleModule(
        argument_spec=dict(
            name=dict(type='str', required=True),
            state=dict(type='str', default='present', choices=['present', 'absent']),
            force=dict(type='bool', default=False),
        ),
        supports_check_mode=True,
    )
    
    name = module.params['name']
    state = module.params['state']
    force = module.params['force']
    
    changed = False
    result = dict(
        name=name,
        state=state,
        changed=changed,
        message=''
    )
    
    # Check mode - don't make changes
    if module.check_mode:
        module.exit_json(**result)
    
    # Your logic here
    if state == 'present':
        # Create/update logic
        changed = True
        result['message'] = f'Resource {name} created'
    else:
        # Delete logic
        changed = True
        result['message'] = f'Resource {name} deleted'
    
    result['changed'] = changed
    module.exit_json(**result)

if __name__ == '__main__':
    main()
```

### Using Custom Module

```yaml
- name: Use custom module
  my_custom_module:
    name: my_resource
    state: present
    force: yes
```

---

## Custom Filters

### Creating Custom Filter

```python
# filter_plugins/my_filters.py

class FilterModule:
    def filters(self):
        return {
            'reverse_string': self.reverse_string,
            'to_environment': self.to_environment,
            'mask_password': self.mask_password,
        }
    
    def reverse_string(self, value):
        return value[::-1]
    
    def to_environment(self, data):
        """Convert dict to environment variable format"""
        return '\n'.join([f'{k.upper()}={v}' for k, v in data.items()])
    
    def mask_password(self, value, visible_chars=2):
        """Mask password showing only first N characters"""
        if len(value) <= visible_chars:
            return '*' * len(value)
        return value[:visible_chars] + '*' * (len(value) - visible_chars)
```

### Using Custom Filters

```yaml
- debug:
    msg: "{{ 'hello' | reverse_string }}"  # olleh

- debug:
    msg: "{{ db_config | to_environment }}"

- debug:
    msg: "{{ password | mask_password(2) }}"  # Se*****
```

---

## Callback Plugins

### Custom Callback for Notifications

```python
# callback_plugins/slack_notify.py

from ansible.plugins.callback import CallbackBase
import requests

class CallbackModule(CallbackBase):
    CALLBACK_VERSION = 2.0
    CALLBACK_TYPE = 'notification'
    CALLBACK_NAME = 'slack_notify'
    CALLBACK_NEEDS_WHITELIST = True
    
    def __init__(self):
        super().__init__()
        self.webhook_url = os.environ.get('SLACK_WEBHOOK_URL')
    
    def v2_playbook_on_stats(self, stats):
        hosts = sorted(stats.processed.keys())
        summary = []
        
        for host in hosts:
            s = stats.summarize(host)
            summary.append(f"{host}: ok={s['ok']} changed={s['changed']} failed={s['failures']}")
        
        message = {
            'text': f"Ansible Playbook Complete\n" + '\n'.join(summary)
        }
        
        if self.webhook_url:
            requests.post(self.webhook_url, json=message)
```

---

## Dynamic Inventory

### Custom Dynamic Inventory Script

```python
#!/usr/bin/env python3
# inventory/dynamic_inventory.py

import json
import argparse
import boto3

def get_inventory():
    """Generate inventory from AWS EC2"""
    ec2 = boto3.client('ec2')
    instances = ec2.describe_instances(
        Filters=[{'Name': 'instance-state-name', 'Values': ['running']}]
    )
    
    inventory = {
        '_meta': {'hostvars': {}},
        'all': {'hosts': [], 'children': ['webservers', 'dbservers']},
        'webservers': {'hosts': []},
        'dbservers': {'hosts': []},
    }
    
    for reservation in instances['Reservations']:
        for instance in reservation['Instances']:
            hostname = instance.get('PublicIpAddress') or instance['PrivateIpAddress']
            
            # Add to all hosts
            inventory['all']['hosts'].append(hostname)
            
            # Add to groups based on tags
            for tag in instance.get('Tags', []):
                if tag['Key'] == 'Role':
                    role = tag['Value'].lower()
                    if role in inventory:
                        inventory[role]['hosts'].append(hostname)
            
            # Add host variables
            inventory['_meta']['hostvars'][hostname] = {
                'ansible_host': hostname,
                'instance_id': instance['InstanceId'],
                'instance_type': instance['InstanceType'],
                'availability_zone': instance['Placement']['AvailabilityZone'],
            }
    
    return inventory

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument('--list', action='store_true')
    parser.add_argument('--host', default=None)
    args = parser.parse_args()
    
    inventory = get_inventory()
    
    if args.list:
        print(json.dumps(inventory, indent=2))
    elif args.host:
        print(json.dumps(inventory['_meta']['hostvars'].get(args.host, {})))

if __name__ == '__main__':
    main()
```

### Usage

```bash
chmod +x inventory/dynamic_inventory.py
ansible-playbook -i inventory/dynamic_inventory.py playbook.yml
```

---

## Ansible Tower/AWX

### Key Concepts

```
┌─────────────────────────────────────────────────────────────┐
│                    AWX / Ansible Tower                       │
├──────────────┬──────────────┬──────────────┬───────────────┤
│ Organizations│   Projects   │  Inventories │   Credentials  │
├──────────────┴──────────────┴──────────────┴───────────────┤
│                    Job Templates                             │
│  (Playbook + Inventory + Credentials + Variables)          │
├─────────────────────────────────────────────────────────────┤
│                     Workflows                                │
│  (Chain multiple Job Templates)                             │
├─────────────────────────────────────────────────────────────┤
│              Schedules & Notifications                       │
└─────────────────────────────────────────────────────────────┘
```

### Installing AWX

```bash
# Clone AWX
git clone https://github.com/ansible/awx.git
cd awx/installer

# Configure inventory
vi inventory
# Set passwords and configurations

# Run installer
ansible-playbook -i inventory install.yml
```

---

## Ansible Collections

### Creating a Collection

```bash
# Create collection skeleton
ansible-galaxy collection init my_namespace.my_collection

# Structure
my_namespace/my_collection/
├── docs/
├── galaxy.yml
├── plugins/
│   ├── modules/
│   ├── inventory/
│   ├── filters/
│   └── callback/
├── roles/
├── playbooks/
└── README.md
```

### galaxy.yml

```yaml
namespace: my_namespace
name: my_collection
version: 1.0.0
readme: README.md
authors:
  - Your Name <email@example.com>
description: My custom Ansible collection
license:
  - MIT
dependencies:
  community.general: ">=2.0.0"
tags:
  - infrastructure
  - automation
repository: https://github.com/org/repo
```

### Building & Publishing

```bash
# Build collection
ansible-galaxy collection build

# Install from galaxy
ansible-galaxy collection install my_namespace.my_collection

# Publish to Galaxy
ansible-galaxy collection publish ./my_namespace-my_collection-1.0.0.tar.gz
```

---

## Molecule Testing

### Setup

```bash
pip install molecule molecule-docker ansible-lint
cd roles/nginx
molecule init scenario -r nginx -d docker
```

### molecule/default/molecule.yml

```yaml
---
dependency:
  name: galaxy
driver:
  name: docker
platforms:
  - name: ubuntu22
    image: ubuntu:22.04
    pre_build_image: true
  - name: centos8
    image: centos:8
    pre_build_image: true
provisioner:
  name: ansible
  options:
    vvv: true
verifier:
  name: ansible
lint: |
  set -e
  ansible-lint
```

### molecule/default/converge.yml

```yaml
---
- name: Converge
  hosts: all
  become: true
  
  roles:
    - role: nginx
      vars:
        nginx_port: 8080
```

### molecule/default/verify.yml

```yaml
---
- name: Verify
  hosts: all
  become: true
  
  tasks:
    - name: Check nginx is installed
      package:
        name: nginx
        state: present
      check_mode: true
      register: pkg
      failed_when: pkg.changed
    
    - name: Check nginx is running
      service:
        name: nginx
        state: started
      check_mode: true
      register: svc
      failed_when: svc.changed
    
    - name: Check nginx responds
      uri:
        url: http://localhost:8080
        status_code: 200
```

### Running Tests

```bash
# Full test cycle
molecule test

# Individual stages
molecule create    # Create containers
molecule converge  # Run playbook
molecule verify    # Run tests
molecule destroy   # Cleanup
```

---

## Delegation & Local Actions

### Delegate to Localhost

```yaml
- name: Add host to load balancer
  uri:
    url: "http://lb.example.com/api/add"
    method: POST
    body: "{{ inventory_hostname }}"
  delegate_to: localhost

- name: Update DNS record
  route53:
    zone: example.com
    record: "{{ inventory_hostname }}"
    type: A
    value: "{{ ansible_default_ipv4.address }}"
  delegate_to: localhost
```

### Delegate to Another Host

```yaml
- name: Get database backup from primary
  command: pg_dump mydb > /tmp/backup.sql
  delegate_to: "{{ groups['dbservers'][0] }}"
```

### Run Once

```yaml
- name: Initialize cluster (run only once)
  command: /opt/cluster/init.sh
  run_once: true
  delegate_to: "{{ groups['cluster_masters'][0] }}"
```

---

## Asynchronous Actions

### Fire and Forget

```yaml
- name: Start long-running process
  command: /opt/long_process.sh
  async: 3600        # Max runtime (1 hour)
  poll: 0            # Don't wait
  register: long_job

# Later, check status
- name: Check on long process
  async_status:
    jid: "{{ long_job.ansible_job_id }}"
  register: job_result
  until: job_result.finished
  retries: 60
  delay: 60
```

### Parallel Tasks on Same Host

```yaml
- name: Run multiple tasks in parallel
  command: "{{ item }}"
  async: 300
  poll: 0
  loop:
    - /opt/task1.sh
    - /opt/task2.sh
    - /opt/task3.sh
  register: parallel_tasks

- name: Wait for all tasks
  async_status:
    jid: "{{ item.ansible_job_id }}"
  loop: "{{ parallel_tasks.results }}"
  register: results
  until: results.finished
  retries: 30
  delay: 10
```

---

## Include vs Import

### Static Import

```yaml
# Processed at playbook parse time
- import_tasks: install.yml
- import_playbook: webservers.yml
- import_role:
    name: nginx

# Conditionals apply to each imported task
- import_tasks: install.yml
  when: install_packages
```

### Dynamic Include

```yaml
# Processed at runtime
- include_tasks: "{{ ansible_os_family | lower }}.yml"
- include_role:
    name: "{{ webserver }}"
  
# Conditionals apply to include itself
- include_tasks: install.yml
  when: install_packages
```

### When to Use Which

| Use Import When | Use Include When |
|-----------------|------------------|
| Tasks are static | Tasks depend on runtime data |
| Need task tags to propagate | Using loops |
| Parse-time variables | Dynamic file names |
| Better performance | Conditional includes |

---

## Strategies

### Linear (Default)

```yaml
# All hosts complete task before next task starts
- hosts: all
  strategy: linear
```

### Free

```yaml
# Each host runs independently
- hosts: all
  strategy: free
```

### Debug

```yaml
# Interactive debugging
- hosts: all
  strategy: debug

# Then use debugger commands
# p task.args - print task arguments
# c - continue
# q - quit
```

---

## Connection Types

```yaml
# SSH (default)
- hosts: remote_servers
  connection: ssh

# Local (for localhost)
- hosts: localhost
  connection: local

# Docker container
- hosts: container1
  connection: docker

# WinRM (Windows)
- hosts: windows_servers
  connection: winrm
  vars:
    ansible_winrm_transport: basic
    ansible_winrm_server_cert_validation: ignore
```

---

## Next Steps

Continue to **14_RealWorld_Projects** for hands-on real-world scenarios.
