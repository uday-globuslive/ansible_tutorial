# Custom Ansible Module Development Guide

## Understanding Custom Modules (Beginner Explanation)

### When Would You Need a Custom Module?

Ansible has 3000+ modules, but sometimes you need one that doesn't exist:

```
┌───────────────────────────────────────────────────────┐
│  SHOULD I CREATE A CUSTOM MODULE?                    │
├───────────────────────────────────────────────────────┤
│                                                       │
│  1. Is there an existing module?                     │
│     └─ Search: ansible-doc -l | grep "keyword"       │
│     └─ Check Ansible Galaxy for community modules    │
│                                                       │
│  2. Can shell/command modules do it?                 │
│     └─ Sometimes a simple script is enough           │
│                                                       │
│  3. Is it complex enough to warrant a module?        │
│     └─ Need idempotency?                             │
│     └─ Need to share across teams?                   │
│     └─ Complex logic or state management?            │
│                                                       │
│  If YES to question 3 and NO to 1 and 2:             │
│  → Create a custom module!                           │
└───────────────────────────────────────────────────────┘
```

### How Custom Modules Work

```
┌────────────────┐      ┌────────────────┐      ┌────────────────┐
│   Playbook     │      │  Your Module   │      │  Managed Node  │
│                │      │  (Python)      │      │                │
│ - my_module:  │────▶│                │────▶│  Executes the  │
│     name: x   │      │ Receives args  │      │  actual work   │
│     state: y  │      │ Does the work  │      │                │
│               │◄────│ Returns JSON   │◄────│  Returns       │
│  Shows result │      │                │      │  result        │
└────────────────┘      └────────────────┘      └────────────────┘
```

## Why Create Custom Modules?

| Reason | Example |
|--------|---------|
| **Proprietary Systems** | Internal APIs, legacy systems |
| **Complex Logic** | Business rules too complex for playbooks |
| **Performance** | Operations needing optimization |
| **Reusability** | Share across teams/projects |
| **Idempotency** | Need fine-grained state control |
| **Error Handling** | Custom error messages and recovery |

---

## Part 1: Module Fundamentals

### Basic Module Structure

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

# Copyright: (c) 2024, Your Name <your.email@example.com>
# GNU General Public License v3.0+

DOCUMENTATION = r'''
---
module: my_custom_module
short_description: Does something useful
version_added: "1.0.0"
description:
    - This module performs a specific action.
    - It supports check mode.
options:
    name:
        description: The name of the resource.
        required: true
        type: str
    state:
        description: Desired state of the resource.
        choices: ['present', 'absent']
        default: present
        type: str
    options:
        description: Additional options.
        type: dict
        default: {}
author:
    - Your Name (@github_username)
'''

EXAMPLES = r'''
- name: Create a resource
  my_custom_module:
    name: myresource
    state: present
    
- name: Remove a resource
  my_custom_module:
    name: myresource
    state: absent
'''

RETURN = r'''
changed:
    description: Whether the module made changes.
    type: bool
    returned: always
message:
    description: Status message.
    type: str
    returned: always
data:
    description: Data returned from the operation.
    type: dict
    returned: success
'''

from ansible.module_utils.basic import AnsibleModule


def main():
    # Define module arguments
    module_args = dict(
        name=dict(type='str', required=True),
        state=dict(type='str', default='present', choices=['present', 'absent']),
        options=dict(type='dict', default={})
    )

    # Initialize module
    module = AnsibleModule(
        argument_spec=module_args,
        supports_check_mode=True
    )

    # Get parameters
    name = module.params['name']
    state = module.params['state']
    options = module.params['options']

    # Initialize result
    result = dict(
        changed=False,
        message='',
        data={}
    )

    # Check mode - report what would change
    if module.check_mode:
        result['changed'] = True
        result['message'] = f"Would {'create' if state == 'present' else 'remove'} {name}"
        module.exit_json(**result)

    # Actual logic
    try:
        if state == 'present':
            # Create or update resource
            result['changed'] = True
            result['message'] = f"Created {name}"
            result['data'] = {'name': name, 'status': 'created'}
        else:
            # Remove resource
            result['changed'] = True
            result['message'] = f"Removed {name}"

        module.exit_json(**result)

    except Exception as e:
        module.fail_json(msg=f"Error: {str(e)}", **result)


if __name__ == '__main__':
    main()
```

---

## Part 2: Real-World Module Use Cases

### Use Case 1: API Integration Module

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

DOCUMENTATION = r'''
---
module: internal_api
short_description: Manage resources via internal REST API
description:
    - Create, update, and delete resources through company's internal API.
    - Handles authentication automatically.
options:
    endpoint:
        description: API endpoint path.
        required: true
        type: str
    method:
        description: HTTP method.
        choices: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH']
        default: GET
        type: str
    data:
        description: Request body data.
        type: dict
        default: {}
    api_token:
        description: API authentication token.
        required: true
        type: str
        no_log: true
    base_url:
        description: API base URL.
        default: https://api.internal.company.com
        type: str
    timeout:
        description: Request timeout in seconds.
        default: 30
        type: int
    state:
        description: Desired resource state.
        choices: ['present', 'absent', 'query']
        default: present
        type: str
'''

EXAMPLES = r'''
- name: Create user via API
  internal_api:
    endpoint: /users
    method: POST
    data:
      username: johndoe
      email: john@company.com
      department: engineering
    api_token: "{{ vault_api_token }}"
    state: present

- name: Query user
  internal_api:
    endpoint: /users/johndoe
    method: GET
    api_token: "{{ vault_api_token }}"
    state: query
  register: user_info

- name: Delete user
  internal_api:
    endpoint: /users/johndoe
    method: DELETE
    api_token: "{{ vault_api_token }}"
    state: absent
'''

RETURN = r'''
status_code:
    description: HTTP status code.
    type: int
    returned: always
response:
    description: API response body.
    type: dict
    returned: success
'''

import json
from ansible.module_utils.basic import AnsibleModule
from ansible.module_utils.urls import fetch_url


def api_request(module, url, method, data, headers, timeout):
    """Make API request and handle response."""
    body = json.dumps(data) if data else None
    
    response, info = fetch_url(
        module,
        url,
        method=method,
        data=body,
        headers=headers,
        timeout=timeout
    )
    
    status_code = info['status']
    
    if response:
        body = response.read()
        try:
            response_data = json.loads(body)
        except json.JSONDecodeError:
            response_data = {'raw': body.decode('utf-8')}
    else:
        response_data = {}
        
    return status_code, response_data, info


def check_resource_exists(module, url, headers, timeout):
    """Check if resource exists."""
    status_code, response, info = api_request(
        module, url, 'GET', None, headers, timeout
    )
    return status_code == 200, response


def main():
    module_args = dict(
        endpoint=dict(type='str', required=True),
        method=dict(type='str', default='GET',
                   choices=['GET', 'POST', 'PUT', 'DELETE', 'PATCH']),
        data=dict(type='dict', default={}),
        api_token=dict(type='str', required=True, no_log=True),
        base_url=dict(type='str', default='https://api.internal.company.com'),
        timeout=dict(type='int', default=30),
        state=dict(type='str', default='present',
                  choices=['present', 'absent', 'query'])
    )

    module = AnsibleModule(
        argument_spec=module_args,
        supports_check_mode=True
    )

    endpoint = module.params['endpoint']
    method = module.params['method']
    data = module.params['data']
    api_token = module.params['api_token']
    base_url = module.params['base_url'].rstrip('/')
    timeout = module.params['timeout']
    state = module.params['state']

    url = f"{base_url}{endpoint}"
    headers = {
        'Authorization': f'Bearer {api_token}',
        'Content-Type': 'application/json',
        'Accept': 'application/json'
    }

    result = dict(
        changed=False,
        status_code=0,
        response={}
    )

    try:
        # Check mode
        if module.check_mode:
            exists, _ = check_resource_exists(module, url, headers, timeout)
            if state == 'present' and not exists:
                result['changed'] = True
            elif state == 'absent' and exists:
                result['changed'] = True
            module.exit_json(**result)

        # Query state
        if state == 'query':
            status_code, response, info = api_request(
                module, url, 'GET', None, headers, timeout
            )
            result['status_code'] = status_code
            result['response'] = response
            
            if status_code >= 400:
                module.fail_json(msg=f"Query failed: {info.get('msg', 'Unknown error')}", **result)
            
            module.exit_json(**result)

        # Present state
        if state == 'present':
            exists, current = check_resource_exists(module, url, headers, timeout)
            
            if exists:
                # Update if data differs
                if data and data != current:
                    status_code, response, info = api_request(
                        module, url, 'PUT', data, headers, timeout
                    )
                    result['changed'] = True
                else:
                    status_code = 200
                    response = current
            else:
                # Create new
                status_code, response, info = api_request(
                    module, url, 'POST', data, headers, timeout
                )
                result['changed'] = True
                
            result['status_code'] = status_code
            result['response'] = response

        # Absent state
        elif state == 'absent':
            exists, _ = check_resource_exists(module, url, headers, timeout)
            
            if exists:
                status_code, response, info = api_request(
                    module, url, 'DELETE', None, headers, timeout
                )
                result['changed'] = True
                result['status_code'] = status_code
                result['response'] = response
            else:
                result['status_code'] = 200
                result['response'] = {'message': 'Resource does not exist'}

        module.exit_json(**result)

    except Exception as e:
        module.fail_json(msg=f"API error: {str(e)}", **result)


if __name__ == '__main__':
    main()
```

---

### Use Case 2: Database Management Module

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

DOCUMENTATION = r'''
---
module: database_manager
short_description: Manage database schemas and users
description:
    - Create and manage database schemas, users, and permissions.
    - Supports PostgreSQL and MySQL.
options:
    db_type:
        description: Database type.
        choices: ['postgresql', 'mysql']
        required: true
        type: str
    host:
        description: Database host.
        default: localhost
        type: str
    port:
        description: Database port.
        type: int
    admin_user:
        description: Admin username.
        required: true
        type: str
    admin_password:
        description: Admin password.
        required: true
        type: str
        no_log: true
    operation:
        description: Operation to perform.
        choices: ['create_db', 'create_user', 'grant_permissions', 'backup', 'restore']
        required: true
        type: str
    database:
        description: Target database name.
        type: str
    username:
        description: Username for user operations.
        type: str
    password:
        description: Password for new user.
        type: str
        no_log: true
    permissions:
        description: List of permissions to grant.
        type: list
        elements: str
        default: ['SELECT', 'INSERT', 'UPDATE', 'DELETE']
    backup_path:
        description: Path for backup/restore operations.
        type: str
'''

EXAMPLES = r'''
- name: Create database
  database_manager:
    db_type: postgresql
    host: db.example.com
    admin_user: postgres
    admin_password: "{{ vault_db_admin_pass }}"
    operation: create_db
    database: myapp_production

- name: Create application user
  database_manager:
    db_type: postgresql
    admin_user: postgres
    admin_password: "{{ vault_db_admin_pass }}"
    operation: create_user
    database: myapp_production
    username: app_user
    password: "{{ vault_app_db_pass }}"
    permissions:
      - SELECT
      - INSERT
      - UPDATE
      - DELETE

- name: Backup database
  database_manager:
    db_type: postgresql
    admin_user: postgres
    admin_password: "{{ vault_db_admin_pass }}"
    operation: backup
    database: myapp_production
    backup_path: /backups/myapp_{{ ansible_date_time.date }}.sql
'''

RETURN = r'''
database:
    description: Database name.
    type: str
    returned: when applicable
user_created:
    description: Username created.
    type: str
    returned: when creating user
backup_file:
    description: Backup file path.
    type: str
    returned: when performing backup
'''

import subprocess
import os
from ansible.module_utils.basic import AnsibleModule


class DatabaseManager:
    def __init__(self, module):
        self.module = module
        self.db_type = module.params['db_type']
        self.host = module.params['host']
        self.port = module.params['port']
        self.admin_user = module.params['admin_user']
        self.admin_password = module.params['admin_password']
        
        # Set default ports
        if not self.port:
            self.port = 5432 if self.db_type == 'postgresql' else 3306
            
    def execute_sql(self, sql, database=None):
        """Execute SQL command."""
        if self.db_type == 'postgresql':
            return self._pg_execute(sql, database)
        else:
            return self._mysql_execute(sql, database)
            
    def _pg_execute(self, sql, database=None):
        """Execute PostgreSQL command."""
        env = os.environ.copy()
        env['PGPASSWORD'] = self.admin_password
        
        cmd = [
            'psql',
            '-h', self.host,
            '-p', str(self.port),
            '-U', self.admin_user,
            '-c', sql
        ]
        
        if database:
            cmd.extend(['-d', database])
            
        result = subprocess.run(
            cmd,
            env=env,
            capture_output=True,
            text=True
        )
        
        return result.returncode == 0, result.stdout, result.stderr
        
    def _mysql_execute(self, sql, database=None):
        """Execute MySQL command."""
        cmd = [
            'mysql',
            '-h', self.host,
            '-P', str(self.port),
            '-u', self.admin_user,
            f'-p{self.admin_password}',
            '-e', sql
        ]
        
        if database:
            cmd.append(database)
            
        result = subprocess.run(
            cmd,
            capture_output=True,
            text=True
        )
        
        return result.returncode == 0, result.stdout, result.stderr
        
    def database_exists(self, database):
        """Check if database exists."""
        if self.db_type == 'postgresql':
            sql = f"SELECT 1 FROM pg_database WHERE datname = '{database}'"
        else:
            sql = f"SELECT SCHEMA_NAME FROM INFORMATION_SCHEMA.SCHEMATA WHERE SCHEMA_NAME = '{database}'"
            
        success, stdout, _ = self.execute_sql(sql)
        return success and database in stdout
        
    def user_exists(self, username):
        """Check if user exists."""
        if self.db_type == 'postgresql':
            sql = f"SELECT 1 FROM pg_roles WHERE rolname = '{username}'"
        else:
            sql = f"SELECT User FROM mysql.user WHERE User = '{username}'"
            
        success, stdout, _ = self.execute_sql(sql)
        return success and username in stdout
        
    def create_database(self, database):
        """Create database."""
        if self.database_exists(database):
            return False, f"Database {database} already exists"
            
        if self.db_type == 'postgresql':
            sql = f"CREATE DATABASE {database}"
        else:
            sql = f"CREATE DATABASE {database} CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci"
            
        success, _, stderr = self.execute_sql(sql)
        
        if success:
            return True, f"Database {database} created"
        else:
            self.module.fail_json(msg=f"Failed to create database: {stderr}")
            
    def create_user(self, username, password, database):
        """Create database user."""
        if self.user_exists(username):
            return False, f"User {username} already exists"
            
        if self.db_type == 'postgresql':
            sql = f"CREATE USER {username} WITH PASSWORD '{password}'"
        else:
            sql = f"CREATE USER '{username}'@'%' IDENTIFIED BY '{password}'"
            
        success, _, stderr = self.execute_sql(sql)
        
        if not success:
            self.module.fail_json(msg=f"Failed to create user: {stderr}")
            
        return True, f"User {username} created"
        
    def grant_permissions(self, username, database, permissions):
        """Grant permissions to user."""
        perms = ', '.join(permissions)
        
        if self.db_type == 'postgresql':
            # Grant connect
            sql = f"GRANT CONNECT ON DATABASE {database} TO {username}"
            self.execute_sql(sql)
            
            # Grant table permissions
            sql = f"GRANT {perms} ON ALL TABLES IN SCHEMA public TO {username}"
            self.execute_sql(sql, database)
            
            # Set default privileges
            sql = f"ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT {perms} ON TABLES TO {username}"
            self.execute_sql(sql, database)
        else:
            sql = f"GRANT {perms} ON {database}.* TO '{username}'@'%'"
            self.execute_sql(sql)
            self.execute_sql("FLUSH PRIVILEGES")
            
        return True, f"Granted {perms} on {database} to {username}"
        
    def backup(self, database, backup_path):
        """Backup database."""
        env = os.environ.copy()
        
        if self.db_type == 'postgresql':
            env['PGPASSWORD'] = self.admin_password
            cmd = [
                'pg_dump',
                '-h', self.host,
                '-p', str(self.port),
                '-U', self.admin_user,
                '-Fc',  # Custom format for pg_restore
                '-f', backup_path,
                database
            ]
        else:
            cmd = [
                'mysqldump',
                '-h', self.host,
                '-P', str(self.port),
                '-u', self.admin_user,
                f'-p{self.admin_password}',
                '--single-transaction',
                '--routines',
                '--triggers',
                database
            ]
            
        result = subprocess.run(
            cmd,
            env=env,
            capture_output=True,
            text=True
        )
        
        if self.db_type == 'mysql':
            # Write output to file for MySQL
            with open(backup_path, 'w') as f:
                f.write(result.stdout)
                
        if result.returncode == 0:
            return True, f"Backup saved to {backup_path}"
        else:
            self.module.fail_json(msg=f"Backup failed: {result.stderr}")
            
    def restore(self, database, backup_path):
        """Restore database."""
        if not os.path.exists(backup_path):
            self.module.fail_json(msg=f"Backup file not found: {backup_path}")
            
        env = os.environ.copy()
        
        if self.db_type == 'postgresql':
            env['PGPASSWORD'] = self.admin_password
            cmd = [
                'pg_restore',
                '-h', self.host,
                '-p', str(self.port),
                '-U', self.admin_user,
                '-d', database,
                '--clean',
                '--if-exists',
                backup_path
            ]
        else:
            cmd = f"mysql -h {self.host} -P {self.port} -u {self.admin_user} -p{self.admin_password} {database} < {backup_path}"
            
        if self.db_type == 'postgresql':
            result = subprocess.run(cmd, env=env, capture_output=True, text=True)
        else:
            result = subprocess.run(cmd, shell=True, capture_output=True, text=True)
            
        if result.returncode == 0:
            return True, f"Restored {database} from {backup_path}"
        else:
            self.module.fail_json(msg=f"Restore failed: {result.stderr}")


def main():
    module_args = dict(
        db_type=dict(type='str', required=True, choices=['postgresql', 'mysql']),
        host=dict(type='str', default='localhost'),
        port=dict(type='int'),
        admin_user=dict(type='str', required=True),
        admin_password=dict(type='str', required=True, no_log=True),
        operation=dict(type='str', required=True,
                      choices=['create_db', 'create_user', 'grant_permissions', 'backup', 'restore']),
        database=dict(type='str'),
        username=dict(type='str'),
        password=dict(type='str', no_log=True),
        permissions=dict(type='list', elements='str',
                        default=['SELECT', 'INSERT', 'UPDATE', 'DELETE']),
        backup_path=dict(type='str')
    )

    module = AnsibleModule(
        argument_spec=module_args,
        required_if=[
            ('operation', 'create_db', ['database']),
            ('operation', 'create_user', ['database', 'username', 'password']),
            ('operation', 'grant_permissions', ['database', 'username']),
            ('operation', 'backup', ['database', 'backup_path']),
            ('operation', 'restore', ['database', 'backup_path']),
        ],
        supports_check_mode=True
    )

    db_manager = DatabaseManager(module)
    operation = module.params['operation']
    
    result = dict(changed=False)

    # Check mode
    if module.check_mode:
        result['changed'] = True
        module.exit_json(**result)

    try:
        if operation == 'create_db':
            changed, msg = db_manager.create_database(module.params['database'])
            result['changed'] = changed
            result['database'] = module.params['database']
            result['message'] = msg
            
        elif operation == 'create_user':
            changed, msg = db_manager.create_user(
                module.params['username'],
                module.params['password'],
                module.params['database']
            )
            result['changed'] = changed
            result['user_created'] = module.params['username']
            result['message'] = msg
            
            # Also grant permissions
            if changed:
                db_manager.grant_permissions(
                    module.params['username'],
                    module.params['database'],
                    module.params['permissions']
                )
                
        elif operation == 'grant_permissions':
            changed, msg = db_manager.grant_permissions(
                module.params['username'],
                module.params['database'],
                module.params['permissions']
            )
            result['changed'] = changed
            result['message'] = msg
            
        elif operation == 'backup':
            changed, msg = db_manager.backup(
                module.params['database'],
                module.params['backup_path']
            )
            result['changed'] = changed
            result['backup_file'] = module.params['backup_path']
            result['message'] = msg
            
        elif operation == 'restore':
            changed, msg = db_manager.restore(
                module.params['database'],
                module.params['backup_path']
            )
            result['changed'] = changed
            result['message'] = msg

        module.exit_json(**result)

    except Exception as e:
        module.fail_json(msg=str(e), **result)


if __name__ == '__main__':
    main()
```

---

### Use Case 3: Service Health Monitor Module

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

DOCUMENTATION = r'''
---
module: service_health
short_description: Monitor and manage service health
description:
    - Check health of services via multiple protocols.
    - Trigger alerts or recovery actions.
options:
    name:
        description: Service name for identification.
        required: true
        type: str
    check_type:
        description: Type of health check.
        choices: ['http', 'tcp', 'dns', 'process', 'disk', 'memory']
        default: http
        type: str
    endpoint:
        description: Endpoint to check (URL or host:port).
        type: str
    expected_status:
        description: Expected HTTP status code.
        default: 200
        type: int
    expected_response:
        description: String expected in response body.
        type: str
    timeout:
        description: Check timeout in seconds.
        default: 10
        type: int
    process_name:
        description: Process name for process check.
        type: str
    threshold_warning:
        description: Warning threshold percentage.
        type: int
        default: 80
    threshold_critical:
        description: Critical threshold percentage.
        type: int
        default: 90
    action_on_failure:
        description: Action to take on failure.
        choices: ['none', 'restart', 'alert', 'custom']
        default: alert
        type: str
    restart_command:
        description: Command to restart service.
        type: str
    alert_webhook:
        description: Webhook URL for alerts.
        type: str
'''

EXAMPLES = r'''
- name: Check web application health
  service_health:
    name: webapp
    check_type: http
    endpoint: http://localhost:8080/health
    expected_status: 200
    expected_response: "healthy"
    action_on_failure: restart
    restart_command: "systemctl restart webapp"

- name: Check database connectivity
  service_health:
    name: postgresql
    check_type: tcp
    endpoint: "localhost:5432"
    timeout: 5
    action_on_failure: alert
    alert_webhook: "https://hooks.slack.com/services/xxx"

- name: Check disk space
  service_health:
    name: disk_root
    check_type: disk
    endpoint: "/"
    threshold_warning: 80
    threshold_critical: 90

- name: Check memory usage
  service_health:
    name: memory
    check_type: memory
    threshold_warning: 75
    threshold_critical: 90
'''

RETURN = r'''
healthy:
    description: Whether service is healthy.
    type: bool
    returned: always
status:
    description: Health status.
    type: str
    returned: always
    sample: "healthy"
details:
    description: Detailed health information.
    type: dict
    returned: always
action_taken:
    description: Action taken on failure.
    type: str
    returned: when action taken
'''

import socket
import subprocess
import os
import json
from ansible.module_utils.basic import AnsibleModule
from ansible.module_utils.urls import fetch_url


class HealthChecker:
    def __init__(self, module):
        self.module = module
        
    def check_http(self, endpoint, expected_status, expected_response, timeout):
        """HTTP health check."""
        try:
            response, info = fetch_url(
                self.module,
                endpoint,
                timeout=timeout
            )
            
            status_code = info['status']
            body = response.read().decode('utf-8') if response else ''
            
            healthy = status_code == expected_status
            
            if healthy and expected_response:
                healthy = expected_response in body
                
            return {
                'healthy': healthy,
                'status': 'healthy' if healthy else 'unhealthy',
                'details': {
                    'status_code': status_code,
                    'response_contains_expected': expected_response in body if expected_response else True
                }
            }
            
        except Exception as e:
            return {
                'healthy': False,
                'status': 'error',
                'details': {'error': str(e)}
            }
            
    def check_tcp(self, endpoint, timeout):
        """TCP port health check."""
        try:
            host, port = endpoint.rsplit(':', 1)
            port = int(port)
            
            sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            sock.settimeout(timeout)
            result = sock.connect_ex((host, port))
            sock.close()
            
            healthy = result == 0
            
            return {
                'healthy': healthy,
                'status': 'healthy' if healthy else 'unhealthy',
                'details': {
                    'host': host,
                    'port': port,
                    'connection': 'successful' if healthy else 'refused'
                }
            }
            
        except Exception as e:
            return {
                'healthy': False,
                'status': 'error',
                'details': {'error': str(e)}
            }
            
    def check_process(self, process_name):
        """Process health check."""
        try:
            result = subprocess.run(
                ['pgrep', '-f', process_name],
                capture_output=True,
                text=True
            )
            
            healthy = result.returncode == 0
            pids = result.stdout.strip().split('\n') if healthy else []
            
            return {
                'healthy': healthy,
                'status': 'healthy' if healthy else 'unhealthy',
                'details': {
                    'process': process_name,
                    'running': healthy,
                    'pids': pids
                }
            }
            
        except Exception as e:
            return {
                'healthy': False,
                'status': 'error',
                'details': {'error': str(e)}
            }
            
    def check_disk(self, mount_point, warning, critical):
        """Disk space health check."""
        try:
            stat = os.statvfs(mount_point)
            total = stat.f_blocks * stat.f_frsize
            free = stat.f_bavail * stat.f_frsize
            used = total - free
            usage_percent = (used / total) * 100
            
            if usage_percent >= critical:
                status = 'critical'
                healthy = False
            elif usage_percent >= warning:
                status = 'warning'
                healthy = True
            else:
                status = 'healthy'
                healthy = True
                
            return {
                'healthy': healthy,
                'status': status,
                'details': {
                    'mount_point': mount_point,
                    'total_gb': round(total / (1024**3), 2),
                    'used_gb': round(used / (1024**3), 2),
                    'free_gb': round(free / (1024**3), 2),
                    'usage_percent': round(usage_percent, 2)
                }
            }
            
        except Exception as e:
            return {
                'healthy': False,
                'status': 'error',
                'details': {'error': str(e)}
            }
            
    def check_memory(self, warning, critical):
        """Memory usage health check."""
        try:
            with open('/proc/meminfo', 'r') as f:
                meminfo = {}
                for line in f:
                    parts = line.split(':')
                    if len(parts) == 2:
                        key = parts[0].strip()
                        value = int(parts[1].strip().split()[0])
                        meminfo[key] = value
                        
            total = meminfo.get('MemTotal', 0)
            available = meminfo.get('MemAvailable', meminfo.get('MemFree', 0))
            used = total - available
            usage_percent = (used / total) * 100 if total > 0 else 0
            
            if usage_percent >= critical:
                status = 'critical'
                healthy = False
            elif usage_percent >= warning:
                status = 'warning'
                healthy = True
            else:
                status = 'healthy'
                healthy = True
                
            return {
                'healthy': healthy,
                'status': status,
                'details': {
                    'total_mb': round(total / 1024, 2),
                    'used_mb': round(used / 1024, 2),
                    'available_mb': round(available / 1024, 2),
                    'usage_percent': round(usage_percent, 2)
                }
            }
            
        except Exception as e:
            return {
                'healthy': False,
                'status': 'error',
                'details': {'error': str(e)}
            }
            
    def restart_service(self, command):
        """Restart service."""
        try:
            result = subprocess.run(
                command,
                shell=True,
                capture_output=True,
                text=True
            )
            return result.returncode == 0
        except:
            return False
            
    def send_alert(self, webhook_url, service_name, health_result):
        """Send alert to webhook."""
        try:
            alert_data = {
                'service': service_name,
                'status': health_result['status'],
                'details': health_result['details'],
                'timestamp': subprocess.run(
                    ['date', '-Iseconds'],
                    capture_output=True,
                    text=True
                ).stdout.strip()
            }
            
            fetch_url(
                self.module,
                webhook_url,
                method='POST',
                data=json.dumps(alert_data).encode('utf-8'),
                headers={'Content-Type': 'application/json'}
            )
            return True
        except:
            return False


def main():
    module_args = dict(
        name=dict(type='str', required=True),
        check_type=dict(type='str', default='http',
                       choices=['http', 'tcp', 'dns', 'process', 'disk', 'memory']),
        endpoint=dict(type='str'),
        expected_status=dict(type='int', default=200),
        expected_response=dict(type='str'),
        timeout=dict(type='int', default=10),
        process_name=dict(type='str'),
        threshold_warning=dict(type='int', default=80),
        threshold_critical=dict(type='int', default=90),
        action_on_failure=dict(type='str', default='alert',
                              choices=['none', 'restart', 'alert', 'custom']),
        restart_command=dict(type='str'),
        alert_webhook=dict(type='str')
    )

    module = AnsibleModule(
        argument_spec=module_args,
        required_if=[
            ('check_type', 'http', ['endpoint']),
            ('check_type', 'tcp', ['endpoint']),
            ('check_type', 'disk', ['endpoint']),
            ('check_type', 'process', ['process_name']),
            ('action_on_failure', 'restart', ['restart_command']),
            ('action_on_failure', 'alert', ['alert_webhook']),
        ],
        supports_check_mode=True
    )

    checker = HealthChecker(module)
    check_type = module.params['check_type']
    name = module.params['name']
    
    # Perform health check
    if check_type == 'http':
        health_result = checker.check_http(
            module.params['endpoint'],
            module.params['expected_status'],
            module.params['expected_response'],
            module.params['timeout']
        )
    elif check_type == 'tcp':
        health_result = checker.check_tcp(
            module.params['endpoint'],
            module.params['timeout']
        )
    elif check_type == 'process':
        health_result = checker.check_process(module.params['process_name'])
    elif check_type == 'disk':
        health_result = checker.check_disk(
            module.params['endpoint'],
            module.params['threshold_warning'],
            module.params['threshold_critical']
        )
    elif check_type == 'memory':
        health_result = checker.check_memory(
            module.params['threshold_warning'],
            module.params['threshold_critical']
        )
    else:
        module.fail_json(msg=f"Unsupported check type: {check_type}")

    result = {
        'changed': False,
        'healthy': health_result['healthy'],
        'status': health_result['status'],
        'details': health_result['details']
    }

    # Check mode - don't take actions
    if module.check_mode:
        module.exit_json(**result)

    # Take action on failure
    if not health_result['healthy']:
        action = module.params['action_on_failure']
        
        if action == 'restart':
            if checker.restart_service(module.params['restart_command']):
                result['changed'] = True
                result['action_taken'] = 'service_restarted'
            else:
                result['action_taken'] = 'restart_failed'
                
        elif action == 'alert':
            if checker.send_alert(
                module.params['alert_webhook'],
                name,
                health_result
            ):
                result['action_taken'] = 'alert_sent'
            else:
                result['action_taken'] = 'alert_failed'

    module.exit_json(**result)


if __name__ == '__main__':
    main()
```

---

### Use Case 4: Configuration Drift Detector

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

DOCUMENTATION = r'''
---
module: config_drift
short_description: Detect configuration drift from baseline
description:
    - Compare current system configuration against a baseline.
    - Report differences and optionally remediate.
options:
    baseline_file:
        description: Path to baseline configuration file.
        required: true
        type: str
    config_type:
        description: Type of configuration to check.
        choices: ['packages', 'files', 'users', 'services', 'sysctl']
        required: true
        type: str
    remediate:
        description: Automatically fix drift.
        default: false
        type: bool
    ignore_patterns:
        description: Patterns to ignore.
        type: list
        elements: str
        default: []
'''

EXAMPLES = r'''
- name: Check package drift
  config_drift:
    baseline_file: /etc/ansible/baselines/packages.json
    config_type: packages
    remediate: true

- name: Check file permissions drift
  config_drift:
    baseline_file: /etc/ansible/baselines/files.json
    config_type: files
    ignore_patterns:
      - "*.log"
      - "/tmp/*"
'''

RETURN = r'''
compliant:
    description: Whether system is compliant.
    type: bool
    returned: always
drift_found:
    description: List of drift items found.
    type: list
    returned: always
remediated:
    description: Items that were remediated.
    type: list
    returned: when remediate=true
'''

import json
import subprocess
import os
import pwd
import grp
import fnmatch
from ansible.module_utils.basic import AnsibleModule


class DriftDetector:
    def __init__(self, module, baseline, ignore_patterns):
        self.module = module
        self.baseline = baseline
        self.ignore_patterns = ignore_patterns
        
    def should_ignore(self, item):
        """Check if item matches ignore patterns."""
        for pattern in self.ignore_patterns:
            if fnmatch.fnmatch(item, pattern):
                return True
        return False
        
    def check_packages(self):
        """Check installed packages against baseline."""
        drift = []
        
        # Get current packages (Debian/Ubuntu)
        result = subprocess.run(
            ['dpkg-query', '-W', '-f=${Package}=${Version}\n'],
            capture_output=True,
            text=True
        )
        
        current_packages = {}
        for line in result.stdout.strip().split('\n'):
            if '=' in line:
                name, version = line.split('=', 1)
                current_packages[name] = version
                
        # Compare with baseline
        for pkg, expected_version in self.baseline.get('packages', {}).items():
            if self.should_ignore(pkg):
                continue
                
            current_version = current_packages.get(pkg)
            
            if current_version is None:
                drift.append({
                    'type': 'missing',
                    'package': pkg,
                    'expected': expected_version
                })
            elif expected_version != '*' and current_version != expected_version:
                drift.append({
                    'type': 'version_mismatch',
                    'package': pkg,
                    'expected': expected_version,
                    'current': current_version
                })
                
        # Check for unexpected packages
        expected_packages = set(self.baseline.get('packages', {}).keys())
        for pkg in current_packages:
            if pkg not in expected_packages and not self.should_ignore(pkg):
                if self.baseline.get('strict_packages', False):
                    drift.append({
                        'type': 'unexpected',
                        'package': pkg,
                        'current': current_packages[pkg]
                    })
                    
        return drift
        
    def check_files(self):
        """Check file permissions and ownership against baseline."""
        drift = []
        
        for file_path, expected in self.baseline.get('files', {}).items():
            if self.should_ignore(file_path):
                continue
                
            if not os.path.exists(file_path):
                drift.append({
                    'type': 'missing',
                    'file': file_path
                })
                continue
                
            stat = os.stat(file_path)
            current_mode = oct(stat.st_mode)[-3:]
            current_owner = pwd.getpwuid(stat.st_uid).pw_name
            current_group = grp.getgrgid(stat.st_gid).gr_name
            
            if 'mode' in expected and current_mode != expected['mode']:
                drift.append({
                    'type': 'mode_mismatch',
                    'file': file_path,
                    'expected': expected['mode'],
                    'current': current_mode
                })
                
            if 'owner' in expected and current_owner != expected['owner']:
                drift.append({
                    'type': 'owner_mismatch',
                    'file': file_path,
                    'expected': expected['owner'],
                    'current': current_owner
                })
                
            if 'group' in expected and current_group != expected['group']:
                drift.append({
                    'type': 'group_mismatch',
                    'file': file_path,
                    'expected': expected['group'],
                    'current': current_group
                })
                
        return drift
        
    def check_services(self):
        """Check service states against baseline."""
        drift = []
        
        for service, expected_state in self.baseline.get('services', {}).items():
            if self.should_ignore(service):
                continue
                
            # Check if service is running
            result = subprocess.run(
                ['systemctl', 'is-active', service],
                capture_output=True,
                text=True
            )
            is_running = result.returncode == 0
            
            # Check if service is enabled
            result = subprocess.run(
                ['systemctl', 'is-enabled', service],
                capture_output=True,
                text=True
            )
            is_enabled = result.stdout.strip() == 'enabled'
            
            if expected_state.get('running', True) != is_running:
                drift.append({
                    'type': 'running_mismatch',
                    'service': service,
                    'expected': expected_state.get('running', True),
                    'current': is_running
                })
                
            if expected_state.get('enabled', True) != is_enabled:
                drift.append({
                    'type': 'enabled_mismatch',
                    'service': service,
                    'expected': expected_state.get('enabled', True),
                    'current': is_enabled
                })
                
        return drift
        
    def remediate_packages(self, drift_items):
        """Remediate package drift."""
        remediated = []
        
        for item in drift_items:
            if item['type'] == 'missing':
                result = subprocess.run(
                    ['apt-get', 'install', '-y', f"{item['package']}={item['expected']}"],
                    capture_output=True
                )
                if result.returncode == 0:
                    remediated.append(item)
            elif item['type'] == 'version_mismatch':
                result = subprocess.run(
                    ['apt-get', 'install', '-y', f"{item['package']}={item['expected']}"],
                    capture_output=True
                )
                if result.returncode == 0:
                    remediated.append(item)
                    
        return remediated
        
    def remediate_files(self, drift_items):
        """Remediate file drift."""
        remediated = []
        
        for item in drift_items:
            file_path = item['file']
            
            if item['type'] == 'mode_mismatch':
                try:
                    os.chmod(file_path, int(item['expected'], 8))
                    remediated.append(item)
                except:
                    pass
            elif item['type'] in ['owner_mismatch', 'group_mismatch']:
                baseline = self.baseline['files'][file_path]
                owner = baseline.get('owner', pwd.getpwuid(os.stat(file_path).st_uid).pw_name)
                group = baseline.get('group', grp.getgrgid(os.stat(file_path).st_gid).gr_name)
                try:
                    uid = pwd.getpwnam(owner).pw_uid
                    gid = grp.getgrnam(group).gr_gid
                    os.chown(file_path, uid, gid)
                    remediated.append(item)
                except:
                    pass
                    
        return remediated


def main():
    module_args = dict(
        baseline_file=dict(type='str', required=True),
        config_type=dict(type='str', required=True,
                        choices=['packages', 'files', 'users', 'services', 'sysctl']),
        remediate=dict(type='bool', default=False),
        ignore_patterns=dict(type='list', elements='str', default=[])
    )

    module = AnsibleModule(
        argument_spec=module_args,
        supports_check_mode=True
    )

    baseline_file = module.params['baseline_file']
    config_type = module.params['config_type']
    remediate = module.params['remediate']
    ignore_patterns = module.params['ignore_patterns']

    # Load baseline
    try:
        with open(baseline_file, 'r') as f:
            baseline = json.load(f)
    except Exception as e:
        module.fail_json(msg=f"Failed to load baseline: {str(e)}")

    detector = DriftDetector(module, baseline, ignore_patterns)
    
    result = {
        'changed': False,
        'compliant': True,
        'drift_found': [],
        'remediated': []
    }

    # Detect drift
    if config_type == 'packages':
        drift = detector.check_packages()
    elif config_type == 'files':
        drift = detector.check_files()
    elif config_type == 'services':
        drift = detector.check_services()
    else:
        module.fail_json(msg=f"Unsupported config type: {config_type}")

    result['drift_found'] = drift
    result['compliant'] = len(drift) == 0

    # Check mode - don't remediate
    if module.check_mode:
        module.exit_json(**result)

    # Remediate if requested
    if remediate and drift:
        if config_type == 'packages':
            result['remediated'] = detector.remediate_packages(drift)
        elif config_type == 'files':
            result['remediated'] = detector.remediate_files(drift)
            
        if result['remediated']:
            result['changed'] = True
            # Re-check compliance after remediation
            result['compliant'] = len(drift) == len(result['remediated'])

    module.exit_json(**result)


if __name__ == '__main__':
    main()
```

---

## Part 3: Module Development Best Practices

### Directory Structure

```
my_collection/
├── plugins/
│   └── modules/
│       ├── __init__.py
│       ├── my_module.py
│       └── my_other_module.py
│   └── module_utils/
│       ├── __init__.py
│       └── common.py        # Shared utilities
├── tests/
│   └── unit/
│       └── plugins/
│           └── modules/
│               └── test_my_module.py
└── galaxy.yml
```

### Shared Module Utilities

```python
# plugins/module_utils/common.py

class APIClient:
    """Shared API client for all modules."""
    
    def __init__(self, base_url, token, timeout=30):
        self.base_url = base_url.rstrip('/')
        self.token = token
        self.timeout = timeout
        self.headers = {
            'Authorization': f'Bearer {token}',
            'Content-Type': 'application/json'
        }
        
    def request(self, method, endpoint, data=None):
        """Make API request."""
        import json
        from ansible.module_utils.urls import open_url
        
        url = f"{self.base_url}/{endpoint.lstrip('/')}"
        body = json.dumps(data) if data else None
        
        response = open_url(
            url,
            method=method,
            data=body,
            headers=self.headers,
            timeout=self.timeout
        )
        
        return json.loads(response.read())


# In your module:
from ansible_collections.myorg.mytools.plugins.module_utils.common import APIClient
```

### Unit Testing

```python
# tests/unit/plugins/modules/test_my_module.py

import pytest
from unittest.mock import MagicMock, patch
from ansible_collections.myorg.mytools.plugins.modules import my_module


@pytest.fixture
def module_args():
    return {
        'name': 'test_resource',
        'state': 'present',
        'options': {}
    }


def test_create_resource(module_args):
    """Test resource creation."""
    with patch.object(my_module.AnsibleModule, '__init__', return_value=None):
        with patch.object(my_module.AnsibleModule, 'params', module_args):
            with patch.object(my_module.AnsibleModule, 'exit_json') as mock_exit:
                my_module.main()
                
                mock_exit.assert_called_once()
                call_args = mock_exit.call_args[1]
                assert call_args['changed'] is True


def test_check_mode(module_args):
    """Test check mode."""
    with patch.object(my_module.AnsibleModule, '__init__', return_value=None):
        with patch.object(my_module.AnsibleModule, 'params', module_args):
            with patch.object(my_module.AnsibleModule, 'check_mode', True):
                with patch.object(my_module.AnsibleModule, 'exit_json') as mock_exit:
                    my_module.main()
                    
                    call_args = mock_exit.call_args[1]
                    assert 'Would' in call_args.get('message', '')
```

---

## Quick Reference

### Module Skeleton Template

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

DOCUMENTATION = r'''
---
module: module_name
short_description: One line description
description:
    - Detailed description
options:
    param1:
        description: Parameter description
        required: true
        type: str
author:
    - Your Name (@github)
'''

EXAMPLES = r'''
- name: Example usage
  module_name:
    param1: value
'''

RETURN = r'''
result:
    description: Description
    type: str
    returned: always
'''

from ansible.module_utils.basic import AnsibleModule


def main():
    module = AnsibleModule(
        argument_spec=dict(
            param1=dict(type='str', required=True)
        ),
        supports_check_mode=True
    )
    
    result = dict(changed=False)
    
    try:
        # Your logic here
        module.exit_json(**result)
    except Exception as e:
        module.fail_json(msg=str(e), **result)


if __name__ == '__main__':
    main()
```

### Common Patterns

```python
# Idempotency check
def resource_exists(name):
    # Check if resource exists
    return True/False

if state == 'present':
    if not resource_exists(name):
        if not module.check_mode:
            create_resource(name)
        result['changed'] = True
else:
    if resource_exists(name):
        if not module.check_mode:
            delete_resource(name)
        result['changed'] = True

# Error handling
try:
    do_something()
except SpecificError as e:
    module.fail_json(msg=f"Operation failed: {e}")
except Exception as e:
    module.fail_json(msg=f"Unexpected error: {e}")

# Secure password handling
password=dict(type='str', no_log=True)
```

---

*Next: [18_Interview_Questions](../18_Interview_Questions/README.md) - Ansible interview preparation*
