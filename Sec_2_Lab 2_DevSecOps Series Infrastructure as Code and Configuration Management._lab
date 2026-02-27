Lab 2: DevSecOps Series - Infrastructure as Code and Configuration Management
Lab Overview
This lab introduces students to Infrastructure as Code (IaC) and Configuration Management principles using Terraform and Ansible. Students will learn to create secure, version-controlled infrastructure templates and automate configuration management while implementing security best practices.

Learning Objectives
By the end of this lab, students will be able to:

• Create and deploy infrastructure using Terraform templates • Implement version control for Infrastructure as Code using GitHub • Perform security scanning of Terraform configurations using Checkov • Write Ansible playbooks for configuration management • Secure sensitive information using Ansible Vault • Understand the integration between IaC and configuration management tools • Apply DevSecOps principles to infrastructure automation

Prerequisites
Before starting this lab, students should have:

• Basic understanding of Linux command line operations • Familiarity with YAML and JSON file formats • Basic knowledge of cloud computing concepts • Understanding of version control systems (Git) • Access to a GitHub account

Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines with all necessary tools installed. Simply click Start Lab to access your environment - no need to build your own VM or install software.

Your lab environment includes: • Ubuntu 20.04 LTS with root access • Terraform v1.6.0 or later • Ansible v2.15 or later • Git client • Python 3.8+ • Checkov security scanner • Text editor (nano/vim)

Lab Task 1: Create Terraform Deployment Template with Security Scanning
Task 1.1: Initialize Terraform Project Structure
First, let's create a proper directory structure for our Terraform project.

# Create project directory
mkdir -p ~/terraform-lab
cd ~/terraform-lab

# Create subdirectories for organization
mkdir -p {modules,environments/dev,environments/prod}

# Initialize Terraform
terraform init
Task 1.2: Create Main Terraform Configuration
Create the main Terraform configuration file that will deploy basic AWS infrastructure.

# Create main.tf file
nano main.tf
Add the following content to main.tf:

# Configure the AWS Provider
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

# Create VPC
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "${var.project_name}-vpc"
    Environment = var.environment
    Project     = var.project_name
  }
}

# Create Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name        = "${var.project_name}-igw"
    Environment = var.environment
  }
}

# Create public subnet
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnet_cidr
  availability_zone       = data.aws_availability_zones.available.names[0]
  map_public_ip_on_launch = true

  tags = {
    Name        = "${var.project_name}-public-subnet"
    Environment = var.environment
    Type        = "Public"
  }
}

# Create private subnet
resource "aws_subnet" "private" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnet_cidr
  availability_zone = data.aws_availability_zones.available.names[1]

  tags = {
    Name        = "${var.project_name}-private-subnet"
    Environment = var.environment
    Type        = "Private"
  }
}

# Create route table for public subnet
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name        = "${var.project_name}-public-rt"
    Environment = var.environment
  }
}

# Associate public subnet with route table
resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

# Create security group for web servers
resource "aws_security_group" "web" {
  name_prefix = "${var.project_name}-web-"
  vpc_id      = aws_vpc.main.id
  description = "Security group for web servers"

  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTPS"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "${var.project_name}-web-sg"
    Environment = var.environment
  }
}

# Data source for availability zones
data "aws_availability_zones" "available" {
  state = "available"
}

# Data source for latest Amazon Linux AMI
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# Create EC2 instance
resource "aws_instance" "web" {
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = var.instance_type
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.web.id]
  key_name               = var.key_pair_name

  user_data = base64encode(templatefile("${path.module}/user_data.sh", {
    project_name = var.project_name
  }))

  tags = {
    Name        = "${var.project_name}-web-server"
    Environment = var.environment
    Type        = "WebServer"
  }
}
Task 1.3: Create Variables File
Create a variables file to make the configuration flexible and reusable.

# Create variables.tf file
nano variables.tf
Add the following content:

variable "aws_region" {
  description = "AWS region for resources"
  type        = string
  default     = "us-east-1"
}

variable "project_name" {
  description = "Name of the project"
  type        = string
  default     = "devsecops-lab"
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"
}

variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"
  
  validation {
    condition     = can(cidrhost(var.vpc_cidr, 0))
    error_message = "VPC CIDR must be a valid IPv4 CIDR block."
  }
}

variable "public_subnet_cidr" {
  description = "CIDR block for public subnet"
  type        = string
  default     = "10.0.1.0/24"
}

variable "private_subnet_cidr" {
  description = "CIDR block for private subnet"
  type        = string
  default     = "10.0.2.0/24"
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.micro"
  
  validation {
    condition = contains([
      "t3.micro", "t3.small", "t3.medium",
      "t2.micro", "t2.small", "t2.medium"
    ], var.instance_type)
    error_message = "Instance type must be a valid t2 or t3 instance type."
  }
}

variable "key_pair_name" {
  description = "Name of the AWS key pair"
  type        = string
  default     = "my-key-pair"
}
Task 1.4: Create Outputs File
Create an outputs file to display important information after deployment.

# Create outputs.tf file
nano outputs.tf
Add the following content:

output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.main.id
}

output "public_subnet_id" {
  description = "ID of the public subnet"
  value       = aws_subnet.public.id
}

output "private_subnet_id" {
  description = "ID of the private subnet"
  value       = aws_subnet.private.id
}

output "web_instance_id" {
  description = "ID of the web server instance"
  value       = aws_instance.web.id
}

output "web_instance_public_ip" {
  description = "Public IP address of the web server"
  value       = aws_instance.web.public_ip
}

output "web_instance_private_ip" {
  description = "Private IP address of the web server"
  value       = aws_instance.web.private_ip
}

output "security_group_id" {
  description = "ID of the web security group"
  value       = aws_security_group.web.id
}
Task 1.5: Create User Data Script
Create a user data script for EC2 instance initialization.

# Create user_data.sh file
nano user_data.sh
Add the following content:

#!/bin/bash
yum update -y
yum install -y httpd

# Start and enable Apache
systemctl start httpd
systemctl enable httpd

# Create a simple web page
cat > /var/www/html/index.html << EOF
<!DOCTYPE html>
<html>
<head>
    <title>${project_name} - DevSecOps Lab</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 40px; }
        .header { color: #2c3e50; }
        .info { background-color: #ecf0f1; padding: 20px; border-radius: 5px; }
    </style>
</head>
<body>
    <h1 class="header">Welcome to ${project_name}</h1>
    <div class="info">
        <h2>DevSecOps Infrastructure Lab</h2>
        <p>This server was deployed using Terraform Infrastructure as Code.</p>
        <p>Server deployed at: $(date)</p>
        <p>Hostname: $(hostname)</p>
    </div>
</body>
</html>
EOF

# Set proper permissions
chown apache:apache /var/www/html/index.html
Task 1.6: Create Terraform Configuration File
Create a terraform.tfvars file for environment-specific values.

# Create terraform.tfvars file
nano terraform.tfvars
Add the following content:

# Environment Configuration
project_name = "devsecops-lab"
environment  = "development"
aws_region   = "us-east-1"

# Network Configuration
vpc_cidr            = "10.0.0.0/16"
public_subnet_cidr  = "10.0.1.0/24"
private_subnet_cidr = "10.0.2.0/24"

# Instance Configuration
instance_type   = "t3.micro"
key_pair_name   = "lab-key-pair"
Task 1.7: Initialize and Validate Terraform
Now let's initialize and validate our Terraform configuration.

# Initialize Terraform (download providers)
terraform init

# Format Terraform files
terraform fmt

# Validate configuration
terraform validate

# Create execution plan
terraform plan
Task 1.8: Set Up GitHub Repository
Create a GitHub repository to store and version control your Terraform code.

# Initialize Git repository
git init

# Create .gitignore file
nano .gitignore
Add the following content to .gitignore:

# Terraform files
*.tfstate
*.tfstate.*
*.tfvars
.terraform/
.terraform.lock.hcl
crash.log
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# IDE files
.vscode/
.idea/
*.swp
*.swo

# OS files
.DS_Store
Thumbs.db

# Sensitive files
*.pem
*.key
secrets.txt
# Add files to Git
git add .
git commit -m "Initial Terraform infrastructure configuration"

# Add remote repository (replace with your GitHub repo URL)
git remote add origin https://github.com/yourusername/terraform-devsecops-lab.git

# Push to GitHub
git branch -M main
git push -u origin main
Task 1.9: Install and Configure Checkov
Install Checkov for security scanning of Terraform configurations.

# Install Checkov using pip
pip3 install checkov

# Verify installation
checkov --version
Task 1.10: Scan Terraform Configuration with Checkov
Run Checkov to scan your Terraform configuration for security issues.

# Run Checkov scan on current directory
checkov -d . --framework terraform

# Run Checkov with specific output format
checkov -d . --framework terraform --output json > checkov-report.json

# Run Checkov with specific checks
checkov -d . --framework terraform --check CKV_AWS_79,CKV_AWS_23

# Generate HTML report
checkov -d . --framework terraform --output cli --output html --output-file-path checkov-report.html
Task 1.11: Fix Security Issues
Based on Checkov results, let's fix common security issues. Update your main.tf file:

# Edit main.tf to fix security issues
nano main.tf
Add these security improvements to your security group:

# Update security group with more restrictive rules
resource "aws_security_group" "web" {
  name_prefix = "${var.project_name}-web-"
  vpc_id      = aws_vpc.main.id
  description = "Security group for web servers"

  # More restrictive SSH access
  ingress {
    description = "SSH from specific CIDR"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.allowed_ssh_cidr]  # Add this variable
  }

  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTPS"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    description = "All outbound traffic"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "${var.project_name}-web-sg"
    Environment = var.environment
  }
}

# Add encryption to EBS volumes
resource "aws_instance" "web" {
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = var.instance_type
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.web.id]
  key_name               = var.key_pair_name

  root_block_device {
    volume_type           = "gp3"
    volume_size           = 20
    encrypted             = true
    delete_on_termination = true
  }

  metadata_options {
    http_endpoint = "enabled"
    http_tokens   = "required"
  }

  user_data = base64encode(templatefile("${path.module}/user_data.sh", {
    project_name = var.project_name
  }))

  tags = {
    Name        = "${var.project_name}-web-server"
    Environment = var.environment
    Type        = "WebServer"
  }
}
Add the new variable to variables.tf:

variable "allowed_ssh_cidr" {
  description = "CIDR block allowed for SSH access"
  type        = string
  default     = "10.0.0.0/16"
}
Task 1.12: Re-scan and Commit Changes
# Re-scan with Checkov
checkov -d . --framework terraform

# Validate updated configuration
terraform validate
terraform plan

# Commit security improvements
git add .
git commit -m "Fix security issues identified by Checkov scan"
git push origin main
Lab Task 2: Write Ansible Playbook with Ansible Vault
Task 2.1: Set Up Ansible Directory Structure
Create a proper Ansible project structure.

# Create Ansible project directory
mkdir -p ~/ansible-lab
cd ~/ansible-lab

# Create Ansible directory structure
mkdir -p {playbooks,roles,inventory,group_vars,host_vars,vault}

# Create basic inventory file
nano inventory/hosts.yml
Add the following content to hosts.yml:

all:
  children:
    webservers:
      hosts:
        web1:
          ansible_host: 10.0.1.100
          ansible_user: ec2-user
          ansible_ssh_private_key_file: ~/.ssh/lab-key-pair.pem
        web2:
          ansible_host: 10.0.1.101
          ansible_user: ec2-user
          ansible_ssh_private_key_file: ~/.ssh/lab-key-pair.pem
    databases:
      hosts:
        db1:
          ansible_host: 10.0.2.100
          ansible_user: ec2-user
          ansible_ssh_private_key_file: ~/.ssh/lab-key-pair.pem
  vars:
    ansible_ssh_common_args: '-o StrictHostKeyChecking=no'
Task 2.2: Create Ansible Configuration File
Create an Ansible configuration file for project-specific settings.

# Create ansible.cfg file
nano ansible.cfg
Add the following content:

[defaults]
inventory = inventory/hosts.yml
remote_user = ec2-user
private_key_file = ~/.ssh/lab-key-pair.pem
host_key_checking = False
retry_files_enabled = False
gathering = smart
fact_caching = memory
stdout_callback = yaml
bin_ansible_callbacks = True

[inventory]
enable_plugins = yaml, ini, auto

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s -o StrictHostKeyChecking=no
pipelining = True
Task 2.3: Create Sensitive Variables File
Create a file with sensitive information that we'll encrypt with Ansible Vault.

# Create sensitive variables file
nano vault/secrets.yml
Add the following sensitive content:

# Database Configuration
db_root_password: "SuperSecretRootPassword123!"
db_app_password: "AppUserPassword456!"
db_backup_password: "BackupPassword789!"

# Application Configuration
app_secret_key: "your-super-secret-application-key-here"
jwt_secret: "jwt-signing-secret-key-very-long-and-secure"

# API Keys
aws_access_key: "AKIAIOSFODNN7EXAMPLE"
aws_secret_key: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"

# SSL Certificate Information
ssl_cert_password: "SSLCertificatePassword123!"

# Monitoring Credentials
monitoring_username: "monitor_user"
monitoring_password: "MonitoringPassword456!"

# Backup Configuration
backup_encryption_key: "backup-encryption-key-32-characters"
Task 2.4: Encrypt Sensitive File with Ansible Vault
Use Ansible Vault to encrypt the sensitive information.

# Encrypt the secrets file with Ansible Vault
ansible-vault encrypt vault/secrets.yml

# You'll be prompted to enter a vault password
# Use a strong password like: VaultPassword123!

# Verify the file is encrypted
cat vault/secrets.yml

# View encrypted file content (will prompt for password)
ansible-vault view vault/secrets.yml
Task 2.5: Create Vault Password File
For automation purposes, create a vault password file (in production, this should be secured properly).

# Create vault password file
echo "VaultPassword123!" > vault/.vault_pass

# Secure the password file
chmod 600 vault/.vault_pass

# Update ansible.cfg to use vault password file
echo "vault_password_file = vault/.vault_pass" >> ansible.cfg
Task 2.6: Create Main Playbook
Create the main Ansible playbook that will configure web servers.

# Create main playbook
nano playbooks/site.yml
Add the following content:

---
- name: Configure Web Servers
  hosts: webservers
  become: yes
  vars_files:
    - ../vault/secrets.yml
  vars:
    app_name: "devsecops-webapp"
    app_port: 8080
    nginx_port: 80
    
  tasks:
    - name: Update system packages
      yum:
        name: "*"
        state: latest
        update_cache: yes
      tags: [system, updates]

    - name: Install required packages
      yum:
        name:
          - nginx
          - python3
          - python3-pip
          - git
          - htop
          - curl
          - wget
        state: present
      tags: [packages]

    - name: Install Python packages
      pip:
        name:
          - flask
          - gunicorn
          - requests
        executable: pip3
      tags: [python]

    - name: Create application user
      user:
        name: "{{ app_name }}"
        system: yes
        shell: /bin/bash
        home: "/opt/{{ app_name }}"
        create_home: yes
      tags: [users]

    - name: Create application directories
      file:
        path: "{{ item }}"
        state: directory
        owner: "{{ app_name }}"
        group: "{{ app_name }}"
        mode: '0755'
      loop:
        - "/opt/{{ app_name }}/app"
        - "/opt/{{ app_name }}/logs"
        - "/opt/{{ app_name }}/config"
      tags: [directories]

    - name: Create Flask application
      copy:
        content: |
          from flask import Flask, jsonify
          import os
          
          app = Flask(__name__)
          app.config['SECRET_KEY'] = '{{ app_secret_key }}'
          
          @app.route('/')
          def home():
              return jsonify({
                  'message': 'DevSecOps Lab Application',
                  'status': 'running',
                  'environment': 'production'
              })
          
          @app.route('/health')
          def health():
              return jsonify({'status': 'healthy'})
          
          if __name__ == '__main__':
              app.run(host='0.0.0.0', port={{ app_port }})
        dest: "/opt/{{ app_name }}/app/app.py"
        owner: "{{ app_name }}"
        group: "{{ app_name }}"
        mode: '0644'
      tags: [application]

    - name: Create Gunicorn configuration
      copy:
        content: |
          bind = "127.0.0.1:{{ app_port }}"
          workers = 2
          worker_class = "sync"
          worker_connections = 1000
          max_requests = 1000
          max_requests_jitter = 100
          timeout = 30
          keepalive = 2
          user = "{{ app_name }}"
          group = "{{ app_name }}"
          tmp_upload_dir = None
          logfile = "/opt/{{ app_name }}/logs/gunicorn.log"
          loglevel = "info"
          accesslog = "/opt/{{ app_name }}/logs/access.log"
          access_log_format = '%(h)s %(l)s %(u)s %(t)s "%(r)s" %(s)s %(b)s "%(f)s" "%(a)s"'
        dest: "/opt/{{ app_name }}/config/gunicorn.conf.py"
        owner: "{{ app_name }}"
        group: "{{ app_name }}"
        mode: '0644'
      tags: [configuration]

    - name: Create systemd service file
      copy:
        content: |
          [Unit]
          Description={{ app_name }} Flask Application
          After=network.target
          
          [Service]
          Type=exec
          User={{ app_name }}
          Group={{ app_name }}
          WorkingDirectory=/opt/{{ app_name }}/app
          Environment=PATH=/usr/local/bin:/usr/bin:/bin
          ExecStart=/usr/local/bin/gunicorn --config /opt/{{ app_name }}/config/gunicorn.conf.py app:app
          ExecReload=/bin/kill -s HUP $MAINPID
          Restart=always
          RestartSec=3
          
          [Install]
          WantedBy=multi-user.target
        dest: "/etc/systemd/system/{{ app_name }}.service"
        mode: '0644'
      notify: reload systemd
      tags: [systemd]

    - name: Configure Nginx
      copy:
        content: |
          server {
              listen {{ nginx_port }};
              server_name _;
              
              location / {
                  proxy_pass http://127.0.0.1:{{ app_port }};
                  proxy_set_header Host $host;
                  proxy_set_header X-Real-IP $remote_addr;
                  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                  proxy_set_header X-Forwarded-Proto $scheme;
              }
              
              location /health {
                  proxy_pass http://127.0.0.1:{{ app_port }}/health;
                  access_log off;
              }
          }
        dest: "/etc/nginx/conf.d/{{ app_name }}.conf"
        backup: yes
      notify: restart nginx
      tags: [nginx]

    - name: Remove default Nginx configuration
      file:
        path: /etc/nginx/conf.d/default.conf
        state: absent
      notify: restart nginx
      tags: [nginx]

    - name: Start and enable services
      systemd:
        name: "{{ item }}"
        state: started
        enabled: yes
        daemon_reload: yes
      loop:
        - nginx
        - "{{ app_name }}"
      tags: [services]

    - name: Configure firewall
      firewalld:
        port: "{{ item }}"
        permanent: yes
        state: enabled
        immediate: yes
      loop:
        - "{{ nginx_port }}/tcp"
        - "22/tcp"
      ignore_errors: yes
      tags: [firewall]

  handlers:
    - name: reload systemd
      systemd:
        daemon_reload: yes

    - name: restart nginx
      systemd:
        name: nginx
        state: restarted

    - name: restart application
      systemd:
        name: "{{ app_name }}"
        state: restarted
Task 2.7: Create Database Configuration Playbook
Create a separate playbook for database configuration.

# Create database playbook
nano playbooks/database.yml
Add the following content:

---
- name: Configure Database Servers
  hosts: databases
  become: yes
  vars_files:
    - ../vault/secrets.yml
  vars:
    mysql_root_password: "{{ db_root_password }}"
    mysql_app_user: "appuser"
    mysql_app_password: "{{ db_app_password }}"
    mysql_database: "devsecops_app"

  tasks:
    - name: Install MySQL/MariaDB
      yum:
        name:
          - mariadb-server
          - mariadb
          - python3-PyMySQL
        state: present
      tags: [database, packages]

    - name: Start and enable MariaDB
      systemd:
        name: mariadb
        state: started
        enabled: yes
      tags: [database, services]

    - name: Set MySQL root password
      mysql_user:
        name: root
        password: "{{ mysql_root_password }}"
        login_unix_socket: /var/lib/mysql/mysql.sock
        state: present
      tags: [database, security]

    - name: Create MySQL configuration file
      copy:
        content: |
          [client]
          user=root
          password={{ mysql_root_password }}
        dest: /root/.my.cnf
        mode: '0600'
        owner: root
        group: root
      tags: [database, configuration]

    - name: Remove anonymous MySQL users
      mysql_user:
        name: ""
        host_all: yes
        state: absent
      tags: [database, security]

    - name: Remove MySQL test database
      mysql_db:
        name: test
        state: absent
      tags: [database, security]

    - name: Create application database
      mysql_db:
        name: "{{ mysql_database }}"
        state: present
      tags: [database, application]

    - name: Create application database user
      mysql_user:
        name: "{{ mysql_app_user }}"
        password: "{{ mysql_app_password }}"
        priv: "{{ mysql_database }}.*:ALL"
        host: "%"
        state: present
      tags: [database, application]

    - name: Configure MySQL for remote connections
      lineinfile:
        path: /etc/my.cnf.d/server.cnf
        regexp: '^bind-address'
        line: 'bind-address = 0.0.0.0'
        insertafter: '^\[mysqld\]'
      notify: restart mariadb
      tags: [database, configuration]

    - name: Configure firewall for MySQL
      firewalld:
        port: 3306/tcp
        permanent: yes
        state: enabled
        immediate: yes
      ignore_errors: yes
      tags: [database, firewall]

  handlers:
    - name: restart mariadb
      systemd:
        name: mariadb
        state: restarted
Task 2.8: Create Group Variables
Create group-specific variables for better organization.

# Create group variables for webservers
nano group_vars/webservers.yml
Add the following content:

---
# Web server specific variables
nginx_worker_processes: auto
nginx_worker_connections: 1024
nginx_keepalive_timeout: 65

# Application settings
app_environment: production
app_debug: false
app_log_level: info

# Security settings
ssl_enabled: false
security_headers: true
# Create group variables for databases
nano group_vars/
