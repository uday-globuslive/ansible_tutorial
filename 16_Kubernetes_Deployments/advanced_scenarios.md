# Advanced Kubernetes Scenarios with Ansible

## Production-Ready Deployments

---

## Scenario 1: StatefulSet Deployment with PVC

Deploy stateful applications like databases with persistent storage.

```yaml
# playbooks/kubernetes/statefulset_deployment.yml
---
- name: Deploy StatefulSet Application
  hosts: localhost
  connection: local
  gather_facts: false
  
  vars:
    app_name: "mongodb"
    namespace: "databases"
    replicas: 3
    storage_class: "gp2"
    storage_size: "100Gi"
    
  tasks:
    - name: Create namespace
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: v1
          kind: Namespace
          metadata:
            name: "{{ namespace }}"
            
    - name: Create headless service
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: v1
          kind: Service
          metadata:
            name: "{{ app_name }}-headless"
            namespace: "{{ namespace }}"
            labels:
              app: "{{ app_name }}"
          spec:
            clusterIP: None
            selector:
              app: "{{ app_name }}"
            ports:
              - port: 27017
                name: mongodb
                
    - name: Create StatefulSet
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: apps/v1
          kind: StatefulSet
          metadata:
            name: "{{ app_name }}"
            namespace: "{{ namespace }}"
          spec:
            serviceName: "{{ app_name }}-headless"
            replicas: "{{ replicas }}"
            selector:
              matchLabels:
                app: "{{ app_name }}"
            template:
              metadata:
                labels:
                  app: "{{ app_name }}"
              spec:
                terminationGracePeriodSeconds: 30
                containers:
                  - name: mongodb
                    image: mongo:6.0
                    ports:
                      - containerPort: 27017
                    env:
                      - name: MONGO_INITDB_ROOT_USERNAME
                        valueFrom:
                          secretKeyRef:
                            name: "{{ app_name }}-secret"
                            key: username
                      - name: MONGO_INITDB_ROOT_PASSWORD
                        valueFrom:
                          secretKeyRef:
                            name: "{{ app_name }}-secret"
                            key: password
                    volumeMounts:
                      - name: data
                        mountPath: /data/db
                    resources:
                      requests:
                        memory: "1Gi"
                        cpu: "500m"
                      limits:
                        memory: "2Gi"
                        cpu: "1000m"
                    livenessProbe:
                      exec:
                        command:
                          - mongosh
                          - --eval
                          - "db.adminCommand('ping')"
                      initialDelaySeconds: 30
                      periodSeconds: 10
                    readinessProbe:
                      exec:
                        command:
                          - mongosh
                          - --eval
                          - "db.adminCommand('ping')"
                      initialDelaySeconds: 5
                      periodSeconds: 5
            volumeClaimTemplates:
              - metadata:
                  name: data
                spec:
                  accessModes: ["ReadWriteOnce"]
                  storageClassName: "{{ storage_class }}"
                  resources:
                    requests:
                      storage: "{{ storage_size }}"
                      
    - name: Wait for StatefulSet pods
      kubernetes.core.k8s_info:
        kind: StatefulSet
        name: "{{ app_name }}"
        namespace: "{{ namespace }}"
      register: statefulset
      until: >-
        statefulset.resources[0].status.readyReplicas | default(0) 
        == replicas
      retries: 60
      delay: 10
```

---

## Scenario 2: CronJob for Scheduled Tasks

```yaml
# playbooks/kubernetes/cronjob_deployment.yml
---
- name: Deploy CronJob for Database Backup
  hosts: localhost
  connection: local
  gather_facts: false
  
  vars:
    job_name: "db-backup"
    namespace: "databases"
    schedule: "0 2 * * *"  # Daily at 2 AM
    backup_bucket: "s3://company-backups/postgres"
    
  tasks:
    - name: Create ServiceAccount
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: v1
          kind: ServiceAccount
          metadata:
            name: "{{ job_name }}-sa"
            namespace: "{{ namespace }}"
            annotations:
              eks.amazonaws.com/role-arn: "arn:aws:iam::123456789:role/BackupRole"
              
    - name: Create CronJob
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: batch/v1
          kind: CronJob
          metadata:
            name: "{{ job_name }}"
            namespace: "{{ namespace }}"
          spec:
            schedule: "{{ schedule }}"
            concurrencyPolicy: Forbid
            successfulJobsHistoryLimit: 3
            failedJobsHistoryLimit: 3
            jobTemplate:
              spec:
                backoffLimit: 2
                activeDeadlineSeconds: 3600
                template:
                  spec:
                    serviceAccountName: "{{ job_name }}-sa"
                    restartPolicy: OnFailure
                    containers:
                      - name: backup
                        image: postgres:15
                        command:
                          - /bin/bash
                          - -c
                          - |
                            set -e
                            TIMESTAMP=$(date +%Y%m%d_%H%M%S)
                            BACKUP_FILE="/tmp/backup_${TIMESTAMP}.sql.gz"
                            
                            echo "Starting backup at $(date)"
                            pg_dump -h ${DB_HOST} -U ${DB_USER} ${DB_NAME} | gzip > ${BACKUP_FILE}
                            
                            echo "Uploading to S3..."
                            aws s3 cp ${BACKUP_FILE} {{ backup_bucket }}/
                            
                            echo "Backup completed successfully"
                        env:
                          - name: DB_HOST
                            value: "postgres-primary.databases.svc.cluster.local"
                          - name: DB_NAME
                            value: "production"
                          - name: DB_USER
                            valueFrom:
                              secretKeyRef:
                                name: postgres-credentials
                                key: username
                          - name: PGPASSWORD
                            valueFrom:
                              secretKeyRef:
                                name: postgres-credentials
                                key: password
                        resources:
                          requests:
                            memory: "256Mi"
                            cpu: "100m"
                          limits:
                            memory: "512Mi"
                            cpu: "500m"
```

---

## Scenario 3: Network Policies

```yaml
# playbooks/kubernetes/network_policies.yml
---
- name: Apply Network Policies
  hosts: localhost
  connection: local
  gather_facts: false
  
  vars:
    namespace: "production"
    
  tasks:
    - name: Default deny all ingress
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: networking.k8s.io/v1
          kind: NetworkPolicy
          metadata:
            name: default-deny-ingress
            namespace: "{{ namespace }}"
          spec:
            podSelector: {}
            policyTypes:
              - Ingress
              
    - name: Allow traffic from ingress controller
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: networking.k8s.io/v1
          kind: NetworkPolicy
          metadata:
            name: allow-ingress-controller
            namespace: "{{ namespace }}"
          spec:
            podSelector:
              matchLabels:
                app: webapp
            policyTypes:
              - Ingress
            ingress:
              - from:
                  - namespaceSelector:
                      matchLabels:
                        name: ingress-nginx
                    podSelector:
                      matchLabels:
                        app.kubernetes.io/name: ingress-nginx
                ports:
                  - protocol: TCP
                    port: 8080
                    
    - name: Allow webapp to database
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: networking.k8s.io/v1
          kind: NetworkPolicy
          metadata:
            name: webapp-to-database
            namespace: "{{ namespace }}"
          spec:
            podSelector:
              matchLabels:
                app: postgres
            policyTypes:
              - Ingress
            ingress:
              - from:
                  - podSelector:
                      matchLabels:
                        app: webapp
                ports:
                  - protocol: TCP
                    port: 5432
                    
    - name: Allow monitoring
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: networking.k8s.io/v1
          kind: NetworkPolicy
          metadata:
            name: allow-prometheus
            namespace: "{{ namespace }}"
          spec:
            podSelector: {}
            policyTypes:
              - Ingress
            ingress:
              - from:
                  - namespaceSelector:
                      matchLabels:
                        name: monitoring
                    podSelector:
                      matchLabels:
                        app: prometheus
                ports:
                  - protocol: TCP
                    port: 9090
```

---

## Scenario 4: Horizontal Pod Autoscaler

```yaml
# playbooks/kubernetes/hpa_deployment.yml
---
- name: Deploy Application with HPA
  hosts: localhost
  connection: local
  gather_facts: false
  
  vars:
    app_name: "api-service"
    namespace: "production"
    min_replicas: 3
    max_replicas: 20
    target_cpu: 70
    target_memory: 80
    
  tasks:
    - name: Deploy application
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: apps/v1
          kind: Deployment
          metadata:
            name: "{{ app_name }}"
            namespace: "{{ namespace }}"
          spec:
            replicas: "{{ min_replicas }}"
            selector:
              matchLabels:
                app: "{{ app_name }}"
            template:
              metadata:
                labels:
                  app: "{{ app_name }}"
              spec:
                containers:
                  - name: api
                    image: "myregistry/{{ app_name }}:latest"
                    ports:
                      - containerPort: 8080
                    resources:
                      requests:
                        memory: "256Mi"
                        cpu: "250m"
                      limits:
                        memory: "512Mi"
                        cpu: "500m"
                        
    - name: Create HPA
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: autoscaling/v2
          kind: HorizontalPodAutoscaler
          metadata:
            name: "{{ app_name }}-hpa"
            namespace: "{{ namespace }}"
          spec:
            scaleTargetRef:
              apiVersion: apps/v1
              kind: Deployment
              name: "{{ app_name }}"
            minReplicas: "{{ min_replicas }}"
            maxReplicas: "{{ max_replicas }}"
            metrics:
              - type: Resource
                resource:
                  name: cpu
                  target:
                    type: Utilization
                    averageUtilization: "{{ target_cpu }}"
              - type: Resource
                resource:
                  name: memory
                  target:
                    type: Utilization
                    averageUtilization: "{{ target_memory }}"
            behavior:
              scaleDown:
                stabilizationWindowSeconds: 300
                policies:
                  - type: Percent
                    value: 10
                    periodSeconds: 60
              scaleUp:
                stabilizationWindowSeconds: 0
                policies:
                  - type: Percent
                    value: 100
                    periodSeconds: 15
                  - type: Pods
                    value: 4
                    periodSeconds: 15
                selectPolicy: Max
```

---

## Scenario 5: Service Mesh (Istio) Configuration

```yaml
# playbooks/kubernetes/istio_configuration.yml
---
- name: Configure Istio Service Mesh
  hosts: localhost
  connection: local
  gather_facts: false
  
  vars:
    namespace: "production"
    app_name: "webapp"
    
  tasks:
    - name: Enable Istio injection
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: v1
          kind: Namespace
          metadata:
            name: "{{ namespace }}"
            labels:
              istio-injection: enabled
              
    - name: Create DestinationRule
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: networking.istio.io/v1beta1
          kind: DestinationRule
          metadata:
            name: "{{ app_name }}"
            namespace: "{{ namespace }}"
          spec:
            host: "{{ app_name }}"
            trafficPolicy:
              connectionPool:
                tcp:
                  maxConnections: 100
                http:
                  h2UpgradePolicy: UPGRADE
                  http1MaxPendingRequests: 100
                  http2MaxRequests: 1000
              loadBalancer:
                simple: LEAST_CONN
              outlierDetection:
                consecutive5xxErrors: 5
                interval: 30s
                baseEjectionTime: 30s
                maxEjectionPercent: 50
            subsets:
              - name: stable
                labels:
                  version: stable
              - name: canary
                labels:
                  version: canary
                  
    - name: Create VirtualService with retry
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
              - "{{ app_name }}.example.com"
            gateways:
              - mesh
              - istio-system/main-gateway
            http:
              - match:
                  - headers:
                      x-canary:
                        exact: "true"
                route:
                  - destination:
                      host: "{{ app_name }}"
                      subset: canary
              - route:
                  - destination:
                      host: "{{ app_name }}"
                      subset: stable
                    weight: 95
                  - destination:
                      host: "{{ app_name }}"
                      subset: canary
                    weight: 5
                retries:
                  attempts: 3
                  perTryTimeout: 2s
                  retryOn: connect-failure,refused-stream,unavailable,cancelled,retriable-4xx
                timeout: 10s
                fault:
                  delay:
                    percentage:
                      value: 0.1
                    fixedDelay: 5s
                    
    - name: Create Gateway
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: networking.istio.io/v1beta1
          kind: Gateway
          metadata:
            name: main-gateway
            namespace: istio-system
          spec:
            selector:
              istio: ingressgateway
            servers:
              - port:
                  number: 443
                  name: https
                  protocol: HTTPS
                tls:
                  mode: SIMPLE
                  credentialName: webapp-tls
                hosts:
                  - "{{ app_name }}.example.com"
```

---

## Scenario 6: Pod Disruption Budget

```yaml
# playbooks/kubernetes/pdb_deployment.yml
---
- name: Configure Pod Disruption Budget
  hosts: localhost
  connection: local
  gather_facts: false
  
  vars:
    apps:
      - name: webapp
        namespace: production
        min_available: 2
      - name: api
        namespace: production
        max_unavailable: 1
      - name: worker
        namespace: production
        min_available: "50%"
        
  tasks:
    - name: Create PDB for applications
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: policy/v1
          kind: PodDisruptionBudget
          metadata:
            name: "{{ item.name }}-pdb"
            namespace: "{{ item.namespace }}"
          spec:
            selector:
              matchLabels:
                app: "{{ item.name }}"
            minAvailable: "{{ item.min_available | default(omit) }}"
            maxUnavailable: "{{ item.max_unavailable | default(omit) }}"
      loop: "{{ apps }}"
      when: item.min_available is defined or item.max_unavailable is defined
```

---

## Scenario 7: ConfigMap and Secret Management

```yaml
# playbooks/kubernetes/config_management.yml
---
- name: Manage Application Configuration
  hosts: localhost
  connection: local
  gather_facts: false
  
  vars:
    namespace: "production"
    app_name: "webapp"
    environment: "production"
    
  tasks:
    - name: Load configuration from file
      set_fact:
        app_config: "{{ lookup('file', 'configs/{{ environment }}.json') | from_json }}"
        
    - name: Create ConfigMap from template
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: v1
          kind: ConfigMap
          metadata:
            name: "{{ app_name }}-config"
            namespace: "{{ namespace }}"
            labels:
              app: "{{ app_name }}"
              environment: "{{ environment }}"
          data:
            config.json: |
              {{ app_config | to_nice_json }}
            APP_ENV: "{{ environment }}"
            LOG_LEVEL: "{{ app_config.log_level | default('info') }}"
            FEATURE_FLAGS: "{{ app_config.features | to_json }}"
            
    - name: Create Secret from Vault
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: v1
          kind: Secret
          metadata:
            name: "{{ app_name }}-secrets"
            namespace: "{{ namespace }}"
          type: Opaque
          stringData:
            DATABASE_URL: "postgresql://{{ vault_db_user }}:{{ vault_db_pass }}@postgres:5432/{{ environment }}"
            REDIS_URL: "redis://:{{ vault_redis_pass }}@redis:6379/0"
            API_KEY: "{{ vault_api_key }}"
            JWT_SECRET: "{{ vault_jwt_secret }}"
            
    - name: Create TLS Secret
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: v1
          kind: Secret
          metadata:
            name: "{{ app_name }}-tls"
            namespace: "{{ namespace }}"
          type: kubernetes.io/tls
          data:
            tls.crt: "{{ lookup('file', 'certs/{{ environment }}/tls.crt') | b64encode }}"
            tls.key: "{{ lookup('file', 'certs/{{ environment }}/tls.key') | b64encode }}"
            
    - name: Trigger rolling restart on config change
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: apps/v1
          kind: Deployment
          metadata:
            name: "{{ app_name }}"
            namespace: "{{ namespace }}"
          spec:
            template:
              metadata:
                annotations:
                  configmap-hash: "{{ app_config | to_json | hash('sha256') }}"
```

---

## Scenario 8: Multi-Cluster Sync

```yaml
# playbooks/kubernetes/multi_cluster_sync.yml
---
- name: Sync Resources Across Clusters
  hosts: localhost
  connection: local
  gather_facts: false
  
  vars:
    clusters:
      - name: us-east-1
        kubeconfig: "~/.kube/config-us-east"
        context: "us-east-1-prod"
      - name: eu-west-1
        kubeconfig: "~/.kube/config-eu-west"
        context: "eu-west-1-prod"
      - name: ap-southeast-1
        kubeconfig: "~/.kube/config-ap-southeast"
        context: "ap-southeast-1-prod"
        
    sync_namespaces:
      - production
      - staging
      
    sync_resources:
      - kind: ConfigMap
        names: ["global-config", "feature-flags"]
      - kind: Secret
        names: ["shared-secrets"]
        
  tasks:
    - name: Get resources from primary cluster
      kubernetes.core.k8s_info:
        kind: "{{ item.0.kind }}"
        name: "{{ item.1 }}"
        namespace: "{{ item.2 }}"
        kubeconfig: "{{ clusters[0].kubeconfig }}"
        context: "{{ clusters[0].context }}"
      loop: "{{ sync_resources | subelements('names') | product(sync_namespaces) | list }}"
      register: source_resources
      
    - name: Sync to other clusters
      kubernetes.core.k8s:
        state: present
        definition: "{{ item.0.resources[0] | combine({'metadata': {'resourceVersion': omit}}, recursive=True) }}"
        kubeconfig: "{{ item.1.kubeconfig }}"
        context: "{{ item.1.context }}"
      loop: "{{ source_resources.results | product(clusters[1:]) | list }}"
      when: item.0.resources | length > 0
```

---

## Complete Workflow Example

```yaml
# playbooks/kubernetes/full_deployment_workflow.yml
---
- name: Complete Application Deployment Workflow
  hosts: localhost
  connection: local
  gather_facts: false
  
  vars:
    app_name: "myapp"
    version: "{{ deploy_version | default('latest') }}"
    namespace: "production"
    
  pre_tasks:
    - name: Validate deployment prerequisites
      block:
        - name: Check cluster connectivity
          kubernetes.core.k8s_cluster_info:
          register: cluster_info
          
        - name: Verify namespace exists
          kubernetes.core.k8s_info:
            kind: Namespace
            name: "{{ namespace }}"
          register: ns_check
          failed_when: ns_check.resources | length == 0
          
        - name: Check resource quotas
          kubernetes.core.k8s_info:
            kind: ResourceQuota
            namespace: "{{ namespace }}"
          register: quotas
          
  tasks:
    - name: Create/Update ConfigMaps
      include_tasks: tasks/create_configmaps.yml
      
    - name: Create/Update Secrets
      include_tasks: tasks/create_secrets.yml
      
    - name: Deploy application
      kubernetes.core.k8s:
        state: present
        template: templates/deployment.yml.j2
        wait: true
        wait_timeout: 300
      register: deployment
      
    - name: Create Service
      kubernetes.core.k8s:
        state: present
        template: templates/service.yml.j2
        
    - name: Configure Ingress
      kubernetes.core.k8s:
        state: present
        template: templates/ingress.yml.j2
        
    - name: Setup HPA
      kubernetes.core.k8s:
        state: present
        template: templates/hpa.yml.j2
        
    - name: Apply Network Policies
      kubernetes.core.k8s:
        state: present
        template: templates/network-policy.yml.j2
        
  post_tasks:
    - name: Run smoke tests
      include_tasks: tasks/smoke_tests.yml
      
    - name: Send deployment notification
      uri:
        url: "{{ slack_webhook }}"
        method: POST
        body_format: json
        body:
          text: "Deployed {{ app_name }}:{{ version }} to {{ namespace }}"
      when: slack_webhook is defined
      
  handlers:
    - name: Rollback deployment
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: apps/v1
          kind: Deployment
          metadata:
            name: "{{ app_name }}"
            namespace: "{{ namespace }}"
          spec:
            template:
              spec:
                containers:
                  - name: "{{ app_name }}"
                    image: "{{ previous_image }}"
```

---

## Kubernetes + Ansible: Decision Matrix

| Scenario | Use Ansible | Use kubectl/Helm | Use Both |
|----------|-------------|------------------|----------|
| Cluster provisioning | ✅ | ❌ | ✅ |
| Single app deployment | ❌ | ✅ | ❌ |
| Multi-cluster sync | ✅ | ❌ | ✅ |
| CI/CD pipeline | ❌ | ✅ | ✅ |
| Complex orchestration | ✅ | ❌ | ✅ |
| GitOps | ❌ | ✅ (ArgoCD) | ✅ |
| Hybrid infrastructure | ✅ | ❌ | ✅ |
| Secret rotation | ✅ | ❌ | ✅ |

---

*Previous: [16_Kubernetes_Deployments](../16_Kubernetes_Deployments/README.md)*  
*Next: [17_Custom_Modules](../17_Custom_Modules/README.md)*
