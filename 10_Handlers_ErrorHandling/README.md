# Ansible Handlers and Error Handling

## What are Handlers?

**Handlers** are special tasks that run only when triggered by a `notify` directive. They're typically used for service restarts, cache clearing, or any action that should happen once after multiple changes.

```
Key Points:
- Handlers run at the END of a play
- They run only ONCE even if notified multiple times
- They run in the ORDER defined, not notification order
```

---

## Basic Handler Usage

### Defining and Triggering Handlers

```yaml
---
- name: Configure Web Server
  hosts: webservers
  become: true
  
  tasks:
    - name: Update nginx.conf
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: Restart nginx        # Trigger handler
    
    - name: Update site config
      template:
        src: site.conf.j2
        dest: /etc/nginx/sites-enabled/mysite
      notify: Restart nginx        # Same handler triggered again
  
  handlers:
    - name: Restart nginx          # Handler definition
      service:
        name: nginx
        state: restarted
```

**Result**: Even though `notify` is called twice, the handler runs only **once** at the end.

---

## Handler Execution Order

### Handlers Run at Play End

```yaml
tasks:
  - name: Task 1
    copy:
      src: file1.txt
      dest: /tmp/file1.txt
    notify: Handler A
  
  - name: Task 2             # This runs BEFORE Handler A
    copy:
      src: file2.txt
      dest: /tmp/file2.txt
    notify: Handler B

handlers:
  - name: Handler A          # Runs after ALL tasks
    debug:
      msg: "Handler A"
  
  - name: Handler B          # Runs after Handler A
    debug:
      msg: "Handler B"
```

### Force Handlers to Run Mid-Play

```yaml
tasks:
  - name: Update config
    template:
      src: config.j2
      dest: /etc/app/config.yml
    notify: Restart app
  
  - name: Flush handlers now
    meta: flush_handlers     # Run handlers here
  
  - name: Continue with app running
    uri:
      url: http://localhost:8080/health
```

---

## Multiple Handlers

### Notify Multiple Handlers

```yaml
tasks:
  - name: Update SSL certificate
    copy:
      src: server.crt
      dest: /etc/ssl/certs/server.crt
    notify:
      - Restart nginx
      - Restart haproxy
      - Clear cache

handlers:
  - name: Restart nginx
    service:
      name: nginx
      state: restarted
  
  - name: Restart haproxy
    service:
      name: haproxy
      state: restarted
  
  - name: Clear cache
    command: /usr/local/bin/clear_cache.sh
```

### Handler Chains

```yaml
handlers:
  - name: Restart nginx
    service:
      name: nginx
      state: restarted
    notify: Check nginx status    # Chain another handler
  
  - name: Check nginx status
    uri:
      url: http://localhost/health
      status_code: 200
    register: health_check
    retries: 5
    delay: 5
```

---

## Listen Directive

The `listen` directive allows multiple handlers to respond to the same event:

```yaml
tasks:
  - name: Update application config
    template:
      src: app.conf.j2
      dest: /etc/app/config.yml
    notify: "restart web services"   # Generic event name

handlers:
  - name: Restart nginx
    service:
      name: nginx
      state: restarted
    listen: "restart web services"   # Responds to event
  
  - name: Restart php-fpm
    service:
      name: php-fpm
      state: restarted
    listen: "restart web services"   # Also responds
  
  - name: Clear opcache
    command: /usr/bin/cachetool opcache:reset
    listen: "restart web services"   # Also responds
```

---

## Handlers with Conditions

```yaml
handlers:
  - name: Restart nginx
    service:
      name: nginx
      state: restarted
    when: nginx_installed | default(true)
  
  - name: Restart apache
    service:
      name: apache2
      state: restarted
    when: apache_installed | default(false)
```

---

## Handlers in Roles

### roles/nginx/handlers/main.yml

```yaml
---
- name: Restart nginx
  service:
    name: nginx
    state: restarted
  listen: "restart nginx"

- name: Reload nginx
  service:
    name: nginx
    state: reloaded
  listen: "reload nginx"

- name: Test nginx config
  command: nginx -t
  listen: "test nginx"
```

### Using Role Handlers

```yaml
# roles/nginx/tasks/main.yml
- name: Deploy nginx config
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  notify: 
    - test nginx
    - reload nginx
```

---

# Error Handling

## Ignoring Errors

```yaml
- name: This might fail
  command: /opt/might_not_exist.sh
  ignore_errors: yes
  register: result

- name: Continue regardless
  debug:
    msg: "Previous task status: {{ 'failed' if result.failed else 'succeeded' }}"
```

## Custom Failure Conditions

```yaml
- name: Run health check
  command: /opt/health_check.sh
  register: health
  failed_when: "'CRITICAL' in health.stdout"

- name: Check exit code
  command: /opt/app/status.sh
  register: status
  failed_when: status.rc not in [0, 2]  # Accept 0 or 2

- name: Complex failure condition
  command: /opt/check.sh
  register: check
  failed_when:
    - check.rc != 0
    - "'IGNORE' not in check.stdout"
```

## Custom Changed Conditions

```yaml
- name: Run migration
  command: /opt/app/migrate.sh
  register: migration
  changed_when: "'Migrated' in migration.stdout"

- name: Check status only
  command: systemctl is-active nginx
  register: nginx_status
  changed_when: false  # Never mark as changed
```

## Block Error Handling

```yaml
- name: Application deployment
  block:
    - name: Download application
      get_url:
        url: "{{ app_url }}"
        dest: /tmp/app.tar.gz
    
    - name: Extract application
      unarchive:
        src: /tmp/app.tar.gz
        dest: /opt/app
        remote_src: yes
    
    - name: Start application
      service:
        name: myapp
        state: started

  rescue:
    - name: Log failure
      debug:
        msg: "Deployment failed, initiating rollback"
    
    - name: Rollback to previous version
      command: /opt/scripts/rollback.sh
    
    - name: Send alert
      mail:
        to: "admin@example.com"
        subject: "Deployment Failed"
        body: "Deployment to {{ inventory_hostname }} failed"
      delegate_to: localhost

  always:
    - name: Cleanup temp files
      file:
        path: /tmp/app.tar.gz
        state: absent
    
    - name: Record deployment attempt
      lineinfile:
        path: /var/log/deployments.log
        line: "{{ ansible_date_time.iso8601 }} - Deployment {{ 'failed' if ansible_failed_task is defined else 'succeeded' }}"
```

## any_errors_fatal

Stop entire play if any host fails:

```yaml
- name: Critical deployment
  hosts: webservers
  any_errors_fatal: true  # Stop if ANY host fails
  
  tasks:
    - name: Deploy application
      copy:
        src: app.war
        dest: /opt/tomcat/webapps/
```

## max_fail_percentage

```yaml
- name: Rolling update
  hosts: webservers
  serial: 5
  max_fail_percentage: 30  # Fail if >30% of batch fails
  
  tasks:
    - name: Update application
      apt:
        name: myapp
        state: latest
```

## Assertions

```yaml
- name: Validate prerequisites
  assert:
    that:
      - ansible_memtotal_mb >= 2048
      - ansible_processor_cores >= 2
      - disk_space_mb >= 10240
    fail_msg: "Server does not meet minimum requirements"
    success_msg: "Server meets all requirements"

- name: Validate configuration
  assert:
    that:
      - database_host is defined
      - database_port | int > 0
      - database_name | length > 0
    fail_msg: "Invalid database configuration"
```

## Fail Module

```yaml
- name: Check environment
  fail:
    msg: "This playbook only runs in production!"
  when: environment != 'production'

- name: Validate input
  fail:
    msg: "Version must be specified with -e version=X.Y.Z"
  when: version is not defined
```

## Rescue with Re-raise

```yaml
- block:
    - name: Risky operation
      command: /opt/risky.sh
  
  rescue:
    - name: Log error
      debug:
        msg: "Error: {{ ansible_failed_result }}"
    
    - name: Re-raise the error
      fail:
        msg: "Failing after logging: {{ ansible_failed_result.msg }}"
```

---

## Retries

```yaml
- name: Wait for service
  uri:
    url: http://localhost:8080/health
    status_code: 200
  register: result
  until: result.status == 200
  retries: 30          # Number of retries
  delay: 10            # Seconds between retries

- name: Wait for file
  wait_for:
    path: /tmp/app_ready
    state: present
  retries: 5
  delay: 10
```

---

## Complete Error Handling Example

```yaml
---
- name: Production Deployment
  hosts: webservers
  become: true
  serial: 1
  max_fail_percentage: 0
  
  vars:
    deployment_id: "{{ ansible_date_time.epoch }}"
  
  pre_tasks:
    - name: Validate deployment
      assert:
        that:
          - version is defined
          - version is match('^[0-9]+\.[0-9]+\.[0-9]+$')
        fail_msg: "Invalid version format"
  
  tasks:
    - name: Deployment block
      block:
        - name: Remove from load balancer
          uri:
            url: "http://lb.example.com/api/drain/{{ inventory_hostname }}"
            method: POST
          delegate_to: localhost
        
        - name: Wait for connections to drain
          pause:
            seconds: 30
        
        - name: Backup current version
          archive:
            path: /opt/app
            dest: "/backup/app-{{ deployment_id }}.tar.gz"
        
        - name: Deploy new version
          unarchive:
            src: "https://releases.example.com/app-{{ version }}.tar.gz"
            dest: /opt/app
            remote_src: yes
          notify: Restart application
        
        - name: Flush handlers
          meta: flush_handlers
        
        - name: Health check
          uri:
            url: "http://localhost:8080/health"
            status_code: 200
          register: health
          until: health.status == 200
          retries: 30
          delay: 5
        
        - name: Add back to load balancer
          uri:
            url: "http://lb.example.com/api/enable/{{ inventory_hostname }}"
            method: POST
          delegate_to: localhost
      
      rescue:
        - name: Log deployment failure
          debug:
            msg: "Deployment failed: {{ ansible_failed_result.msg | default('Unknown error') }}"
        
        - name: Restore from backup
          unarchive:
            src: "/backup/app-{{ deployment_id }}.tar.gz"
            dest: /opt/app
            remote_src: yes
          ignore_errors: yes
          notify: Restart application
        
        - name: Notify on failure
          slack:
            token: "{{ slack_token }}"
            channel: "#deployments"
            msg: "❌ Deployment failed on {{ inventory_hostname }}"
          delegate_to: localhost
          ignore_errors: yes
        
        - name: Fail the play
          fail:
            msg: "Deployment failed, rolled back to previous version"
      
      always:
        - name: Cleanup temp files
          file:
            path: "/tmp/app-{{ version }}.tar.gz"
            state: absent
          ignore_errors: yes
  
  handlers:
    - name: Restart application
      service:
        name: myapp
        state: restarted
```

---

## Next Steps

Continue to **11_Vault** to learn about securing sensitive data with Ansible Vault.
