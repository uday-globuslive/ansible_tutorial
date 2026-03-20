# HTC Global Services - Ansible Developer Interview Prep Guide

## 📋 Job Description Summary

**Role:** Ansible Development Engineer  
**Company:** HTC Global Services  
**Location:** Chennai, Tamil Nadu, India  
**Experience:** 5-10 years

### Key Focus Areas (What They REALLY Want)
```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        HTC GLOBAL SERVICES - PRIORITIES                         │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│   🔴 TIER 1 (Must Have - They'll definitely ask about these):                  │
│   ├── Ansible Playbook Development (5+ years)                                  │
│   ├── Day 2 Operations Automation                                              │
│   ├── Self-Healing Automation                                                  │
│   ├── ServiceNow Integration                                                   │
│   └── Azure VM Management                                                      │
│                                                                                 │
│   🟡 TIER 2 (Important - Likely to ask):                                       │
│   ├── New Relic Integration                                                    │
│   ├── Moogsoft Integration                                                     │
│   ├── Linux Execution Environments                                             │
│   └── Python/Bash/PowerShell Scripting                                         │
│                                                                                 │
│   🟢 TIER 3 (Nice to Have - Bonus points):                                     │
│   ├── CI/CD Pipeline Integration                                               │
│   ├── Docker/Kubernetes                                                        │
│   └── PowerBI Integration                                                      │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

# Part 1: Understanding Day 2 Operations (Critical!)

## What Are Day 0, Day 1, Day 2 Operations?

This is a key concept the interviewer will expect you to know inside-out.

### The Timeline Analogy

Think of it like buying and owning a car:

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                            INFRASTRUCTURE LIFECYCLE                              │
├──────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  DAY 0: PLANNING                     DAY 1: DEPLOYMENT        DAY 2: OPERATIONS │
│  ════════════════                    ════════════════         ═══════════════════│
│                                                                                  │
│  🚗 Car Analogy:                     🚗 Car Analogy:          🚗 Car Analogy:    │
│  "Deciding which car               "Buying the car          "Maintaining the    │
│   to buy, budget,                   and driving it           car - oil changes, │
│   features needed"                  home"                    tire rotation,     │
│                                                              insurance renewal" │
│                                                                                  │
│  💻 Infrastructure:                 💻 Infrastructure:       💻 Infrastructure: │
│  • Architecture design             • Provision VMs          • Patching/Updates │
│  • Capacity planning               • Deploy apps            • Backup/Recovery  │
│  • Tool selection                  • Configure network      • Monitoring/Alerts│
│  • Security requirements           • Set up databases       • Scaling          │
│  • Cost estimation                 • Initial hardening      • Compliance checks│
│                                                              • Certificate mgmt │
│                                                              • Log rotation     │
│                                                              • User management  │
│                                                                                  │
│  📅 Timing:                         📅 Timing:               📅 Timing:         │
│  Before deployment                 One-time activity        Ongoing, forever   │
│                                                                                  │
│  ⏱️ Duration:                       ⏱️ Duration:             ⏱️ Duration:        │
│  Weeks of planning                 Hours/Days               Months/Years       │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### Why Day 2 Matters Most (Interview Talking Point)

> **Interview Answer:** "Day 2 Operations is where organizations spend 80% of their infrastructure effort. Anyone can provision a VM, but keeping 500 VMs patched, compliant, and operational requires sophisticated automation. That's where Ansible excels - it brings consistency and auditability to ongoing operations."

---

## Day 2 Operations Categories

```
                              DAY 2 OPERATIONS
                                     │
        ┌────────────┬─────────┬─────┴─────┬──────────┬────────────┐
        │            │         │           │          │            │
        ▼            ▼         ▼           ▼          ▼            ▼
   ┌─────────┐  ┌─────────┐ ┌────────┐ ┌────────┐ ┌─────────┐ ┌─────────┐
   │PATCHING │  │ BACKUP  │ │SCALING │ │MONITOR │ │SECURITY │ │ CONFIG  │
   │         │  │         │ │        │ │        │ │         │ │ DRIFT   │
   └────┬────┘  └────┬────┘ └───┬────┘ └───┬────┘ └────┬────┘ └────┬────┘
        │            │          │          │           │           │
   OS updates   Scheduled   Add/remove   Health     Compliance   Enforce
   App updates  snapshots   resources    checks     audits       baselines
   Hotfixes     Off-site    Auto-scale   Alerting   Hardening    Drift fix
   Reboots      Retention   Load balance Metrics    Cert renewal State sync
```

---

# Part 2: Day 2 Operations Playbooks (With Full Code)

## 2.1 Automated Patching System

### The Business Problem
> "Our 200 Linux servers need monthly security patches. Manual patching takes 3 days and sometimes breaks things. We need automated, staged patching with rollback capability."

### Architecture Overview
```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                        AUTOMATED PATCHING WORKFLOW                               │
├──────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐    │
│  │   STAGE 1   │     │   STAGE 2   │     │   STAGE 3   │     │   STAGE 4   │    │
│  │    DEV      │────▶│    TEST     │────▶│   STAGING   │────▶│   PROD      │    │
│  │  (10 VMs)   │     │  (20 VMs)   │     │  (50 VMs)   │     │  (120 VMs)  │    │
│  └─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘    │
│        │                   │                   │                   │            │
│        ▼                   ▼                   ▼                   ▼            │
│   ┌─────────┐         ┌─────────┐         ┌─────────┐         ┌─────────┐       │
│   │Validate │         │Validate │         │Validate │         │Validate │       │
│   │& Report │         │& Report │         │& Report │         │& Report │       │
│   └─────────┘         └─────────┘         └─────────┘         └─────────┘       │
│        │                   │                   │                   │            │
│        └───────────────────┴───────────────────┴───────────────────┘            │
│                                    │                                            │
│                                    ▼                                            │
│                         ┌───────────────────┐                                   │
│                         │   ServiceNow      │                                   │
│                         │   Change Ticket   │                                   │
│                         │   Auto-Updated    │                                   │
│                         └───────────────────┘                                   │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### Complete Patching Playbook

```yaml
# playbooks/day2_operations/patching/main_patching.yml
---
- name: "Day 2 Operations - Automated Patching Workflow"
  hosts: "{{ target_environment | default('dev') }}"
  become: true
  gather_facts: true
  serial: "{{ batch_size | default('20%') }}"  # Patch 20% at a time
  max_fail_percentage: 10  # Abort if >10% fail
  
  vars:
    # Patching configuration
    patching_window: "{{ lookup('env', 'PATCHING_WINDOW') | default('Saturday 02:00-06:00') }}"
    reboot_required: true
    pre_patch_snapshot: true
    servicenow_enabled: true
    rollback_on_failure: true
    
    # Exclusions (don't patch these packages)
    package_exclusions:
      - kernel*  # Kernel updates need special handling
      - docker*  # Docker updates might break containers
    
    # Health check endpoints
    health_checks:
      - name: "Web Server"
        url: "http://localhost:80/health"
        expected_status: 200
      - name: "API Service"
        url: "http://localhost:8080/api/status"
        expected_status: 200

  pre_tasks:
    # ╔═══════════════════════════════════════════════════════════════╗
    # ║ PRE-PATCHING PHASE: Prepare and validate                     ║
    # ╚═══════════════════════════════════════════════════════════════╝
    
    - name: "PRE-PATCH | Create ServiceNow change request"
      uri:
        url: "https://{{ snow_instance }}.service-now.com/api/now/table/change_request"
        method: POST
        user: "{{ snow_username }}"
        password: "{{ snow_password }}"
        force_basic_auth: true
        body_format: json
        body:
          short_description: "Automated patching - {{ inventory_hostname }}"
          description: "Day 2 Operations: Automated security patching"
          category: "Software"
          priority: 3
          assignment_group: "Linux Admins"
          cmdb_ci: "{{ inventory_hostname }}"
          start_date: "{{ ansible_date_time.iso8601 }}"
          state: "implement"
        status_code: 201
      register: snow_change
      delegate_to: localhost
      run_once: true
      when: servicenow_enabled | bool
      tags: [servicenow, pre-patch]

    - name: "PRE-PATCH | Store change request number"
      set_fact:
        change_number: "{{ snow_change.json.result.number }}"
      when: servicenow_enabled | bool and snow_change is defined

    - name: "PRE-PATCH | Create VM snapshot for rollback"
      # This would use Azure or VMware module depending on environment
      include_role:
        name: vm_snapshot
      vars:
        snapshot_name: "pre-patch-{{ ansible_date_time.date }}"
        snapshot_description: "Snapshot before patching - {{ change_number | default('manual') }}"
      when: pre_patch_snapshot | bool
      tags: [snapshot, pre-patch]

    - name: "PRE-PATCH | Drain connections (for web servers)"
      uri:
        url: "http://localhost:8080/admin/drain"
        method: POST
        timeout: 60
      when: "'webservers' in group_names"
      ignore_errors: true
      tags: [drain, pre-patch]

    - name: "PRE-PATCH | Wait for active connections to drain"
      wait_for:
        host: "{{ inventory_hostname }}"
        port: 80
        state: drained
        timeout: 300
      when: "'webservers' in group_names"
      ignore_errors: true

    - name: "PRE-PATCH | Record current package versions"
      shell: |
        rpm -qa --qf '%{NAME}-%{VERSION}-%{RELEASE}.%{ARCH}\n' | sort > /var/log/patches/pre-patch-packages-{{ ansible_date_time.date }}.txt
      args:
        creates: "/var/log/patches/pre-patch-packages-{{ ansible_date_time.date }}.txt"
      tags: [audit, pre-patch]

  tasks:
    # ╔═══════════════════════════════════════════════════════════════╗
    # ║ PATCHING PHASE: Apply updates                                 ║
    # ╚═══════════════════════════════════════════════════════════════╝
    
    - name: "PATCH | Update package cache (RHEL/CentOS)"
      yum:
        update_cache: true
        name: "*"
        state: latest
        exclude: "{{ package_exclusions | join(',') }}"
        security: true  # Only security updates
      register: patch_result
      when: ansible_os_family == "RedHat"
      tags: [patch]

    - name: "PATCH | Update packages (Ubuntu/Debian)"
      apt:
        update_cache: true
        upgrade: safe
        autoremove: true
      register: patch_result
      when: ansible_os_family == "Debian"
      tags: [patch]

    - name: "PATCH | Record what was updated"
      copy:
        content: |
          Patching Report - {{ inventory_hostname }}
          Date: {{ ansible_date_time.iso8601 }}
          Change Request: {{ change_number | default('N/A') }}
          
          Updated Packages:
          {{ patch_result.results | default(patch_result.stdout_lines) | to_nice_yaml }}
        dest: "/var/log/patches/patch-report-{{ ansible_date_time.date }}.txt"
      tags: [report, patch]

    - name: "PATCH | Check if reboot is required"
      stat:
        path: /var/run/reboot-required
      register: reboot_file
      when: ansible_os_family == "Debian"

    - name: "PATCH | Check if reboot needed (RHEL)"
      command: needs-restarting -r
      register: reboot_check
      failed_when: false
      changed_when: false
      when: ansible_os_family == "RedHat"

    - name: "PATCH | Reboot if required"
      reboot:
        msg: "Ansible Day 2 Ops: Rebooting for patching"
        pre_reboot_delay: 30
        post_reboot_delay: 60
        reboot_timeout: 600
      when: >
        reboot_required and
        ((reboot_file.stat.exists | default(false)) or 
         (reboot_check.rc | default(1) == 1))
      register: reboot_result
      tags: [reboot, patch]

  post_tasks:
    # ╔═══════════════════════════════════════════════════════════════╗
    # ║ POST-PATCHING PHASE: Validate and report                     ║
    # ╚═══════════════════════════════════════════════════════════════╝
    
    - name: "POST-PATCH | Run health checks"
      uri:
        url: "{{ item.url }}"
        method: GET
        status_code: "{{ item.expected_status }}"
        timeout: 30
      loop: "{{ health_checks }}"
      loop_control:
        label: "{{ item.name }}"
      register: health_check_results
      ignore_errors: true
      tags: [validate, post-patch]

    - name: "POST-PATCH | Determine patch success"
      set_fact:
        patch_success: "{{ health_check_results.results | selectattr('failed', 'equalto', false) | list | length == health_checks | length }}"

    - name: "POST-PATCH | Rollback if health checks failed"
      include_role:
        name: vm_restore_snapshot
      vars:
        snapshot_name: "pre-patch-{{ ansible_date_time.date }}"
      when: 
        - not patch_success
        - rollback_on_failure | bool

    - name: "POST-PATCH | Update ServiceNow with results"
      uri:
        url: "https://{{ snow_instance }}.service-now.com/api/now/table/change_request/{{ snow_change.json.result.sys_id }}"
        method: PATCH
        user: "{{ snow_username }}"
        password: "{{ snow_password }}"
        force_basic_auth: true
        body_format: json
        body:
          state: "{{ 'closed' if patch_success else 'cancelled' }}"
          close_code: "{{ 'successful' if patch_success else 'unsuccessful' }}"
          close_notes: |
            Automated patching completed
            Host: {{ inventory_hostname }}
            Packages Updated: {{ patch_result.results | default([]) | length }}
            Reboot: {{ reboot_result.rebooted | default(false) }}
            Health Checks: {{ 'PASSED' if patch_success else 'FAILED' }}
      delegate_to: localhost
      when: servicenow_enabled | bool
      tags: [servicenow, post-patch]

    - name: "POST-PATCH | Send notification"
      mail:
        host: "{{ smtp_server }}"
        to: "{{ notification_email }}"
        subject: "Patching {{ 'Completed' if patch_success else 'FAILED' }} - {{ inventory_hostname }}"
        body: |
          Day 2 Operations Patching Report
          ================================
          
          Host: {{ inventory_hostname }}
          Date: {{ ansible_date_time.iso8601 }}
          Change Request: {{ change_number | default('N/A') }}
          
          Status: {{ 'SUCCESS' if patch_success else 'FAILED - ROLLBACK EXECUTED' }}
          
          Packages Updated: {{ patch_result.results | default([]) | length }}
          Reboot Required: {{ reboot_result.rebooted | default(false) }}
      delegate_to: localhost
      ignore_errors: true
      tags: [notify, post-patch]
```

### Interview Talking Points for Patching
```
┌─────────────────────────────────────────────────────────────────────────────────┐
│ 💬 TALKING POINTS: Automated Patching                                           │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│ Q: "Tell me about your experience with Day 2 Operations automation."            │
│                                                                                 │
│ A: "I developed an automated patching system using Ansible that handles         │
│    our entire fleet of 200+ Linux servers. Key features I implemented:          │
│                                                                                 │
│    1. STAGED ROLLOUT - Patches go Dev → Test → Staging → Prod over 4 days      │
│       using 'serial' and 'max_fail_percentage' to limit blast radius            │
│                                                                                 │
│    2. AUTO-SNAPSHOTS - Pre-patch snapshots for instant rollback                │
│                                                                                 │
│    3. HEALTH CHECKS - Post-patch validation hits critical endpoints             │
│       If checks fail, automatic rollback triggers                               │
│                                                                                 │
│    4. SERVICENOW INTEGRATION - Change requests auto-created and closed          │
│       Full audit trail for compliance                                           │
│                                                                                 │
│    RESULTS: Reduced patching time from 3 days manual to 4 hours automated       │
│             Zero unplanned outages from patching in 18 months"                  │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2.2 Automated Backup System

### Complete Backup Playbook

```yaml
# playbooks/day2_operations/backup/automated_backup.yml
---
- name: "Day 2 Operations - Automated Backup System"
  hosts: "{{ target_hosts | default('all') }}"
  become: true
  gather_facts: true
  
  vars:
    backup_base_dir: "/backup"
    retention_days: 30
    backup_method: "incremental"  # full, incremental, differential
    
    # What to backup
    backup_targets:
      - "/etc"
      - "/var/www"
      - "/opt/applications"
      - "/home"
    
    # Exclusions
    backup_exclusions:
      - "*.log"
      - "*.tmp"
      - "cache/*"
      - "*.swap"
    
    # Remote storage
    remote_storage:
      type: "azure_blob"  # azure_blob, s3, nfs
      container: "server-backups"
      connection_string: "{{ vault_azure_connection_string }}"
    
    # Notification settings
    notify_on_failure: true
    notify_email: "ops-team@company.com"

  tasks:
    # ╔═══════════════════════════════════════════════════════════════╗
    # ║ PHASE 1: Pre-backup preparation                               ║
    # ╚═══════════════════════════════════════════════════════════════╝
    
    - name: "BACKUP | Create backup directory structure"
      file:
        path: "{{ backup_base_dir }}/{{ item }}"
        state: directory
        mode: '0750'
        owner: root
        group: root
      loop:
        - "daily"
        - "weekly"
        - "monthly"
        - "logs"

    - name: "BACKUP | Stop application services for consistent backup"
      service:
        name: "{{ item }}"
        state: stopped
      loop: "{{ services_to_stop | default([]) }}"
      when: consistent_backup | default(false)
      register: stopped_services

    - name: "BACKUP | Create database dump (if applicable)"
      include_tasks: backup_database.yml
      when: "'db_servers' in group_names"

    # ╔═══════════════════════════════════════════════════════════════╗
    # ║ PHASE 2: Execute backup                                       ║
    # ╚═══════════════════════════════════════════════════════════════╝
    
    - name: "BACKUP | Calculate backup filename"
      set_fact:
        backup_filename: "{{ inventory_hostname }}-{{ ansible_date_time.date }}-{{ backup_method }}.tar.gz"
        backup_path: "{{ backup_base_dir }}/daily/{{ inventory_hostname }}-{{ ansible_date_time.date }}-{{ backup_method }}.tar.gz"

    - name: "BACKUP | Create incremental backup marker"
      copy:
        content: "{{ ansible_date_time.iso8601 }}"
        dest: "{{ backup_base_dir }}/.last_backup_{{ backup_method }}"
      when: backup_method == "incremental"

    - name: "BACKUP | Execute backup with tar"
      shell: |
        tar --listed-incremental={{ backup_base_dir }}/.snapshot-{{ backup_method }} \
            {{ backup_exclusions | map('regex_replace', '^', '--exclude=') | join(' ') }} \
            -czf {{ backup_path }} \
            {{ backup_targets | join(' ') }} \
            2>&1 | tee {{ backup_base_dir }}/logs/backup-{{ ansible_date_time.date }}.log
      args:
        creates: "{{ backup_path }}"
      register: backup_result

    - name: "BACKUP | Calculate backup size and checksum"
      stat:
        path: "{{ backup_path }}"
        checksum_algorithm: sha256
      register: backup_stats

    - name: "BACKUP | Record backup metadata"
      copy:
        content: |
          {
            "hostname": "{{ inventory_hostname }}",
            "timestamp": "{{ ansible_date_time.iso8601 }}",
            "backup_type": "{{ backup_method }}",
            "filename": "{{ backup_filename }}",
            "size_bytes": {{ backup_stats.stat.size }},
            "size_human": "{{ (backup_stats.stat.size / 1024 / 1024) | round(2) }} MB",
            "checksum_sha256": "{{ backup_stats.stat.checksum }}",
            "targets": {{ backup_targets | to_json }},
            "exclusions": {{ backup_exclusions | to_json }}
          }
        dest: "{{ backup_path }}.metadata.json"

    # ╔═══════════════════════════════════════════════════════════════╗
    # ║ PHASE 3: Upload to remote storage                             ║
    # ╚═══════════════════════════════════════════════════════════════╝
    
    - name: "BACKUP | Upload to Azure Blob Storage"
      # Using az CLI (can also use azure.azcollection modules)
      shell: |
        az storage blob upload \
          --account-name {{ azure_storage_account }} \
          --container-name {{ remote_storage.container }} \
          --file {{ backup_path }} \
          --name {{ ansible_date_time.year }}/{{ ansible_date_time.month }}/{{ backup_filename }} \
          --auth-mode login
      environment:
        AZURE_STORAGE_CONNECTION_STRING: "{{ remote_storage.connection_string }}"
      when: remote_storage.type == "azure_blob"
      register: upload_result

    - name: "BACKUP | Upload to AWS S3"
      amazon.aws.s3_object:
        bucket: "{{ remote_storage.bucket }}"
        object: "{{ ansible_date_time.year }}/{{ ansible_date_time.month }}/{{ backup_filename }}"
        src: "{{ backup_path }}"
        mode: put
      when: remote_storage.type == "s3"

    # ╔═══════════════════════════════════════════════════════════════╗
    # ║ PHASE 4: Cleanup and verification                             ║
    # ╚═══════════════════════════════════════════════════════════════╝
    
    - name: "BACKUP | Restart stopped services"
      service:
        name: "{{ item.item }}"
        state: started
      loop: "{{ stopped_services.results | default([]) }}"
      when: 
        - consistent_backup | default(false)
        - stopped_services is defined

    - name: "BACKUP | Clean up old local backups"
      find:
        paths: "{{ backup_base_dir }}/daily"
        age: "{{ retention_days }}d"
        file_type: file
      register: old_backups

    - name: "BACKUP | Remove old backups"
      file:
        path: "{{ item.path }}"
        state: absent
      loop: "{{ old_backups.files }}"
      loop_control:
        label: "{{ item.path | basename }}"

    - name: "BACKUP | Update ServiceNow with backup status"
      uri:
        url: "https://{{ snow_instance }}.service-now.com/api/now/table/x_backup_log"
        method: POST
        user: "{{ snow_username }}"
        password: "{{ snow_password }}"
        force_basic_auth: true
        body_format: json
        body:
          server: "{{ inventory_hostname }}"
          backup_date: "{{ ansible_date_time.iso8601 }}"
          backup_size: "{{ backup_stats.stat.size }}"
          backup_type: "{{ backup_method }}"
          remote_path: "{{ remote_storage.container }}/{{ ansible_date_time.year }}/{{ ansible_date_time.month }}/{{ backup_filename }}"
          status: "success"
          checksum: "{{ backup_stats.stat.checksum }}"
      delegate_to: localhost
      when: servicenow_enabled | default(false)

    - name: "BACKUP | Generate backup report"
      template:
        src: backup_report.html.j2
        dest: "{{ backup_base_dir }}/logs/backup-report-{{ ansible_date_time.date }}.html"
      vars:
        report_data:
          host: "{{ inventory_hostname }}"
          timestamp: "{{ ansible_date_time.iso8601 }}"
          size: "{{ (backup_stats.stat.size / 1024 / 1024) | round(2) }} MB"
          checksum: "{{ backup_stats.stat.checksum }}"
          status: "SUCCESS"
```

---

## 2.3 Auto-Scaling Playbook

```yaml
# playbooks/day2_operations/scaling/auto_scale.yml
---
- name: "Day 2 Operations - Auto-Scaling Azure VMs"
  hosts: localhost
  gather_facts: false
  
  vars:
    resource_group: "production-rg"
    vmss_name: "webapp-vmss"
    
    # Scaling thresholds
    scale_out_cpu_threshold: 75  # Add VMs when CPU > 75%
    scale_in_cpu_threshold: 25   # Remove VMs when CPU < 25%
    min_instances: 2
    max_instances: 10
    cooldown_minutes: 5

  tasks:
    - name: "SCALE | Get current VMSS metrics from Azure Monitor"
      azure.azcollection.azure_rm_resource_info:
        resource_group: "{{ resource_group }}"
        provider: monitor
        resource_type: metrics
        resource_name: "{{ vmss_name }}"
        api_version: "2021-05-01"
        subresource:
          - type: default
            name: "metricDefinitions"
      register: metrics_info

    - name: "SCALE | Get current CPU percentage"
      # In practice, use azure_rm_virtualmachinescaleset_info or Azure Monitor API
      # This is simplified for demonstration
      set_fact:
        current_cpu: "{{ lookup('env', 'CURRENT_CPU_PERCENT') | default(50) | int }}"
        current_instance_count: "{{ lookup('env', 'CURRENT_INSTANCES') | default(2) | int }}"

    - name: "SCALE | Determine scaling action"
      set_fact:
        scale_action: >-
          {% if current_cpu | int > scale_out_cpu_threshold and current_instance_count | int < max_instances %}
          scale_out
          {% elif current_cpu | int < scale_in_cpu_threshold and current_instance_count | int > min_instances %}
          scale_in
          {% else %}
          none
          {% endif %}

    - name: "SCALE | Calculate new instance count"
      set_fact:
        new_instance_count: >-
          {% if scale_action == 'scale_out' %}
          {{ [current_instance_count | int + 2, max_instances] | min }}
          {% elif scale_action == 'scale_in' %}
          {{ [current_instance_count | int - 1, min_instances] | max }}
          {% else %}
          {{ current_instance_count }}
          {% endif %}

    - name: "SCALE | Execute scale-out on Azure VMSS"
      azure.azcollection.azure_rm_virtualmachinescaleset:
        resource_group: "{{ resource_group }}"
        name: "{{ vmss_name }}"
        capacity: "{{ new_instance_count }}"
      when: scale_action != "none"
      register: scale_result

    - name: "SCALE | Wait for new instances to be ready"
      azure.azcollection.azure_rm_virtualmachinescaleset_info:
        resource_group: "{{ resource_group }}"
        name: "{{ vmss_name }}"
      register: vmss_status
      until: vmss_status.vmss[0].provisioning_state == "Succeeded"
      retries: 20
      delay: 30
      when: scale_action == "scale_out"

    - name: "SCALE | Log scaling event to ServiceNow"
      uri:
        url: "https://{{ snow_instance }}.service-now.com/api/now/table/incident"
        method: POST
        user: "{{ snow_username }}"
        password: "{{ snow_password }}"
        force_basic_auth: true
        body_format: json
        body:
          short_description: "Auto-scaling event: {{ scale_action }}"
          description: |
            Auto-scaling triggered by Ansible Day 2 Operations
            
            VMSS: {{ vmss_name }}
            Previous Instance Count: {{ current_instance_count }}
            New Instance Count: {{ new_instance_count }}
            Trigger: CPU at {{ current_cpu }}%
            Action: {{ scale_action }}
          category: "Cloud"
          priority: 4
          assignment_group: "Cloud Operations"
      delegate_to: localhost
      when: scale_action != "none"

    - name: "SCALE | Send alert to New Relic"
      uri:
        url: "https://insights-collector.newrelic.com/v1/accounts/{{ newrelic_account_id }}/events"
        method: POST
        headers:
          X-Insert-Key: "{{ newrelic_insights_key }}"
          Content-Type: "application/json"
        body_format: json
        body:
          eventType: "AnsibleScalingEvent"
          vmss: "{{ vmss_name }}"
          action: "{{ scale_action }}"
          previous_count: "{{ current_instance_count }}"
          new_count: "{{ new_instance_count }}"
          cpu_trigger: "{{ current_cpu }}"
          timestamp: "{{ ansible_date_time.epoch }}"
      when: scale_action != "none"
```

---

# Part 3: Self-Healing Automation (Critical for This JD!)

## What is Self-Healing?

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                          SELF-HEALING AUTOMATION                                 │
├──────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  Traditional Operations:                                                         │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐        │
│  │ Alert   │───▶│ Ticket  │───▶│  Human  │───▶│  Login  │───▶│  Fix    │        │
│  │ Fires   │    │ Created │    │ Reviews │    │  to     │    │  Issue  │        │
│  │         │    │         │    │ (10min) │    │ Server  │    │ (20min) │        │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘    └─────────┘        │
│                                                                                  │
│  Total Time: 30-60 minutes (if someone is awake!)                               │
│  ════════════════════════════════════════════════════════════════════════       │
│                                                                                  │
│                                                                                  │
│  Self-Healing Automation:                                                        │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐        │
│  │ Alert   │───▶│ Ansible │───▶│ Auto    │───▶│ Verify  │───▶│ Close   │        │
│  │ Fires   │    │ Webhook │    │Remediate│    │ Fixed   │    │ Ticket  │        │
│  │         │    │ Trigger │    │ (2min)  │    │         │    │         │        │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘    └─────────┘        │
│                                                                                  │
│  Total Time: 2-5 minutes (24/7 with no human needed!)                           │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

## Self-Healing Architecture with New Relic, Moogsoft, ServiceNow

```
                        ┌───────────────────────────────────────────────────────┐
                        │              SELF-HEALING ECOSYSTEM                    │
                        └───────────────────────────────────────────────────────┘

┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   NEW RELIC     │     │   MOOGSOFT      │     │   APPLICATION   │
│   APM/INFRA     │     │   AIOPS         │     │   LOGS          │
│   Monitoring    │     │   Correlation   │     │   (ELK/Splunk)  │
└────────┬────────┘     └────────┬────────┘     └────────┬────────┘
         │                       │                       │
         │ Webhook               │ Webhook               │ Webhook
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                 │
                                 ▼
                  ┌──────────────────────────────┐
                  │     ANSIBLE AUTOMATION       │
                  │     CONTROLLER (AAP/AWX)     │
                  │                              │
                  │  ┌────────────────────────┐  │
                  │  │   Webhook Receiver     │  │
                  │  │   (Job Template API)   │  │
                  │  └───────────┬────────────┘  │
                  │              │               │
                  │              ▼               │
                  │  ┌────────────────────────┐  │
                  │  │   Decision Engine      │  │
                  │  │   (Runbook Selection)  │  │
                  │  └───────────┬────────────┘  │
                  │              │               │
                  │              ▼               │
                  │  ┌────────────────────────┐  │
                  │  │   Remediation Playbook │  │
                  │  │   Execution            │  │
                  │  └───────────┬────────────┘  │
                  │              │               │
                  └──────────────┼───────────────┘
                                 │
                                 ▼
                  ┌──────────────────────────────┐
                  │       SERVICENOW             │
                  │                              │        ┌─────────────┐
                  │  • Incident auto-created    │───────▶│   AUDIT     │
                  │  • Work notes updated        │        │   TRAIL     │
                  │  • Auto-resolved if fixed    │        │   CREATED   │
                  │  • Escalate if unresolved    │        └─────────────┘
                  │                              │
                  └──────────────────────────────┘
```

## 3.1 New Relic Integration Module

### Understanding New Relic

**What is New Relic?**
- Application Performance Monitoring (APM) tool
- Monitors server health, application performance, database queries
- Sends "alerts" when thresholds are breached (CPU > 80%, response time > 2s)

### Custom Module: New Relic Alert Handler

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

DOCUMENTATION = r'''
---
module: newrelic_alert_handler
short_description: Listen for and process New Relic alerts
description:
    - Receives New Relic webhook alerts and triggers remediation
    - Supports infrastructure and APM alerts
    - Integrates with ServiceNow for ticket management
version_added: "1.0.0"
author: "Your Name (@yourhandle)"

options:
    alert_id:
        description: New Relic alert policy ID
        required: true
        type: str
    api_key:
        description: New Relic API key
        required: true
        type: str
        no_log: true
    account_id:
        description: New Relic account ID
        required: true
        type: str
    action:
        description: Action to perform
        type: str
        choices: ['acknowledge', 'close', 'get_details', 'list_violations']
        default: 'get_details'
    
requirements:
    - requests
'''

EXAMPLES = r'''
# Get alert details
- name: Get New Relic alert details
  newrelic_alert_handler:
    api_key: "{{ newrelic_api_key }}"
    account_id: "12345"
    alert_id: "abc123"
    action: get_details
  register: alert_info

# Acknowledge alert after remediation
- name: Acknowledge alert
  newrelic_alert_handler:
    api_key: "{{ newrelic_api_key }}"
    account_id: "12345"
    alert_id: "abc123"
    action: acknowledge
'''

RETURN = r'''
alert:
    description: Alert details
    type: dict
    returned: always
    sample:
        id: "abc123"
        condition_name: "High CPU"
        entity_name: "webserver01"
        severity: "critical"
        started_at: "2024-01-15T10:30:00Z"
        violation_url: "https://alerts.newrelic.com/..."
'''

from ansible.module_utils.basic import AnsibleModule
import json

try:
    import requests
    HAS_REQUESTS = True
except ImportError:
    HAS_REQUESTS = False


class NewRelicAlertClient:
    """Client for New Relic Alert API operations."""
    
    def __init__(self, api_key, account_id):
        self.api_key = api_key
        self.account_id = account_id
        self.base_url = "https://api.newrelic.com/v2"
        self.graphql_url = "https://api.newrelic.com/graphql"
        self.headers = {
            "Api-Key": api_key,
            "Content-Type": "application/json"
        }
    
    def get_alert_details(self, alert_id):
        """Get detailed information about a specific alert."""
        # Using NerdGraph (New Relic's GraphQL API)
        query = """
        {
            actor {
                account(id: %s) {
                    alerts {
                        violation(id: "%s") {
                            id
                            alertConditionName
                            policyName
                            openedAt
                            closedAt
                            entity {
                                name
                                type
                            }
                            label
                            priority
                        }
                    }
                }
            }
        }
        """ % (self.account_id, alert_id)
        
        response = requests.post(
            self.graphql_url,
            headers=self.headers,
            json={"query": query}
        )
        
        if response.status_code == 200:
            return response.json()
        else:
            raise Exception(f"API error: {response.status_code} - {response.text}")
    
    def list_open_violations(self):
        """List all current open alert violations."""
        query = """
        {
            actor {
                account(id: %s) {
                    alerts {
                        openViolations {
                            violationId
                            alertPolicyId
                            alertPolicyName
                            entity {
                                name
                                type
                            }
                            openedAt
                            priority
                        }
                    }
                }
            }
        }
        """ % self.account_id
        
        response = requests.post(
            self.graphql_url, 
            headers=self.headers,
            json={"query": query}
        )
        
        if response.status_code == 200:
            return response.json()
        else:
            raise Exception(f"API error: {response.status_code}")
    
    def acknowledge_alert(self, alert_id):
        """Acknowledge an alert (marks it as being worked on)."""
        # Note: Actual acknowledgment depends on your New Relic setup
        # This might post to a workflow or incident management
        mutation = """
        mutation {
            aiWorkflowsAcknowledgeIncident(
                accountId: %s
                incidentId: "%s"
            ) {
                incident {
                    id
                    state
                }
            }
        }
        """ % (self.account_id, alert_id)
        
        response = requests.post(
            self.graphql_url,
            headers=self.headers,
            json={"query": mutation}
        )
        
        return response.json()


def run_module():
    """Main module execution."""
    module_args = dict(
        api_key=dict(type='str', required=True, no_log=True),
        account_id=dict(type='str', required=True),
        alert_id=dict(type='str', required=False),
        action=dict(
            type='str',
            default='get_details',
            choices=['acknowledge', 'close', 'get_details', 'list_violations']
        )
    )
    
    result = dict(
        changed=False,
        alert={},
        violations=[]
    )
    
    module = AnsibleModule(
        argument_spec=module_args,
        required_if=[
            ('action', 'get_details', ['alert_id']),
            ('action', 'acknowledge', ['alert_id']),
        ],
        supports_check_mode=True
    )
    
    if not HAS_REQUESTS:
        module.fail_json(msg="The 'requests' library is required")
    
    client = NewRelicAlertClient(
        api_key=module.params['api_key'],
        account_id=module.params['account_id']
    )
    
    try:
        action = module.params['action']
        
        if action == 'get_details':
            alert_data = client.get_alert_details(module.params['alert_id'])
            result['alert'] = alert_data
            
        elif action == 'list_violations':
            violations = client.list_open_violations()
            result['violations'] = violations
            
        elif action == 'acknowledge':
            if not module.check_mode:
                ack_result = client.acknowledge_alert(module.params['alert_id'])
                result['changed'] = True
                result['alert'] = ack_result
        
        module.exit_json(**result)
        
    except Exception as e:
        module.fail_json(msg=str(e), **result)


def main():
    run_module()


if __name__ == '__main__':
    main()
```

## 3.2 Moogsoft Integration Module

### Understanding Moogsoft

**What is Moogsoft?**
- AIOps platform (Artificial Intelligence for IT Operations)
- Collects alerts from multiple sources (New Relic, Datadog, CloudWatch, etc.)
- Uses AI to correlate related alerts into "Situations"
- Reduces alert noise by grouping related issues

### Custom Module: Moogsoft Situation Handler

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

DOCUMENTATION = r'''
---
module: moogsoft_situation
short_description: Manage Moogsoft situations and alerts
description:
    - Interact with Moogsoft AIOps platform
    - Retrieve situation details for remediation
    - Update situation status after remediation
    - Link situations to ServiceNow incidents
version_added: "1.0.0"
author: "Your Name (@yourhandle)"

options:
    api_url:
        description: Moogsoft API URL
        required: true
        type: str
    api_key:
        description: Moogsoft API key
        required: true
        type: str
        no_log: true
    situation_id:
        description: Moogsoft situation ID
        type: str
    action:
        description: Action to perform
        type: str
        choices: ['get', 'list_open', 'add_comment', 'resolve', 'get_alerts']
        default: 'get'
    comment:
        description: Comment to add (for add_comment action)
        type: str
'''

EXAMPLES = r'''
# Get open situations
- name: Get all open Moogsoft situations
  moogsoft_situation:
    api_url: "https://moogsoft.company.com/api/v1"
    api_key: "{{ moogsoft_api_key }}"
    action: list_open
  register: situations

# Get correlated alerts for a situation
- name: Get alerts in situation
  moogsoft_situation:
    api_url: "https://moogsoft.company.com/api/v1"
    api_key: "{{ moogsoft_api_key }}"
    situation_id: "{{ situation_id }}"
    action: get_alerts
  register: correlated_alerts

# Resolve situation after remediation
- name: Resolve situation
  moogsoft_situation:
    api_url: "https://moogsoft.company.com/api/v1"
    api_key: "{{ moogsoft_api_key }}"
    situation_id: "{{ situation_id }}"
    action: resolve
    comment: "Auto-resolved by Ansible playbook"
'''

RETURN = r'''
situation:
    description: Situation details
    type: dict
    returned: when action is get
    sample:
        id: "SIT-12345"
        description: "Multiple database connection failures"
        severity: 5
        services: ["payment-api", "inventory-service"]
        status: "Opened"
        alert_count: 15
alerts:
    description: Correlated alerts in the situation
    type: list
    returned: when action is get_alerts
'''

from ansible.module_utils.basic import AnsibleModule

try:
    import requests
    HAS_REQUESTS = True
except ImportError:
    HAS_REQUESTS = False


class MoogsoftClient:
    """Client for Moogsoft API operations."""
    
    def __init__(self, api_url, api_key):
        self.api_url = api_url.rstrip('/')
        self.headers = {
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json",
            "Accept": "application/json"
        }
    
    def get_situation(self, situation_id):
        """Get situation details."""
        response = requests.get(
            f"{self.api_url}/situations/{situation_id}",
            headers=self.headers
        )
        response.raise_for_status()
        return response.json()
    
    def list_open_situations(self):
        """List all open situations."""
        params = {
            "status": "Opened",
            "limit": 100
        }
        response = requests.get(
            f"{self.api_url}/situations",
            headers=self.headers,
            params=params
        )
        response.raise_for_status()
        return response.json()
    
    def get_alerts_in_situation(self, situation_id):
        """Get all alerts correlated into this situation."""
        response = requests.get(
            f"{self.api_url}/situations/{situation_id}/alerts",
            headers=self.headers
        )
        response.raise_for_status()
        return response.json()
    
    def add_comment(self, situation_id, comment):
        """Add a comment/work note to situation."""
        payload = {
            "comment": comment,
            "type": "automation"
        }
        response = requests.post(
            f"{self.api_url}/situations/{situation_id}/comment",
            headers=self.headers,
            json=payload
        )
        response.raise_for_status()
        return response.json()
    
    def resolve_situation(self, situation_id, resolution_comment):
        """Resolve/close a situation."""
        payload = {
            "status": "Resolved",
            "resolution": resolution_comment
        }
        response = requests.patch(
            f"{self.api_url}/situations/{situation_id}",
            headers=self.headers,
            json=payload
        )
        response.raise_for_status()
        return response.json()
    
    def extract_affected_hosts(self, situation):
        """Extract host/server names from situation alerts."""
        hosts = set()
        alerts = situation.get('alerts', [])
        for alert in alerts:
            # Extract hostname from various possible fields
            host = alert.get('source') or alert.get('node') or alert.get('manager')
            if host:
                hosts.add(host)
        return list(hosts)


def run_module():
    """Main module execution."""
    module_args = dict(
        api_url=dict(type='str', required=True),
        api_key=dict(type='str', required=True, no_log=True),
        situation_id=dict(type='str', required=False),
        action=dict(
            type='str',
            default='get',
            choices=['get', 'list_open', 'add_comment', 'resolve', 'get_alerts']
        ),
        comment=dict(type='str', required=False)
    )
    
    result = dict(
        changed=False,
        situation={},
        situations=[],
        alerts=[],
        affected_hosts=[]
    )
    
    module = AnsibleModule(
        argument_spec=module_args,
        required_if=[
            ('action', 'get', ['situation_id']),
            ('action', 'get_alerts', ['situation_id']),
            ('action', 'add_comment', ['situation_id', 'comment']),
            ('action', 'resolve', ['situation_id']),
        ],
        supports_check_mode=True
    )
    
    if not HAS_REQUESTS:
        module.fail_json(msg="The 'requests' library is required")
    
    client = MoogsoftClient(
        api_url=module.params['api_url'],
        api_key=module.params['api_key']
    )
    
    try:
        action = module.params['action']
        
        if action == 'get':
            situation = client.get_situation(module.params['situation_id'])
            result['situation'] = situation
            result['affected_hosts'] = client.extract_affected_hosts(situation)
            
        elif action == 'list_open':
            situations = client.list_open_situations()
            result['situations'] = situations
            
        elif action == 'get_alerts':
            alerts = client.get_alerts_in_situation(module.params['situation_id'])
            result['alerts'] = alerts
            
        elif action == 'add_comment':
            if not module.check_mode:
                client.add_comment(
                    module.params['situation_id'],
                    module.params['comment']
                )
                result['changed'] = True
                
        elif action == 'resolve':
            if not module.check_mode:
                comment = module.params.get('comment', 'Resolved by Ansible automation')
                client.resolve_situation(module.params['situation_id'], comment)
                result['changed'] = True
        
        module.exit_json(**result)
        
    except Exception as e:
        module.fail_json(msg=str(e), **result)


def main():
    run_module()


if __name__ == '__main__':
    main()
```

## 3.3 Complete Self-Healing Playbook

This playbook ties together New Relic, Moogsoft, and ServiceNow:

```yaml
# playbooks/self_healing/auto_remediation.yml
---
# ╔═══════════════════════════════════════════════════════════════════════════════╗
# ║ SELF-HEALING AUTOMATION PLAYBOOK                                              ║
# ║                                                                               ║
# ║ Triggered by: Webhook from New Relic/Moogsoft/ServiceNow                      ║
# ║ Flow: Receive Alert → Diagnose → Remediate → Verify → Document                ║
# ╚═══════════════════════════════════════════════════════════════════════════════╝

- name: "Self-Healing - Process and Remediate Alert"
  hosts: localhost
  gather_facts: false
  
  vars:
    # Passed from webhook
    alert_source: "{{ lookup('env', 'ALERT_SOURCE') }}"  # newrelic, moogsoft, snow
    alert_id: "{{ lookup('env', 'ALERT_ID') }}"
    alert_type: "{{ lookup('env', 'ALERT_TYPE') }}"      # cpu_high, disk_full, service_down, etc.
    affected_host: "{{ lookup('env', 'AFFECTED_HOST') }}"
    
    # Remediation config
    max_remediation_attempts: 3
    escalate_after_failures: true
    
    # Runbook mapping - which playbook handles which alert type
    remediation_runbooks:
      cpu_high: "remediate_high_cpu.yml"
      disk_full: "remediate_disk_full.yml"
      memory_low: "remediate_memory.yml"
      service_down: "remediate_service.yml"
      certificate_expiring: "remediate_certificate.yml"
      connection_pool_exhausted: "remediate_db_connections.yml"

  tasks:
    # ╔═══════════════════════════════════════════════════════════════╗
    # ║ PHASE 1: TRIAGE - Gather alert context                        ║
    # ╚═══════════════════════════════════════════════════════════════╝
    
    - name: "TRIAGE | Log incoming alert"
      debug:
        msg: |
          ═══════════════════════════════════════════════════════════
          SELF-HEALING AUTOMATION TRIGGERED
          ═══════════════════════════════════════════════════════════
          Source: {{ alert_source }}
          Alert ID: {{ alert_id }}
          Type: {{ alert_type }}
          Affected Host: {{ affected_host }}
          Timestamp: {{ ansible_date_time.iso8601 | default('N/A') }}
          ═══════════════════════════════════════════════════════════

    - name: "TRIAGE | Get alert details from New Relic"
      newrelic_alert_handler:
        api_key: "{{ newrelic_api_key }}"
        account_id: "{{ newrelic_account_id }}"
        alert_id: "{{ alert_id }}"
        action: get_details
      register: newrelic_alert
      when: alert_source == "newrelic"

    - name: "TRIAGE | Get situation details from Moogsoft"
      moogsoft_situation:
        api_url: "{{ moogsoft_api_url }}"
        api_key: "{{ moogsoft_api_key }}"
        situation_id: "{{ alert_id }}"
        action: get
      register: moogsoft_situation
      when: alert_source == "moogsoft"

    # ╔═══════════════════════════════════════════════════════════════╗
    # ║ PHASE 2: CREATE SERVICENOW INCIDENT                           ║
    # ╚═══════════════════════════════════════════════════════════════╝
    
    - name: "SNOW | Create incident for tracking"
      servicenow.itsm.incident:
        instance:
          host: "{{ snow_instance }}.service-now.com"
          username: "{{ snow_username }}"
          password: "{{ snow_password }}"
        state: present
        data:
          short_description: "Auto-Remediation: {{ alert_type }} on {{ affected_host }}"
          description: |
            Automated incident created by self-healing system
            
            Alert Source: {{ alert_source }}
            Alert ID: {{ alert_id }}
            Alert Type: {{ alert_type }}
            Affected Host: {{ affected_host }}
            
            Remediation will be attempted automatically.
          caller_id: "ansible_automation"
          category: "Software"
          subcategory: "Operating System"
          impact: 2
          urgency: 2
          assignment_group: "Linux Admins"
          cmdb_ci: "{{ affected_host }}"
      register: snow_incident
      delegate_to: localhost

    - name: "SNOW | Store incident number"
      set_fact:
        incident_number: "{{ snow_incident.record.number }}"
        incident_sys_id: "{{ snow_incident.record.sys_id }}"

    # ╔═══════════════════════════════════════════════════════════════╗
    # ║ PHASE 3: EXECUTE REMEDIATION                                  ║
    # ╚═══════════════════════════════════════════════════════════════╝
    
    - name: "REMEDIATE | Check if runbook exists for this alert type"
      set_fact:
        runbook_path: "{{ remediation_runbooks[alert_type] | default('') }}"
    
    - name: "REMEDIATE | Fail if no runbook found"
      fail:
        msg: "No remediation runbook found for alert type: {{ alert_type }}"
      when: runbook_path == ""

    - name: "REMEDIATE | Execute remediation playbook"
      include_tasks: "{{ runbook_path }}"
      vars:
        target_host: "{{ affected_host }}"
        incident_id: "{{ incident_number }}"
      register: remediation_result

    # ╔═══════════════════════════════════════════════════════════════╗
    # ║ PHASE 4: VERIFY REMEDIATION SUCCESS                           ║
    # ╚═══════════════════════════════════════════════════════════════╝
    
    - name: "VERIFY | Wait for metrics to stabilize"
      pause:
        seconds: 60
      when: not ansible_check_mode

    - name: "VERIFY | Check if alert has cleared in New Relic"
      newrelic_alert_handler:
        api_key: "{{ newrelic_api_key }}"
        account_id: "{{ newrelic_account_id }}"
        alert_id: "{{ alert_id }}"
        action: get_details
      register: alert_check
      when: alert_source == "newrelic"
      ignore_errors: true

    - name: "VERIFY | Determine if remediation was successful"
      set_fact:
        remediation_successful: "{{ alert_check.alert.status | default('open') == 'closed' }}"
      when: alert_source == "newrelic"

    # ╔═══════════════════════════════════════════════════════════════╗
    # ║ PHASE 5: CLOSE OR ESCALATE                                    ║
    # ╚═══════════════════════════════════════════════════════════════╝
    
    - name: "CLOSE | Update ServiceNow - Success"
      servicenow.itsm.incident:
        instance:
          host: "{{ snow_instance }}.service-now.com"
          username: "{{ snow_username }}"
          password: "{{ snow_password }}"
        state: present
        number: "{{ incident_number }}"
        data:
          state: resolved
          close_code: "Solved (Permanently)"
          close_notes: |
            Resolved automatically by Ansible self-healing.
            
            Remediation Applied: {{ runbook_path }}
            Execution Time: {{ ansible_date_time.iso8601 }}
            
            No human intervention required.
      when: remediation_successful | default(false)

    - name: "ESCALATE | Update ServiceNow - Requires Human"
      servicenow.itsm.incident:
        instance:
          host: "{{ snow_instance }}.service-now.com"
          username: "{{ snow_username }}"
          password: "{{ snow_password }}"
        state: present
        number: "{{ incident_number }}"
        data:
          state: in_progress
          urgency: 1
          impact: 1
          work_notes: |
            Automated remediation UNSUCCESSFUL
            
            Remediation Attempted: {{ runbook_path }}
            Manual intervention required.
            
            Please review and escalate as needed.
      when: not (remediation_successful | default(false))

    - name: "ESCALATE | Notify on-call via PagerDuty"
      uri:
        url: "https://events.pagerduty.com/v2/enqueue"
        method: POST
        body_format: json
        body:
          routing_key: "{{ pagerduty_routing_key }}"
          event_action: "trigger"
          payload:
            summary: "Ansible auto-remediation failed: {{ alert_type }} on {{ affected_host }}"
            severity: "critical"
            source: "ansible-selfhealing"
            custom_details:
              alert_id: "{{ alert_id }}"
              incident: "{{ incident_number }}"
              remediation_attempted: "{{ runbook_path }}"
      when: not (remediation_successful | default(false)) and escalate_after_failures

    - name: "NOTIFY | Send metrics to New Relic"
      uri:
        url: "https://insights-collector.newrelic.com/v1/accounts/{{ newrelic_account_id }}/events"
        method: POST
        headers:
          X-Insert-Key: "{{ newrelic_insights_key }}"
        body_format: json
        body:
          - eventType: "AnsibleSelfHealing"
            alertType: "{{ alert_type }}"
            affectedHost: "{{ affected_host }}"
            remediationPlaybook: "{{ runbook_path }}"
            success: "{{ remediation_successful | default(false) }}"
            incidentNumber: "{{ incident_number }}"
            durationSeconds: "{{ ansible_date_time.epoch | int - start_time | default(0) | int }}"
```

## 3.4 Remediation Runbook Examples

### High CPU Remediation

```yaml
# playbooks/self_healing/remediate_high_cpu.yml
---
- name: "Remediate High CPU on {{ target_host }}"
  hosts: "{{ target_host }}"
  become: true
  gather_facts: true
  
  tasks:
    - name: "DIAGNOSE | Get top CPU consumers"
      shell: |
        ps aux --sort=-%cpu | head -20
      register: top_processes

    - name: "DIAGNOSE | Log top processes"
      debug:
        msg: |
          Top CPU consumers:
          {{ top_processes.stdout }}

    - name: "REMEDIATE | Kill known runaway processes"
      shell: |
        # Kill any zombie processes
        kill -9 $(ps aux | awk '{ if ($8 == "Z") print $2 }') 2>/dev/null || true
        
        # Kill known CPU hogs (customize this list)
        pkill -9 -f "known_runaway_script.sh" 2>/dev/null || true
      ignore_errors: true

    - name: "REMEDIATE | Clear system caches"
      shell: |
        sync
        echo 3 > /proc/sys/vm/drop_caches
      when: ansible_memfree_mb | int < 500

    - name: "REMEDIATE | Restart high-memory services if needed"
      service:
        name: "{{ item }}"
        state: restarted
      loop:
        - httpd
        - nginx
      when: "'webservers' in group_names"
      ignore_errors: true

    - name: "VERIFY | Check CPU after remediation"
      shell: |
        top -bn1 | grep "Cpu(s)" | awk '{print $2}'
      register: cpu_after

    - name: "REPORT | Set remediation result"
      set_fact:
        remediation_result:
          status: "{{ 'success' if (cpu_after.stdout | float < 70) else 'partial' }}"
          cpu_before: "{{ top_processes.stdout_lines[0] | default('unknown') }}"
          cpu_after: "{{ cpu_after.stdout }}"
```

### Disk Full Remediation

```yaml
# playbooks/self_healing/remediate_disk_full.yml
---
- name: "Remediate Disk Full on {{ target_host }}"
  hosts: "{{ target_host }}"
  become: true
  gather_facts: true
  
  vars:
    cleanup_threshold_mb: 5000  # Clean until 5GB free
    
  tasks:
    - name: "DIAGNOSE | Check current disk usage"
      shell: df -h / | tail -1 | awk '{print $5}' | sed 's/%//'
      register: disk_usage_before

    - name: "DIAGNOSE | Find large files"
      shell: |
        find /var/log -type f -size +100M -exec ls -lh {} \; 2>/dev/null | head -20
      register: large_files

    - name: "REMEDIATE | Rotate and compress old logs"
      shell: |
        logrotate -f /etc/logrotate.conf
      ignore_errors: true

    - name: "REMEDIATE | Clean old journal logs"
      shell: |
        journalctl --vacuum-time=7d
        journalctl --vacuum-size=500M
      when: ansible_service_mgr == "systemd"

    - name: "REMEDIATE | Clean package manager cache"
      shell: |
        yum clean all 2>/dev/null || apt-get clean 2>/dev/null || true

    - name: "REMEDIATE | Remove old temp files"
      shell: |
        find /tmp -type f -atime +7 -delete 2>/dev/null || true
        find /var/tmp -type f -atime +7 -delete 2>/dev/null || true

    - name: "REMEDIATE | Compress log files older than 1 day"
      shell: |
        find /var/log -type f -name "*.log" -mtime +1 ! -name "*.gz" -exec gzip {} \;
      ignore_errors: true

    - name: "VERIFY | Check disk usage after"
      shell: df -h / | tail -1 | awk '{print $5}' | sed 's/%//'
      register: disk_usage_after

    - name: "REPORT | Set remediation result"
      set_fact:
        remediation_result:
          status: "{{ 'success' if (disk_usage_after.stdout | int < 85) else 'needs_attention' }}"
          disk_before: "{{ disk_usage_before.stdout }}%"
          disk_after: "{{ disk_usage_after.stdout }}%"
          space_recovered: "{{ (disk_usage_before.stdout | int) - (disk_usage_after.stdout | int) }}%"
```

---

# Part 4: Azure VM Automation (Critical for This JD!)

## 4.1 Azure VM Lifecycle Management

### Inventory Plugin Configuration

```yaml
# inventory/azure_rm.yml
---
plugin: azure.azcollection.azure_rm
include_vm_resource_groups:
  - production-rg
  - staging-rg
  - dev-rg

auth_source: auto  # Uses environment vars or Azure CLI login

keyed_groups:
  # Group by tags
  - key: tags.environment
    prefix: env
  - key: tags.application
    prefix: app
  - key: tags.team
    prefix: team
    
  # Group by location
  - key: location
    prefix: region
    
  # Group by OS
  - key: os_profile.system
    prefix: os

hostvar_expressions:
  # Make VM info easily accessible
  ansible_host: public_ip_address | default(private_ip_address)
  vm_size: hardware_profile.vm_size
  os_disk_size: storage_profile.os_disk.disk_size_gb
  
conditional_groups:
  # Create groups based on conditions
  linux_vms: "'Linux' in os_profile.system"
  windows_vms: "'Windows' in os_profile.system"
  large_vms: "'Standard_D' in hardware_profile.vm_size"
```

### Azure VM Provisioning Playbook

```yaml
# playbooks/azure/provision_vm.yml
---
- name: "Azure VM Provisioning - Day 1 Operations"
  hosts: localhost
  gather_facts: false
  
  vars:
    azure_region: "eastus"
    resource_group: "production-rg"
    vm_size: "Standard_D2s_v3"
    admin_username: "azureadmin"
    
    # VM configuration
    vm_config:
      name: "webapp-{{ 100 | random }}"
      os_type: "linux"
      image:
        publisher: "RedHat"
        offer: "RHEL"
        sku: "8_4"
        version: "latest"
      tags:
        environment: "production"
        application: "webapp"
        team: "platform"
        managed_by: "ansible"

  tasks:
    - name: "NETWORK | Create virtual network"
      azure.azcollection.azure_rm_virtualnetwork:
        resource_group: "{{ resource_group }}"
        name: "{{ resource_group }}-vnet"
        address_prefixes: "10.0.0.0/16"
        state: present
      register: vnet

    - name: "NETWORK | Create subnet"
      azure.azcollection.azure_rm_subnet:
        resource_group: "{{ resource_group }}"
        virtual_network_name: "{{ resource_group }}-vnet"
        name: "default"
        address_prefix_cidr: "10.0.1.0/24"
        state: present
      register: subnet

    - name: "NETWORK | Create public IP"
      azure.azcollection.azure_rm_publicipaddress:
        resource_group: "{{ resource_group }}"
        name: "{{ vm_config.name }}-pip"
        allocation_method: static
        sku: standard
        state: present
      register: public_ip

    - name: "NETWORK | Create network security group"
      azure.azcollection.azure_rm_securitygroup:
        resource_group: "{{ resource_group }}"
        name: "{{ vm_config.name }}-nsg"
        rules:
          - name: SSH
            protocol: Tcp
            destination_port_range: 22
            access: Allow
            priority: 100
            direction: Inbound
            source_address_prefix: "{{ allowed_ssh_cidr | default('10.0.0.0/8') }}"
          - name: HTTP
            protocol: Tcp
            destination_port_range: 80
            access: Allow
            priority: 200
            direction: Inbound
          - name: HTTPS
            protocol: Tcp
            destination_port_range: 443
            access: Allow
            priority: 201
            direction: Inbound
      register: nsg

    - name: "NETWORK | Create NIC"
      azure.azcollection.azure_rm_networkinterface:
        resource_group: "{{ resource_group }}"
        name: "{{ vm_config.name }}-nic"
        virtual_network: "{{ resource_group }}-vnet"
        subnet: default
        public_ip_name: "{{ vm_config.name }}-pip"
        security_group: "{{ vm_config.name }}-nsg"
        state: present
      register: nic

    - name: "VM | Create virtual machine"
      azure.azcollection.azure_rm_virtualmachine:
        resource_group: "{{ resource_group }}"
        name: "{{ vm_config.name }}"
        vm_size: "{{ vm_size }}"
        admin_username: "{{ admin_username }}"
        ssh_password_enabled: false
        ssh_public_keys:
          - path: "/home/{{ admin_username }}/.ssh/authorized_keys"
            key_data: "{{ ssh_public_key }}"
        network_interfaces: "{{ vm_config.name }}-nic"
        image: "{{ vm_config.image }}"
        os_disk_caching: ReadWrite
        managed_disk_type: Premium_LRS
        tags: "{{ vm_config.tags }}"
        state: present
      register: vm_result

    - name: "POST | Wait for VM to be accessible"
      wait_for:
        host: "{{ public_ip.state.ip_address }}"
        port: 22
        timeout: 300

    - name: "POST | Add to dynamic inventory"
      add_host:
        name: "{{ public_ip.state.ip_address }}"
        groups:
          - new_vms
          - "{{ vm_config.tags.environment }}"
        ansible_user: "{{ admin_username }}"

    - name: "SNOW | Create CI in CMDB"
      uri:
        url: "https://{{ snow_instance }}.service-now.com/api/now/table/cmdb_ci_server"
        method: POST
        user: "{{ snow_username }}"
        password: "{{ snow_password }}"
        force_basic_auth: true
        body_format: json
        body:
          name: "{{ vm_config.name }}"
          ip_address: "{{ public_ip.state.ip_address }}"
          os: "{{ vm_config.image.offer }} {{ vm_config.image.sku }}"
          location: "{{ azure_region }}"
          environment: "{{ vm_config.tags.environment }}"
          managed_by: "Ansible"
          install_status: "Installed"
      delegate_to: localhost


# Day 1 Configuration (on the new VM)
- name: "Configure New Azure VM"
  hosts: new_vms
  become: true
  gather_facts: true
  
  roles:
    - common-hardening
    - newrelic-agent
    - log-shipping
    
  post_tasks:
    - name: "Notify New Relic of deployment"
      uri:
        url: "https://api.newrelic.com/v2/applications/{{ newrelic_app_id }}/deployments.json"
        method: POST
        headers:
          X-Api-Key: "{{ newrelic_api_key }}"
        body_format: json
        body:
          deployment:
            revision: "{{ vm_config.name }}"
            description: "New VM provisioned by Ansible"
      delegate_to: localhost
```

## 4.2 Azure PaaS Day 2 Operations

### App Service Management

```yaml
# playbooks/azure/paas_day2_operations.yml
---
- name: "Azure PaaS - Day 2 Operations"
  hosts: localhost
  gather_facts: false
  
  vars:
    resource_group: "production-rg"
    app_service_name: "mywebapp"
    
  tasks:
    # ╔═══════════════════════════════════════════════════════════════╗
    # ║ APP SERVICE SLOT MANAGEMENT                                   ║
    # ╚═══════════════════════════════════════════════════════════════╝
    
    - name: "SLOTS | Create staging slot"
      azure.azcollection.azure_rm_webapp_slot:
        resource_group: "{{ resource_group }}"
        webapp_name: "{{ app_service_name }}"
        name: staging
        configuration_source: "{{ app_service_name }}"
        state: present
      register: staging_slot

    - name: "SLOTS | Swap staging to production"
      azure.azcollection.azure_rm_webapp_swap:
        resource_group: "{{ resource_group }}"
        webapp_name: "{{ app_service_name }}"
        slot: staging
        action: swap
      when: ready_for_production | default(false)

    # ╔═══════════════════════════════════════════════════════════════╗
    # ║ AUTO-SCALING CONFIGURATION                                    ║
    # ╚═══════════════════════════════════════════════════════════════╝
    
    - name: "SCALE | Configure auto-scaling rules"
      azure.azcollection.azure_rm_autoscale:
        resource_group: "{{ resource_group }}"
        name: "{{ app_service_name }}-autoscale"
        target_resource_uri: "/subscriptions/{{ subscription_id }}/resourceGroups/{{ resource_group }}/providers/Microsoft.Web/serverfarms/{{ app_service_plan }}"
        enabled: true
        profiles:
          - name: "business-hours"
            count: 3
            min_count: 2
            max_count: 10
            recurrence:
              frequency: week
              schedule:
                days: ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday"]
                hours: [9]
                minutes: [0]
            rules:
              - metric_name: CpuPercentage
                operator: GreaterThan
                threshold: 75
                direction: Increase
                change_count: 2
                cooldown: 5
              - metric_name: CpuPercentage
                operator: LessThan
                threshold: 25
                direction: Decrease
                change_count: 1
                cooldown: 10

    # ╔═══════════════════════════════════════════════════════════════╗
    # ║ SSL CERTIFICATE MANAGEMENT                                    ║
    # ╚═══════════════════════════════════════════════════════════════╝
    
    - name: "SSL | Get certificate from Key Vault"
      azure.azcollection.azure_rm_keyvaultsecret_info:
        vault_uri: "https://{{ key_vault_name }}.vault.azure.net"
        name: "{{ ssl_cert_name }}"
      register: ssl_cert

    - name: "SSL | Upload certificate to App Service"
      azure.azcollection.azure_rm_webappcertificate:
        resource_group: "{{ resource_group }}"
        name: "{{ ssl_cert_name }}"
        key_vault_secret_id: "{{ ssl_cert.secrets[0].id }}"
        state: present

    - name: "SSL | Bind certificate to custom domain"
      azure.azcollection.azure_rm_webapp:
        resource_group: "{{ resource_group }}"
        name: "{{ app_service_name }}"
        https_only: true
        custom_domain_verification_id: "{{ domain_verification_id }}"
        host_name_ssl_states:
          - name: "www.example.com"
            ssl_state: SniEnabled
            thumbprint: "{{ ssl_cert.secrets[0].thumbprint }}"

    # ╔═══════════════════════════════════════════════════════════════╗
    # ║ APPLICATION INSIGHTS INTEGRATION                              ║
    # ╚═══════════════════════════════════════════════════════════════╝
    
    - name: "MONITOR | Configure Application Insights"
      azure.azcollection.azure_rm_webapp:
        resource_group: "{{ resource_group }}"
        name: "{{ app_service_name }}"
        app_settings:
          APPLICATIONINSIGHTS_CONNECTION_STRING: "{{ app_insights_connection_string }}"
          ApplicationInsightsAgent_EXTENSION_VERSION: "~3"
          APPINSIGHTS_INSTRUMENTATIONKEY: "{{ app_insights_instrumentation_key }}"
```

---

# Part 5: Interview Q&A Aligned to JD

## 5.1 Day 2 Operations Questions

### Q: "What is your experience with Day 2 Operations?"

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│ 💬 SAMPLE ANSWER                                                                │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│ "Day 2 Operations has been the core of my Ansible work for the past 3 years.   │
│  I've built comprehensive automation for:                                       │
│                                                                                 │
│  1. AUTOMATED PATCHING                                                          │
│     • Staged rollout across 200+ servers (dev → test → staging → prod)         │
│     • Pre-patch snapshots for rollback capability                              │
│     • Post-patch health checks with automatic rollback if checks fail          │
│     • ServiceNow integration for change management compliance                   │
│     • RESULT: 95% reduction in patching time, zero unplanned outages           │
│                                                                                 │
│  2. BACKUP AUTOMATION                                                           │
│     • Incremental backups with 30-day retention                                │
│     • Azure Blob Storage archival                                              │
│     • Automated backup verification and restore testing                        │
│     • RESULT: 100% backup success rate, RPO of 4 hours                         │
│                                                                                 │
│  3. COMPLIANCE & DRIFT DETECTION                                               │
│     • Weekly compliance scans against CIS benchmarks                           │
│     • Automatic remediation of drift from baseline                             │
│     • Audit reports generated for security team                                │
│     • RESULT: Improved audit scores from 72% to 98%                            │
│                                                                                 │
│  The key with Day 2 is idempotency - I design every playbook to be safely      │
│  re-runnable, using check mode extensively during development."                │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

### Q: "How do you handle integration with monitoring tools?"

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│ 💬 SAMPLE ANSWER                                                                │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│ "I've integrated Ansible with three major monitoring platforms:                 │
│                                                                                 │
│  NEW RELIC:                                                                     │
│  • Developed custom module using NerdGraph API (GraphQL)                        │
│  • Webhook receiver on AAP triggers remediation playbooks                       │
│  • Deployment markers sent after each successful deploy                        │
│  • Custom dashboards pulling Ansible execution metrics                         │
│                                                                                 │
│  MOOGSOFT:                                                                      │
│  • Built module to fetch situation details and affected hosts                  │
│  • Auto-acknowledgment when remediation starts                                 │
│  • Resolution sent back when remediation succeeds                              │
│  • Correlation data used to identify root cause                                │
│                                                                                 │
│  SERVICENOW:                                                                    │
│  • Using servicenow.itsm collection for incident management                    │
│  • Change requests auto-created before any Day 2 operation                     │
│  • CMDB automatically updated when VMs are provisioned                         │
│  • Incident auto-resolution when self-healing succeeds                         │
│                                                                                 │
│  The integration pattern I follow:                                             │
│  Alert → Webhook → AWX Job Template → Remediation → Verify → Update ITSM      │
│                                                                                 │
│  RESULT: MTTR reduced from 45 minutes to under 5 minutes for common issues"   │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

### Q: "Describe your experience with self-healing automation"

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│ 💬 SAMPLE ANSWER                                                                │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│ "Self-healing is my specialty. I designed a system where:                       │
│                                                                                 │
│  ARCHITECTURE:                                                                  │
│  • New Relic and Moogsoft send webhooks to Ansible Automation Controller       │
│  • A 'triage' playbook determines the alert type and selects the runbook       │
│  • Runbooks exist for: high CPU, disk full, service down, DB connections       │
│  • After remediation, verification checks confirm the fix worked               │
│  • ServiceNow incident auto-resolved or escalated based on outcome             │
│                                                                                 │
│  KEY DESIGN DECISIONS:                                                          │
│  1. Idempotent runbooks - can run multiple times safely                        │
│  2. Maximum 3 retries before escalation                                        │
│  3. Always create audit trail in ServiceNow                                    │
│  4. Cooldown periods to prevent remediation storms                             │
│  5. Human escalation path for complex issues                                   │
│                                                                                 │
│  RUNBOOKS I'VE BUILT:                                                          │
│  • High CPU: Kill zombies, restart known memory hogs, clear caches             │
│  • Disk full: Log rotation, clean temp files, archive old data                 │
│  • Service down: Graceful restart, dependency check, port validation           │
│  • Certificate expiry: Auto-renew Let's Encrypt, notify for manual certs       │
│                                                                                 │
│  RESULTS:                                                                       │
│  • 70% of incidents now auto-remediated                                        │
│  • On-call team only paged for complex issues                                  │
│  • Customer satisfaction improved (fewer visible outages)"                     │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

### Q: "How do you manage Azure VMs with Ansible?"

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│ 💬 SAMPLE ANSWER                                                                │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│ "I use the azure.azcollection extensively for Azure automation:                 │
│                                                                                 │
│  DYNAMIC INVENTORY:                                                             │
│  • azure_rm plugin auto-discovers VMs across resource groups                   │
│  • Grouping by tags (environment, application, team)                           │
│  • Conditional groups for OS type (linux_vms, windows_vms)                     │
│  • Private IP used for internal networks, public for bastions                  │
│                                                                                 │
│  DAY 1 - PROVISIONING:                                                          │
│  • azure_rm_virtualmachine for VM creation                                     │
│  • azure_rm_securitygroup for NSG rules                                        │
│  • azure_rm_loadbalancer for load balancer setup                               │
│  • Post-provisioning roles apply hardening and monitoring                      │
│                                                                                 │
│  DAY 2 - OPERATIONS:                                                            │
│  • VMSS scaling based on metrics                                               │
│  • Snapshot management before patching                                         │
│  • Disk expansion playbooks                                                    │
│  • Start/stop schedules for cost optimization                                  │
│                                                                                 │
│  PAAS MANAGEMENT:                                                               │
│  • App Service deployment slot swaps                                           │
│  • Azure SQL maintenance windows                                               │
│  • Key Vault secret rotation                                                   │
│  • Application Insights configuration                                          │
│                                                                                 │
│  BEST PRACTICE: I always use service principals with minimal permissions       │
│  and store credentials in Ansible Vault or Azure Key Vault."                   │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 5.2 Technical Deep Dive Questions

### Q: "Walk me through how you'd build a module from scratch"

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│ 💬 SAMPLE ANSWER                                                                │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│ "I follow a structured approach:                                                │
│                                                                                 │
│  1. DEFINE THE INTERFACE                                                        │
│     • What parameters are needed? (required vs optional)                       │
│     • What should be returned?                                                 │
│     • What does check_mode do?                                                 │
│                                                                                 │
│  2. WRITE DOCUMENTATION FIRST                                                   │
│     • DOCUMENTATION block with all parameters                                  │
│     • EXAMPLES block with realistic use cases                                  │
│     • RETURN block with expected outputs                                       │
│                                                                                 │
│  3. IMPLEMENT THE MODULE                                                        │
│     • Import AnsibleModule from ansible.module_utils.basic                     │
│     • Define argument_spec with types and validation                           │
│     • Implement main logic in a class for testability                          │
│     • Handle check_mode (report what would change)                             │
│     • Use module.exit_json() for success                                       │
│     • Use module.fail_json() for errors                                        │
│                                                                                 │
│  4. TEST THOROUGHLY                                                             │
│     • Unit tests with pytest                                                   │
│     • Integration tests with molecule                                          │
│     • Test idempotency (run twice, second should report no changes)            │
│                                                                                 │
│  5. PACKAGE AND DISTRIBUTE                                                      │
│     • If internal: place in library/ or roles/*/library/                       │
│     • If sharing: create Ansible Collection with galaxy.yml                    │
│     • Document installation and usage"                                         │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

### Q: "How do you handle errors and rollback?"

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│ 💬 SAMPLE ANSWER                                                                │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│ "Error handling is critical for production automation:                          │
│                                                                                 │
│  PLAYBOOK-LEVEL:                                                                │
│  • block/rescue/always for try-catch-finally patterns                          │
│  • ignore_errors for non-critical tasks (with result checking)                 │
│  • any_errors_fatal for critical batch operations                              │
│  • max_fail_percentage to limit blast radius                                   │
│                                                                                 │
│  Example pattern I use:                                                         │
│                                                                                 │
│  - name: Safe deployment with rollback                                         │
│    block:                                                                       │
│      - name: Create snapshot                                                    │
│        include_role: snapshot                                                   │
│      - name: Deploy application                                                 │
│        include_role: deploy                                                     │
│      - name: Run health checks                                                  │
│        include_tasks: health_check.yml                                          │
│        register: health                                                         │
│      - name: Fail if unhealthy                                                  │
│        fail:                                                                    │
│          msg: 'Health check failed'                                             │
│        when: not health.passed                                                  │
│    rescue:                                                                      │
│      - name: Restore from snapshot                                              │
│        include_role: restore_snapshot                                           │
│      - name: Notify team                                                        │
│        include_tasks: send_alert.yml                                            │
│    always:                                                                      │
│      - name: Update ServiceNow                                                  │
│        include_tasks: update_ticket.yml                                         │
│                                                                                 │
│  MODULE-LEVEL:                                                                  │
│  • Comprehensive try/except in Python                                          │
│  • Return meaningful error messages                                            │
│  • Support check_mode for dry runs"                                            │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 5.3 Agile/Scrum Questions (From JD)

### Q: "How do you work in an Agile environment?"

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│ 💬 SAMPLE ANSWER                                                                │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│ "I've worked in Agile teams for the past 4 years:                               │
│                                                                                 │
│  SPRINT PLANNING:                                                               │
│  • I help estimate automation stories using story points                       │
│  • I advocate for automation debt tickets in the backlog                       │
│  • I ensure automation is part of Definition of Done                           │
│                                                                                 │
│  DAILY STANDUPS:                                                                │
│  • I report on playbook development progress                                   │
│  • I flag blockers like API changes or permission issues                       │
│  • I share automation wins with the team                                       │
│                                                                                 │
│  SPRINT DEMOS:                                                                  │
│  • I demonstrate new automation capabilities                                   │
│  • I show before/after metrics (time saved, errors prevented)                  │
│                                                                                 │
│  RETROSPECTIVES:                                                                │
│  • I analyze failed automations                                                │
│  • I propose process improvements                                              │
│  • I document lessons learned in the wiki                                      │
│                                                                                 │
│  USER STORIES FOR AUTOMATION:                                                   │
│  • 'As an operator, I want automated patching so I can focus on improvements'  │
│  • Include acceptance criteria (idempotent, documented, tested)                │
│  • Track velocity to improve estimation"                                       │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 5.4 Community of Practice Questions (From JD)

### Q: "How would you contribute to an Ansible Community of Practice?"

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│ 💬 SAMPLE ANSWER                                                                │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│ "I've actively participated in and led automation communities:                  │
│                                                                                 │
│  KNOWLEDGE SHARING:                                                             │
│  • Monthly brown-bag sessions on Ansible topics                                │
│  • 'Module of the Month' presentations                                         │
│  • Troubleshooting workshops                                                   │
│                                                                                 │
│  DOCUMENTATION:                                                                 │
│  • Maintained internal Ansible style guide                                     │
│  • Created playbook templates for common patterns                              │
│  • Built sample project repository for new team members                        │
│                                                                                 │
│  CODE REVIEW:                                                                   │
│  • Weekly pair programming sessions                                            │
│  • PR reviews focusing on best practices                                       │
│  • Linting standards with ansible-lint                                         │
│                                                                                 │
│  MENTORING:                                                                     │
│  • Onboarded 5 engineers to Ansible                                            │
│  • Created 'Ansible from Zero to Hero' internal course                         │
│  • Office hours for troubleshooting help                                       │
│                                                                                 │
│  INITIATIVES I'D BRING:                                                         │
│  • Playbook 'Cookbook' with tested patterns                                    │
│  • Monthly 'Automation Challenge' competitions                                 │
│  • Guest speakers from Red Hat or community"                                   │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

# Part 6: Execution Environments (From JD)

## What Are Execution Environments?

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                         EXECUTION ENVIRONMENTS (EE)                             │
├──────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  OLD WAY (ansible-playbook on control node):                                    │
│  ┌─────────────────────────────────────────┐                                    │
│  │ Control Node                            │                                    │
│  │ ┌─────────────────────────────────────┐ │                                    │
│  │ │ • Python 3.8                        │ │                                    │
│  │ │ • Ansible 2.9                       │ │                                    │
│  │ │ • azure-cli                         │ │                                    │
│  │ │ • boto3                             │ │                                    │
│  │ │ • requests                          │ │  ← Everything installed globally  │
│  │ │ • pywinrm                           │ │     Dependency conflicts!         │
│  │ │ • ...dozens more...                 │ │     "Works on my machine!"        │
│  │ └─────────────────────────────────────┘ │                                    │
│  └─────────────────────────────────────────┘                                    │
│                                                                                  │
│  ═══════════════════════════════════════════════════════════════════════════     │
│                                                                                  │
│  NEW WAY (Execution Environments - containerized):                              │
│  ┌─────────────────────────────────────────┐                                    │
│  │ AAP / AWX Controller                   │                                    │
│  │ ┌───────────┐ ┌───────────┐ ┌───────────┐                                    │
│  │ │   EE 1    │ │   EE 2    │ │   EE 3    │                                    │
│  │ │ ───────── │ │ ───────── │ │ ───────── │                                    │
│  │ │ Azure     │ │ AWS       │ │ Network   │  ← Isolated containers            │
│  │ │ • az-cli  │ │ • boto3   │ │ • napalm  │     Each has ONLY what it needs   │
│  │ │ • pywinrm │ │ • aws-cli │ │ • netmiko │     No conflicts!                 │
│  │ │ • ansible │ │ • ansible │ │ • ansible │     Reproducible everywhere       │
│  │ └───────────┘ └───────────┘ └───────────┘                                    │
│  └─────────────────────────────────────────┘                                    │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

## Building Execution Environments

### Step 1: Install ansible-builder

```bash
pip install ansible-builder
```

### Step 2: Create execution-environment.yml

```yaml
# execution-environment.yml
---
version: 3

# Based on Red Hat's UBI minimal image
images:
  base_image:
    name: registry.redhat.io/ansible-automation-platform-24/ee-minimal-rhel9:latest

dependencies:
  # Ansible collections to include
  galaxy: requirements.yml
  
  # Python packages to install
  python: requirements.txt
  
  # System packages (dnf/yum)
  system: bindep.txt

additional_build_steps:
  # Run after dependencies are installed
  append_final:
    - RUN echo "Custom EE for HTC Global Services"
```

### Step 3: Define dependencies

```yaml
# requirements.yml - Ansible Collections
---
collections:
  - name: azure.azcollection
    version: ">=1.15.0"
  - name: servicenow.itsm
    version: ">=2.0.0"
  - name: community.general
    version: ">=6.0.0"
  - name: ansible.posix
```

```text
# requirements.txt - Python packages
azure-mgmt-compute>=29.0.0
azure-identity>=1.12.0
msrest>=0.7.1
requests>=2.28.0
pysnow>=0.7.17
```

```text
# bindep.txt - System packages
python39-devel [platform:centos-9 platform:rhel-9]
gcc [compile]
```

### Step 4: Build the EE

```bash
# Build the execution environment
ansible-builder build \
  --tag my-company/azure-ee:1.0 \
  --container-runtime podman \
  --file execution-environment.yml

# Push to registry
podman push my-company/azure-ee:1.0 registry.company.com/ansible/azure-ee:1.0
```

### Interview Talking Point

```
"I've built custom execution environments for different use cases:

• AZURE EE: Contains az-collection, pywinrm for Windows, azure-identity
• AWS EE: Contains boto3, amazon.aws collection
• NETWORK EE: Contains napalm, netmiko for network automation

Using ansible-builder, I define dependencies declaratively and build
reproducible container images. This solved our 'works on my machine'
problems and made our automation portable between dev, test, and prod AAP."
```

---

# Part 7: Quick Reference - JD Mapping

| JD Requirement | Where to Study | Interview Talking Points |
|---------------|----------------|--------------------------|
| **5+ years Ansible** | This tutorial | Mention specific projects, metrics |
| **VM/Azure VM automation** | Part 4 | Dynamic inventory, VMSS scaling |
| **Day 2 Operations** | Part 1-2 | Patching, backup, scaling playbooks |
| **ServiceNow integration** | Part 3 | Custom modules, incident management |
| **New Relic integration** | Part 3.1 | NerdGraph API, webhooks |
| **Moogsoft integration** | Part 3.2 | Situation handling, correlation |
| **Self-healing** | Part 3.3-3.4 | End-to-end auto-remediation |
| **Python/Bash scripting** | Module development | Show your custom modules |
| **Linux environments** | Part 6 | Execution environments |
| **Agile/Scrum** | Part 5.3 | Sprint participation stories |
| **CI/CD** | Beginner tutorial | Test pipelines, Galaxy publishing |
| **Docker/Kubernetes** | Development guide | K8s validation module |

---

# Next Steps

1. **Practice speaking through each scenario** - Don't just read, rehearse out loud
2. **Build your own examples** - Modify these playbooks for your resume projects
3. **Prepare metrics** - "Reduced X by Y%" is powerful
4. **Know the architecture** - Be ready to whiteboard the self-healing flow
5. **Ask questions back** - "What monitoring tools does HTC use currently?"

Good luck with your HTC interview! 🎯
