# Ansible Vault - Secrets Management

## What is Ansible Vault? (Beginner Explanation)

**Ansible Vault** encrypts sensitive data like passwords, API keys, and certificates so they can be safely stored in version control (like Git).

### The Problem: Secrets in Plain Text

Imagine committing this to Git:
```yaml
# ❌ DANGER! Anyone with repo access sees this!
db_password: "SuperSecret123"
aws_secret_key: "AKIAIOSFODNN7EXAMPLE"
```

This is like writing your house key code on a public billboard!

### The Solution: Vault Encryption

```
┌─────────────────────┐     Vault Encrypt     ┌─────────────────────────┐
│ secrets.yml         │   ───────────────→  │ secrets.yml             │
│                     │                   │                         │
│ db_password:        │                   │ $ANSIBLE_VAULT;1.1;AES  │
│   SuperSecret123    │                   │ 38613033346465636234... │
│                     │                   │ 62343639626234383134... │
│ (READABLE!)         │                   │ (ENCRYPTED - SAFE!)     │
└─────────────────────┘                   └─────────────────────────┘
                                                     │
                                                     ▼
                                          ✅ Safe to commit to Git!
```

### Analogy: A Locked Diary

Vault is like a **diary with a lock**:
- Without the key (vault password): You see gibberish
- With the key: You can read the contents
- You can carry the locked diary anywhere safely

```
Plain Text → Vault Encryption → Encrypted File → Safe to Commit
```

---

## Basic Vault Commands

### Encrypting Files

```bash
# Create new encrypted file
ansible-vault create secrets.yml

# Encrypt existing file
ansible-vault encrypt secrets.yml

# Encrypt multiple files
ansible-vault encrypt secret1.yml secret2.yml

# Encrypt with specific vault ID
ansible-vault encrypt --vault-id prod@prompt secrets.yml
```

### Decrypting Files

```bash
# Decrypt file permanently
ansible-vault decrypt secrets.yml

# View encrypted file (without decrypting)
ansible-vault view secrets.yml
```

### Editing Encrypted Files

```bash
# Edit encrypted file
ansible-vault edit secrets.yml

# Opens in default editor, re-encrypts on save
```

### Changing Passwords

```bash
# Change vault password
ansible-vault rekey secrets.yml

# Rekey with new vault ID
ansible-vault rekey --new-vault-id prod@prompt secrets.yml
```

---

## Using Vault in Playbooks

### Method 1: Separate Encrypted File

```yaml
# group_vars/all/vault.yml (encrypted)
---
vault_db_password: "super_secret_password"
vault_api_key: "sk-1234567890abcdef"
vault_ssl_key: |
  -----BEGIN PRIVATE KEY-----
  MIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEA...
  -----END PRIVATE KEY-----
```

```yaml
# group_vars/all/vars.yml (plain text)
---
db_password: "{{ vault_db_password }}"
api_key: "{{ vault_api_key }}"
```

```yaml
# playbook.yml
---
- name: Deploy Application
  hosts: webservers
  vars_files:
    - group_vars/all/vault.yml
  
  tasks:
    - name: Configure database
      template:
        src: db.conf.j2
        dest: /etc/app/db.conf
      # Uses db_password which references vault_db_password
```

### Method 2: Inline Encrypted Variables

```yaml
# vars.yml - Use ansible-vault encrypt_string
---
db_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  61626364656667686970716b6c6d6e6f70717273747576777879...
  7a30313233343536373839306162636465666768696a6b6c6d...

plain_variable: "This is not encrypted"
```

### Encrypting Individual Strings

```bash
# Encrypt a string
ansible-vault encrypt_string 'my_secret_password' --name 'db_password'

# Output:
# db_password: !vault |
#           $ANSIBLE_VAULT;1.1;AES256
#           61626364656...

# With vault ID
ansible-vault encrypt_string 'secret' --name 'var_name' --vault-id prod@prompt

# From stdin (for scripts)
echo -n 'my_secret' | ansible-vault encrypt_string --stdin-name 'password'
```

---

## Running Playbooks with Vault

### Ask for Password

```bash
# Prompt for vault password
ansible-playbook playbook.yml --ask-vault-pass

# Short form
ansible-playbook playbook.yml -J
```

### Password File

```bash
# Create password file
echo "my_vault_password" > ~/.vault_pass
chmod 600 ~/.vault_pass

# Use password file
ansible-playbook playbook.yml --vault-password-file ~/.vault_pass

# Or in ansible.cfg
# [defaults]
# vault_password_file = ~/.vault_pass
```

### Password Script

```bash
#!/bin/bash
# vault_pass.sh - Get password from secure storage
# Example: AWS Secrets Manager, HashiCorp Vault, etc.

aws secretsmanager get-secret-value \
  --secret-id ansible-vault-password \
  --query SecretString \
  --output text
```

```bash
chmod +x vault_pass.sh
ansible-playbook playbook.yml --vault-password-file ./vault_pass.sh
```

### Environment Variable

```bash
# Set vault password in environment
export ANSIBLE_VAULT_PASSWORD_FILE=~/.vault_pass

# Now run without specifying password
ansible-playbook playbook.yml
```

---

## Multiple Vault Passwords (Vault IDs)

### Creating with Vault IDs

```bash
# Create with vault ID
ansible-vault create --vault-id dev@prompt secrets_dev.yml
ansible-vault create --vault-id prod@~/.vault_pass_prod secrets_prod.yml

# Encrypt with vault ID
ansible-vault encrypt --vault-id staging@prompt staging_secrets.yml
```

### Using Multiple Vault IDs

```bash
# Run with multiple vault passwords
ansible-playbook playbook.yml \
  --vault-id dev@~/.vault_pass_dev \
  --vault-id prod@~/.vault_pass_prod
```

### Vault ID in Files

```yaml
# File header shows vault ID
$ANSIBLE_VAULT;1.2;AES256;prod
61626364656667686970716b6c6d6e6f70...
```

---

## Best Practices

### 1. Use Descriptive Variable Names

```yaml
# Good - clearly indicates encrypted
vault_database_password: "secret123"
vault_api_token: "token-xyz"

# Reference in regular vars
database_password: "{{ vault_database_password }}"
```

### 2. Separate Vault Files

```
group_vars/
├── all/
│   ├── vars.yml         # Plain variables
│   └── vault.yml        # Encrypted (includes only secrets)
├── production/
│   ├── vars.yml
│   └── vault.yml
└── staging/
    ├── vars.yml
    └── vault.yml
```

### 3. Never Commit Unencrypted Secrets

```bash
# .gitignore
*.vault_pass
.vault_pass*
*_unencrypted.yml
```

### 4. Use Different Passwords per Environment

```bash
# Different passwords for different environments
ansible-playbook playbook.yml \
  --vault-id production@~/.vault_pass_prod \
  --vault-id staging@~/.vault_pass_staging
```

### 5. Git Pre-commit Hook

```bash
#!/bin/bash
# .git/hooks/pre-commit

# Check for unencrypted vault files
for file in $(git diff --cached --name-only | grep vault); do
    if ! head -n1 "$file" | grep -q '$ANSIBLE_VAULT'; then
        echo "ERROR: $file is not encrypted!"
        exit 1
    fi
done
```

---

## Complete Vault Structure Example

### Directory Layout

```
ansible-project/
├── ansible.cfg
├── inventory/
│   ├── production
│   └── staging
├── group_vars/
│   ├── all/
│   │   ├── vars.yml
│   │   └── vault.yml       # Encrypted
│   ├── production/
│   │   ├── vars.yml
│   │   └── vault.yml       # Encrypted
│   └── staging/
│       ├── vars.yml
│       └── vault.yml       # Encrypted
├── playbooks/
│   └── deploy.yml
└── .vault_pass             # NOT in version control
```

### group_vars/all/vault.yml

```yaml
# ansible-vault edit group_vars/all/vault.yml
---
vault_common_ssh_key: |
  -----BEGIN OPENSSH PRIVATE KEY-----
  b3BlbnNzaC1rZXktdjEAAAAA...
  -----END OPENSSH PRIVATE KEY-----
```

### group_vars/production/vault.yml

```yaml
# Encrypted - production secrets
---
vault_db_password: "prod_super_secret_123"
vault_api_key: "prod-api-key-xyz"
vault_ssl_certificate_key: |
  -----BEGIN PRIVATE KEY-----
  MIIEvQIBADANBgkqhkiG9w0BAQEFA...
  -----END PRIVATE KEY-----
```

### group_vars/production/vars.yml

```yaml
# Plain text - references vault variables
---
environment: production
db_host: db.prod.example.com
db_port: 5432
db_password: "{{ vault_db_password }}"
api_key: "{{ vault_api_key }}"
ssl_key: "{{ vault_ssl_certificate_key }}"
```

### ansible.cfg

```ini
[defaults]
vault_password_file = .vault_pass
# Or use vault identity
# vault_identity_list = production@~/.vault_pass_prod, staging@~/.vault_pass_staging
```

---

## Integration with External Secret Managers

### HashiCorp Vault (Example)

```yaml
# Using community.hashi_vault collection
- name: Get secret from HashiCorp Vault
  set_fact:
    db_password: "{{ lookup('community.hashi_vault.hashi_vault', 
                    'secret/data/database:password',
                    url='https://vault.example.com',
                    token=lookup('env', 'VAULT_TOKEN')) }}"
```

### AWS Secrets Manager

```yaml
# Using amazon.aws collection
- name: Get secret from AWS Secrets Manager
  set_fact:
    db_password: "{{ lookup('amazon.aws.aws_secret', 
                    'myapp/database/password',
                    region='us-east-1') }}"
```

### Azure Key Vault

```yaml
# Using azure.azcollection
- name: Get secret from Azure Key Vault
  azure_rm_keyvaultsecret_info:
    vault_uri: https://myvault.vault.azure.net
    name: database-password
  register: secret

- set_fact:
    db_password: "{{ secret.secrets[0].secret }}"
```

---

## Vault Security Checklist

```
✅ Vault password files have 600 permissions
✅ Vault passwords are NOT in version control
✅ Different vault passwords per environment
✅ Pre-commit hooks check encryption status
✅ Regular vault password rotation schedule
✅ Secrets are prefixed with vault_
✅ No hardcoded secrets in templates
✅ All team members have access to required passwords
✅ Audit log of vault file changes
```

---

## Quick Reference

| Command | Purpose |
|---------|---------|
| `ansible-vault create file.yml` | Create new encrypted file |
| `ansible-vault encrypt file.yml` | Encrypt existing file |
| `ansible-vault decrypt file.yml` | Decrypt file |
| `ansible-vault edit file.yml` | Edit encrypted file |
| `ansible-vault view file.yml` | View decrypted content |
| `ansible-vault rekey file.yml` | Change password |
| `ansible-vault encrypt_string` | Encrypt single value |

---

## Next Steps

Continue to **12_Best_Practices** to learn Ansible coding standards and optimization techniques.
