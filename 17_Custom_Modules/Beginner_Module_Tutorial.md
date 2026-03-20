# Ansible Custom Module Development - Beginner Tutorial

## Start Here: Your First Custom Module

This guide takes you from **zero to custom module** with clear explanations at every step.

---

## Table of Contents

1. [Understanding What a Module Is](#understanding-what-a-module-is)
2. [Your First Module (Hello World)](#your-first-module-hello-world)
3. [Module Building Blocks Explained](#module-building-blocks-explained)
4. [Beginner Examples (5 Progressive Examples)](#beginner-examples)
5. [Intermediate Examples](#intermediate-examples)
6. [Packaging Your Modules](#packaging-your-modules)
7. [Deploying to Production](#deploying-to-production)

---

## Understanding What a Module Is

### What Happens When You Run a Playbook?

```
YOU write this:                 Ansible does this:
─────────────────              ────────────────────

playbook.yml                    1. Reads your playbook
│                               2. Finds the module (apt)
├─ - name: Install nginx        3. Copies module to target server
│   apt:                        4. Runs module with your parameters
│     name: nginx               5. Module returns JSON result
│     state: present            6. Ansible shows you the result
```

### A Module is Just a Python Script

```python
# That's it! A module is a Python script that:
# 1. Receives input (parameters from playbook)
# 2. Does something (install package, create file, call API)
# 3. Returns output (success/failure, what changed)
```

### Simple Analogy: Modules are Like Kitchen Appliances

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   PLAYBOOK = Recipe                                             │
│   "Bake a cake at 350°F for 30 minutes"                        │
│                                                                 │
│   MODULE = Oven                                                 │
│   • Receives: temperature=350, time=30                          │
│   • Does: Actually bakes the cake                               │
│   • Returns: "Cake is done" or "Error: oven broken"            │
│                                                                 │
│   Built-in modules: apt, file, copy, template (many ovens!)    │
│   Custom modules: YOUR special appliance for YOUR needs        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Your First Module (Hello World)

Let's create the simplest possible module to understand the structure.

### Step 1: Create the Module File

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

# File: library/hello_world.py
# This is your first custom module!

from ansible.module_utils.basic import AnsibleModule

def main():
    # Step 1: Define what parameters this module accepts
    module = AnsibleModule(
        argument_spec=dict(
            name=dict(type='str', required=True)
        )
    )
    
    # Step 2: Get the parameter value
    name = module.params['name']
    
    # Step 3: Return the result
    module.exit_json(
        changed=False,
        message=f"Hello, {name}!"
    )

if __name__ == '__main__':
    main()
```

### Step 2: Create Your Directory Structure

```
my_ansible_project/
├── library/                    # ← Ansible looks here for custom modules
│   └── hello_world.py          # ← Your module
├── playbook.yml                # ← Your playbook
└── inventory                   # ← Your inventory
```

### Step 3: Use Your Module in a Playbook

```yaml
# playbook.yml
---
- name: Test my first module
  hosts: localhost
  connection: local
  
  tasks:
    - name: Say hello
      hello_world:
        name: "World"
      register: result
      
    - name: Show the result
      debug:
        var: result.message
```

### Step 4: Run It!

```bash
ansible-playbook playbook.yml

# Output:
# TASK [Say hello] ****
# ok: [localhost]
#
# TASK [Show the result] ****
# ok: [localhost] => {
#     "result.message": "Hello, World!"
# }
```

🎉 **Congratulations!** You've created your first custom module!

---

## Module Building Blocks Explained

Every module has these parts. Let's understand each one:

### The Complete Template

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

# ╔═══════════════════════════════════════════════════════════════════╗
# ║ BLOCK 1: DOCUMENTATION                                           ║
# ║ This appears when you run: ansible-doc my_module                  ║
# ╚═══════════════════════════════════════════════════════════════════╝

DOCUMENTATION = r'''
---
module: my_module
short_description: A one-line description
description:
    - A longer description
    - Can have multiple lines
options:
    name:
        description: What is this parameter for
        required: true
        type: str
    state:
        description: What should happen
        required: false
        type: str
        choices: ['present', 'absent']
        default: present
author:
    - Your Name (@yourgithub)
'''

# ╔═══════════════════════════════════════════════════════════════════╗
# ║ BLOCK 2: EXAMPLES                                                 ║
# ║ Shows users how to use your module                                ║
# ╚═══════════════════════════════════════════════════════════════════╝

EXAMPLES = r'''
- name: Create something
  my_module:
    name: example
    state: present

- name: Remove something
  my_module:
    name: example
    state: absent
'''

# ╔═══════════════════════════════════════════════════════════════════╗
# ║ BLOCK 3: RETURN VALUES                                            ║
# ║ Documents what your module returns                                ║
# ╚═══════════════════════════════════════════════════════════════════╝

RETURN = r'''
message:
    description: Status message
    type: str
    returned: always
    sample: "Resource created successfully"
data:
    description: The resource data
    type: dict
    returned: success
'''

# ╔═══════════════════════════════════════════════════════════════════╗
# ║ BLOCK 4: IMPORTS                                                  ║
# ║ Python libraries you need                                         ║
# ╚═══════════════════════════════════════════════════════════════════╝

from ansible.module_utils.basic import AnsibleModule

# ╔═══════════════════════════════════════════════════════════════════╗
# ║ BLOCK 5: MAIN FUNCTION                                            ║
# ║ Where all the work happens                                        ║
# ╚═══════════════════════════════════════════════════════════════════╝

def main():
    # Define parameters (argument_spec)
    module_args = dict(
        name=dict(type='str', required=True),
        state=dict(type='str', default='present', choices=['present', 'absent'])
    )
    
    # Initialize module
    module = AnsibleModule(
        argument_spec=module_args,
        supports_check_mode=True  # Best practice!
    )
    
    # Get parameters
    name = module.params['name']
    state = module.params['state']
    
    # Prepare result
    result = dict(
        changed=False,
        message='',
        data={}
    )
    
    # Check mode = dry run (show what would happen)
    if module.check_mode:
        result['message'] = f"Would {'create' if state == 'present' else 'remove'} {name}"
        module.exit_json(**result)
    
    # Your actual logic here
    try:
        if state == 'present':
            result['changed'] = True
            result['message'] = f"Created {name}"
        else:
            result['changed'] = True
            result['message'] = f"Removed {name}"
            
        module.exit_json(**result)
        
    except Exception as e:
        module.fail_json(msg=f"Error: {str(e)}", **result)


if __name__ == '__main__':
    main()
```

### Understanding Each Part

| Block | Purpose | Required? |
|-------|---------|-----------|
| `DOCUMENTATION` | Shows in `ansible-doc` | Recommended |
| `EXAMPLES` | Usage examples | Recommended |
| `RETURN` | What module returns | Recommended |
| `argument_spec` | Define parameters | **Yes** |
| `AnsibleModule()` | Initialize module | **Yes** |
| `module.params` | Access parameters | **Yes** |
| `module.exit_json()` | Return success | **Yes** |
| `module.fail_json()` | Return failure | **Yes** |

---

## Beginner Examples

### Example 1: File Existence Checker

**Purpose**: Check if a file exists and return information about it.

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

# library/file_checker.py
# A simple module that checks if a file exists

DOCUMENTATION = r'''
---
module: file_checker
short_description: Check if a file exists
description:
    - Checks if a file exists and returns its info
options:
    path:
        description: Path to the file to check
        required: true
        type: str
'''

EXAMPLES = r'''
- name: Check if config file exists
  file_checker:
    path: /etc/nginx/nginx.conf
  register: result

- name: Show result
  debug:
    msg: "File exists: {{ result.exists }}"
'''

RETURN = r'''
exists:
    description: Whether the file exists
    type: bool
    returned: always
size:
    description: File size in bytes
    type: int
    returned: when file exists
'''

import os
from ansible.module_utils.basic import AnsibleModule


def main():
    # What parameters does this module accept?
    module_args = dict(
        path=dict(type='str', required=True)
    )
    
    # Initialize the module
    module = AnsibleModule(
        argument_spec=module_args,
        supports_check_mode=True  # This module doesn't change anything
    )
    
    # Get the path parameter
    path = module.params['path']
    
    # Prepare our result
    result = dict(
        changed=False,  # This module never changes anything
        exists=False,
        path=path
    )
    
    # Check if file exists
    if os.path.exists(path):
        result['exists'] = True
        result['size'] = os.path.getsize(path)
        result['is_file'] = os.path.isfile(path)
        result['is_directory'] = os.path.isdir(path)
    
    # Return the result
    module.exit_json(**result)


if __name__ == '__main__':
    main()
```

**How to use it:**
```yaml
- name: Check if nginx is installed
  file_checker:
    path: /etc/nginx/nginx.conf
  register: nginx_check

- name: Install nginx if not present
  apt:
    name: nginx
    state: present
  when: not nginx_check.exists
```

---

### Example 2: Simple Calculator

**Purpose**: Learn how to handle different parameter types.

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

# library/calculator.py
# A module that performs basic math operations

DOCUMENTATION = r'''
---
module: calculator
short_description: Perform basic math operations
description:
    - Add, subtract, multiply, or divide two numbers
options:
    operation:
        description: Math operation to perform
        required: true
        type: str
        choices: ['add', 'subtract', 'multiply', 'divide']
    num1:
        description: First number
        required: true
        type: float
    num2:
        description: Second number
        required: true
        type: float
'''

EXAMPLES = r'''
- name: Add two numbers
  calculator:
    operation: add
    num1: 10
    num2: 5
  register: result

- name: Show result
  debug:
    msg: "10 + 5 = {{ result.answer }}"
'''

RETURN = r'''
answer:
    description: The calculation result
    type: float
    returned: success
expression:
    description: The full expression
    type: str
    returned: success
'''

from ansible.module_utils.basic import AnsibleModule


def main():
    module_args = dict(
        operation=dict(
            type='str', 
            required=True,
            choices=['add', 'subtract', 'multiply', 'divide']
        ),
        num1=dict(type='float', required=True),
        num2=dict(type='float', required=True)
    )
    
    module = AnsibleModule(
        argument_spec=module_args,
        supports_check_mode=True
    )
    
    # Get parameters
    operation = module.params['operation']
    num1 = module.params['num1']
    num2 = module.params['num2']
    
    result = dict(changed=False)
    
    # Perform the calculation
    try:
        if operation == 'add':
            answer = num1 + num2
            symbol = '+'
        elif operation == 'subtract':
            answer = num1 - num2
            symbol = '-'
        elif operation == 'multiply':
            answer = num1 * num2
            symbol = '*'
        elif operation == 'divide':
            if num2 == 0:
                module.fail_json(msg="Cannot divide by zero!")
            answer = num1 / num2
            symbol = '/'
            
        result['answer'] = answer
        result['expression'] = f"{num1} {symbol} {num2} = {answer}"
        
        module.exit_json(**result)
        
    except Exception as e:
        module.fail_json(msg=str(e))


if __name__ == '__main__':
    main()
```

---

### Example 3: Environment Variable Manager

**Purpose**: Learn about state management (present/absent pattern).

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

# library/env_manager.py
# Manage environment variables in a file

DOCUMENTATION = r'''
---
module: env_manager
short_description: Manage environment variables in a .env file
description:
    - Add or remove environment variables from a file
    - Commonly used pattern in Ansible modules
options:
    path:
        description: Path to the .env file
        required: true
        type: str
    name:
        description: Variable name
        required: true
        type: str
    value:
        description: Variable value (required when state=present)
        type: str
    state:
        description: Whether the variable should exist
        type: str
        choices: ['present', 'absent']
        default: present
'''

EXAMPLES = r'''
- name: Set database URL
  env_manager:
    path: /opt/app/.env
    name: DATABASE_URL
    value: "postgresql://localhost:5432/mydb"
    state: present

- name: Remove old variable
  env_manager:
    path: /opt/app/.env
    name: OLD_SETTING
    state: absent
'''

import os
import re
from ansible.module_utils.basic import AnsibleModule


def read_env_file(path):
    """Read and parse .env file"""
    env_vars = {}
    if os.path.exists(path):
        with open(path, 'r') as f:
            for line in f:
                line = line.strip()
                if line and not line.startswith('#') and '=' in line:
                    key, value = line.split('=', 1)
                    env_vars[key.strip()] = value.strip()
    return env_vars


def write_env_file(path, env_vars):
    """Write environment variables to file"""
    with open(path, 'w') as f:
        for key, value in sorted(env_vars.items()):
            f.write(f"{key}={value}\n")


def main():
    module_args = dict(
        path=dict(type='str', required=True),
        name=dict(type='str', required=True),
        value=dict(type='str'),
        state=dict(type='str', default='present', choices=['present', 'absent'])
    )
    
    module = AnsibleModule(
        argument_spec=module_args,
        supports_check_mode=True,
        # If state is 'present', value is required
        required_if=[
            ('state', 'present', ['value'])
        ]
    )
    
    path = module.params['path']
    name = module.params['name']
    value = module.params['value']
    state = module.params['state']
    
    result = dict(
        changed=False,
        path=path,
        name=name
    )
    
    # Read current state
    env_vars = read_env_file(path)
    
    if state == 'present':
        # Check if we need to make changes
        if name not in env_vars or env_vars[name] != value:
            result['changed'] = True
            result['old_value'] = env_vars.get(name)
            result['new_value'] = value
            
            if not module.check_mode:
                env_vars[name] = value
                write_env_file(path, env_vars)
                
    elif state == 'absent':
        if name in env_vars:
            result['changed'] = True
            result['removed_value'] = env_vars[name]
            
            if not module.check_mode:
                del env_vars[name]
                write_env_file(path, env_vars)
    
    module.exit_json(**result)


if __name__ == '__main__':
    main()
```

---

### Example 4: Service Health Checker

**Purpose**: Learn how to run system commands from a module.

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

# library/service_health.py
# Check health of system services

DOCUMENTATION = r'''
---
module: service_health
short_description: Check health of a system service
description:
    - Checks if a service is running
    - Checks if service is enabled
    - Returns comprehensive health status
options:
    name:
        description: Service name
        required: true
        type: str
    check_port:
        description: Optional port to check
        type: int
'''

EXAMPLES = r'''
- name: Check nginx health
  service_health:
    name: nginx
    check_port: 80
  register: health

- name: Alert if unhealthy
  debug:
    msg: "ALERT: {{ health.name }} is not running!"
  when: not health.is_running
'''

RETURN = r'''
is_running:
    description: Whether service is running
    type: bool
is_enabled:
    description: Whether service is enabled at boot
    type: bool
port_open:
    description: Whether the port is open (if check_port specified)
    type: bool
status_output:
    description: Raw status output
    type: str
'''

import socket
from ansible.module_utils.basic import AnsibleModule


def check_port(port, host='localhost', timeout=3):
    """Check if a port is open"""
    try:
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.settimeout(timeout)
        result = sock.connect_ex((host, port))
        sock.close()
        return result == 0
    except:
        return False


def main():
    module_args = dict(
        name=dict(type='str', required=True),
        check_port=dict(type='int')
    )
    
    module = AnsibleModule(
        argument_spec=module_args,
        supports_check_mode=True
    )
    
    name = module.params['name']
    port = module.params['check_port']
    
    result = dict(
        changed=False,
        name=name,
        is_running=False,
        is_enabled=False
    )
    
    # Check if service is running using systemctl
    # module.run_command() is the safe way to run commands
    rc, stdout, stderr = module.run_command(['systemctl', 'is-active', name])
    result['is_running'] = (rc == 0)
    result['status_output'] = stdout.strip()
    
    # Check if service is enabled
    rc, stdout, stderr = module.run_command(['systemctl', 'is-enabled', name])
    result['is_enabled'] = (rc == 0)
    
    # Check port if specified
    if port:
        result['port_open'] = check_port(port)
        result['checked_port'] = port
    
    # Overall health
    result['healthy'] = result['is_running']
    if port:
        result['healthy'] = result['healthy'] and result['port_open']
    
    module.exit_json(**result)


if __name__ == '__main__':
    main()
```

---

### Example 5: User Input Validator

**Purpose**: Learn about input validation and error handling.

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

# library/validate_input.py
# Validate various types of user input

DOCUMENTATION = r'''
---
module: validate_input
short_description: Validate user input against rules
description:
    - Validate email addresses, URLs, IP addresses, etc.
    - Use this to verify configuration before deployment
options:
    value:
        description: The value to validate
        required: true
        type: str
    type:
        description: Type of validation to perform
        required: true
        type: str
        choices: ['email', 'url', 'ip', 'hostname', 'port', 'username']
    fail_on_invalid:
        description: Whether to fail if validation fails
        type: bool
        default: false
'''

EXAMPLES = r'''
- name: Validate email address
  validate_input:
    value: "user@example.com"
    type: email
  register: email_check

- name: Validate IP address (fail if invalid)
  validate_input:
    value: "{{ server_ip }}"
    type: ip
    fail_on_invalid: true
'''

RETURN = r'''
valid:
    description: Whether the value is valid
    type: bool
value:
    description: The validated value
    type: str
error:
    description: Error message if invalid
    type: str
'''

import re
from ansible.module_utils.basic import AnsibleModule


def validate_email(value):
    """Validate email address format"""
    pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    if re.match(pattern, value):
        return True, None
    return False, f"'{value}' is not a valid email address"


def validate_ip(value):
    """Validate IPv4 address"""
    pattern = r'^(\d{1,3}\.){3}\d{1,3}$'
    if re.match(pattern, value):
        parts = value.split('.')
        if all(0 <= int(part) <= 255 for part in parts):
            return True, None
    return False, f"'{value}' is not a valid IP address"


def validate_url(value):
    """Validate URL format"""
    pattern = r'^https?://[^\s/$.?#].[^\s]*$'
    if re.match(pattern, value):
        return True, None
    return False, f"'{value}' is not a valid URL"


def validate_hostname(value):
    """Validate hostname format"""
    pattern = r'^[a-zA-Z0-9]([a-zA-Z0-9\-]{0,61}[a-zA-Z0-9])?(\.[a-zA-Z]{2,})+$'
    if re.match(pattern, value):
        return True, None
    return False, f"'{value}' is not a valid hostname"


def validate_port(value):
    """Validate port number"""
    try:
        port = int(value)
        if 1 <= port <= 65535:
            return True, None
        return False, f"Port {value} is out of range (1-65535)"
    except ValueError:
        return False, f"'{value}' is not a valid port number"


def validate_username(value):
    """Validate username format"""
    pattern = r'^[a-z_][a-z0-9_-]{0,31}$'
    if re.match(pattern, value):
        return True, None
    return False, f"'{value}' is not a valid username (lowercase, start with letter)"


def main():
    validators = {
        'email': validate_email,
        'ip': validate_ip,
        'url': validate_url,
        'hostname': validate_hostname,
        'port': validate_port,
        'username': validate_username
    }
    
    module_args = dict(
        value=dict(type='str', required=True),
        type=dict(type='str', required=True, choices=list(validators.keys())),
        fail_on_invalid=dict(type='bool', default=False)
    )
    
    module = AnsibleModule(
        argument_spec=module_args,
        supports_check_mode=True
    )
    
    value = module.params['value']
    val_type = module.params['type']
    fail_on_invalid = module.params['fail_on_invalid']
    
    # Run the appropriate validator
    validator = validators[val_type]
    is_valid, error = validator(value)
    
    result = dict(
        changed=False,
        valid=is_valid,
        value=value,
        type=val_type
    )
    
    if not is_valid:
        result['error'] = error
        if fail_on_invalid:
            module.fail_json(msg=error, **result)
    
    module.exit_json(**result)


if __name__ == '__main__':
    main()
```

---

## Intermediate Examples

### Example 6: REST API Client Module

**Purpose**: Learn how to interact with external APIs.

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

# library/api_resource.py
# Manage resources through a REST API

DOCUMENTATION = r'''
---
module: api_resource
short_description: Manage resources via REST API
description:
    - Create, read, update, delete resources through a REST API
    - Generic module for any REST endpoint
options:
    url:
        description: API endpoint URL
        required: true
        type: str
    method:
        description: HTTP method
        type: str
        choices: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH']
        default: GET
    headers:
        description: HTTP headers
        type: dict
        default: {}
    body:
        description: Request body (for POST/PUT/PATCH)
        type: dict
    auth_token:
        description: Bearer token for authentication
        type: str
        no_log: true
    timeout:
        description: Request timeout in seconds
        type: int
        default: 30
    validate_certs:
        description: Validate SSL certificates
        type: bool
        default: true
'''

EXAMPLES = r'''
# GET request
- name: Get user info
  api_resource:
    url: https://api.example.com/users/123
    method: GET
    auth_token: "{{ api_token }}"
  register: user

# POST request
- name: Create new user
  api_resource:
    url: https://api.example.com/users
    method: POST
    auth_token: "{{ api_token }}"
    body:
      name: John Doe
      email: john@example.com
  register: new_user

# DELETE request
- name: Delete user
  api_resource:
    url: https://api.example.com/users/123
    method: DELETE
    auth_token: "{{ api_token }}"
'''

RETURN = r'''
status_code:
    description: HTTP status code
    type: int
response:
    description: Response body (parsed JSON)
    type: dict
headers:
    description: Response headers
    type: dict
'''

import json
from ansible.module_utils.basic import AnsibleModule
from ansible.module_utils.urls import fetch_url


def main():
    module_args = dict(
        url=dict(type='str', required=True),
        method=dict(type='str', default='GET', 
                   choices=['GET', 'POST', 'PUT', 'DELETE', 'PATCH']),
        headers=dict(type='dict', default={}),
        body=dict(type='dict'),
        auth_token=dict(type='str', no_log=True),  # no_log hides from output!
        timeout=dict(type='int', default=30),
        validate_certs=dict(type='bool', default=True)
    )
    
    module = AnsibleModule(
        argument_spec=module_args,
        supports_check_mode=True
    )
    
    url = module.params['url']
    method = module.params['method']
    headers = module.params['headers'].copy()
    body = module.params['body']
    auth_token = module.params['auth_token']
    timeout = module.params['timeout']
    
    # Add content-type header
    headers['Content-Type'] = 'application/json'
    
    # Add authorization header if token provided
    if auth_token:
        headers['Authorization'] = f'Bearer {auth_token}'
    
    # Check mode - don't make actual request for mutating operations
    if module.check_mode and method in ['POST', 'PUT', 'DELETE', 'PATCH']:
        module.exit_json(
            changed=True,
            msg=f"Would {method} to {url}",
            check_mode=True
        )
    
    # Prepare body
    data = json.dumps(body) if body else None
    
    # Make the request using Ansible's fetch_url
    # This handles SSL, proxies, and other connection settings
    response, info = fetch_url(
        module,
        url,
        method=method,
        headers=headers,
        data=data,
        timeout=timeout
    )
    
    status_code = info['status']
    
    result = dict(
        changed=method in ['POST', 'PUT', 'DELETE', 'PATCH'],
        status_code=status_code,
        url=url,
        method=method
    )
    
    # Parse response body
    if response:
        response_body = response.read()
        try:
            result['response'] = json.loads(response_body)
        except json.JSONDecodeError:
            result['response'] = response_body.decode('utf-8')
    else:
        result['response'] = None
    
    # Check for errors
    if status_code >= 400:
        module.fail_json(
            msg=f"API request failed with status {status_code}",
            **result
        )
    
    module.exit_json(**result)


if __name__ == '__main__':
    main()
```

---

### Example 7: Configuration File Manager

**Purpose**: Learn how to manage files idempotently.

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

# library/config_manager.py
# Manage configuration files (JSON, YAML, INI)

DOCUMENTATION = r'''
---
module: config_manager
short_description: Manage configuration files
description:
    - Read, write, and update configuration files
    - Supports JSON, YAML, and INI formats
    - Idempotent - only changes when necessary
options:
    path:
        description: Path to the configuration file
        required: true
        type: path
    format:
        description: File format
        required: true
        type: str
        choices: ['json', 'yaml', 'ini']
    settings:
        description: Settings to ensure exist in the file
        type: dict
        default: {}
    state:
        description: State of the file
        type: str
        choices: ['present', 'absent']
        default: present
    backup:
        description: Create backup before modifying
        type: bool
        default: true
'''

EXAMPLES = r'''
# Update JSON config
- name: Update app configuration
  config_manager:
    path: /opt/app/config.json
    format: json
    settings:
      database:
        host: localhost
        port: 5432
      logging:
        level: info
    backup: true

# Update YAML config
- name: Update Kubernetes config
  config_manager:
    path: /etc/myapp/config.yaml
    format: yaml
    settings:
      replicas: 3
      image: myapp:v2.0
'''

import os
import json
import shutil
from datetime import datetime
from ansible.module_utils.basic import AnsibleModule

# Try to import yaml (might not be available)
try:
    import yaml
    HAS_YAML = True
except ImportError:
    HAS_YAML = False


def deep_merge(base, override):
    """Deep merge two dictionaries"""
    result = base.copy()
    for key, value in override.items():
        if key in result and isinstance(result[key], dict) and isinstance(value, dict):
            result[key] = deep_merge(result[key], value)
        else:
            result[key] = value
    return result


def read_config(path, fmt):
    """Read configuration file"""
    if not os.path.exists(path):
        return {}
        
    with open(path, 'r') as f:
        content = f.read()
        
    if fmt == 'json':
        return json.loads(content) if content.strip() else {}
    elif fmt == 'yaml':
        return yaml.safe_load(content) or {}
    elif fmt == 'ini':
        # Simple INI parser
        config = {}
        section = 'default'
        for line in content.split('\n'):
            line = line.strip()
            if line.startswith('[') and line.endswith(']'):
                section = line[1:-1]
                config[section] = {}
            elif '=' in line and not line.startswith('#'):
                key, value = line.split('=', 1)
                if section not in config:
                    config[section] = {}
                config[section][key.strip()] = value.strip()
        return config
    
    return {}


def write_config(path, data, fmt, backup=True):
    """Write configuration file"""
    # Create backup if file exists
    if backup and os.path.exists(path):
        timestamp = datetime.now().strftime('%Y%m%d%H%M%S')
        backup_path = f"{path}.{timestamp}.bak"
        shutil.copy2(path, backup_path)
    
    # Ensure directory exists
    os.makedirs(os.path.dirname(path), exist_ok=True)
    
    with open(path, 'w') as f:
        if fmt == 'json':
            json.dump(data, f, indent=2)
        elif fmt == 'yaml':
            yaml.safe_dump(data, f, default_flow_style=False)
        elif fmt == 'ini':
            for section, values in data.items():
                f.write(f"[{section}]\n")
                for key, value in values.items():
                    f.write(f"{key}={value}\n")
                f.write("\n")


def main():
    module_args = dict(
        path=dict(type='path', required=True),
        format=dict(type='str', required=True, choices=['json', 'yaml', 'ini']),
        settings=dict(type='dict', default={}),
        state=dict(type='str', default='present', choices=['present', 'absent']),
        backup=dict(type='bool', default=True)
    )
    
    module = AnsibleModule(
        argument_spec=module_args,
        supports_check_mode=True
    )
    
    path = module.params['path']
    fmt = module.params['format']
    settings = module.params['settings']
    state = module.params['state']
    backup = module.params['backup']
    
    # Check for yaml support
    if fmt == 'yaml' and not HAS_YAML:
        module.fail_json(msg="PyYAML is required for YAML format")
    
    result = dict(
        changed=False,
        path=path,
        format=fmt
    )
    
    if state == 'absent':
        if os.path.exists(path):
            result['changed'] = True
            if not module.check_mode:
                os.remove(path)
        module.exit_json(**result)
    
    # Read current config
    current = read_config(path, fmt)
    
    # Merge with new settings
    desired = deep_merge(current, settings)
    
    # Check if changes needed
    if current != desired:
        result['changed'] = True
        result['diff'] = {
            'before': current,
            'after': desired
        }
        
        if not module.check_mode:
            write_config(path, desired, fmt, backup)
    
    result['config'] = desired
    module.exit_json(**result)


if __name__ == '__main__':
    main()
```

---

## Packaging Your Modules

### Method 1: Local Library Folder (Quick Start)

```
my_project/
├── library/                # Ansible looks here automatically
│   ├── my_module.py
│   └── another_module.py
├── playbook.yml
└── ansible.cfg
```

**ansible.cfg:**
```ini
[defaults]
library = ./library
```

### Method 2: Ansible Collection (Recommended for Sharing)

**Step 1: Create Collection Structure**
```bash
# Create collection skeleton
ansible-galaxy collection init mycompany.mymodules

# This creates:
mycompany/
└── mymodules/
    ├── README.md
    ├── galaxy.yml              # Collection metadata
    ├── plugins/
    │   └── modules/            # Your modules go here
    │       ├── __init__.py
    │       └── my_module.py
    ├── roles/
    ├── playbooks/
    └── docs/
```

**Step 2: Configure galaxy.yml**
```yaml
# galaxy.yml
namespace: mycompany
name: mymodules
version: 1.0.0
readme: README.md
authors:
  - Your Name <your.email@example.com>
description: My custom Ansible modules
license_file: LICENSE
tags:
  - automation
  - infrastructure
dependencies: {}
repository: https://github.com/yourname/ansible-mymodules
```

**Step 3: Add Your Modules**
```bash
# Copy your modules
cp my_module.py mycompany/mymodules/plugins/modules/
```

**Step 4: Build the Collection**
```bash
cd mycompany/mymodules
ansible-galaxy collection build

# Creates: mycompany-mymodules-1.0.0.tar.gz
```

**Step 5: Install Locally**
```bash
ansible-galaxy collection install mycompany-mymodules-1.0.0.tar.gz
```

**Step 6: Use in Playbook**
```yaml
---
- hosts: all
  tasks:
    - name: Use my custom module
      mycompany.mymodules.my_module:
        param1: value1
```

### Method 3: Publish to Ansible Galaxy

```bash
# 1. Create account at galaxy.ansible.com

# 2. Get your API token from galaxy.ansible.com/me/preferences

# 3. Publish
ansible-galaxy collection publish mycompany-mymodules-1.0.0.tar.gz --api-key YOUR_API_KEY
```

---

## Deploying to Production

### Option 1: Include in Your Repository

```
your_ansible_repo/
├── collections/
│   └── ansible_collections/
│       └── mycompany/
│           └── mymodules/
│               └── plugins/modules/
│                   └── my_module.py
├── playbooks/
├── inventory/
└── requirements.yml
```

**requirements.yml:**
```yaml
collections:
  - name: ./collections/ansible_collections/mycompany/mymodules
    type: dir
```

### Option 2: Git Repository

**requirements.yml:**
```yaml
collections:
  - name: https://github.com/yourcompany/ansible-modules.git
    type: git
    version: v1.0.0
```

**Install:**
```bash
ansible-galaxy collection install -r requirements.yml
```

### Option 3: Private Galaxy Server (Enterprise)

```yaml
# requirements.yml
collections:
  - name: mycompany.mymodules
    version: ">=1.0.0"
    source: https://galaxy.internal.company.com
```

### Option 4: Automation Hub (Red Hat)

For Red Hat customers using Ansible Automation Platform:
```yaml
collections:
  - name: mycompany.mymodules
    source: https://console.redhat.com/api/automation-hub/
```

---

## Complete Project Example

Here's a complete example showing everything together:

### Directory Structure
```
my_ansible_modules/
├── README.md
├── galaxy.yml
├── requirements.txt              # Python dependencies
├── plugins/
│   └── modules/
│       ├── __init__.py
│       ├── file_checker.py
│       ├── env_manager.py
│       ├── api_resource.py
│       └── config_manager.py
├── tests/
│   ├── unit/
│   │   └── test_file_checker.py
│   └── integration/
│       ├── inventory
│       └── test_modules.yml
├── docs/
│   ├── file_checker.md
│   └── env_manager.md
└── examples/
    └── example_playbook.yml
```

### requirements.txt
```
pyyaml>=5.0
requests>=2.25.0
```

### tests/integration/test_modules.yml
```yaml
---
- name: Test custom modules
  hosts: localhost
  connection: local
  
  tasks:
    - name: Test file_checker
      file_checker:
        path: /etc/hosts
      register: result
      
    - name: Verify result
      assert:
        that:
          - result.exists == true
        fail_msg: "file_checker module failed"
        
    - name: Test env_manager
      env_manager:
        path: /tmp/test.env
        name: TEST_VAR
        value: "hello"
        state: present
      register: env_result
      
    - name: Verify env was created
      assert:
        that:
          - env_result.changed == true
```

### Running Tests
```bash
# Unit tests
python -m pytest tests/unit/

# Integration tests
ansible-playbook tests/integration/test_modules.yml -v

# Test idempotency (run twice)
ansible-playbook tests/integration/test_modules.yml
ansible-playbook tests/integration/test_modules.yml  # Should show changed=0
```

---

## Summary Checklist

### Before Your Module is Ready

- [ ] Has DOCUMENTATION block
- [ ] Has EXAMPLES block
- [ ] Has RETURN block
- [ ] Supports check_mode
- [ ] Uses `no_log=True` for sensitive parameters
- [ ] Has proper error handling with `fail_json()`
- [ ] Is idempotent (running twice doesn't cause issues)
- [ ] Has unit tests
- [ ] Has integration tests
- [ ] Documentation is written

### Module Quality Levels

| Level | Requirements |
|-------|--------------|
| **Basic** | Works correctly, has documentation |
| **Good** | + Check mode, error handling, tests |
| **Production** | + Idempotent, no_log for secrets, CI/CD |
| **Professional** | + Published collection, community feedback |

---

*Now you're ready to build, package, and deploy your own Ansible modules!*
