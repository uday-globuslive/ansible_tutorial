# Ansible for Kubernetes Deployments

## Understanding Kubernetes + Ansible (Beginner Explanation)

### What is Kubernetes?

Before diving in, let's understand what Kubernetes (K8s) is:

**Simple Analogy**: Kubernetes is like a **shipping port manager**:
- Your applications = Shipping containers
- Kubernetes = The manager who decides where containers go
- Nodes (servers) = Ships that carry containers

```
┌──────────────────────────────────────────────────┐
│              KUBERNETES CLUSTER                    │
│                                                    │
│  ┌────────────┐    ┌────────────┐    ┌────────────┐  │
│  │   Node 1   │    │   Node 2   │    │   Node 3   │  │
│  │            │    │            │    │            │  │
│  │ ┌───┐┌───┐ │    │ ┌───┐┌───┐ │    │ ┌───┐┌───┐ │  │
│  │ │App││App│ │    │ │App││App│ │    │ │App││App│ │  │
│  │ └───┘└───┘ │    │ └───┘└───┘ │    │ └───┘└───┘ │  │
│  └────────────┘    └────────────┘    └────────────┘  │
│                                                    │
│       K8s automatically manages where apps run     │
└──────────────────────────────────────────────────┘
```

### Why Use Ansible WITH Kubernetes?

**"Wait, doesn't Kubernetes manage itself?"** Yes, but:

| Kubernetes Handles | Ansible Handles |
|-------------------|----------------|
| Running containers | Setting up the cluster initially |
| Scaling apps | Installing K8s on bare metal |
| Service discovery | Managing multiple clusters |
| Load balancing | Hybrid cloud/VM environments |
| Self-healing | Configuration before K8s exists |

**Think of it this way:**
- **Kubernetes** = Manages apps INSIDE the cluster
- **Ansible** = Sets up the cluster, manages things OUTSIDE

## Why Use Ansible with Kubernetes?

While Kubernetes has its own declarative configuration (kubectl, Helm), Ansible adds value in several ways:

### Use Cases for Ansible + Kubernetes

| Use Case | Why Ansible? |
|----------|-------------|
| **Cluster Provisioning** | Set up K8s clusters on bare metal or cloud |
| **Pre-deployment Setup** | Configure nodes before joining cluster |
| **Multi-cluster Management** | Manage multiple clusters consistently |
| **Hybrid Deployments** | Deploy both K8s resources AND traditional VMs |
| **Configuration Management** | Manage ConfigMaps, Secrets at scale |
| **CI/CD Integration** | Orchestrate complex deployment pipelines |
| **Compliance & Auditing** | Ensure consistent cluster configurations |
| **Disaster Recovery** | Automate backup and restore procedures |

---

## Part 1: Setting Up Kubernetes with Ansible

### Prerequisites

```yaml
# requirements.yml
---
collections:
  - name: kubernetes.core
    version: ">=2.4.0"
  - name: cloud.common
    version: ">=2.1.0"
```

```bash
# Install collections
ansible-galaxy collection install -r requirements.yml

# Install Python dependencies
pip install kubernetes openshift PyYAML
```

### Cluster Installation Playbook

```yaml
# playbooks/kubernetes/install_cluster.yml
---
- name: Install Kubernetes Cluster
  hosts: k8s_all
  become: true
  vars:
    kubernetes_version: "1.29"
    container_runtime: "containerd"
    pod_network_cidr: "10.244.0.0/16"
    service_cidr: "10.96.0.0/12"
    
  pre_tasks:
    - name: Disable swap
      command: swapoff -a
      changed_when: false
      
    - name: Remove swap from fstab
      lineinfile:
        path: /etc/fstab
        regexp: '.*swap.*'
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
        dest: /etc/modules-load.d/k8s.conf
        content: |
          overlay
          br_netfilter
          
    - name: Set sysctl parameters
      sysctl:
        name: "{{ item.key }}"
        value: "{{ item.value }}"
        sysctl_file: /etc/sysctl.d/k8s.conf
        reload: yes
      loop:
        - { key: 'net.bridge.bridge-nf-call-iptables', value: '1' }
        - { key: 'net.bridge.bridge-nf-call-ip6tables', value: '1' }
        - { key: 'net.ipv4.ip_forward', value: '1' }

  tasks:
    - name: Install containerd
      include_tasks: tasks/install_containerd.yml
      
    - name: Install Kubernetes packages
      include_tasks: tasks/install_k8s_packages.yml

# Role for master nodes
- name: Initialize Kubernetes Master
  hosts: k8s_masters
  become: true
  
  tasks:
    - name: Check if cluster is initialized
      stat:
        path: /etc/kubernetes/admin.conf
      register: k8s_init
      
    - name: Initialize the cluster
      command: >
        kubeadm init 
        --pod-network-cidr={{ pod_network_cidr }}
        --service-cidr={{ service_cidr }}
        --apiserver-advertise-address={{ ansible_host }}
      when: not k8s_init.stat.exists
      register: kubeadm_init
      
    - name: Create .kube directory
      file:
        path: "{{ ansible_env.HOME }}/.kube"
        state: directory
        mode: '0755'
        
    - name: Copy admin.conf
      copy:
        src: /etc/kubernetes/admin.conf
        dest: "{{ ansible_env.HOME }}/.kube/config"
        remote_src: yes
        mode: '0600'
        
    - name: Get join command
      command: kubeadm token create --print-join-command
      register: join_command
      changed_when: false
      
    - name: Store join command
      set_fact:
        kubeadm_join_command: "{{ join_command.stdout }}"
        
    - name: Install Calico CNI
      kubernetes.core.k8s:
        state: present
        src: https://docs.projectcalico.org/manifests/calico.yaml

# Join worker nodes
- name: Join Worker Nodes
  hosts: k8s_workers
  become: true
  
  tasks:
    - name: Check if node is joined
      stat:
        path: /etc/kubernetes/kubelet.conf
      register: kubelet_conf
      
    - name: Join cluster
      command: "{{ hostvars[groups['k8s_masters'][0]]['kubeadm_join_command'] }}"
      when: not kubelet_conf.stat.exists
```

---

## Part 2: Deploying Applications to Kubernetes

### Basic Deployment

```yaml
# playbooks/kubernetes/deploy_app.yml
---
- name: Deploy Application to Kubernetes
  hosts: localhost
  connection: local
  gather_facts: false
  
  vars:
    app_name: "myapp"
    app_namespace: "production"
    app_image: "myregistry/myapp:v1.2.3"
    app_replicas: 3
    app_port: 8080
    
  tasks:
    - name: Create namespace
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: v1
          kind: Namespace
          metadata:
            name: "{{ app_namespace }}"
            labels:
              environment: production
              managed-by: ansible
              
    - name: Create ConfigMap
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: v1
          kind: ConfigMap
          metadata:
            name: "{{ app_name }}-config"
            namespace: "{{ app_namespace }}"
          data:
            APP_ENV: "production"
            LOG_LEVEL: "info"
            DATABASE_HOST: "postgres.database.svc.cluster.local"
            
    - name: Create Secret
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: v1
          kind: Secret
          metadata:
            name: "{{ app_name }}-secrets"
            namespace: "{{ app_namespace }}"
          type: Opaque
          stringData:
            DATABASE_PASSWORD: "{{ vault_db_password }}"
            API_KEY: "{{ vault_api_key }}"
            
    - name: Create Deployment
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: apps/v1
          kind: Deployment
          metadata:
            name: "{{ app_name }}"
            namespace: "{{ app_namespace }}"
            labels:
              app: "{{ app_name }}"
          spec:
            replicas: "{{ app_replicas }}"
            selector:
              matchLabels:
                app: "{{ app_name }}"
            strategy:
              type: RollingUpdate
              rollingUpdate:
                maxSurge: 1
                maxUnavailable: 0
            template:
              metadata:
                labels:
                  app: "{{ app_name }}"
              spec:
                containers:
                  - name: "{{ app_name }}"
                    image: "{{ app_image }}"
                    ports:
                      - containerPort: "{{ app_port }}"
                    envFrom:
                      - configMapRef:
                          name: "{{ app_name }}-config"
                      - secretRef:
                          name: "{{ app_name }}-secrets"
                    resources:
                      requests:
                        memory: "256Mi"
                        cpu: "250m"
                      limits:
                        memory: "512Mi"
                        cpu: "500m"
                    livenessProbe:
                      httpGet:
                        path: /health
                        port: "{{ app_port }}"
                      initialDelaySeconds: 30
                      periodSeconds: 10
                    readinessProbe:
                      httpGet:
                        path: /ready
                        port: "{{ app_port }}"
                      initialDelaySeconds: 5
                      periodSeconds: 5
                      
    - name: Create Service
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: v1
          kind: Service
          metadata:
            name: "{{ app_name }}"
            namespace: "{{ app_namespace }}"
          spec:
            selector:
              app: "{{ app_name }}"
            ports:
              - protocol: TCP
                port: 80
                targetPort: "{{ app_port }}"
            type: ClusterIP
            
    - name: Create Ingress
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: networking.k8s.io/v1
          kind: Ingress
          metadata:
            name: "{{ app_name }}"
            namespace: "{{ app_namespace }}"
            annotations:
              kubernetes.io/ingress.class: nginx
              cert-manager.io/cluster-issuer: letsencrypt-prod
          spec:
            tls:
              - hosts:
                  - "{{ app_name }}.example.com"
                secretName: "{{ app_name }}-tls"
            rules:
              - host: "{{ app_name }}.example.com"
                http:
                  paths:
                    - path: /
                      pathType: Prefix
                      backend:
                        service:
                          name: "{{ app_name }}"
                          port:
                            number: 80
```

---

## Part 3: Helm Chart Deployment with Ansible

### Install Helm Charts

```yaml
# playbooks/kubernetes/deploy_helm.yml
---
- name: Deploy Applications via Helm
  hosts: localhost
  connection: local
  gather_facts: false
  
  vars:
    helm_charts:
      - name: nginx-ingress
        chart: ingress-nginx/ingress-nginx
        namespace: ingress-nginx
        create_namespace: true
        values:
          controller:
            replicaCount: 2
            service:
              type: LoadBalancer
              
      - name: prometheus
        chart: prometheus-community/prometheus
        namespace: monitoring
        create_namespace: true
        values:
          server:
            persistentVolume:
              size: 50Gi
          alertmanager:
            enabled: true
            
      - name: grafana
        chart: grafana/grafana
        namespace: monitoring
        values:
          adminPassword: "{{ vault_grafana_password }}"
          persistence:
            enabled: true
            size: 10Gi
            
  pre_tasks:
    - name: Add Helm repositories
      kubernetes.core.helm_repository:
        name: "{{ item.name }}"
        repo_url: "{{ item.url }}"
      loop:
        - { name: ingress-nginx, url: "https://kubernetes.github.io/ingress-nginx" }
        - { name: prometheus-community, url: "https://prometheus-community.github.io/helm-charts" }
        - { name: grafana, url: "https://grafana.github.io/helm-charts" }
        
  tasks:
    - name: Deploy Helm charts
      kubernetes.core.helm:
        name: "{{ item.name }}"
        chart_ref: "{{ item.chart }}"
        release_namespace: "{{ item.namespace }}"
        create_namespace: "{{ item.create_namespace | default(false) }}"
        values: "{{ item.values | default({}) }}"
        wait: true
        wait_timeout: 600s
      loop: "{{ helm_charts }}"
```

---

## Part 4: Managing Multiple Kubernetes Clusters

### Multi-Cluster Inventory

```yaml
# inventory/kubernetes_clusters.yml
---
all:
  children:
    k8s_clusters:
      children:
        production:
          hosts:
            prod_cluster:
              kubeconfig: "~/.kube/config-prod"
              context: "production"
              
        staging:
          hosts:
            staging_cluster:
              kubeconfig: "~/.kube/config-staging"
              context: "staging"
              
        development:
          hosts:
            dev_cluster:
              kubeconfig: "~/.kube/config-dev"
              context: "development"
              
  vars:
    ansible_connection: local
    ansible_python_interpreter: "{{ ansible_playbook_python }}"
```

### Multi-Cluster Deployment

```yaml
# playbooks/kubernetes/multi_cluster_deploy.yml
---
- name: Deploy to Multiple Kubernetes Clusters
  hosts: k8s_clusters
  gather_facts: false
  serial: 1  # Deploy one cluster at a time
  
  vars:
    app_name: "api-gateway"
    app_version: "2.1.0"
    
  environment:
    KUBECONFIG: "{{ kubeconfig }}"
    
  tasks:
    - name: Set cluster context
      command: kubectl config use-context {{ context }}
      changed_when: false
      
    - name: Verify cluster connectivity
      kubernetes.core.k8s_cluster_info:
      register: cluster_info
      
    - name: Display cluster info
      debug:
        msg: "Deploying to {{ context }} - K8s version {{ cluster_info.version.server.kubernetes.gitVersion }}"
        
    - name: Deploy application
      kubernetes.core.k8s:
        state: present
        template: templates/deployment.yml.j2
        
    - name: Wait for deployment rollout
      kubernetes.core.k8s_info:
        kind: Deployment
        name: "{{ app_name }}"
        namespace: "{{ app_namespace }}"
      register: deployment_status
      until: >-
        deployment_status.resources[0].status.readyReplicas | default(0) 
        == deployment_status.resources[0].spec.replicas
      retries: 30
      delay: 10
      
    - name: Verify deployment health
      uri:
        url: "https://{{ app_endpoint }}/health"
        validate_certs: "{{ validate_certs | default(true) }}"
      register: health_check
      failed_when: health_check.status != 200
```

---

## Part 5: Kubernetes Operators with Ansible

### What is an Ansible-based Operator?

Ansible Operators use Ansible playbooks to manage Kubernetes custom resources.

```
Custom Resource Change → Operator SDK → Ansible Playbook → K8s API
```

### Creating an Operator

```bash
# Install Operator SDK
curl -LO https://github.com/operator-framework/operator-sdk/releases/latest/download/operator-sdk_linux_amd64
chmod +x operator-sdk_linux_amd64
sudo mv operator-sdk_linux_amd64 /usr/local/bin/operator-sdk

# Initialize Ansible Operator
operator-sdk init --plugins=ansible --domain=example.com

# Create API
operator-sdk create api --group=app --version=v1 --kind=MyApp --generate-role
```

### Operator Role

```yaml
# roles/myapp/tasks/main.yml
---
- name: Create MyApp Deployment
  kubernetes.core.k8s:
    state: "{{ state }}"
    definition:
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: "{{ ansible_operator_meta.name }}"
        namespace: "{{ ansible_operator_meta.namespace }}"
      spec:
        replicas: "{{ replicas | default(1) }}"
        selector:
          matchLabels:
            app: "{{ ansible_operator_meta.name }}"
        template:
          metadata:
            labels:
              app: "{{ ansible_operator_meta.name }}"
          spec:
            containers:
              - name: app
                image: "{{ image }}"
                ports:
                  - containerPort: "{{ port | default(8080) }}"
                resources:
                  requests:
                    memory: "{{ memory_request | default('128Mi') }}"
                    cpu: "{{ cpu_request | default('100m') }}"
                    
- name: Create MyApp Service
  kubernetes.core.k8s:
    state: "{{ state }}"
    definition:
      apiVersion: v1
      kind: Service
      metadata:
        name: "{{ ansible_operator_meta.name }}"
        namespace: "{{ ansible_operator_meta.namespace }}"
      spec:
        selector:
          app: "{{ ansible_operator_meta.name }}"
        ports:
          - port: 80
            targetPort: "{{ port | default(8080) }}"
```

### Custom Resource Definition

```yaml
# config/samples/app_v1_myapp.yaml
apiVersion: app.example.com/v1
kind: MyApp
metadata:
  name: myapp-sample
spec:
  replicas: 3
  image: nginx:latest
  port: 80
  memory_request: "256Mi"
  cpu_request: "200m"
```

---

## Part 6: Real-World Kubernetes Use Cases

### Use Case 1: Blue-Green Deployment

```yaml
# playbooks/kubernetes/blue_green_deployment.yml
---
- name: Blue-Green Deployment
  hosts: localhost
  connection: local
  gather_facts: false
  
  vars:
    app_name: "webapp"
    namespace: "production"
    new_version: "v2.0.0"
    old_version: "v1.9.0"
    
  tasks:
    - name: Determine current active deployment
      kubernetes.core.k8s_info:
        kind: Service
        name: "{{ app_name }}"
        namespace: "{{ namespace }}"
      register: current_service
      
    - name: Set deployment colors
      set_fact:
        current_color: "{{ current_service.resources[0].spec.selector.color | default('blue') }}"
        new_color: "{{ 'green' if current_service.resources[0].spec.selector.color | default('blue') == 'blue' else 'blue' }}"
        
    - name: Deploy new version ({{ new_color }})
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: apps/v1
          kind: Deployment
          metadata:
            name: "{{ app_name }}-{{ new_color }}"
            namespace: "{{ namespace }}"
            labels:
              app: "{{ app_name }}"
              color: "{{ new_color }}"
              version: "{{ new_version }}"
          spec:
            replicas: 3
            selector:
              matchLabels:
                app: "{{ app_name }}"
                color: "{{ new_color }}"
            template:
              metadata:
                labels:
                  app: "{{ app_name }}"
                  color: "{{ new_color }}"
              spec:
                containers:
                  - name: app
                    image: "myregistry/{{ app_name }}:{{ new_version }}"
                    ports:
                      - containerPort: 8080
                      
    - name: Wait for new deployment to be ready
      kubernetes.core.k8s_info:
        kind: Deployment
        name: "{{ app_name }}-{{ new_color }}"
        namespace: "{{ namespace }}"
      register: new_deployment
      until: >-
        new_deployment.resources[0].status.readyReplicas | default(0) 
        == new_deployment.resources[0].spec.replicas
      retries: 60
      delay: 10
      
    - name: Run smoke tests on new deployment
      uri:
        url: "http://{{ app_name }}-{{ new_color }}.{{ namespace }}.svc.cluster.local/health"
      register: smoke_test
      retries: 5
      delay: 5
      until: smoke_test.status == 200
      
    - name: Switch traffic to new deployment
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: v1
          kind: Service
          metadata:
            name: "{{ app_name }}"
            namespace: "{{ namespace }}"
          spec:
            selector:
              app: "{{ app_name }}"
              color: "{{ new_color }}"
            ports:
              - port: 80
                targetPort: 8080
                
    - name: Wait for traffic switch verification
      pause:
        seconds: 30
        prompt: "Verifying traffic switch... Check application health"
        
    - name: Remove old deployment
      kubernetes.core.k8s:
        state: absent
        kind: Deployment
        name: "{{ app_name }}-{{ current_color }}"
        namespace: "{{ namespace }}"
      when: cleanup_old | default(true)
```

### Use Case 2: Canary Deployment

```yaml
# playbooks/kubernetes/canary_deployment.yml
---
- name: Canary Deployment with Traffic Splitting
  hosts: localhost
  connection: local
  gather_facts: false
  
  vars:
    app_name: "api"
    namespace: "production"
    canary_weight: 10  # 10% traffic to canary
    stable_weight: 90  # 90% traffic to stable
    
  tasks:
    - name: Deploy canary version
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: apps/v1
          kind: Deployment
          metadata:
            name: "{{ app_name }}-canary"
            namespace: "{{ namespace }}"
            labels:
              app: "{{ app_name }}"
              track: canary
          spec:
            replicas: 1
            selector:
              matchLabels:
                app: "{{ app_name }}"
                track: canary
            template:
              metadata:
                labels:
                  app: "{{ app_name }}"
                  track: canary
              spec:
                containers:
                  - name: app
                    image: "myregistry/{{ app_name }}:{{ canary_version }}"
                    
    - name: Configure Istio VirtualService for traffic splitting
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: networking.istio.io/v1beta1
          kind: VirtualService
          metadata:
            name: "{{ app_name }}"
            namespace: "{{ namespace }}"
          spec:
            hosts:
              - "{{ app_name }}"
            http:
              - route:
                  - destination:
                      host: "{{ app_name }}"
                      subset: stable
                    weight: "{{ stable_weight }}"
                  - destination:
                      host: "{{ app_name }}"
                      subset: canary
                    weight: "{{ canary_weight }}"
                    
    - name: Monitor canary metrics
      block:
        - name: Wait and gather metrics
          pause:
            minutes: 5
            prompt: "Monitoring canary deployment..."
            
        - name: Check error rate
          uri:
            url: "http://prometheus:9090/api/v1/query"
            method: GET
            body_format: json
            body:
              query: 'sum(rate(http_requests_total{app="{{ app_name }}",track="canary",status=~"5.."}[5m])) / sum(rate(http_requests_total{app="{{ app_name }}",track="canary"}[5m]))'
          register: error_rate
          
        - name: Evaluate canary success
          set_fact:
            canary_success: "{{ error_rate.json.data.result[0].value[1] | float < 0.01 }}"
            
      rescue:
        - name: Canary failed - rollback
          set_fact:
            canary_success: false
            
    - name: Promote or rollback
      include_tasks: "{{ 'promote_canary.yml' if canary_success else 'rollback_canary.yml' }}"
```

### Use Case 3: GitOps with ArgoCD

```yaml
# playbooks/kubernetes/setup_argocd.yml
---
- name: Setup ArgoCD for GitOps
  hosts: localhost
  connection: local
  gather_facts: false
  
  tasks:
    - name: Create ArgoCD namespace
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: v1
          kind: Namespace
          metadata:
            name: argocd
            
    - name: Install ArgoCD
      kubernetes.core.k8s:
        state: present
        src: https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
        namespace: argocd
        
    - name: Wait for ArgoCD to be ready
      kubernetes.core.k8s_info:
        kind: Deployment
        namespace: argocd
        name: argocd-server
      register: argocd_server
      until: >-
        argocd_server.resources[0].status.readyReplicas | default(0) >= 1
      retries: 30
      delay: 10
      
    - name: Create ArgoCD Application
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: argoproj.io/v1alpha1
          kind: Application
          metadata:
            name: "{{ app_name }}"
            namespace: argocd
          spec:
            project: default
            source:
              repoURL: "{{ git_repo_url }}"
              targetRevision: HEAD
              path: "kubernetes/{{ app_name }}"
            destination:
              server: https://kubernetes.default.svc
              namespace: "{{ app_namespace }}"
            syncPolicy:
              automated:
                prune: true
                selfHeal: true
              syncOptions:
                - CreateNamespace=true
```

---

## Part 7: Kubernetes Troubleshooting with Ansible

### Automated Diagnostics

```yaml
# playbooks/kubernetes/diagnose_cluster.yml
---
- name: Kubernetes Cluster Diagnostics
  hosts: localhost
  connection: local
  gather_facts: false
  
  tasks:
    - name: Get cluster info
      kubernetes.core.k8s_cluster_info:
      register: cluster_info
      
    - name: Check node status
      kubernetes.core.k8s_info:
        kind: Node
      register: nodes
      
    - name: Identify unhealthy nodes
      set_fact:
        unhealthy_nodes: >-
          {{ nodes.resources | selectattr('status.conditions', 'defined') |
             map(attribute='metadata.name') | list }}
             
    - name: Get pod status across namespaces
      kubernetes.core.k8s_info:
        kind: Pod
        field_selectors:
          - status.phase!=Running
          - status.phase!=Succeeded
      register: problem_pods
      
    - name: List problem pods
      debug:
        msg: |
          Problem pods found:
          {% for pod in problem_pods.resources %}
          - {{ pod.metadata.namespace }}/{{ pod.metadata.name }}: {{ pod.status.phase }}
          {% endfor %}
          
    - name: Check persistent volume claims
      kubernetes.core.k8s_info:
        kind: PersistentVolumeClaim
      register: pvcs
      
    - name: Identify pending PVCs
      debug:
        msg: "Pending PVC: {{ item.metadata.namespace }}/{{ item.metadata.name }}"
      loop: "{{ pvcs.resources }}"
      when: item.status.phase == 'Pending'
      
    - name: Generate diagnostic report
      template:
        src: templates/diagnostic_report.md.j2
        dest: "/tmp/k8s_diagnostic_{{ ansible_date_time.date }}.md"
```

---

## Best Practices

### 1. Use Kubernetes Module Best Practices

```yaml
# Good: Use wait and state tracking
- name: Deploy with verification
  kubernetes.core.k8s:
    state: present
    definition: "{{ lookup('template', 'deployment.yml.j2') }}"
    wait: true
    wait_timeout: 300
    wait_condition:
      type: Available
      status: "True"

# Good: Use k8s_info for status checks
- name: Verify deployment
  kubernetes.core.k8s_info:
    kind: Deployment
    name: myapp
    namespace: production
  register: deployment
  until: deployment.resources[0].status.availableReplicas == deployment.resources[0].spec.replicas
  retries: 30
  delay: 10
```

### 2. Secret Management

```yaml
# Use Vault for sensitive data
- name: Create secret from Vault
  kubernetes.core.k8s:
    state: present
    definition:
      apiVersion: v1
      kind: Secret
      metadata:
        name: app-secrets
        namespace: production
      type: Opaque
      stringData:
        DB_PASSWORD: "{{ lookup('hashi_vault', 'secret=secret/data/app:db_password') }}"
```

### 3. Idempotent Operations

```yaml
# Always check before creating
- name: Check if resource exists
  kubernetes.core.k8s_info:
    kind: ConfigMap
    name: app-config
    namespace: default
  register: existing_config

- name: Create or update config
  kubernetes.core.k8s:
    state: present
    definition: "{{ config_definition }}"
  when: existing_config.resources | length == 0 or force_update | default(false)
```

---

## Summary

| Feature | Ansible Strength |
|---------|------------------|
| Cluster Setup | Full automation from bare metal |
| Multi-cluster | Consistent management across environments |
| Hybrid | Mix K8s and traditional deployments |
| Operators | Ansible-based custom controllers |
| CI/CD | Integration with existing pipelines |
| Compliance | Audit-ready automation |

---

*Next: [17_Custom_Modules](../17_Custom_Modules/README.md) - Creating custom Ansible modules*
