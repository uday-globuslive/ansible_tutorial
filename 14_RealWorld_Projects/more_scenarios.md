# More Real-World Scenarios

## Understanding These Advanced Scenarios (Beginner's Guide)

These projects build on basic Ansible knowledge. They solve problems you'll encounter in real IT jobs.

### Scenario Difficulty Levels

| Project | Difficulty | Key Concepts |
|---------|------------|--------------|
| Disaster Recovery | ⭐⭐⭐ | Backups, S3, Scheduled tasks |
| Complete Monitoring | ⭐⭐⭐⭐ | Multi-component systems |
| CI/CD Pipeline | ⭐⭐⭐⭐ | Integration with Jenkins/GitLab |
| Multi-Tier App | ⭐⭐⭐⭐⭐ | Complex architectures |

---

## Project 6: Disaster Recovery Automation

### What Is Disaster Recovery?

```
DISASTER RECOVERY = Having a plan when things go wrong
═══════════════════════════════════════════════════════════════

   😊 Normal Day                     💥 Disaster Strikes!
   ─────────────                     ────────────────────
   
   ┌─────────────┐                   ┌─────────────┐
   │   Server    │                   │   Server    │ ← Server dies!
   │   Working   │     ──────►       │   DEAD ☠️   │
   │   Happily   │                   │             │
   └─────────────┘                   └─────────────┘
         │                                  │
         ▼                                  ▼
   ┌─────────────┐                   ┌─────────────┐
   │   Backups   │     ──────►       │  RESTORE    │ ← We can recover!
   │   Created   │                   │  from       │
   │   Daily     │                   │  Backups    │
   └─────────────┘                   └─────────────┘

   This playbook AUTOMATES the backup process!
```

### Automated Backup and Restore

```yaml
# playbooks/disaster_recovery.yml
# ═══════════════════════════════════════════════════════════════
# AUTOMATED DISASTER RECOVERY
# ═══════════════════════════════════════════════════════════════
# 
# What this playbook does:
#   1. Creates compressed backups of important files
#   2. Uploads backups to Amazon S3 (cloud storage)
#   3. Deletes old backups to save space
#   4. Backs up databases separately
#
# How to run:
#   ansible-playbook -i inventory disaster_recovery.yml
#
# Tip: Schedule this with cron to run daily at 2 AM:
#   0 2 * * * ansible-playbook -i /path/to/inventory /path/to/disaster_recovery.yml
# ═══════════════════════════════════════════════════════════════
---
- name: Disaster Recovery - Create Backups
  hosts: all
  become: true
  
  vars:
    backup_root: /backup                    # Where to store backups locally
    backup_retention_days: 7                # Keep backups for 7 days
    s3_bucket: company-backups              # Your S3 bucket name
    
    # What to backup - organized by type
    backup_types:
      - name: system_config                 # System configuration files
        paths:
          - /etc/nginx                      # Web server config
          - /etc/ssh                        # SSH config
          - /etc/hosts                      # Hostnames
          
      - name: application                   # Your application files
        paths:
          - /opt/app/config                 # App settings
          - /opt/app/data                   # App data
          
      - name: logs                          # Important logs
        paths:
          - /var/log/app                    # Application logs
  
  tasks:
    # ───────────────────────────────────────────────────────────────
    # STEP 1: Create today's backup folder
    # ───────────────────────────────────────────────────────────────
    - name: Create backup directory
      file:
        path: "{{ backup_root }}/{{ ansible_date_time.date }}"
        state: directory
        mode: '0700'                        # Only root can access
    
    # ───────────────────────────────────────────────────────────────
    # STEP 2: Create compressed archives of each backup type
    # ───────────────────────────────────────────────────────────────
    - name: Create backups
      archive:
        path: "{{ item.paths }}"            # Files/folders to backup
        dest: "{{ backup_root }}/{{ ansible_date_time.date }}/{{ item.name }}.tar.gz"
        format: gz                          # Compressed with gzip
      loop: "{{ backup_types }}"            # Do this for each type
      register: backup_results              # Save results for next step
    
    # ───────────────────────────────────────────────────────────────
    # STEP 3: Upload to cloud (S3) for safekeeping
    # ───────────────────────────────────────────────────────────────
    - name: Upload to S3
      amazon.aws.s3_object:
        bucket: "{{ s3_bucket }}"
        src: "{{ backup_root }}/{{ ansible_date_time.date }}/{{ item.item.name }}.tar.gz"
        object: "{{ inventory_hostname }}/{{ ansible_date_time.date }}/{{ item.item.name }}.tar.gz"
        mode: put
      loop: "{{ backup_results.results }}"
      delegate_to: localhost                # Run from Ansible controller
    
    # ───────────────────────────────────────────────────────────────
    # STEP 4: Clean up old backups (save disk space)
    # ───────────────────────────────────────────────────────────────
    - name: Remove old local backups
      find:
        paths: "{{ backup_root }}"
        file_type: directory
        age: "{{ backup_retention_days }}d" # Older than X days
      register: old_backups
    
    - name: Delete old backups
      file:
        path: "{{ item.path }}"
        state: absent
      loop: "{{ old_backups.files }}"

# ═══════════════════════════════════════════════════════════════
# DATABASE BACKUPS (separate because they need special handling)
# ═══════════════════════════════════════════════════════════════
- name: Backup Databases
  hosts: dbservers
  become: true
  
  tasks:
    - name: Create PostgreSQL backup
      postgresql_db:
        name: "{{ item }}"                  # Database name
        state: dump                         # Create dump file
        target: "{{ backup_root }}/{{ ansible_date_time.date }}/{{ item }}.sql.gz"
      loop: "{{ databases }}"
      become_user: postgres
    
    - name: Upload DB backups to S3
      amazon.aws.s3_object:
        bucket: "{{ s3_bucket }}"
        src: "{{ backup_root }}/{{ ansible_date_time.date }}/{{ item }}.sql.gz"
        object: "databases/{{ ansible_date_time.date }}/{{ item }}.sql.gz"
        mode: put
      loop: "{{ databases }}"
      delegate_to: localhost

# Restore Playbook
- name: Disaster Recovery - Restore
  hosts: "{{ target_host }}"
  become: true
  
  vars_prompt:
    - name: restore_date
      prompt: "Enter backup date (YYYY-MM-DD)"
      private: no
    - name: backup_type
      prompt: "Backup type to restore (system_config/application/database)"
      private: no
  
  tasks:
    - name: Download backup from S3
      amazon.aws.s3_object:
        bucket: "{{ s3_bucket }}"
        object: "{{ inventory_hostname }}/{{ restore_date }}/{{ backup_type }}.tar.gz"
        dest: "/tmp/restore_{{ backup_type }}.tar.gz"
        mode: get
      delegate_to: localhost
    
    - name: Extract backup
      unarchive:
        src: "/tmp/restore_{{ backup_type }}.tar.gz"
        dest: /
        remote_src: yes
    
    - name: Restore database (if applicable)
      postgresql_db:
        name: "{{ db_name }}"
        state: restore
        target: "/tmp/restore_database.sql.gz"
      become_user: postgres
      when: backup_type == "database"
```

---

## Project 7: Multi-Cloud Deployment

### Deploy to AWS, Azure, and GCP

```yaml
# playbooks/multi_cloud_deploy.yml
---
- name: Deploy to AWS
  hosts: localhost
  gather_facts: no
  
  vars:
    aws_region: us-east-1
    instance_type: t3.medium
    ami_id: ami-0123456789abcdef0
  
  tasks:
    - name: Create AWS security group
      amazon.aws.ec2_security_group:
        name: webapp-sg
        description: Web application security group
        region: "{{ aws_region }}"
        rules:
          - proto: tcp
            ports: [22, 80, 443]
            cidr_ip: 0.0.0.0/0
      register: aws_sg
    
    - name: Launch AWS instances
      amazon.aws.ec2_instance:
        name: "webapp-{{ item }}"
        instance_type: "{{ instance_type }}"
        image_id: "{{ ami_id }}"
        region: "{{ aws_region }}"
        security_groups:
          - "{{ aws_sg.group_id }}"
        wait: yes
        count: 1
        tags:
          Environment: production
          Application: webapp
      loop: "{{ range(1, aws_instance_count + 1) | list }}"
      register: aws_instances
    
    - name: Add AWS instances to inventory
      add_host:
        name: "{{ item.instances[0].public_ip_address }}"
        groups: aws_servers
        ansible_user: ubuntu
        cloud_provider: aws
      loop: "{{ aws_instances.results }}"

- name: Deploy to Azure
  hosts: localhost
  gather_facts: no
  
  vars:
    azure_location: eastus
    azure_vm_size: Standard_B2s
  
  tasks:
    - name: Create Azure resource group
      azure.azcollection.azure_rm_resourcegroup:
        name: webapp-rg
        location: "{{ azure_location }}"
    
    - name: Create Azure virtual network
      azure.azcollection.azure_rm_virtualnetwork:
        resource_group: webapp-rg
        name: webapp-vnet
        address_prefixes: "10.0.0.0/16"
    
    - name: Create Azure subnet
      azure.azcollection.azure_rm_subnet:
        resource_group: webapp-rg
        name: webapp-subnet
        virtual_network: webapp-vnet
        address_prefix: "10.0.1.0/24"
    
    - name: Create Azure VM
      azure.azcollection.azure_rm_virtualmachine:
        resource_group: webapp-rg
        name: "webapp-azure-{{ item }}"
        vm_size: "{{ azure_vm_size }}"
        admin_username: azureuser
        ssh_password_enabled: false
        ssh_public_keys:
          - path: /home/azureuser/.ssh/authorized_keys
            key_data: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
        image:
          offer: 0001-com-ubuntu-server-jammy
          publisher: Canonical
          sku: 22_04-lts
          version: latest
        os_disk_name: "webapp-osdisk-{{ item }}"
        virtual_network: webapp-vnet
        subnet: webapp-subnet
      loop: "{{ range(1, azure_instance_count + 1) | list }}"
      register: azure_vms
    
    - name: Add Azure VMs to inventory
      add_host:
        name: "{{ item.ansible_facts.azure_vm.properties.networkProfile.networkInterfaces[0].properties.ipConfigurations[0].properties.publicIPAddress.properties.ipAddress }}"
        groups: azure_servers
        ansible_user: azureuser
        cloud_provider: azure
      loop: "{{ azure_vms.results }}"

- name: Deploy to GCP
  hosts: localhost
  gather_facts: no
  
  vars:
    gcp_project: my-project-id
    gcp_zone: us-central1-a
    gcp_machine_type: e2-medium
  
  tasks:
    - name: Create GCP instances
      google.cloud.gcp_compute_instance:
        name: "webapp-gcp-{{ item }}"
        machine_type: "{{ gcp_machine_type }}"
        zone: "{{ gcp_zone }}"
        project: "{{ gcp_project }}"
        disks:
          - auto_delete: true
            boot: true
            initialize_params:
              source_image: projects/ubuntu-os-cloud/global/images/family/ubuntu-2204-lts
        network_interfaces:
          - access_configs:
              - name: External NAT
                type: ONE_TO_ONE_NAT
        state: present
      loop: "{{ range(1, gcp_instance_count + 1) | list }}"
      register: gcp_instances
    
    - name: Add GCP instances to inventory
      add_host:
        name: "{{ item.networkInterfaces[0].accessConfigs[0].natIP }}"
        groups: gcp_servers
        ansible_user: ubuntu
        cloud_provider: gcp
      loop: "{{ gcp_instances.results }}"

# Now configure all cloud servers
- name: Configure All Cloud Servers
  hosts: aws_servers:azure_servers:gcp_servers
  become: true
  
  tasks:
    - name: Wait for servers to be ready
      wait_for_connection:
        timeout: 300
    
    - name: Update system
      apt:
        update_cache: yes
        upgrade: yes
    
    - name: Install common packages
      apt:
        name:
          - nginx
          - docker.io
          - python3-pip
        state: present
    
    - name: Tag with cloud provider
      debug:
        msg: "Configured server on {{ cloud_provider }}"
```

---

## Project 8: Zero-Downtime Blue-Green Deployment

```yaml
# playbooks/blue_green_deploy.yml
---
- name: Blue-Green Deployment
  hosts: localhost
  gather_facts: no
  
  vars:
    app_name: myapp
    current_color: "{{ lookup('file', 'current_color.txt', errors='ignore') | default('blue') }}"
    new_color: "{{ 'green' if current_color == 'blue' else 'blue' }}"
  
  tasks:
    - name: Display deployment info
      debug:
        msg: "Deploying from {{ current_color }} to {{ new_color }}"

- name: Deploy to New Environment
  hosts: "{{ new_color }}_servers"
  become: true
  serial: "50%"
  
  tasks:
    - name: Stop old application
      systemd:
        name: "{{ app_name }}"
        state: stopped
      ignore_errors: yes
    
    - name: Deploy new version
      git:
        repo: "{{ git_repo }}"
        dest: "/opt/{{ app_name }}"
        version: "{{ version }}"
    
    - name: Install dependencies
      command: npm ci
      args:
        chdir: "/opt/{{ app_name }}"
    
    - name: Build application
      command: npm run build
      args:
        chdir: "/opt/{{ app_name }}"
    
    - name: Start application
      systemd:
        name: "{{ app_name }}"
        state: started
    
    - name: Health check
      uri:
        url: "http://localhost:{{ app_port }}/health"
        status_code: 200
      register: health
      until: health.status == 200
      retries: 30
      delay: 5

- name: Switch Traffic
  hosts: loadbalancers
  become: true
  
  tasks:
    - name: Update HAProxy backend
      template:
        src: templates/haproxy_{{ new_color }}.cfg.j2
        dest: /etc/haproxy/haproxy.cfg
        validate: haproxy -c -f %s
      notify: Reload HAProxy
    
    - name: Gradual traffic shift (10% increments)
      include_tasks: shift_traffic.yml
      loop: "{{ range(10, 110, 10) | list }}"
      loop_control:
        loop_var: traffic_percent
  
  handlers:
    - name: Reload HAProxy
      systemd:
        name: haproxy
        state: reloaded

- name: Update Color State
  hosts: localhost
  
  tasks:
    - name: Record new color
      copy:
        content: "{{ new_color }}"
        dest: current_color.txt
    
    - name: Notify deployment complete
      slack:
        token: "{{ slack_token }}"
        channel: "#deployments"
        msg: |
          ✅ Blue-Green Deployment Complete
          New Active: {{ new_color }}
          Version: {{ version }}

# Rollback playbook
- name: Blue-Green Rollback
  hosts: localhost
  gather_facts: no
  
  vars:
    current_color: "{{ lookup('file', 'current_color.txt') }}"
    rollback_color: "{{ 'blue' if current_color == 'green' else 'green' }}"
  
  tasks:
    - name: Switch traffic back immediately
      include_tasks: switch_lb.yml
      vars:
        target_color: "{{ rollback_color }}"
    
    - name: Update color state
      copy:
        content: "{{ rollback_color }}"
        dest: current_color.txt
```

---

## Project 9: Database Migration Automation

```yaml
# playbooks/database_migration.yml
---
- name: Database Migration
  hosts: dbservers
  become: true
  serial: 1
  
  vars:
    db_name: production_db
    migration_dir: /opt/migrations
    backup_dir: /backup/db
  
  pre_tasks:
    - name: Create backup before migration
      postgresql_db:
        name: "{{ db_name }}"
        state: dump
        target: "{{ backup_dir }}/{{ db_name }}_pre_migration_{{ ansible_date_time.epoch }}.sql.gz"
      become_user: postgres
    
    - name: Get current schema version
      postgresql_query:
        db: "{{ db_name }}"
        query: "SELECT version FROM schema_migrations ORDER BY version DESC LIMIT 1"
      become_user: postgres
      register: current_version
    
    - name: Display current version
      debug:
        msg: "Current schema version: {{ current_version.query_result[0].version | default('none') }}"
  
  tasks:
    - name: Copy migration files
      copy:
        src: "{{ item }}"
        dest: "{{ migration_dir }}/"
      with_fileglob:
        - "migrations/*.sql"
    
    - name: Get pending migrations
      find:
        paths: "{{ migration_dir }}"
        patterns: "*.sql"
      register: migration_files
    
    - name: Run migrations
      block:
        - name: Apply migration
          postgresql_query:
            db: "{{ db_name }}"
            path_to_script: "{{ item.path }}"
          become_user: postgres
          loop: "{{ migration_files.files | sort(attribute='path') }}"
          register: migration_results
        
        - name: Record migration
          postgresql_query:
            db: "{{ db_name }}"
            query: "INSERT INTO schema_migrations (version, applied_at) VALUES (%s, NOW())"
            positional_args:
              - "{{ item.path | basename | regex_replace('\\.sql$', '') }}"
          become_user: postgres
          loop: "{{ migration_files.files | sort(attribute='path') }}"
      
      rescue:
        - name: Migration failed - restore backup
          postgresql_db:
            name: "{{ db_name }}"
            state: restore
            target: "{{ backup_dir }}/{{ db_name }}_pre_migration_{{ ansible_date_time.epoch }}.sql.gz"
          become_user: postgres
        
        - name: Fail with error
          fail:
            msg: "Migration failed and rolled back. Check logs for details."
  
  post_tasks:
    - name: Verify migration
      postgresql_query:
        db: "{{ db_name }}"
        query: "SELECT version FROM schema_migrations ORDER BY version DESC"
      become_user: postgres
      register: final_version
    
    - name: Display migration results
      debug:
        msg: "Migrations complete. Latest version: {{ final_version.query_result[0].version }}"
    
    - name: Notify team
      slack:
        token: "{{ slack_token }}"
        channel: "#database"
        msg: |
          ✅ Database migration complete
          Database: {{ db_name }}
          Server: {{ inventory_hostname }}
          Migrations applied: {{ migration_files.files | length }}
```

---

## Project 10: Compliance and Audit Automation

```yaml
# playbooks/compliance_audit.yml
---
- name: Compliance Audit
  hosts: all
  become: true
  
  vars:
    audit_report_dir: /var/log/compliance
    compliance_standards:
      - CIS
      - PCI-DSS
      - HIPAA
  
  tasks:
    - name: Create audit directory
      file:
        path: "{{ audit_report_dir }}"
        state: directory
        mode: '0700'
    
    # CIS Benchmark Checks
    - name: Check SSH configuration
      block:
        - name: Verify SSH root login disabled
          lineinfile:
            path: /etc/ssh/sshd_config
            regexp: '^PermitRootLogin'
            line: 'PermitRootLogin no'
          check_mode: yes
          register: ssh_root
          
        - name: Verify password authentication disabled
          lineinfile:
            path: /etc/ssh/sshd_config
            regexp: '^PasswordAuthentication'
            line: 'PasswordAuthentication no'
          check_mode: yes
          register: ssh_password
    
    - name: Check file permissions
      stat:
        path: "{{ item }}"
      loop:
        - /etc/passwd
        - /etc/shadow
        - /etc/group
        - /etc/gshadow
      register: file_perms
    
    - name: Verify file permissions
      assert:
        that:
          - item.stat.mode == '0644' or item.stat.mode == '0600' or item.stat.mode == '0640'
        quiet: yes
      loop: "{{ file_perms.results }}"
      register: perm_check
      ignore_errors: yes
    
    - name: Check for unowned files
      command: find / -nouser -o -nogroup -type f 2>/dev/null
      register: unowned_files
      changed_when: false
      failed_when: false
    
    - name: Check password policy
      lineinfile:
        path: /etc/login.defs
        regexp: "{{ item.regexp }}"
        line: "{{ item.line }}"
      check_mode: yes
      loop:
        - { regexp: '^PASS_MAX_DAYS', line: 'PASS_MAX_DAYS 90' }
        - { regexp: '^PASS_MIN_DAYS', line: 'PASS_MIN_DAYS 7' }
        - { regexp: '^PASS_WARN_AGE', line: 'PASS_WARN_AGE 14' }
      register: password_policy
    
    - name: Check for listening services
      command: ss -tuln
      register: listening_services
      changed_when: false
    
    - name: Check firewall status
      command: ufw status
      register: firewall_status
      changed_when: false
      failed_when: false
    
    - name: Generate compliance report
      template:
        src: templates/compliance_report.j2
        dest: "{{ audit_report_dir }}/report_{{ ansible_date_time.date }}.html"
      vars:
        checks:
          ssh_root_login: "{{ not ssh_root.changed }}"
          ssh_password_auth: "{{ not ssh_password.changed }}"
          file_permissions: "{{ perm_check.results | selectattr('failed', 'equalto', false) | list | length == perm_check.results | length }}"
          no_unowned_files: "{{ unowned_files.stdout_lines | length == 0 }}"
          firewall_enabled: "{{ 'active' in firewall_status.stdout | lower }}"
    
    - name: Fetch compliance report
      fetch:
        src: "{{ audit_report_dir }}/report_{{ ansible_date_time.date }}.html"
        dest: "reports/{{ inventory_hostname }}/"
        flat: yes

- name: Aggregate Compliance Reports
  hosts: localhost
  
  tasks:
    - name: Generate summary report
      template:
        src: templates/compliance_summary.j2
        dest: "reports/summary_{{ ansible_date_time.date }}.html"
    
    - name: Send compliance report
      mail:
        host: "{{ smtp_server }}"
        to: "{{ compliance_team_email }}"
        subject: "Compliance Audit Report - {{ ansible_date_time.date }}"
        body: "Please find attached the compliance audit report."
        attach:
          - "reports/summary_{{ ansible_date_time.date }}.html"
```

---

## Next Steps

Continue to **15_Troubleshooting** for debugging tips and solutions to common problems.
