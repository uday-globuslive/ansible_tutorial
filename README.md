# Ansible Tutorial - Complete Learning Path

Welcome to the comprehensive Ansible tutorial! This guide will take you from beginner to master level.

---

## 🚀 NEW: Ansible Development (For Job Interviews!)

> **Important**: The interviewer mentioned they're looking for **Ansible development experience**, not just infrastructure knowledge. Check out these new guides:

| Guide | Description | Best For |
|-------|-------------|----------|
| [🔧 Ansible Development Guide](17_Custom_Modules/Ansible_Development_Guide.md) | **Complete guide to building custom modules** with 10+ production scenarios, architecture deep-dives, and interview Q&A | **Ansible developers, job interviews** |
| [💼 Real-World Module Scenarios](17_Custom_Modules/Real_World_Module_Scenarios.md) | **12 complete module implementations** including ServiceNow CMDB, Vault secret rotation, K8s validation, compliance scanning | **Portfolio building, showcasing skills** |

### What's Covered in Development Guides:
- ✅ Module architecture and Python structure
- ✅ ServiceNow CMDB integration
- ✅ Database migration automation
- ✅ HashiCorp Vault secret rotation
- ✅ Kubernetes manifest validation
- ✅ CIS compliance scanning
- ✅ Multi-cloud resource management
- ✅ Feature flag management
- ✅ Incident response automation
- ✅ FinOps cost allocation tagging
- ✅ Interview questions with sample answers
- ✅ Testing strategies (Molecule, unit tests)

---

## 🌟 Supplementary Guides (Highly Recommended!)

| Guide | Description | Best For |
|-------|-------------|----------|
| [📘 Complete Beginner Guide](Complete_Beginner_Guide.md) | **Every concept explained in simple terms** with analogies, visuals, and step-by-step examples | Absolute beginners, visual learners |
| [🎯 Interview Prep Supplement](Interview_Prep_Supplement.md) | Topics NOT covered in main tutorial + common interview questions with answers | Job seekers, interview preparation |

---

## 📚 Table of Contents

### Part 1: Fundamentals

| # | Topic | Description | Level |
|---|-------|-------------|-------|
| 01 | [Introduction](01_Introduction/README.md) | What is Ansible, architecture, key concepts | Beginner |
| 02 | [Installation](02_Installation/README.md) | Installing Ansible, configuration, commands | Beginner |
| 03 | [Inventory](03_Inventory/README.md) | Managing hosts, groups, variables | Beginner |
| 04 | [Playbooks](04_Playbooks/README.md) | Writing and running playbooks | Beginner |

### Part 2: Core Concepts

| # | Topic | Description | Level |
|---|-------|-------------|-------|
| 05 | [Modules](05_Modules/README.md) | Built-in modules, common operations | Beginner |
| 06 | [Variables](06_Variables/README.md) | Variables, facts, filters | Intermediate |
| 07 | [Conditionals & Loops](07_Conditionals_Loops/README.md) | When, loops, blocks | Intermediate |

### Part 3: Organization & Reuse

| # | Topic | Description | Level |
|---|-------|-------------|-------|
| 08 | [Roles](08_Roles/README.md) | Creating and using roles | Intermediate |
| 09 | [Templates](09_Templates/README.md) | Jinja2 templating | Intermediate |
| 10 | [Handlers & Error Handling](10_Handlers_ErrorHandling/README.md) | Event-driven tasks, error handling | Intermediate |

### Part 4: Security & Best Practices

| # | Topic | Description | Level |
|---|-------|-------------|-------|
| 11 | [Vault](11_Vault/README.md) | Secrets management | Intermediate |
| 12 | [Best Practices](12_Best_Practices/README.md) | Coding standards, optimization | Intermediate |

### Part 5: Advanced Topics

| # | Topic | Description | Level |
|---|-------|-------------|-------|
| 13 | [Advanced Topics](13_Advanced_Topics/README.md) | Custom modules, collections, testing | Advanced |
| 14 | [Real-World Projects](14_RealWorld_Projects/README.md) | Complete production projects | Advanced |
| 15 | [Troubleshooting](15_Troubleshooting/README.md) | Debugging and problem solving | All Levels |

### Part 6: Specialized Topics

| # | Topic | Description | Level |
|---|-------|-------------|-------|
| 16 | [Kubernetes Deployments](16_Kubernetes_Deployments/README.md) | K8s cluster setup, app deployment, operators | Advanced |
| 17 | [Custom Modules](17_Custom_Modules/README.md) | Building your own Ansible modules | Expert |

### 🔥 Ansible Development (Interview Focus!)

| # | Guide | Description | Level |
|---|-------|-------------|-------|
| NEW | [🔧 Ansible Development Guide](17_Custom_Modules/Ansible_Development_Guide.md) | Complete module development with 10+ scenarios | Expert |
| NEW | [💼 Real-World Module Scenarios](17_Custom_Modules/Real_World_Module_Scenarios.md) | 12 production module implementations | Expert |

---

## 🎯 Learning Paths

### Path 1: Quick Start (1-2 days)
For those who need to start using Ansible quickly:
1. Introduction (30 min)
2. Installation (30 min)
3. Inventory Basics (30 min)
4. Your First Playbook (1 hour)
5. Essential Modules (1 hour)

### Path 2: Standard Learning (1-2 weeks)
For a thorough understanding:
1. Complete sections 01-06
2. Practice with sample playbooks
3. Continue with sections 07-11
4. Build a small project

### Path 3: Mastery (1 month)
For becoming an Ansible expert:
1. Complete all sections
2. Build multiple real-world projects
3. Contribute to Ansible Galaxy
4. Implement CI/CD with Ansible

---

## 🛠️ Practice Projects

### Beginner Projects
- [ ] Configure a web server (Nginx/Apache)
- [ ] Set up user accounts and SSH keys
- [ ] Deploy a static website
- [ ] Configure system packages and updates

### Intermediate Projects
- [ ] Deploy a LAMP/LEMP stack
- [ ] Set up Docker and containers
- [ ] Configure monitoring (Prometheus/Grafana)
- [ ] Implement automated backups

### Advanced Projects
- [ ] Set up a Kubernetes cluster
- [ ] Build CI/CD pipeline integration
- [ ] Multi-cloud deployment
- [ ] Zero-downtime blue-green deployment

### Expert Projects
- [ ] Create custom API integration module
- [ ] Build Ansible-based K8s Operator
- [ ] Implement compliance automation
- [ ] Multi-cluster sync with GitOps

---

## 🎯 Interview Preparation: Ansible Development Focus

Since the interviewer is looking for **Ansible development experience**, here's what to emphasize:

### What to Highlight in Your Interview

| Topic | What to Say | Where to Learn |
|-------|-------------|----------------|
| **Custom Modules** | "I developed custom modules for ServiceNow CMDB integration, reducing manual CMDB updates by 90%" | [Development Guide](17_Custom_Modules/Ansible_Development_Guide.md) |
| **API Integration** | "I built modules that integrate with internal REST APIs using Python's fetch_url" | [Real-World Scenarios](17_Custom_Modules/Real_World_Module_Scenarios.md) |
| **Idempotency** | "I ensure all my modules support check mode and only make changes when necessary" | [Module Architecture](17_Custom_Modules/Ansible_Development_Guide.md#module-architecture-deep-dive) |
| **Testing** | "I use Molecule for integration testing and pytest for unit tests" | [Testing Section](17_Custom_Modules/Ansible_Development_Guide.md#testing) |
| **Security** | "I always use no_log for sensitive parameters and validate all inputs" | All module examples |

### Key Development Skills to Demonstrate

```
┌────────────────────────────────────────────────────────────────────┐
│                    ANSIBLE DEVELOPER SKILLS                       │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  1. PYTHON EXPERTISE                                               │
│     ├── AnsibleModule class usage                                  │
│     ├── Error handling (try/except, fail_json)                     │
│     └── API integration (requests, fetch_url)                      │
│                                                                    │
│  2. MODULE DEVELOPMENT                                             │
│     ├── DOCUMENTATION, EXAMPLES, RETURN blocks                     │
│     ├── Argument specification (types, choices, no_log)            │
│     └── Check mode support                                         │
│                                                                    │
│  3. TESTING                                                        │
│     ├── Unit tests with pytest                                     │
│     ├── Integration tests with Molecule                            │
│     └── Idempotency verification                                   │
│                                                                    │
│  4. REAL-WORLD INTEGRATIONS                                        │
│     ├── ServiceNow, Jira, PagerDuty                               │
│     ├── HashiCorp Vault, AWS Secrets Manager                       │
│     └── Kubernetes, Docker, Cloud providers                        │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 📝 Quick Reference Files

| File | Location | Description |
|------|----------|-------------|
| Commands Cheatsheet | [02_Installation/commands_cheatsheet.md](02_Installation/commands_cheatsheet.md) | All common Ansible commands |
| Sample Inventories | [03_Inventory/sample_inventories.md](03_Inventory/sample_inventories.md) | Ready-to-use inventory examples |
| Sample Playbooks | [04_Playbooks/sample_playbooks.yml](04_Playbooks/sample_playbooks.yml) | Production-ready playbook examples |
| More Scenarios | [14_RealWorld_Projects/more_scenarios.md](14_RealWorld_Projects/more_scenarios.md) | Additional real-world projects |
| K8s Advanced | [16_Kubernetes_Deployments/advanced_scenarios.md](16_Kubernetes_Deployments/advanced_scenarios.md) | StatefulSets, CronJobs, Istio, HPA |
| Module Examples | [17_Custom_Modules/more_use_cases.md](17_Custom_Modules/more_use_cases.md) | API, SSL, Compliance modules |

---

## 💡 Key Concepts Summary

### Ansible Architecture
```
Control Node (Ansible installed)
      ↓ SSH
Managed Nodes (Servers to configure)
```

### Core Components
- **Inventory**: Where (which servers)
- **Playbook**: What (tasks to run)
- **Module**: How (actual commands)
- **Role**: Reusable package of automation

### Essential Commands
```bash
# Test connection
ansible all -m ping

# Run ad-hoc command
ansible webservers -m shell -a "uptime"

# Run playbook
ansible-playbook site.yml

# Dry run
ansible-playbook site.yml --check --diff

# With vault
ansible-playbook site.yml --ask-vault-pass
```

---

## 📖 Additional Resources

### Official Documentation
- [Ansible Documentation](https://docs.ansible.com/)
- [Module Index](https://docs.ansible.com/ansible/latest/modules/modules_by_category.html)
- [Ansible Galaxy](https://galaxy.ansible.com/)

### Community
- [Ansible Reddit](https://www.reddit.com/r/ansible/)
- [Ansible GitHub](https://github.com/ansible/ansible)

### Certifications
- Red Hat Certified Specialist in Ansible Automation
- Red Hat Certified Engineer (RHCE)

---

## ✅ Skills Checklist

### Beginner
- [ ] Install and configure Ansible
- [ ] Create static inventory
- [ ] Write basic playbooks
- [ ] Use common modules (apt, copy, service)
- [ ] Run ad-hoc commands

### Intermediate
- [ ] Use variables and facts
- [ ] Write conditionals and loops
- [ ] Create and use roles
- [ ] Use Jinja2 templates
- [ ] Encrypt with Vault
- [ ] Handle errors gracefully

### Advanced
- [ ] Write custom modules
- [ ] Create collections
- [ ] Dynamic inventory
- [ ] Integration with CI/CD
- [ ] Testing with Molecule
- [ ] Performance optimization

### Kubernetes & Custom Modules
- [ ] Deploy applications to Kubernetes
- [ ] Manage multi-cluster environments
- [ ] Use Helm with Ansible
- [ ] Create Ansible-based K8s Operators
- [ ] Build custom modules for APIs
- [ ] Certificate and compliance automation

---

## 🚀 Getting Started

1. **Install Ansible**
   ```bash
   pip install ansible
   ```

2. **Create your first inventory**
   ```ini
   # inventory/hosts
   [local]
   localhost ansible_connection=local
   ```

3. **Write your first playbook**
   ```yaml
   # hello.yml
   - name: Hello World
     hosts: local
     tasks:
       - name: Print message
         debug:
           msg: "Hello, Ansible!"
   ```

4. **Run it!**
   ```bash
   ansible-playbook -i inventory/hosts hello.yml
   ```

---

*Happy Automating! 🎉*

---

**Created for DevOps engineers who want to master infrastructure automation with Ansible.**
