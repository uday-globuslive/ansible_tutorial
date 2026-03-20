# Ansible Best Practices

## Project Structure

### Recommended Directory Layout

```
ansible-project/
├── ansible.cfg                 # Project configuration
├── requirements.yml            # Role/collection dependencies
├── site.yml                    # Master playbook
│
├── inventory/
│   ├── production/
│   │   ├── hosts.yml          # Production hosts
│   │   ├── group_vars/
│   │   │   ├── all/
│   │   │   │   ├── vars.yml
│   │   │   │   └── vault.yml  # Encrypted
│   │   │   └── webservers.yml
│   │   └── host_vars/
│   │       └── web1.prod.example.com.yml
│   └── staging/
│       ├── hosts.yml
│       └── group_vars/
│           └── all.yml
│
├── playbooks/
│   ├── webservers.yml
│   ├── dbservers.yml
│   ├── deploy.yml
│   └── maintenance/
│       ├── backup.yml
│       └── update.yml
│
├── roles/
│   ├── common/
│   ├── nginx/
│   ├── mysql/
│   └── app/
│
├── library/                    # Custom modules
├── filter_plugins/             # Custom filters
├── callback_plugins/           # Custom callbacks
│
├── files/                      # Static files
├── templates/                  # Global templates
│
├── tests/
│   ├── integration/
│   └── unit/
│
└── docs/
    ├── README.md
    └── RUNBOOK.md
```

---

## Coding Standards

### 1. YAML Formatting

```yaml
# ✅ Good: Consistent 2-space indentation
---
- name: Good formatting example
  hosts: all
  become: true
  
  vars:
    packages:
      - nginx
      - vim
  
  tasks:
    - name: Install packages
      apt:
        name: "{{ packages }}"
        state: present

# ❌ Bad: Inconsistent, no task names
---
- hosts: all
  tasks:
  - apt: name=nginx state=present
  -   copy:
          src: file
          dest: /tmp
```

### 2. Task Naming

```yaml
# ✅ Good: Descriptive, action-oriented names
- name: Install nginx web server
- name: Create application directory with correct permissions
- name: Deploy nginx configuration from template
- name: Ensure nginx service is running and enabled

# ❌ Bad: Vague or missing names
- name: Install
- name: Do stuff
- apt:  # No name at all
    name: nginx
```

### 3. Variable Naming

```yaml
# ✅ Good: Descriptive, snake_case
nginx_worker_processes: 4
database_connection_pool_size: 20
app_deployment_path: /opt/myapp
enable_ssl_redirect: true

# ❌ Bad: Unclear, inconsistent
n: 4
DBpool: 20
appPath: /opt/myapp
SSL: true
```

### 4. Use Native YAML Syntax

```yaml
# ✅ Good: Native YAML
- name: Install packages
  apt:
    name:
      - nginx
      - vim
    state: present
    update_cache: yes

# ❌ Avoid: Key=value syntax (harder to read)
- apt: name=nginx,vim state=present update_cache=yes
```

### 5. Quote Strings When Needed

```yaml
# ✅ Good: Quote when needed
app_version: "1.0.0"        # Looks like float, quote it
port: "8080"                # If you want string
name: "true"                # Actually a string, not boolean
path: "/path/with spaces/file.txt"

# ❌ Risky: Might be interpreted wrong
version: 1.0.0              # Interpreted as 1.0
enabled: yes                # Interpreted as boolean
```

---

## Performance Optimization

### 1. Use Free Strategy for Independent Tasks

```yaml
- name: Fast parallel execution
  hosts: all
  strategy: free  # Tasks run as soon as host is ready
  
  tasks:
    - name: Independent task 1
      command: /opt/task1.sh
    
    - name: Independent task 2
      command: /opt/task2.sh
```

### 2. Reduce SSH Connections

```yaml
# ansible.cfg
[ssh_connection]
pipelining = True
ssh_args = -o ControlMaster=auto -o ControlPersist=60s -o ControlPath=/tmp/ansible-%r@%h:%p
```

### 3. Limit Fact Gathering

```yaml
# Disable entirely when not needed
- name: Fast playbook
  hosts: all
  gather_facts: no
  tasks: [...]

# Or gather specific facts only
- name: Gather minimal facts
  hosts: all
  gather_facts: no
  
  tasks:
    - setup:
        gather_subset:
          - min
          - network
```

### 4. Cache Facts

```ini
# ansible.cfg
[defaults]
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_facts
fact_caching_timeout = 86400
```

### 5. Use Async for Long Tasks

```yaml
- name: Long running task
  command: /opt/long_process.sh
  async: 3600      # Max runtime in seconds
  poll: 0          # Fire and forget

- name: Wait for completion
  async_status:
    jid: "{{ long_task.ansible_job_id }}"
  register: job_result
  until: job_result.finished
  retries: 60
  delay: 60
```

### 6. Optimize Package Installation

```yaml
# ✅ Good: Install all packages at once
- name: Install packages
  apt:
    name:
      - nginx
      - vim
      - git
      - curl
    state: present

# ❌ Bad: Separate tasks for each package
- name: Install nginx
  apt:
    name: nginx
- name: Install vim
  apt:
    name: vim
```

---

## Security Best Practices

### 1. Never Hardcode Secrets

```yaml
# ❌ Never do this
db_password: "SuperSecret123!"

# ✅ Use vault
db_password: "{{ vault_db_password }}"

# ✅ Or environment variables
db_password: "{{ lookup('env', 'DB_PASSWORD') }}"
```

### 2. Limit Privilege Escalation

```yaml
# ✅ Only use become when needed
- name: Read config (no sudo needed)
  command: cat /etc/app/config.yml
  
- name: Update system packages (requires sudo)
  apt:
    upgrade: yes
  become: true

# ❌ Avoid: become: true at play level when not needed
```

### 3. Validate External Input

```yaml
- name: Validate user input
  assert:
    that:
      - app_version is match('^[0-9]+\.[0-9]+\.[0-9]+$')
      - deploy_env in ['staging', 'production']
    fail_msg: "Invalid input parameters"
```

### 4. Use no_log for Sensitive Tasks

```yaml
- name: Create database user
  mysql_user:
    name: "{{ db_user }}"
    password: "{{ db_password }}"
  no_log: true  # Don't log password
```

### 5. Verify File Integrity

```yaml
- name: Download with checksum verification
  get_url:
    url: "{{ app_download_url }}"
    dest: /tmp/app.tar.gz
    checksum: "sha256:{{ app_sha256_checksum }}"
```

---

## Idempotency Guidelines

### 1. Avoid shell/command When Module Exists

```yaml
# ❌ Not idempotent
- name: Install nginx
  shell: apt-get install -y nginx

# ✅ Idempotent
- name: Install nginx
  apt:
    name: nginx
    state: present
```

### 2. Use creates/removes Arguments

```yaml
- name: Initialize application
  command: /opt/app/init.sh
  args:
    creates: /opt/app/.initialized
```

### 3. Check Before Actions

```yaml
- name: Check if already deployed
  stat:
    path: /opt/app/version-{{ version }}
  register: version_check

- name: Deploy only if not already present
  unarchive:
    src: "{{ app_url }}"
    dest: /opt/app
  when: not version_check.stat.exists
```

---

## Testing & Validation

### 1. Always Use Check Mode

```bash
# Test before applying
ansible-playbook playbook.yml --check --diff
```

### 2. Use Syntax Check

```bash
ansible-playbook playbook.yml --syntax-check
```

### 3. Use Ansible Lint

```bash
# Install
pip install ansible-lint

# Run
ansible-lint playbook.yml
ansible-lint roles/

# Configuration: .ansible-lint
skip_list:
  - 'yaml'
  - 'role-name'
```

### 4. Write Tests with Molecule

```bash
# Install
pip install molecule molecule-docker

# Initialize
cd roles/nginx
molecule init scenario -r nginx -d docker

# Test
molecule test
```

---

## Documentation

### 1. Document Variables

```yaml
# defaults/main.yml
---
# nginx_port: HTTP listen port
# Type: integer
# Default: 80
nginx_port: 80

# nginx_server_name: Server hostname
# Type: string
# Required: yes
nginx_server_name: localhost

# nginx_ssl_enabled: Enable SSL/TLS
# Type: boolean
# Default: false
nginx_ssl_enabled: false
```

### 2. README for Roles

```markdown
# Role: nginx

## Description
Installs and configures Nginx web server.

## Requirements
- Ubuntu 20.04+ or CentOS 8+
- Ansible 2.10+

## Variables
| Variable | Default | Description |
|----------|---------|-------------|
| nginx_port | 80 | HTTP port |
| nginx_ssl | false | Enable SSL |

## Dependencies
- common

## Example Playbook
\`\`\`yaml
- hosts: webservers
  roles:
    - nginx
\`\`\`

## License
MIT
```

### 3. Inline Comments

```yaml
tasks:
  - name: Configure nginx
    template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    # NOTE: This template includes ALL virtual hosts
    # If you need site-specific configs, add to sites-enabled/
    notify: Reload nginx
```

---

## Version Control

### 1. .gitignore

```gitignore
# Ansible
*.retry
*.log

# Vault password files
.vault_pass*
*.vault_pass

# Sensitive files
*_unencrypted.yml
secrets_plain.yml

# Local testing
.molecule/
tests/output/

# Python
__pycache__/
*.pyc
.venv/

# IDE
.idea/
.vscode/
*.swp
```

### 2. Branch Strategy

```
main/master    → Production-ready code
develop        → Integration branch
feature/*      → New features
hotfix/*       → Production fixes
```

### 3. Commit Messages

```
feat(nginx): add SSL configuration support
fix(mysql): correct backup script permissions
docs(readme): update installation instructions
refactor(common): simplify package installation loop
test(nginx): add molecule tests for SSL
```

---

## Checklist Summary

### Before Commit
- [ ] `ansible-lint` passes
- [ ] `--syntax-check` passes
- [ ] `--check --diff` runs successfully
- [ ] All vault files encrypted
- [ ] No hardcoded secrets
- [ ] Tasks have descriptive names
- [ ] Variables are documented

### Before Deployment
- [ ] Tested in staging environment
- [ ] Rollback plan documented
- [ ] Backup completed
- [ ] Team notified
- [ ] Change request approved

---

## Next Steps

Continue to **13_Advanced_Topics** for advanced Ansible techniques.
