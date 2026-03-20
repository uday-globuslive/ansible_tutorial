# Real-World Ansible Projects

## Understanding Real-World Projects (Beginner's Guide)

These projects show how professionals use Ansible in production. Don't worry if they seem complex - each builds on concepts you've already learned!

### How These Projects Are Organized

Real-world Ansible projects follow a standard structure:

```
project/
│
├── ansible.cfg              # Project settings
├── inventory/               # WHO - Your servers
│   ├── production           # Production servers
│   └── staging              # Staging servers
│
├── group_vars/              # Variables for groups
│   ├── all.yml              # Variables for everyone
│   ├── webservers.yml       # Variables for webservers
│   └── vault.yml            # Encrypted secrets
│
├── playbooks/               # WHAT TO DO - Your automation scripts
│   ├── site.yml             # Main playbook
│   ├── deploy.yml           # Deploy application
│   └── rollback.yml         # Undo deployment if needed
│
└── roles/                   # HOW TO DO IT - Reusable components
    ├── common/              # Shared setup (users, packages)
    ├── nginx/               # Web server configuration
    └── app/                 # Application-specific tasks
```

### What Makes These "Production-Ready"?

| Feature | Why It Matters |
|---------|----------------|
| **Error handling** | Tasks don't crash silently |
| **Rolling updates** | Update 1 server at a time |
| **Health checks** | Verify things work before moving on |
| **Rollback capability** | Can undo changes if something breaks |
| **Notifications** | Team knows when deployments happen |
| **Secrets management** | Passwords aren't in plain text |

---

This section contains complete, production-ready projects you can learn from and adapt.

---

## Project 1: Complete Web Application Deployment

### Scenario
Deploy a Node.js application with Nginx as reverse proxy, PostgreSQL database, and Redis cache.

### What This Project Does (Visual Overview)

```
                     DEPLOYMENT FLOW
═══════════════════════════════════════════════════════════════

   STEP 1: Database         STEP 2: Web Servers      STEP 3: Notify
   ─────────────────        ─────────────────────    ─────────────
                            
   ┌────────────┐           ┌──────────┐  ┌──────────┐   ┌─────────┐
   │ PostgreSQL │    ──►    │  Web 1   │  │  Web 2   │   │  Slack  │
   │  Database  │           │ (Node.js │  │ (Node.js │   │ Channel │
   └────────────┘           │ + Nginx) │  │ + Nginx) │   └─────────┘
        │                   └────┬─────┘  └────┬─────┘
        ▼                        │             │
   Run migrations          Remove from LB  Wait for #1
                           Deploy code     Then deploy
                           Health check
                           Add back to LB
                           
   ┌────────────┐
   │   Redis    │           SERIAL: One server at a time
   │   Cache    │           NO DOWNTIME: Always some servers running!
   └────────────┘
```

### Directory Structure

```
project_webapp/
├── ansible.cfg
├── inventory/
│   ├── production
│   └── staging
├── group_vars/
│   ├── all.yml
│   ├── webservers.yml
│   ├── dbservers.yml
│   └── vault.yml
├── playbooks/
│   ├── site.yml
│   ├── deploy.yml
│   └── rollback.yml
└── roles/
    ├── common/
    ├── nginx/
    ├── nodejs/
    ├── postgresql/
    ├── redis/
    └── app/
```

### inventory/production

```ini
[webservers]
web1.example.com
web2.example.com

[dbservers]
db1.example.com

[cacheservers]
redis1.example.com

[all:vars]
ansible_user=deploy
ansible_python_interpreter=/usr/bin/python3
environment=production
```

### group_vars/all.yml

```yaml
---
# Common settings
app_name: mywebapp
app_user: appuser
app_group: appgroup
deploy_dir: /opt/{{ app_name }}
log_dir: /var/log/{{ app_name }}

# Repository
git_repo: git@github.com:company/mywebapp.git
git_branch: main

# Domain
domain_name: mywebapp.example.com
```

### playbooks/deploy.yml

```yaml
---
- name: Deploy Web Application
  hosts: all
  become: true
  
  pre_tasks:
    - name: Validate deployment
      assert:
        that:
          - version is defined
          - version is match('^v[0-9]+\.[0-9]+\.[0-9]+$')
        fail_msg: "Version must be specified (e.g., v1.0.0)"
      run_once: true

- name: Deploy Database Changes
  hosts: dbservers
  become: true
  serial: 1
  
  tasks:
    - name: Run database migrations
      command: "{{ deploy_dir }}/bin/migrate"
      become_user: "{{ app_user }}"
      run_once: true
      when: run_migrations | default(true)

- name: Deploy Application to Web Servers
  hosts: webservers
  become: true
  serial: 1
  max_fail_percentage: 0
  
  pre_tasks:
    - name: Remove from load balancer
      uri:
        url: "{{ lb_api }}/drain/{{ inventory_hostname }}"
        method: POST
      delegate_to: localhost
      tags: [deploy, lb]
    
    - name: Wait for connections to drain
      pause:
        seconds: 30
      tags: [deploy, lb]
  
  tasks:
    - name: Create deployment timestamp
      set_fact:
        deploy_timestamp: "{{ ansible_date_time.epoch }}"
    
    - name: Create release directory
      file:
        path: "{{ deploy_dir }}/releases/{{ version }}"
        state: directory
        owner: "{{ app_user }}"
        group: "{{ app_group }}"
        mode: '0755'
    
    - name: Clone repository
      git:
        repo: "{{ git_repo }}"
        dest: "{{ deploy_dir }}/releases/{{ version }}"
        version: "{{ version }}"
        accept_hostkey: yes
      become_user: "{{ app_user }}"
    
    - name: Install Node.js dependencies
      npm:
        path: "{{ deploy_dir }}/releases/{{ version }}"
        production: yes
      become_user: "{{ app_user }}"
    
    - name: Build application
      command: npm run build
      args:
        chdir: "{{ deploy_dir }}/releases/{{ version }}"
      become_user: "{{ app_user }}"
    
    - name: Deploy environment file
      template:
        src: env.production.j2
        dest: "{{ deploy_dir }}/releases/{{ version }}/.env"
        owner: "{{ app_user }}"
        mode: '0600'
    
    - name: Update current symlink
      file:
        src: "{{ deploy_dir }}/releases/{{ version }}"
        dest: "{{ deploy_dir }}/current"
        state: link
        force: yes
      notify: Restart application
    
    - name: Flush handlers
      meta: flush_handlers
    
    - name: Health check
      uri:
        url: "http://localhost:{{ app_port }}/health"
        status_code: 200
      register: health
      until: health.status == 200
      retries: 30
      delay: 5
    
    - name: Cleanup old releases
      shell: |
        cd {{ deploy_dir }}/releases
        ls -t | tail -n +{{ keep_releases + 1 }} | xargs -r rm -rf
      args:
        executable: /bin/bash
  
  post_tasks:
    - name: Add back to load balancer
      uri:
        url: "{{ lb_api }}/enable/{{ inventory_hostname }}"
        method: POST
      delegate_to: localhost
      tags: [deploy, lb]
  
  handlers:
    - name: Restart application
      systemd:
        name: "{{ app_name }}"
        state: restarted
        daemon_reload: yes

- name: Post-Deployment Tasks
  hosts: localhost
  
  tasks:
    - name: Send deployment notification
      slack:
        token: "{{ slack_token }}"
        channel: "#deployments"
        msg: |
          ✅ Deployment Complete
          App: {{ app_name }}
          Version: {{ version }}
          Environment: {{ environment }}
      when: notify_slack | default(true)
    
    - name: Record deployment
      uri:
        url: "{{ metrics_api }}/deployment"
        method: POST
        body_format: json
        body:
          app: "{{ app_name }}"
          version: "{{ version }}"
          environment: "{{ environment }}"
          timestamp: "{{ ansible_date_time.iso8601 }}"
      ignore_errors: yes
```

### playbooks/rollback.yml

```yaml
---
- name: Rollback Application
  hosts: webservers
  become: true
  serial: 1
  
  vars_prompt:
    - name: rollback_version
      prompt: "Enter version to rollback to"
      private: no
  
  tasks:
    - name: Check if version exists
      stat:
        path: "{{ deploy_dir }}/releases/{{ rollback_version }}"
      register: version_check
    
    - name: Fail if version doesn't exist
      fail:
        msg: "Version {{ rollback_version }} not found"
      when: not version_check.stat.exists
    
    - name: Remove from load balancer
      uri:
        url: "{{ lb_api }}/drain/{{ inventory_hostname }}"
        method: POST
      delegate_to: localhost
    
    - name: Update symlink to previous version
      file:
        src: "{{ deploy_dir }}/releases/{{ rollback_version }}"
        dest: "{{ deploy_dir }}/current"
        state: link
        force: yes
    
    - name: Restart application
      systemd:
        name: "{{ app_name }}"
        state: restarted
    
    - name: Health check
      uri:
        url: "http://localhost:{{ app_port }}/health"
        status_code: 200
      register: health
      until: health.status == 200
      retries: 30
      delay: 5
    
    - name: Add back to load balancer
      uri:
        url: "{{ lb_api }}/enable/{{ inventory_hostname }}"
        method: POST
      delegate_to: localhost
```

---

## Project 2: Kubernetes Cluster Setup

### Complete K8s Cluster Provisioning

```yaml
# playbooks/k8s_cluster.yml
---
- name: Prepare All Nodes
  hosts: k8s_cluster
  become: true
  
  tasks:
    - name: Disable swap
      command: swapoff -a
      changed_when: false
    
    - name: Remove swap from fstab
      lineinfile:
        path: /etc/fstab
        regexp: '\sswap\s'
        state: absent
    
    - name: Load required kernel modules
      modprobe:
        name: "{{ item }}"
        state: present
      loop:
        - overlay
        - br_netfilter
    
    - name: Persist kernel modules
      copy:
        content: |
          overlay
          br_netfilter
        dest: /etc/modules-load.d/k8s.conf
    
    - name: Set sysctl parameters
      sysctl:
        name: "{{ item.name }}"
        value: "{{ item.value }}"
        state: present
        sysctl_file: /etc/sysctl.d/k8s.conf
        reload: yes
      loop:
        - { name: 'net.bridge.bridge-nf-call-iptables', value: '1' }
        - { name: 'net.bridge.bridge-nf-call-ip6tables', value: '1' }
        - { name: 'net.ipv4.ip_forward', value: '1' }
    
    - name: Install containerd
      import_role:
        name: containerd
    
    - name: Add Kubernetes repository
      apt_repository:
        repo: "deb https://apt.kubernetes.io/ kubernetes-xenial main"
        state: present
        filename: kubernetes
    
    - name: Install Kubernetes packages
      apt:
        name:
          - kubelet={{ k8s_version }}-00
          - kubeadm={{ k8s_version }}-00
          - kubectl={{ k8s_version }}-00
        state: present
        update_cache: yes
    
    - name: Hold Kubernetes packages
      dpkg_selections:
        name: "{{ item }}"
        selection: hold
      loop: [kubelet, kubeadm, kubectl]

- name: Initialize Kubernetes Master
  hosts: k8s_masters[0]
  become: true
  
  tasks:
    - name: Check if cluster initialized
      stat:
        path: /etc/kubernetes/admin.conf
      register: k8s_init
    
    - name: Initialize cluster
      command: >
        kubeadm init
        --pod-network-cidr={{ pod_network_cidr }}
        --apiserver-advertise-address={{ ansible_default_ipv4.address }}
        --kubernetes-version={{ k8s_version }}
      when: not k8s_init.stat.exists
      register: kubeadm_init
    
    - name: Create .kube directory
      file:
        path: "/home/{{ ansible_user }}/.kube"
        state: directory
        owner: "{{ ansible_user }}"
        group: "{{ ansible_user }}"
        mode: '0755'
    
    - name: Copy admin.conf
      copy:
        src: /etc/kubernetes/admin.conf
        dest: "/home/{{ ansible_user }}/.kube/config"
        remote_src: yes
        owner: "{{ ansible_user }}"
        group: "{{ ansible_user }}"
        mode: '0644'
    
    - name: Install Calico CNI
      command: kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
      become_user: "{{ ansible_user }}"
      when: kubeadm_init.changed
    
    - name: Get join command
      command: kubeadm token create --print-join-command
      register: join_command
      changed_when: false
    
    - name: Store join command
      set_fact:
        k8s_join_command: "{{ join_command.stdout }}"
      delegate_to: localhost
      delegate_facts: true

- name: Join Worker Nodes
  hosts: k8s_workers
  become: true
  
  tasks:
    - name: Check if node already joined
      stat:
        path: /etc/kubernetes/kubelet.conf
      register: kubelet_conf
    
    - name: Join cluster
      command: "{{ hostvars['localhost']['k8s_join_command'] }}"
      when: not kubelet_conf.stat.exists

- name: Post-Installation
  hosts: k8s_masters[0]
  become: true
  become_user: "{{ ansible_user }}"
  
  tasks:
    - name: Wait for all nodes to be ready
      command: kubectl get nodes
      register: nodes
      until: "'NotReady' not in nodes.stdout"
      retries: 30
      delay: 10
    
    - name: Display cluster info
      command: kubectl cluster-info
      register: cluster_info
    
    - name: Show cluster status
      debug:
        var: cluster_info.stdout_lines
```

---

## Project 3: CI/CD Pipeline Integration

### Jenkins Pipeline with Ansible

```groovy
// Jenkinsfile
pipeline {
    agent any
    
    parameters {
        string(name: 'VERSION', description: 'Version to deploy')
        choice(name: 'ENVIRONMENT', choices: ['staging', 'production'])
    }
    
    stages {
        stage('Validate') {
            steps {
                sh '''
                    ansible-playbook playbooks/deploy.yml \
                        --syntax-check \
                        -i inventory/${ENVIRONMENT}
                '''
            }
        }
        
        stage('Dry Run') {
            steps {
                sh '''
                    ansible-playbook playbooks/deploy.yml \
                        -i inventory/${ENVIRONMENT} \
                        -e "version=${VERSION}" \
                        --check --diff
                '''
            }
        }
        
        stage('Deploy') {
            when {
                expression { params.ENVIRONMENT == 'staging' || 
                            currentBuild.rawBuild.getCause(hudson.model.Cause$UserIdCause) }
            }
            steps {
                sh '''
                    ansible-playbook playbooks/deploy.yml \
                        -i inventory/${ENVIRONMENT} \
                        -e "version=${VERSION}"
                '''
            }
        }
        
        stage('Smoke Tests') {
            steps {
                sh '''
                    ansible-playbook playbooks/smoke_tests.yml \
                        -i inventory/${ENVIRONMENT}
                '''
            }
        }
    }
    
    post {
        failure {
            sh '''
                ansible-playbook playbooks/rollback.yml \
                    -i inventory/${ENVIRONMENT}
            '''
        }
    }
}
```

### GitHub Actions with Ansible

```yaml
# .github/workflows/deploy.yml
name: Deploy Application

on:
  push:
    tags:
      - 'v*'

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'
      
      - name: Install Ansible
        run: |
          pip install ansible boto3
          ansible-galaxy install -r requirements.yml
      
      - name: Setup SSH key
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
          ssh-keyscan -H ${{ secrets.HOST }} >> ~/.ssh/known_hosts
      
      - name: Deploy to staging
        run: |
          ansible-playbook playbooks/deploy.yml \
            -i inventory/staging \
            -e "version=${{ github.ref_name }}"
        env:
          ANSIBLE_VAULT_PASSWORD: ${{ secrets.VAULT_PASSWORD }}
      
      - name: Run tests
        run: |
          ansible-playbook playbooks/integration_tests.yml \
            -i inventory/staging
      
      - name: Deploy to production
        if: success()
        run: |
          ansible-playbook playbooks/deploy.yml \
            -i inventory/production \
            -e "version=${{ github.ref_name }}"
        env:
          ANSIBLE_VAULT_PASSWORD: ${{ secrets.VAULT_PASSWORD }}
```

---

## Project 4: Server Hardening

### Security Hardening Playbook

```yaml
# playbooks/hardening.yml
---
- name: Security Hardening
  hosts: all
  become: true
  
  vars:
    ssh_port: 22
    allowed_users: [admin, deploy]
    fail2ban_maxretry: 3
    fail2ban_bantime: 3600
  
  tasks:
    # SSH Hardening
    - name: Configure SSH
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: "{{ item.regexp }}"
        line: "{{ item.line }}"
        state: present
        validate: sshd -t -f %s
      loop:
        - { regexp: '^#?Port', line: 'Port {{ ssh_port }}' }
        - { regexp: '^#?PermitRootLogin', line: 'PermitRootLogin no' }
        - { regexp: '^#?PasswordAuthentication', line: 'PasswordAuthentication no' }
        - { regexp: '^#?PermitEmptyPasswords', line: 'PermitEmptyPasswords no' }
        - { regexp: '^#?X11Forwarding', line: 'X11Forwarding no' }
        - { regexp: '^#?MaxAuthTries', line: 'MaxAuthTries 3' }
        - { regexp: '^#?AllowTcpForwarding', line: 'AllowTcpForwarding no' }
        - { regexp: '^#?ClientAliveInterval', line: 'ClientAliveInterval 300' }
        - { regexp: '^#?ClientAliveCountMax', line: 'ClientAliveCountMax 2' }
      notify: Restart SSH
    
    - name: Set SSH allowed users
      lineinfile:
        path: /etc/ssh/sshd_config
        line: "AllowUsers {{ allowed_users | join(' ') }}"
        state: present
      notify: Restart SSH
    
    # Firewall Configuration
    - name: Install UFW
      apt:
        name: ufw
        state: present
    
    - name: Default deny incoming
      ufw:
        direction: incoming
        policy: deny
    
    - name: Default allow outgoing
      ufw:
        direction: outgoing
        policy: allow
    
    - name: Allow SSH
      ufw:
        rule: allow
        port: "{{ ssh_port }}"
        proto: tcp
    
    - name: Allow HTTP/HTTPS
      ufw:
        rule: allow
        port: "{{ item }}"
        proto: tcp
      loop: [80, 443]
      when: "'webservers' in group_names"
    
    - name: Enable UFW
      ufw:
        state: enabled
    
    # Fail2ban
    - name: Install fail2ban
      apt:
        name: fail2ban
        state: present
    
    - name: Configure fail2ban
      template:
        src: templates/jail.local.j2
        dest: /etc/fail2ban/jail.local
      notify: Restart fail2ban
    
    # System Hardening
    - name: Set secure permissions on sensitive files
      file:
        path: "{{ item.path }}"
        mode: "{{ item.mode }}"
      loop:
        - { path: '/etc/passwd', mode: '0644' }
        - { path: '/etc/shadow', mode: '0600' }
        - { path: '/etc/gshadow', mode: '0600' }
        - { path: '/etc/group', mode: '0644' }
    
    - name: Disable unnecessary services
      service:
        name: "{{ item }}"
        state: stopped
        enabled: no
      loop:
        - avahi-daemon
        - cups
      ignore_errors: yes
    
    # Automatic Security Updates
    - name: Install unattended-upgrades
      apt:
        name: unattended-upgrades
        state: present
    
    - name: Enable automatic security updates
      copy:
        content: |
          APT::Periodic::Update-Package-Lists "1";
          APT::Periodic::Unattended-Upgrade "1";
          APT::Periodic::AutocleanInterval "7";
        dest: /etc/apt/apt.conf.d/20auto-upgrades
    
    # Audit logging
    - name: Install auditd
      apt:
        name: auditd
        state: present
    
    - name: Configure audit rules
      copy:
        src: files/audit.rules
        dest: /etc/audit/rules.d/hardening.rules
      notify: Restart auditd
    
    # Kernel hardening
    - name: Apply kernel hardening parameters
      sysctl:
        name: "{{ item.name }}"
        value: "{{ item.value }}"
        state: present
        sysctl_file: /etc/sysctl.d/99-hardening.conf
        reload: yes
      loop:
        # Network hardening
        - { name: 'net.ipv4.conf.all.accept_redirects', value: '0' }
        - { name: 'net.ipv4.conf.all.send_redirects', value: '0' }
        - { name: 'net.ipv4.conf.all.accept_source_route', value: '0' }
        - { name: 'net.ipv4.icmp_echo_ignore_broadcasts', value: '1' }
        - { name: 'net.ipv4.tcp_syncookies', value: '1' }
        # Memory protection
        - { name: 'kernel.randomize_va_space', value: '2' }
        - { name: 'kernel.exec-shield', value: '1' }
  
  handlers:
    - name: Restart SSH
      service:
        name: sshd
        state: restarted
    
    - name: Restart fail2ban
      service:
        name: fail2ban
        state: restarted
    
    - name: Restart auditd
      service:
        name: auditd
        state: restarted
```

---

## Project 5: Monitoring Stack Deployment

### Prometheus + Grafana + Alertmanager

```yaml
# playbooks/monitoring.yml
---
- name: Deploy Monitoring Stack
  hosts: monitoring
  become: true
  
  vars:
    prometheus_version: "2.45.0"
    grafana_version: "10.0.0"
    alertmanager_version: "0.25.0"
    node_exporter_version: "1.6.0"
    
    prometheus_retention: "15d"
    prometheus_scrape_interval: "15s"
    
    grafana_admin_user: admin
    grafana_admin_password: "{{ vault_grafana_password }}"
    
    alertmanager_slack_webhook: "{{ vault_slack_webhook }}"
  
  tasks:
    - name: Create monitoring users
      user:
        name: "{{ item }}"
        system: yes
        shell: /sbin/nologin
        create_home: no
      loop:
        - prometheus
        - alertmanager
    
    # Prometheus
    - name: Create Prometheus directories
      file:
        path: "{{ item }}"
        state: directory
        owner: prometheus
        group: prometheus
        mode: '0755'
      loop:
        - /etc/prometheus
        - /etc/prometheus/rules
        - /var/lib/prometheus
    
    - name: Download Prometheus
      get_url:
        url: "https://github.com/prometheus/prometheus/releases/download/v{{ prometheus_version }}/prometheus-{{ prometheus_version }}.linux-amd64.tar.gz"
        dest: /tmp/prometheus.tar.gz
    
    - name: Extract Prometheus
      unarchive:
        src: /tmp/prometheus.tar.gz
        dest: /tmp
        remote_src: yes
    
    - name: Install Prometheus binaries
      copy:
        src: "/tmp/prometheus-{{ prometheus_version }}.linux-amd64/{{ item }}"
        dest: "/usr/local/bin/{{ item }}"
        remote_src: yes
        mode: '0755'
      loop:
        - prometheus
        - promtool
    
    - name: Deploy Prometheus config
      template:
        src: templates/prometheus.yml.j2
        dest: /etc/prometheus/prometheus.yml
        owner: prometheus
        validate: promtool check config %s
      notify: Restart Prometheus
    
    - name: Deploy alerting rules
      template:
        src: templates/alert_rules.yml.j2
        dest: /etc/prometheus/rules/alerts.yml
        owner: prometheus
      notify: Restart Prometheus
    
    - name: Deploy Prometheus systemd service
      template:
        src: templates/prometheus.service.j2
        dest: /etc/systemd/system/prometheus.service
      notify:
        - Reload systemd
        - Restart Prometheus
    
    - name: Start Prometheus
      systemd:
        name: prometheus
        state: started
        enabled: yes
    
    # Alertmanager
    - name: Download Alertmanager
      get_url:
        url: "https://github.com/prometheus/alertmanager/releases/download/v{{ alertmanager_version }}/alertmanager-{{ alertmanager_version }}.linux-amd64.tar.gz"
        dest: /tmp/alertmanager.tar.gz
    
    - name: Extract and install Alertmanager
      unarchive:
        src: /tmp/alertmanager.tar.gz
        dest: /tmp
        remote_src: yes
    
    - name: Install Alertmanager binary
      copy:
        src: "/tmp/alertmanager-{{ alertmanager_version }}.linux-amd64/alertmanager"
        dest: /usr/local/bin/alertmanager
        remote_src: yes
        mode: '0755'
    
    - name: Deploy Alertmanager config
      template:
        src: templates/alertmanager.yml.j2
        dest: /etc/alertmanager/alertmanager.yml
      notify: Restart Alertmanager
    
    # Grafana
    - name: Add Grafana repository
      apt_repository:
        repo: "deb https://packages.grafana.com/oss/deb stable main"
        state: present
    
    - name: Install Grafana
      apt:
        name: grafana
        state: present
        update_cache: yes
    
    - name: Deploy Grafana config
      template:
        src: templates/grafana.ini.j2
        dest: /etc/grafana/grafana.ini
      notify: Restart Grafana
    
    - name: Deploy Grafana datasources
      template:
        src: templates/grafana-datasources.yml.j2
        dest: /etc/grafana/provisioning/datasources/prometheus.yml
      notify: Restart Grafana
    
    - name: Deploy Grafana dashboards provisioning
      template:
        src: templates/grafana-dashboards.yml.j2
        dest: /etc/grafana/provisioning/dashboards/default.yml
      notify: Restart Grafana
    
    - name: Copy dashboard JSON files
      copy:
        src: "files/dashboards/"
        dest: /var/lib/grafana/dashboards/
      notify: Restart Grafana
    
    - name: Start Grafana
      systemd:
        name: grafana-server
        state: started
        enabled: yes
  
  handlers:
    - name: Reload systemd
      systemd:
        daemon_reload: yes
    
    - name: Restart Prometheus
      systemd:
        name: prometheus
        state: restarted
    
    - name: Restart Alertmanager
      systemd:
        name: alertmanager
        state: restarted
    
    - name: Restart Grafana
      systemd:
        name: grafana-server
        state: restarted

# Deploy Node Exporter to all servers
- name: Deploy Node Exporter
  hosts: all
  become: true
  
  tasks:
    - name: Download Node Exporter
      get_url:
        url: "https://github.com/prometheus/node_exporter/releases/download/v{{ node_exporter_version }}/node_exporter-{{ node_exporter_version }}.linux-amd64.tar.gz"
        dest: /tmp/node_exporter.tar.gz
    
    - name: Extract and install
      unarchive:
        src: /tmp/node_exporter.tar.gz
        dest: /tmp
        remote_src: yes
    
    - name: Install binary
      copy:
        src: "/tmp/node_exporter-{{ node_exporter_version }}.linux-amd64/node_exporter"
        dest: /usr/local/bin/node_exporter
        remote_src: yes
        mode: '0755'
    
    - name: Create systemd service
      copy:
        content: |
          [Unit]
          Description=Node Exporter
          After=network.target
          
          [Service]
          User=nobody
          ExecStart=/usr/local/bin/node_exporter
          
          [Install]
          WantedBy=multi-user.target
        dest: /etc/systemd/system/node_exporter.service
      notify: Restart Node Exporter
    
    - name: Start Node Exporter
      systemd:
        name: node_exporter
        state: started
        enabled: yes
        daemon_reload: yes
  
  handlers:
    - name: Restart Node Exporter
      systemd:
        name: node_exporter
        state: restarted
```

---

## Next Steps

Continue to **15_Troubleshooting** for debugging tips and common issues.
