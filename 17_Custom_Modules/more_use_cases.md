# More Custom Module Examples

## Industry-Specific Module Use Cases

---

## Use Case 5: Cloud Cost Optimizer Module

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

DOCUMENTATION = r'''
---
module: cloud_cost_optimizer
short_description: Analyze and optimize cloud resource costs
description:
    - Scan cloud resources for cost optimization opportunities.
    - Identify unused, underutilized, or oversized resources.
    - Generate recommendations and optionally apply changes.
options:
    provider:
        description: Cloud provider.
        choices: ['aws', 'azure', 'gcp']
        required: true
        type: str
    regions:
        description: Regions to scan.
        type: list
        elements: str
        default: ['all']
    analysis_types:
        description: Types of analysis to perform.
        type: list
        elements: str
        default: ['unused', 'rightsizing', 'reserved', 'spot']
    min_savings_threshold:
        description: Minimum savings threshold to report (USD).
        type: float
        default: 10.0
    auto_remediate:
        description: Automatically apply safe optimizations.
        type: bool
        default: false
    safe_tags:
        description: Resources with these tags are safe to modify.
        type: list
        elements: str
        default: ['auto-optimize=true']
'''

EXAMPLES = r'''
- name: Analyze AWS costs
  cloud_cost_optimizer:
    provider: aws
    regions: ['us-east-1', 'us-west-2']
    analysis_types:
      - unused
      - rightsizing
    min_savings_threshold: 50.0
  register: cost_analysis

- name: Apply safe optimizations
  cloud_cost_optimizer:
    provider: aws
    auto_remediate: true
    safe_tags:
      - 'Environment=dev'
      - 'auto-optimize=true'
'''

RETURN = r'''
total_monthly_savings:
    description: Total potential monthly savings.
    type: float
    returned: always
recommendations:
    description: List of optimization recommendations.
    type: list
    returned: always
actions_taken:
    description: Actions taken if auto_remediate was true.
    type: list
    returned: when auto_remediate=true
'''

import json
from ansible.module_utils.basic import AnsibleModule

try:
    import boto3
    HAS_BOTO3 = True
except ImportError:
    HAS_BOTO3 = False


class AWSCostOptimizer:
    def __init__(self, regions):
        self.regions = regions if 'all' not in regions else self._get_all_regions()
        self.recommendations = []
        self.total_savings = 0
        
    def _get_all_regions(self):
        ec2 = boto3.client('ec2', region_name='us-east-1')
        regions = [r['RegionName'] for r in ec2.describe_regions()['Regions']]
        return regions
        
    def find_unused_ebs_volumes(self):
        """Find unattached EBS volumes."""
        for region in self.regions:
            ec2 = boto3.client('ec2', region_name=region)
            volumes = ec2.describe_volumes(
                Filters=[{'Name': 'status', 'Values': ['available']}]
            )['Volumes']
            
            for vol in volumes:
                # Estimate cost (simplified)
                size_gb = vol['Size']
                monthly_cost = size_gb * 0.10  # Approximate gp2 cost
                
                self.recommendations.append({
                    'type': 'unused_ebs',
                    'resource_id': vol['VolumeId'],
                    'region': region,
                    'description': f"Unattached EBS volume ({size_gb} GB)",
                    'monthly_savings': monthly_cost,
                    'action': 'delete_volume',
                    'risk': 'low'
                })
                self.total_savings += monthly_cost
                
    def find_unused_elastic_ips(self):
        """Find unassociated Elastic IPs."""
        for region in self.regions:
            ec2 = boto3.client('ec2', region_name=region)
            addresses = ec2.describe_addresses()['Addresses']
            
            for addr in addresses:
                if 'AssociationId' not in addr:
                    monthly_cost = 3.60  # ~$0.005/hour
                    
                    self.recommendations.append({
                        'type': 'unused_eip',
                        'resource_id': addr['AllocationId'],
                        'region': region,
                        'description': f"Unassociated Elastic IP {addr['PublicIp']}",
                        'monthly_savings': monthly_cost,
                        'action': 'release_address',
                        'risk': 'low'
                    })
                    self.total_savings += monthly_cost
                    
    def find_oversized_instances(self):
        """Find EC2 instances with low CPU utilization."""
        for region in self.regions:
            ec2 = boto3.client('ec2', region_name=region)
            cloudwatch = boto3.client('cloudwatch', region_name=region)
            
            instances = ec2.describe_instances(
                Filters=[{'Name': 'instance-state-name', 'Values': ['running']}]
            )
            
            for reservation in instances['Reservations']:
                for instance in reservation['Instances']:
                    instance_id = instance['InstanceId']
                    instance_type = instance['InstanceType']
                    
                    # Get average CPU over last 7 days
                    from datetime import datetime, timedelta
                    response = cloudwatch.get_metric_statistics(
                        Namespace='AWS/EC2',
                        MetricName='CPUUtilization',
                        Dimensions=[{'Name': 'InstanceId', 'Value': instance_id}],
                        StartTime=datetime.utcnow() - timedelta(days=7),
                        EndTime=datetime.utcnow(),
                        Period=86400,
                        Statistics=['Average']
                    )
                    
                    if response['Datapoints']:
                        avg_cpu = sum(d['Average'] for d in response['Datapoints']) / len(response['Datapoints'])
                        
                        if avg_cpu < 10:  # Less than 10% average CPU
                            # Suggest downsizing
                            self.recommendations.append({
                                'type': 'oversized_instance',
                                'resource_id': instance_id,
                                'region': region,
                                'description': f"Instance {instance_type} averaging {avg_cpu:.1f}% CPU",
                                'monthly_savings': self._estimate_downsize_savings(instance_type),
                                'action': 'downsize_instance',
                                'suggested_type': self._suggest_smaller_instance(instance_type),
                                'risk': 'medium'
                            })
                            
    def find_old_snapshots(self):
        """Find EBS snapshots older than 30 days."""
        from datetime import datetime, timezone, timedelta
        
        for region in self.regions:
            ec2 = boto3.client('ec2', region_name=region)
            sts = boto3.client('sts')
            account_id = sts.get_caller_identity()['Account']
            
            snapshots = ec2.describe_snapshots(OwnerIds=[account_id])['Snapshots']
            cutoff_date = datetime.now(timezone.utc) - timedelta(days=30)
            
            for snap in snapshots:
                if snap['StartTime'] < cutoff_date:
                    size_gb = snap['VolumeSize']
                    monthly_cost = size_gb * 0.05  # EBS snapshot pricing
                    
                    self.recommendations.append({
                        'type': 'old_snapshot',
                        'resource_id': snap['SnapshotId'],
                        'region': region,
                        'description': f"Snapshot {size_gb}GB from {snap['StartTime'].strftime('%Y-%m-%d')}",
                        'monthly_savings': monthly_cost,
                        'action': 'delete_snapshot',
                        'risk': 'medium'
                    })
                    self.total_savings += monthly_cost
                    
    def _estimate_downsize_savings(self, instance_type):
        """Estimate monthly savings from downsizing."""
        # Simplified pricing
        prices = {
            't3.large': 60, 't3.medium': 30,
            'm5.large': 70, 'm5.xlarge': 140,
            'c5.large': 62, 'c5.xlarge': 124
        }
        current_price = prices.get(instance_type, 100)
        return current_price * 0.5  # Assume 50% savings
        
    def _suggest_smaller_instance(self, current_type):
        """Suggest a smaller instance type."""
        downsizing_map = {
            't3.large': 't3.medium',
            't3.xlarge': 't3.large',
            'm5.xlarge': 'm5.large',
            'm5.2xlarge': 'm5.xlarge',
            'c5.xlarge': 'c5.large',
            'c5.2xlarge': 'c5.xlarge'
        }
        return downsizing_map.get(current_type, 't3.small')
        
    def run_analysis(self, analysis_types):
        """Run specified analyses."""
        if 'unused' in analysis_types:
            self.find_unused_ebs_volumes()
            self.find_unused_elastic_ips()
            
        if 'rightsizing' in analysis_types:
            self.find_oversized_instances()
            
        if 'snapshots' in analysis_types:
            self.find_old_snapshots()
            
        return self.recommendations, self.total_savings


def main():
    module_args = dict(
        provider=dict(type='str', required=True, choices=['aws', 'azure', 'gcp']),
        regions=dict(type='list', elements='str', default=['all']),
        analysis_types=dict(type='list', elements='str', 
                           default=['unused', 'rightsizing', 'reserved', 'spot']),
        min_savings_threshold=dict(type='float', default=10.0),
        auto_remediate=dict(type='bool', default=False),
        safe_tags=dict(type='list', elements='str', default=['auto-optimize=true'])
    )

    module = AnsibleModule(
        argument_spec=module_args,
        supports_check_mode=True
    )

    if not HAS_BOTO3 and module.params['provider'] == 'aws':
        module.fail_json(msg="boto3 is required for AWS analysis")

    result = dict(
        changed=False,
        total_monthly_savings=0,
        recommendations=[],
        actions_taken=[]
    )

    try:
        if module.params['provider'] == 'aws':
            optimizer = AWSCostOptimizer(module.params['regions'])
            recommendations, total_savings = optimizer.run_analysis(
                module.params['analysis_types']
            )
            
            # Filter by threshold
            result['recommendations'] = [
                r for r in recommendations 
                if r['monthly_savings'] >= module.params['min_savings_threshold']
            ]
            result['total_monthly_savings'] = sum(
                r['monthly_savings'] for r in result['recommendations']
            )
            
        module.exit_json(**result)

    except Exception as e:
        module.fail_json(msg=str(e), **result)


if __name__ == '__main__':
    main()
```

---

## Use Case 6: SSL Certificate Manager Module

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

DOCUMENTATION = r'''
---
module: ssl_certificate_manager
short_description: Manage SSL/TLS certificates
description:
    - Check certificate expiration dates.
    - Generate CSRs.
    - Deploy certificates to servers.
    - Integrate with Let's Encrypt.
options:
    action:
        description: Action to perform.
        choices: ['check', 'generate_csr', 'deploy', 'renew']
        required: true
        type: str
    domain:
        description: Domain name for the certificate.
        type: str
    certificate_path:
        description: Path to certificate file.
        type: str
    key_path:
        description: Path to private key file.
        type: str
    expiry_warning_days:
        description: Days before expiry to warn.
        type: int
        default: 30
    auto_renew:
        description: Automatically renew expiring certificates.
        type: bool
        default: false
    acme_email:
        description: Email for Let's Encrypt registration.
        type: str
    webroot_path:
        description: Webroot path for ACME challenge.
        type: str
'''

EXAMPLES = r'''
- name: Check certificate expiration
  ssl_certificate_manager:
    action: check
    certificate_path: /etc/ssl/certs/mysite.crt
    expiry_warning_days: 30
  register: cert_status

- name: Generate CSR
  ssl_certificate_manager:
    action: generate_csr
    domain: example.com
    key_path: /etc/ssl/private/example.key

- name: Renew with Let's Encrypt
  ssl_certificate_manager:
    action: renew
    domain: example.com
    acme_email: admin@example.com
    webroot_path: /var/www/html
    auto_renew: true
'''

RETURN = r'''
valid:
    description: Whether certificate is valid.
    type: bool
    returned: on check
days_until_expiry:
    description: Days until certificate expires.
    type: int
    returned: on check
subject:
    description: Certificate subject.
    type: str
    returned: on check
issuer:
    description: Certificate issuer.
    type: str
    returned: on check
csr_path:
    description: Path to generated CSR.
    type: str
    returned: on generate_csr
'''

import os
import subprocess
from datetime import datetime
from ansible.module_utils.basic import AnsibleModule

try:
    from cryptography import x509
    from cryptography.x509.oid import NameOID
    from cryptography.hazmat.primitives import hashes, serialization
    from cryptography.hazmat.primitives.asymmetric import rsa
    from cryptography.hazmat.backends import default_backend
    HAS_CRYPTOGRAPHY = True
except ImportError:
    HAS_CRYPTOGRAPHY = False


class SSLManager:
    def __init__(self, module):
        self.module = module
        
    def check_certificate(self, cert_path, warning_days):
        """Check certificate validity and expiration."""
        if not os.path.exists(cert_path):
            return {
                'valid': False,
                'error': f'Certificate file not found: {cert_path}'
            }
            
        with open(cert_path, 'rb') as f:
            cert_data = f.read()
            
        cert = x509.load_pem_x509_certificate(cert_data, default_backend())
        
        now = datetime.utcnow()
        days_until_expiry = (cert.not_valid_after - now).days
        
        # Get subject and issuer
        subject = cert.subject.get_attributes_for_oid(NameOID.COMMON_NAME)
        issuer = cert.issuer.get_attributes_for_oid(NameOID.ORGANIZATION_NAME)
        
        # Get SANs
        try:
            san = cert.extensions.get_extension_for_class(x509.SubjectAlternativeName)
            domains = [name.value for name in san.value.get_values_for_type(x509.DNSName)]
        except x509.ExtensionNotFound:
            domains = []
            
        return {
            'valid': now >= cert.not_valid_before and now <= cert.not_valid_after,
            'days_until_expiry': days_until_expiry,
            'expires_soon': days_until_expiry <= warning_days,
            'not_valid_before': cert.not_valid_before.isoformat(),
            'not_valid_after': cert.not_valid_after.isoformat(),
            'subject': subject[0].value if subject else 'Unknown',
            'issuer': issuer[0].value if issuer else 'Unknown',
            'domains': domains,
            'serial_number': str(cert.serial_number)
        }
        
    def generate_private_key(self, key_path, key_size=4096):
        """Generate a new RSA private key."""
        key = rsa.generate_private_key(
            public_exponent=65537,
            key_size=key_size,
            backend=default_backend()
        )
        
        key_pem = key.private_bytes(
            encoding=serialization.Encoding.PEM,
            format=serialization.PrivateFormat.PKCS8,
            encryption_algorithm=serialization.NoEncryption()
        )
        
        with open(key_path, 'wb') as f:
            f.write(key_pem)
            
        os.chmod(key_path, 0o600)
        return key
        
    def generate_csr(self, domain, key_path, additional_domains=None):
        """Generate a Certificate Signing Request."""
        # Generate or load key
        if os.path.exists(key_path):
            with open(key_path, 'rb') as f:
                key = serialization.load_pem_private_key(
                    f.read(),
                    password=None,
                    backend=default_backend()
                )
        else:
            key = self.generate_private_key(key_path)
            
        # Build CSR
        subject = x509.Name([
            x509.NameAttribute(NameOID.COMMON_NAME, domain),
        ])
        
        # Add SANs
        san_list = [x509.DNSName(domain)]
        if additional_domains:
            san_list.extend([x509.DNSName(d) for d in additional_domains])
            
        csr = x509.CertificateSigningRequestBuilder().subject_name(
            subject
        ).add_extension(
            x509.SubjectAlternativeName(san_list),
            critical=False
        ).sign(key, hashes.SHA256(), default_backend())
        
        csr_path = key_path.replace('.key', '.csr')
        with open(csr_path, 'wb') as f:
            f.write(csr.public_bytes(serialization.Encoding.PEM))
            
        return {
            'csr_path': csr_path,
            'key_path': key_path,
            'domain': domain,
            'additional_domains': additional_domains or []
        }
        
    def renew_letsencrypt(self, domain, email, webroot_path):
        """Renew certificate using Let's Encrypt (certbot)."""
        cmd = [
            'certbot', 'certonly',
            '--webroot',
            '-w', webroot_path,
            '-d', domain,
            '--email', email,
            '--agree-tos',
            '--non-interactive',
            '--keep-until-expiring'
        ]
        
        result = subprocess.run(
            cmd,
            capture_output=True,
            text=True
        )
        
        if result.returncode == 0:
            return {
                'renewed': True,
                'cert_path': f'/etc/letsencrypt/live/{domain}/fullchain.pem',
                'key_path': f'/etc/letsencrypt/live/{domain}/privkey.pem'
            }
        else:
            raise Exception(f"Certbot failed: {result.stderr}")


def main():
    module_args = dict(
        action=dict(type='str', required=True,
                   choices=['check', 'generate_csr', 'deploy', 'renew']),
        domain=dict(type='str'),
        certificate_path=dict(type='str'),
        key_path=dict(type='str'),
        expiry_warning_days=dict(type='int', default=30),
        auto_renew=dict(type='bool', default=False),
        acme_email=dict(type='str'),
        webroot_path=dict(type='str'),
        additional_domains=dict(type='list', elements='str', default=[])
    )

    module = AnsibleModule(
        argument_spec=module_args,
        required_if=[
            ('action', 'check', ['certificate_path']),
            ('action', 'generate_csr', ['domain', 'key_path']),
            ('action', 'renew', ['domain', 'acme_email', 'webroot_path']),
        ],
        supports_check_mode=True
    )

    if not HAS_CRYPTOGRAPHY:
        module.fail_json(msg="cryptography library is required")

    ssl_manager = SSLManager(module)
    action = module.params['action']
    
    result = dict(changed=False)

    try:
        if action == 'check':
            cert_info = ssl_manager.check_certificate(
                module.params['certificate_path'],
                module.params['expiry_warning_days']
            )
            result.update(cert_info)
            
            # Auto-renew if enabled and expiring soon
            if (module.params['auto_renew'] and 
                cert_info.get('expires_soon') and
                not module.check_mode):
                
                if module.params['acme_email'] and module.params['webroot_path']:
                    domain = cert_info['subject']
                    renew_result = ssl_manager.renew_letsencrypt(
                        domain,
                        module.params['acme_email'],
                        module.params['webroot_path']
                    )
                    result['renewal'] = renew_result
                    result['changed'] = True
                    
        elif action == 'generate_csr':
            if not module.check_mode:
                csr_result = ssl_manager.generate_csr(
                    module.params['domain'],
                    module.params['key_path'],
                    module.params['additional_domains']
                )
                result.update(csr_result)
            result['changed'] = True
            
        elif action == 'renew':
            if not module.check_mode:
                renew_result = ssl_manager.renew_letsencrypt(
                    module.params['domain'],
                    module.params['acme_email'],
                    module.params['webroot_path']
                )
                result.update(renew_result)
            result['changed'] = True

        module.exit_json(**result)

    except Exception as e:
        module.fail_json(msg=str(e), **result)


if __name__ == '__main__':
    main()
```

---

## Use Case 7: Log Analysis Module

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

DOCUMENTATION = r'''
---
module: log_analyzer
short_description: Analyze application logs for patterns
description:
    - Parse and analyze log files.
    - Detect error patterns and anomalies.
    - Generate reports and alerts.
options:
    log_path:
        description: Path to log file or directory.
        required: true
        type: str
    log_format:
        description: Log format pattern.
        choices: ['auto', 'apache', 'nginx', 'json', 'syslog', 'custom']
        default: auto
        type: str
    custom_pattern:
        description: Custom regex pattern for parsing.
        type: str
    analysis_type:
        description: Type of analysis.
        choices: ['errors', 'statistics', 'anomaly', 'security']
        default: errors
        type: str
    time_range:
        description: Time range to analyze (e.g., '1h', '24h', '7d').
        type: str
        default: '24h'
    alert_threshold:
        description: Number of errors to trigger alert.
        type: int
        default: 100
'''

EXAMPLES = r'''
- name: Analyze nginx error log
  log_analyzer:
    log_path: /var/log/nginx/error.log
    log_format: nginx
    analysis_type: errors
    time_range: 1h
  register: log_report

- name: Security analysis
  log_analyzer:
    log_path: /var/log/auth.log
    log_format: syslog
    analysis_type: security
    alert_threshold: 10
'''

RETURN = r'''
total_lines:
    description: Total lines analyzed.
    type: int
    returned: always
error_count:
    description: Number of errors found.
    type: int
    returned: always
top_errors:
    description: Most common errors.
    type: list
    returned: on errors analysis
alert_triggered:
    description: Whether alert threshold was exceeded.
    type: bool
    returned: always
'''

import os
import re
import json
from datetime import datetime, timedelta
from collections import Counter, defaultdict
from ansible.module_utils.basic import AnsibleModule


class LogAnalyzer:
    def __init__(self, module):
        self.module = module
        
        self.log_patterns = {
            'apache': r'(?P<ip>\S+) \S+ \S+ \[(?P<time>[^\]]+)\] "(?P<method>\S+) (?P<path>\S+) \S+" (?P<status>\d+) (?P<size>\S+)',
            'nginx': r'(?P<ip>\S+) - \S+ \[(?P<time>[^\]]+)\] "(?P<method>\S+) (?P<path>\S+) \S+" (?P<status>\d+) (?P<size>\d+)',
            'syslog': r'(?P<time>\w+\s+\d+\s+\d+:\d+:\d+) (?P<host>\S+) (?P<process>\S+): (?P<message>.+)',
            'json': None  # Parsed as JSON
        }
        
        self.error_patterns = [
            r'(?i)error',
            r'(?i)exception',
            r'(?i)fatal',
            r'(?i)failed',
            r'(?i)timeout',
            r'(?i)refused',
            r'(?i)denied'
        ]
        
        self.security_patterns = [
            (r'(?i)failed password', 'failed_login'),
            (r'(?i)invalid user', 'invalid_user'),
            (r'(?i)authentication failure', 'auth_failure'),
            (r'(?i)possible break-in', 'break_in_attempt'),
            (r'(?i)connection closed by .* \[preauth\]', 'preauth_close'),
            (r'(?i)too many authentication failures', 'brute_force')
        ]
        
    def parse_time_range(self, time_range):
        """Parse time range string to timedelta."""
        match = re.match(r'(\d+)([hdwm])', time_range)
        if not match:
            return timedelta(hours=24)
            
        value = int(match.group(1))
        unit = match.group(2)
        
        if unit == 'h':
            return timedelta(hours=value)
        elif unit == 'd':
            return timedelta(days=value)
        elif unit == 'w':
            return timedelta(weeks=value)
        elif unit == 'm':
            return timedelta(days=value * 30)
            
    def read_logs(self, log_path, time_range):
        """Read and filter log lines by time range."""
        cutoff_time = datetime.now() - self.parse_time_range(time_range)
        lines = []
        
        if os.path.isdir(log_path):
            log_files = [os.path.join(log_path, f) for f in os.listdir(log_path)
                        if f.endswith('.log')]
        else:
            log_files = [log_path]
            
        for log_file in log_files:
            if not os.path.exists(log_file):
                continue
                
            with open(log_file, 'r', errors='ignore') as f:
                for line in f:
                    lines.append(line.strip())
                    
        return lines
        
    def parse_line(self, line, log_format, custom_pattern=None):
        """Parse a log line according to format."""
        if log_format == 'json':
            try:
                return json.loads(line)
            except json.JSONDecodeError:
                return {'raw': line}
                
        pattern = self.log_patterns.get(log_format)
        if custom_pattern:
            pattern = custom_pattern
            
        if pattern:
            match = re.match(pattern, line)
            if match:
                return match.groupdict()
                
        return {'raw': line}
        
    def analyze_errors(self, lines, log_format):
        """Find and categorize errors."""
        errors = []
        error_types = Counter()
        
        for line in lines:
            for pattern in self.error_patterns:
                if re.search(pattern, line):
                    errors.append(line)
                    
                    # Categorize error
                    match = re.search(pattern, line, re.IGNORECASE)
                    if match:
                        error_types[match.group(0).lower()] += 1
                    break
                    
        # Get top errors by frequency
        error_messages = Counter(errors)
        top_errors = [
            {'message': msg[:200], 'count': count}
            for msg, count in error_messages.most_common(10)
        ]
        
        return {
            'error_count': len(errors),
            'error_types': dict(error_types),
            'top_errors': top_errors
        }
        
    def analyze_statistics(self, lines, log_format):
        """Generate log statistics."""
        parsed = [self.parse_line(l, log_format) for l in lines]
        
        stats = {
            'total_lines': len(lines),
            'unique_ips': set(),
            'status_codes': Counter(),
            'methods': Counter(),
            'top_paths': Counter()
        }
        
        for entry in parsed:
            if 'ip' in entry:
                stats['unique_ips'].add(entry['ip'])
            if 'status' in entry:
                stats['status_codes'][entry['status']] += 1
            if 'method' in entry:
                stats['methods'][entry['method']] += 1
            if 'path' in entry:
                # Normalize path (remove query strings, IDs)
                path = re.sub(r'\?.*', '', entry['path'])
                path = re.sub(r'/\d+', '/{id}', path)
                stats['top_paths'][path] += 1
                
        return {
            'total_lines': stats['total_lines'],
            'unique_ips': len(stats['unique_ips']),
            'status_codes': dict(stats['status_codes']),
            'methods': dict(stats['methods']),
            'top_paths': dict(stats['top_paths'].most_common(20))
        }
        
    def analyze_security(self, lines, log_format):
        """Security-focused analysis."""
        findings = defaultdict(list)
        ip_attempts = Counter()
        
        for line in lines:
            for pattern, category in self.security_patterns:
                if re.search(pattern, line):
                    findings[category].append(line)
                    
                    # Extract IP if present
                    ip_match = re.search(r'\d+\.\d+\.\d+\.\d+', line)
                    if ip_match:
                        ip_attempts[ip_match.group(0)] += 1
                    break
                    
        # Identify potential attackers
        suspicious_ips = [
            {'ip': ip, 'attempts': count}
            for ip, count in ip_attempts.most_common(10)
            if count > 5
        ]
        
        return {
            'findings_by_type': {k: len(v) for k, v in findings.items()},
            'total_security_events': sum(len(v) for v in findings.values()),
            'suspicious_ips': suspicious_ips,
            'sample_events': {
                k: v[:3] for k, v in findings.items()
            }
        }


def main():
    module_args = dict(
        log_path=dict(type='str', required=True),
        log_format=dict(type='str', default='auto',
                       choices=['auto', 'apache', 'nginx', 'json', 'syslog', 'custom']),
        custom_pattern=dict(type='str'),
        analysis_type=dict(type='str', default='errors',
                          choices=['errors', 'statistics', 'anomaly', 'security']),
        time_range=dict(type='str', default='24h'),
        alert_threshold=dict(type='int', default=100)
    )

    module = AnsibleModule(
        argument_spec=module_args,
        supports_check_mode=True
    )

    analyzer = LogAnalyzer(module)
    log_path = module.params['log_path']
    log_format = module.params['log_format']
    analysis_type = module.params['analysis_type']
    alert_threshold = module.params['alert_threshold']
    
    result = dict(
        changed=False,
        alert_triggered=False
    )

    try:
        # Read logs
        lines = analyzer.read_logs(log_path, module.params['time_range'])
        result['total_lines'] = len(lines)
        
        # Perform analysis
        if analysis_type == 'errors':
            analysis = analyzer.analyze_errors(lines, log_format)
            result.update(analysis)
            result['alert_triggered'] = analysis['error_count'] > alert_threshold
            
        elif analysis_type == 'statistics':
            analysis = analyzer.analyze_statistics(lines, log_format)
            result.update(analysis)
            
        elif analysis_type == 'security':
            analysis = analyzer.analyze_security(lines, log_format)
            result.update(analysis)
            result['alert_triggered'] = analysis['total_security_events'] > alert_threshold

        module.exit_json(**result)

    except Exception as e:
        module.fail_json(msg=str(e), **result)


if __name__ == '__main__':
    main()
```

---

## Use Case 8: Compliance Checker Module

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

DOCUMENTATION = r'''
---
module: compliance_checker
short_description: Check system compliance against security standards
description:
    - Verify system configuration against CIS benchmarks.
    - Check for common security misconfigurations.
    - Generate compliance reports.
options:
    standard:
        description: Compliance standard to check.
        choices: ['cis_ubuntu', 'cis_centos', 'pci_dss', 'hipaa', 'custom']
        default: cis_ubuntu
        type: str
    checks:
        description: Specific checks to run.
        type: list
        elements: str
        default: ['all']
    auto_remediate:
        description: Automatically fix non-compliant items.
        type: bool
        default: false
    report_format:
        description: Report output format.
        choices: ['json', 'html', 'csv']
        default: json
        type: str
'''

EXAMPLES = r'''
- name: Run CIS compliance check
  compliance_checker:
    standard: cis_ubuntu
    checks:
      - filesystem
      - permissions
      - network
  register: compliance_report

- name: PCI-DSS check with remediation
  compliance_checker:
    standard: pci_dss
    auto_remediate: true
  register: pci_results
'''

RETURN = r'''
compliant:
    description: Overall compliance status.
    type: bool
    returned: always
score:
    description: Compliance score percentage.
    type: float
    returned: always
passed:
    description: Number of passed checks.
    type: int
    returned: always
failed:
    description: Number of failed checks.
    type: int
    returned: always
findings:
    description: Detailed findings.
    type: list
    returned: always
'''

import os
import subprocess
import stat
from ansible.module_utils.basic import AnsibleModule


class ComplianceChecker:
    def __init__(self, module):
        self.module = module
        self.findings = []
        self.passed = 0
        self.failed = 0
        
    def check_filesystem_permissions(self):
        """Check critical file permissions (CIS 1.x)."""
        critical_files = [
            ('/etc/passwd', '0644', 'root', 'root'),
            ('/etc/shadow', '0640', 'root', 'shadow'),
            ('/etc/group', '0644', 'root', 'root'),
            ('/etc/gshadow', '0640', 'root', 'shadow'),
            ('/etc/ssh/sshd_config', '0600', 'root', 'root'),
            ('/etc/crontab', '0600', 'root', 'root'),
        ]
        
        for filepath, expected_mode, expected_owner, expected_group in critical_files:
            if not os.path.exists(filepath):
                continue
                
            stat_info = os.stat(filepath)
            mode = oct(stat_info.st_mode)[-4:]
            
            import pwd
            import grp
            owner = pwd.getpwuid(stat_info.st_uid).pw_name
            group = grp.getgrgid(stat_info.st_gid).gr_name
            
            finding = {
                'control_id': 'CIS-PERM-001',
                'description': f'File permissions for {filepath}',
                'file': filepath
            }
            
            if mode != expected_mode or owner != expected_owner:
                finding['status'] = 'FAIL'
                finding['current'] = f'mode={mode}, owner={owner}'
                finding['expected'] = f'mode={expected_mode}, owner={expected_owner}'
                finding['remediation'] = f'chmod {expected_mode} {filepath} && chown {expected_owner}:{expected_group} {filepath}'
                self.failed += 1
            else:
                finding['status'] = 'PASS'
                self.passed += 1
                
            self.findings.append(finding)
            
    def check_ssh_config(self):
        """Check SSH server configuration (CIS 5.2.x)."""
        ssh_checks = [
            ('PermitRootLogin', 'no', 'CIS-SSH-001'),
            ('PasswordAuthentication', 'no', 'CIS-SSH-002'),
            ('PermitEmptyPasswords', 'no', 'CIS-SSH-003'),
            ('X11Forwarding', 'no', 'CIS-SSH-004'),
            ('MaxAuthTries', '4', 'CIS-SSH-005'),
            ('Protocol', '2', 'CIS-SSH-006'),
            ('ClientAliveInterval', '300', 'CIS-SSH-007'),
            ('LoginGraceTime', '60', 'CIS-SSH-008'),
        ]
        
        try:
            with open('/etc/ssh/sshd_config', 'r') as f:
                config = f.read()
        except FileNotFoundError:
            return
            
        for param, expected, control_id in ssh_checks:
            import re
            pattern = rf'^{param}\s+(\S+)'
            match = re.search(pattern, config, re.MULTILINE | re.IGNORECASE)
            
            finding = {
                'control_id': control_id,
                'description': f'SSH {param} configuration',
                'expected': expected
            }
            
            if match:
                current = match.group(1).lower()
                if current == expected.lower():
                    finding['status'] = 'PASS'
                    self.passed += 1
                else:
                    finding['status'] = 'FAIL'
                    finding['current'] = current
                    finding['remediation'] = f"sed -i 's/^{param}.*/{param} {expected}/' /etc/ssh/sshd_config"
                    self.failed += 1
            else:
                finding['status'] = 'FAIL'
                finding['current'] = 'not set'
                finding['remediation'] = f"echo '{param} {expected}' >> /etc/ssh/sshd_config"
                self.failed += 1
                
            self.findings.append(finding)
            
    def check_password_policy(self):
        """Check password policy settings (CIS 5.4.x)."""
        try:
            with open('/etc/login.defs', 'r') as f:
                login_defs = f.read()
        except FileNotFoundError:
            return
            
        checks = [
            ('PASS_MAX_DAYS', '90', 'CIS-PWD-001'),
            ('PASS_MIN_DAYS', '7', 'CIS-PWD-002'),
            ('PASS_WARN_AGE', '14', 'CIS-PWD-003'),
        ]
        
        import re
        for param, min_value, control_id in checks:
            pattern = rf'^{param}\s+(\d+)'
            match = re.search(pattern, login_defs, re.MULTILINE)
            
            finding = {
                'control_id': control_id,
                'description': f'Password policy {param}',
                'expected': f'>= {min_value}' if 'MIN' in param else f'<= {min_value}'
            }
            
            if match:
                current = int(match.group(1))
                expected = int(min_value)
                
                if 'MAX' in param and current <= expected:
                    finding['status'] = 'PASS'
                    self.passed += 1
                elif 'MIN' in param or 'WARN' in param:
                    if current >= expected:
                        finding['status'] = 'PASS'
                        self.passed += 1
                    else:
                        finding['status'] = 'FAIL'
                        finding['current'] = str(current)
                        self.failed += 1
                else:
                    finding['status'] = 'FAIL'
                    finding['current'] = str(current)
                    self.failed += 1
            else:
                finding['status'] = 'FAIL'
                finding['current'] = 'not set'
                self.failed += 1
                
            self.findings.append(finding)
            
    def check_network_settings(self):
        """Check network security settings (CIS 3.x)."""
        sysctl_checks = [
            ('net.ipv4.ip_forward', '0', 'CIS-NET-001'),
            ('net.ipv4.conf.all.send_redirects', '0', 'CIS-NET-002'),
            ('net.ipv4.conf.default.send_redirects', '0', 'CIS-NET-003'),
            ('net.ipv4.conf.all.accept_source_route', '0', 'CIS-NET-004'),
            ('net.ipv4.conf.all.accept_redirects', '0', 'CIS-NET-005'),
            ('net.ipv4.conf.all.log_martians', '1', 'CIS-NET-006'),
            ('net.ipv4.icmp_echo_ignore_broadcasts', '1', 'CIS-NET-007'),
        ]
        
        for param, expected, control_id in sysctl_checks:
            try:
                result = subprocess.run(
                    ['sysctl', '-n', param],
                    capture_output=True,
                    text=True
                )
                current = result.stdout.strip()
            except:
                current = 'unknown'
                
            finding = {
                'control_id': control_id,
                'description': f'Kernel parameter {param}',
                'expected': expected
            }
            
            if current == expected:
                finding['status'] = 'PASS'
                self.passed += 1
            else:
                finding['status'] = 'FAIL'
                finding['current'] = current
                finding['remediation'] = f'sysctl -w {param}={expected}'
                self.failed += 1
                
            self.findings.append(finding)
            
    def remediate(self, findings):
        """Apply remediation for failed checks."""
        remediated = []
        
        for finding in findings:
            if finding['status'] == 'FAIL' and 'remediation' in finding:
                try:
                    subprocess.run(
                        finding['remediation'],
                        shell=True,
                        check=True,
                        capture_output=True
                    )
                    remediated.append(finding['control_id'])
                except subprocess.CalledProcessError:
                    pass
                    
        return remediated
        
    def run_checks(self, checks):
        """Run specified compliance checks."""
        if 'all' in checks or 'filesystem' in checks:
            self.check_filesystem_permissions()
            
        if 'all' in checks or 'ssh' in checks:
            self.check_ssh_config()
            
        if 'all' in checks or 'password' in checks:
            self.check_password_policy()
            
        if 'all' in checks or 'network' in checks:
            self.check_network_settings()
            
        total = self.passed + self.failed
        score = (self.passed / total * 100) if total > 0 else 0
        
        return {
            'compliant': self.failed == 0,
            'score': round(score, 2),
            'passed': self.passed,
            'failed': self.failed,
            'findings': self.findings
        }


def main():
    module_args = dict(
        standard=dict(type='str', default='cis_ubuntu',
                     choices=['cis_ubuntu', 'cis_centos', 'pci_dss', 'hipaa', 'custom']),
        checks=dict(type='list', elements='str', default=['all']),
        auto_remediate=dict(type='bool', default=False),
        report_format=dict(type='str', default='json',
                          choices=['json', 'html', 'csv'])
    )

    module = AnsibleModule(
        argument_spec=module_args,
        supports_check_mode=True
    )

    checker = ComplianceChecker(module)
    
    result = dict(changed=False)

    try:
        # Run checks
        check_result = checker.run_checks(module.params['checks'])
        result.update(check_result)
        
        # Remediate if requested
        if module.params['auto_remediate'] and not module.check_mode:
            failed_findings = [f for f in check_result['findings'] if f['status'] == 'FAIL']
            remediated = checker.remediate(failed_findings)
            
            if remediated:
                result['changed'] = True
                result['remediated'] = remediated

        module.exit_json(**result)

    except Exception as e:
        module.fail_json(msg=str(e), **result)


if __name__ == '__main__':
    main()
```

---

## Summary: When to Create Custom Modules

| Scenario | Create Module | Alternative |
|----------|--------------|-------------|
| Proprietary API | ✅ Yes | uri module (limited) |
| Complex business logic | ✅ Yes | Multiple tasks |
| Performance critical | ✅ Yes | shell/command |
| Reuse across projects | ✅ Yes | Roles |
| Need idempotency | ✅ Yes | shell + register |
| Simple one-off task | ❌ No | shell module |
| Standard operations | ❌ No | Built-in modules |

---

*These modules demonstrate patterns for real enterprise use cases. Adapt them to your specific requirements!*
