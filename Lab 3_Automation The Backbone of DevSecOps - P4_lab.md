Lab 3: Automation - The Backbone of DevSecOps - P4
Lab Objectives
By the end of this lab, students will be able to:

• Understand the fundamentals of DevSecOps automation and its importance in modern software development • Set up and configure GitLab for version control and CI/CD pipeline management • Install and configure SonarQube for continuous code quality analysis • Create and manage infrastructure using Terraform templates • Implement security scanning of Terraform templates using TFsec • Build an automated pipeline that integrates security scanning into the development workflow • Demonstrate practical implementation of "shift-left" security practices

Prerequisites
Before starting this lab, students should have:

• Basic understanding of Linux command line operations • Fundamental knowledge of version control concepts (Git) • Basic understanding of cloud computing concepts • Familiarity with YAML syntax • Basic knowledge of infrastructure as code concepts

Note: Al Nafi provides ready-to-use Linux-based cloud machines for this lab. Simply click "Start Lab" to begin - no need to build your own virtual machine.

Lab Environment Setup
Your cloud machine comes pre-configured with: • Ubuntu 20.04 LTS • Docker and Docker Compose • Git • Basic development tools

Task 1: Setting Up GitLab for Version Control and CI/CD
Subtask 1.1: Install GitLab using Docker
First, let's set up GitLab Community Edition using Docker.

# Create a directory for GitLab data
sudo mkdir -p /srv/gitlab

# Create docker-compose file for GitLab
cat > docker-compose-gitlab.yml << 'EOF'
version: '3.6'
services:
  gitlab:
    image: gitlab/gitlab-ce:latest
    container_name: gitlab
    restart: always
    hostname: 'localhost'
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://localhost:8080'
        gitlab_rails['gitlab_shell_ssh_port'] = 2224
    ports:
      - '8080:8080'
      - '2224:22'
    volumes:
      - '/srv/gitlab/config:/etc/gitlab'
      - '/srv/gitlab/logs:/var/log/gitlab'
      - '/srv/gitlab/data:/var/opt/gitlab'
    shm_size: '256m'
EOF

# Start GitLab
docker-compose -f docker-compose-gitlab.yml up -d

# Wait for GitLab to start (this may take 5-10 minutes)
echo "Waiting for GitLab to start..."
sleep 300
Subtask 1.2: Configure GitLab Initial Setup
# Get the initial root password
sudo docker exec -it gitlab grep 'Password:' /etc/gitlab/initial_root_password

# Note: Save this password as you'll need it to login
Access GitLab at http://localhost:8080 using:

Username: root
Password: (from the command above)
Subtask 1.3: Create a Sample Project
Login to GitLab web interface
Click New Project
Choose Create blank project
Fill in project details:
Project name: devsecops-demo
Visibility Level: Private
Click Create project
Subtask 1.4: Clone and Setup Local Repository
# Create a working directory
mkdir -p ~/devsecops-lab
cd ~/devsecops-lab

# Initialize git repository
git init
git remote add origin http://localhost:8080/root/devsecops-demo.git

# Configure git user
git config user.name "DevSecOps Student"
git config user.email "student@example.com"

# Create initial project structure
mkdir -p terraform
mkdir -p .gitlab-ci

# Create a simple README
cat > README.md << 'EOF'
# DevSecOps Demo Project

This project demonstrates automated security scanning in a DevSecOps pipeline.

## Components
- Terraform infrastructure templates
- GitLab CI/CD pipeline
- SonarQube code quality analysis
- TFsec security scanning
EOF

# Commit and push initial files
git add .
git commit -m "Initial project setup"
git push -u origin main
Task 2: Setting Up SonarQube for Code Quality Analysis
Subtask 2.1: Install SonarQube using Docker
# Create SonarQube docker-compose file
cat > docker-compose-sonarqube.yml << 'EOF'
version: '3.7'
services:
  sonarqube:
    image: sonarqube:community
    container_name: sonarqube
    depends_on:
      - db
    environment:
      SONAR_JDBC_URL: jdbc:postgresql://db:5432/sonar
      SONAR_JDBC_USERNAME: sonar
      SONAR_JDBC_PASSWORD: sonar
    volumes:
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_extensions:/opt/sonarqube/extensions
      - sonarqube_logs:/opt/sonarqube/logs
    ports:
      - "9000:9000"
  db:
    image: postgres:13
    container_name: sonarqube-db
    environment:
      POSTGRES_USER: sonar
      POSTGRES_PASSWORD: sonar
      POSTGRES_DB: sonar
    volumes:
      - postgresql:/var/lib/postgresql
      - postgresql_data:/var/lib/postgresql/data

volumes:
  sonarqube_data:
  sonarqube_extensions:
  sonarqube_logs:
  postgresql:
  postgresql_data:
EOF

# Start SonarQube
docker-compose -f docker-compose-sonarqube.yml up -d

# Wait for SonarQube to start
echo "Waiting for SonarQube to start..."
sleep 120
Subtask 2.2: Configure SonarQube
Access SonarQube at http://localhost:9000
Login with default credentials:
Username: admin
Password: admin
Change the default password when prompted
Create a new project:
Project key: devsecops-demo
Display name: DevSecOps Demo
Generate a token for CI/CD integration:
Go to My Account > Security
Generate token named gitlab-ci
Save this token for later use
Task 3: Creating Terraform Templates for Infrastructure
Subtask 3.1: Create Basic Terraform Configuration
# Navigate to terraform directory
cd ~/devsecops-lab/terraform

# Create main Terraform configuration
cat > main.tf << 'EOF'
# Configure the AWS Provider
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  required_version = ">= 1.0"
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

# Associate route table with public subnet
resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

# Create security group
resource "aws_security_group" "web" {
  name_prefix = "${var.project_name}-web-"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # This is intentionally insecure for demonstration
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
EOF

# Create variables file
cat > variables.tf << 'EOF'
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-west-2"
}

variable "project_name" {
  description = "Name of the project"
  type        = string
  default     = "devsecops-demo"
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
}

variable "public_subnet_cidr" {
  description = "CIDR block for public subnet"
  type        = string
  default     = "10.0.1.0/24"
}
EOF

# Create outputs file
cat > outputs.tf << 'EOF'
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.main.id
}

output "public_subnet_id" {
  description = "ID of the public subnet"
  value       = aws_subnet.public.id
}

output "security_group_id" {
  description = "ID of the security group"
  value       = aws_security_group.web.id
}
EOF
Subtask 3.2: Install Terraform and TFsec
# Install Terraform
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform

# Install TFsec
curl -s https://raw.githubusercontent.com/aquasecurity/tfsec/master/scripts/install_linux.sh | bash
sudo mv tfsec /usr/local/bin/

# Verify installations
terraform version
tfsec --version
Task 4: Implementing TFsec for Terraform Security Scanning
Subtask 4.1: Run TFsec Scan Manually
# Navigate to terraform directory
cd ~/devsecops-lab/terraform

# Run TFsec scan
tfsec .

# Run TFsec with JSON output for CI/CD integration
tfsec . --format json > tfsec-results.json

# View the results
cat tfsec-results.json | jq '.'
Subtask 4.2: Create TFsec Configuration
# Create TFsec configuration file
cat > .tfsec.yml << 'EOF'
severity_overrides:
  AWS002: ERROR
  AWS017: ERROR
  AWS018: ERROR

exclude:
  - AWS002  # Temporarily exclude for demo purposes

minimum_severity: MEDIUM

format: json
EOF
Subtask 4.3: Create Security Remediation Script
# Create a script to fix common security issues
cat > fix-security-issues.sh << 'EOF'
#!/bin/bash

echo "Applying security fixes to Terraform configuration..."

# Fix 1: Restrict SSH access to specific IP ranges
sed -i 's/cidr_blocks = \["0.0.0.0\/0"\]  # This is intentionally insecure for demonstration/cidr_blocks = ["10.0.0.0\/8"]  # Restricted to private networks/' main.tf

# Fix 2: Add encryption for any storage resources (if they exist)
# This is a placeholder for more complex fixes

echo "Security fixes applied. Please review the changes."
EOF

chmod +x fix-security-issues.sh
Task 5: Creating GitLab CI/CD Pipeline with Security Integration
Subtask 5.1: Create GitLab CI/CD Pipeline Configuration
# Navigate back to project root
cd ~/devsecops-lab

# Create GitLab CI/CD pipeline configuration
cat > .gitlab-ci.yml << 'EOF'
stages:
  - validate
  - security-scan
  - quality-check
  - deploy

variables:
  TF_ROOT: ${CI_PROJECT_DIR}/terraform
  TF_ADDRESS: ${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/terraform/state/production

cache:
  paths:
    - ${TF_ROOT}/.terraform

before_script:
  - apt-get update -qq && apt-get install -y -qq git curl unzip jq
  - curl -s https://raw.githubusercontent.com/aquasecurity/tfsec/master/scripts/install_linux.sh | bash
  - mv tfsec /usr/local/bin/

terraform-validate:
  stage: validate
  image: hashicorp/terraform:latest
  script:
    - cd ${TF_ROOT}
    - terraform fmt -check
    - terraform init -backend=false
    - terraform validate
  only:
    - merge_requests
    - main

tfsec-scan:
  stage: security-scan
  image: alpine:latest
  script:
    - cd ${TF_ROOT}
    - tfsec . --format json --out tfsec-report.json
    - tfsec . --format junit --out tfsec-junit.xml
    - |
      if [ -s tfsec-report.json ]; then
        echo "Security issues found:"
        cat tfsec-report.json | jq '.results[] | select(.severity == "HIGH" or .severity == "CRITICAL")'
        HIGH_CRITICAL_COUNT=$(cat tfsec-report.json | jq '[.results[] | select(.severity == "HIGH" or .severity == "CRITICAL")] | length')
        if [ "$HIGH_CRITICAL_COUNT" -gt 0 ]; then
          echo "Found $HIGH_CRITICAL_COUNT high/critical security issues. Pipeline will fail."
          exit 1
        fi
      fi
  artifacts:
    reports:
      junit: ${TF_ROOT}/tfsec-junit.xml
    paths:
      - ${TF_ROOT}/tfsec-report.json
    expire_in: 1 week
  only:
    - merge_requests
    - main

sonarqube-check:
  stage: quality-check
  image: sonarsource/sonar-scanner-cli:latest
  variables:
    SONAR_USER_HOME: "${CI_PROJECT_DIR}/.sonar"
    GIT_DEPTH: "0"
  cache:
    key: "${CI_JOB_NAME}"
    paths:
      - .sonar/cache
  script:
    - sonar-scanner
      -Dsonar.projectKey=devsecops-demo
      -Dsonar.sources=.
      -Dsonar.host.url=http://sonarqube:9000
      -Dsonar.login=$SONAR_TOKEN
  allow_failure: true
  only:
    - merge_requests
    - main

terraform-plan:
  stage: deploy
  image: hashicorp/terraform:latest
  script:
    - cd ${TF_ROOT}
    - terraform init
    - terraform plan -out="planfile"
  artifacts:
    paths:
      - ${TF_ROOT}/planfile
    expire_in: 1 week
  only:
    - main
  when: manual

terraform-apply:
  stage: deploy
  image: hashicorp/terraform:latest
  script:
    - cd ${TF_ROOT}
    - terraform init
    - terraform apply -input=false "planfile"
  dependencies:
    - terraform-plan
  only:
    - main
  when: manual
EOF
Subtask 5.2: Create SonarQube Configuration
# Create SonarQube project configuration
cat > sonar-project.properties << 'EOF'
sonar.projectKey=devsecops-demo
sonar.projectName=DevSecOps Demo
sonar.projectVersion=1.0
sonar.sources=.
sonar.exclusions=**/*.tf,**/*.yml,**/*.yaml,**/*.json
sonar.sourceEncoding=UTF-8
EOF
Subtask 5.3: Create Security Policy Documentation
# Create security policy document
cat > SECURITY.md << 'EOF'
# Security Policy

## Automated Security Scanning

This project implements automated security scanning as part of the DevSecOps pipeline:

### Tools Used
- **TFsec**: Scans Terraform templates for security misconfigurations
- **SonarQube**: Analyzes code quality and security vulnerabilities
- **GitLab CI/CD**: Orchestrates the automated pipeline

### Security Gates
1. **Terraform Validation**: Ensures Terraform syntax is correct
2. **Security Scanning**: TFsec scans for security issues
3. **Quality Gates**: SonarQube checks code quality
4. **Manual Approval**: Deployment requires manual approval

### Security Standards
- No HIGH or CRITICAL security issues allowed in production
- All infrastructure changes must pass security scans
- Security scan results are stored as artifacts for review

### Remediation Process
1. Security issues are identified during the pipeline
2. Developer receives feedback with specific issue details
3. Issues must be fixed before merge/deployment
4. Re-run pipeline to verify fixes
EOF
Task 6: Testing the Complete DevSecOps Pipeline
Subtask 6.1: Commit and Push All Changes
# Add all files to git
git add .

# Commit changes
git commit -m "Add complete DevSecOps pipeline with security scanning

- Added Terraform infrastructure templates
- Integrated TFsec for security scanning
- Added GitLab CI/CD pipeline
- Configured SonarQube integration
- Added security policy documentation"

# Push to GitLab
git push origin main
Subtask 6.2: Configure GitLab CI/CD Variables
Go to your GitLab project
Navigate to Settings > CI/CD
Expand Variables section
Add the following variables:
SONAR_TOKEN: (token from SonarQube setup)
AWS_ACCESS_KEY_ID: (if testing with real AWS - optional)
AWS_SECRET_ACCESS_KEY: (if testing with real AWS - optional)
Subtask 6.3: Trigger Pipeline and Monitor Results
Go to CI/CD > Pipelines in GitLab
Click Run Pipeline
Monitor each stage:
Validate: Terraform validation
Security Scan: TFsec scanning
Quality Check: SonarQube analysis
Deploy: Manual deployment steps
Subtask 6.4: Review Security Scan Results
# Download and review TFsec results from GitLab artifacts
# Or run locally to see immediate results
cd ~/devsecops-lab/terraform
tfsec . --format table

# Check for specific security issues
tfsec . --format json | jq '.results[] | select(.severity == "HIGH" or .severity == "CRITICAL")'
Subtask 6.5: Demonstrate Security Issue Remediation
# Intentionally introduce a security issue
cd ~/devsecops-lab/terraform

# Add an insecure S3 bucket to demonstrate scanning
cat >> main.tf << 'EOF'

# Intentionally insecure S3 bucket for demonstration
resource "aws_s3_bucket" "example" {
  bucket = "${var.project_name}-demo-bucket"

  tags = {
    Name        = "${var.project_name}-bucket"
    Environment = var.environment
  }
}

resource "aws_s3_bucket_public_access_block" "example" {
  bucket = aws_s3_bucket.example.id

  block_public_acls       = false  # Insecure setting
  block_public_policy     = false  # Insecure setting
  ignore_public_acls      = false  # Insecure setting
  restrict_public_buckets = false  # Insecure setting
}
EOF

# Run TFsec to see new security issues
tfsec .

# Fix the security issues
cat > s3-security-fix.tf << 'EOF'
# Secure S3 bucket configuration
resource "aws_s3_bucket" "example_secure" {
  bucket = "${var.project_name}-demo-bucket-secure"

  tags = {
    Name        = "${var.project_name}-bucket-secure"
    Environment = var.environment
  }
}

resource "aws_s3_bucket_public_access_block" "example_secure" {
  bucket = aws_s3_bucket.example_secure.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_server_side_encryption_configuration" "example_secure" {
  bucket = aws_s3_bucket.example_secure.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_versioning" "example_secure" {
  bucket = aws_s3_bucket.example_secure.id
  versioning_configuration {
    status = "Enabled"
  }
}
EOF

# Remove the insecure configuration
rm -f main.tf.bak
sed -i '/# Intentionally insecure S3 bucket/,$d' main.tf

# Commit the security improvements
git add .
git commit -m "Fix S3 security issues and add secure configuration

- Removed insecure S3 bucket configuration
- Added secure S3 bucket with encryption and versioning
- Enabled proper public access blocking"

git push origin main
Task 7: Advanced Pipeline Features
Subtask 7.1: Create Security Dashboard Script
# Create a script to generate security dashboard
cat > scripts/security-dashboard.sh << 'EOF'
#!/bin/bash

echo "=== DevSecOps Security Dashboard ==="
echo "Generated on: $(date)"
echo ""

# Check if TFsec results exist
if [ -f "terraform/tfsec-report.json" ]; then
    echo "=== Terraform Security Scan Results ==="
    TOTAL_ISSUES=$(cat terraform/tfsec-report.json | jq '.results | length')
    CRITICAL_ISSUES=$(cat terraform/tfsec-report.json | jq '[.results[] | select(.severity == "CRITICAL")] | length')
    HIGH_ISSUES=$(cat terraform/tfsec-report.json | jq '[.results[] | select(.severity == "HIGH")] | length')
    MEDIUM_ISSUES=$(cat terraform/tfsec-report.json | jq '[.results[] | select(.severity == "MEDIUM")] | length')
    LOW_ISSUES=$(cat terraform/tfsec-report.json | jq '[.results[] | select(.severity == "LOW")] | length')
    
    echo "Total Issues: $TOTAL_ISSUES"
    echo "Critical: $CRITICAL_ISSUES"
    echo "High: $HIGH_ISSUES"
    echo "Medium: $MEDIUM_ISSUES"
    echo "Low: $LOW_ISSUES"
    echo ""
    
    if [ "$CRITICAL_ISSUES" -gt 0 ] || [ "$HIGH_ISSUES" -gt 0 ]; then
        echo "❌ SECURITY GATE: FAILED - High/Critical issues found"
    else
        echo "✅ SECURITY GATE: PASSED - No high/critical issues"
    fi
else
    echo "No TFsec results found. Run 'tfsec terraform/' to generate."
fi

echo ""
echo "=== Pipeline Status ==="
echo "Last commit: $(git log -1 --pretty=format:'%h - %s (%an, %ar)')"
echo "Branch: $(git branch --show-current)"
EOF

chmod +x scripts/security-dashboard.sh
mkdir -p scripts
Subtask 7.2: Create Automated Security Report
# Create automated security report generator
cat > scripts/generate-security-report.py << 'EOF'
#!/usr/bin/env python3
import json
import sys
from datetime import datetime

def generate_security_report(tfsec_file):
    """Generate a comprehensive security report from TFsec results."""
    
    try:
        with open(tfsec_file, 'r') as f:
            data = json.load(f)
    except FileNotFoundError:
        print(f"Error: {tfsec_file} not found")
        return
    
    results = data.get('results', [])
    
    # Generate HTML report
    html_content = f"""
    <!DOCTYPE html>
    <html>
    <head>
        <title>DevSecOps Security Report</title>
        <style>
            body {{ font-family: Arial, sans-serif; margin: 20px; }}
            .header {{ background-color: #f0f0f0; padding: 20px; border-radius: 5px; }}
            .critical {{ color: #d32f2f; }}
            .high {{ color: #f57c00; }}
            .medium {{ color: #fbc02d; }}
            .low {{ color: #388e3c; }}
            .issue {{ margin: 10px 0; padding: 10px; border-left: 4px solid #ccc; }}
            .summary {{ background-color: #e3f2fd; padding: 15px; border-radius: 5px; margin: 20px 0; }}
        </style>
    </head>
    <body>
        <div class="header">
            <h1>DevSecOps Security Report</h1>
            <p>Generated on: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}</p>
        </div>
        
        <div class="summary">
            <h2>Summary</h2>
            <p>Total Issues Found: {len(results)}</p>
    """
    
    # Count issues by severity
    severity_counts = {}
    for result in results:
        severity = result.get('severity', 'UNKNOWN')
        severity_counts[severity] = severity_counts.get(severity, 0) + 1
    
    for severity, count in severity_counts.items():
        html_content += f"<p class='{severity.lower()}'>{severity}: {count}</p>"
    
    html_content += "</div><h2>Detailed Issues</h2>"
    
    # Add detailed issues
    for result in results:
        severity = result.get('severity', 'UNKNOWN')
        rule_id = result.get('rule_id', 'N/A')
        description = result.get('description', 'No description')
        location = result.get('location', {})
        filename = location.get('filename', 'Unknown file')
        
        html_content += f"""
        <div class="issue">
            <h3 class="{severity.lower()}">{rule_id} - {severity}</h3>
            <p><strong>File:</strong> {filename}</p>
            <p><strong>Description:</strong> {description}</p>
        </div>
        """
    
    html_content += """
    </body>
    </html>
    """
    
    # Save HTML report
    with open('security-report.html', 'w') as f:
        f.write(html_content)
    
    print("Security report generated: security-report.html")

if __name__ == "__main__":
    if len(sys.argv) != 2:
        print("Usage: python3 generate-security-report.py <tfsec-results.json>")
        sys.exit(1)
    
    generate_security_report(sys.argv[1])
EOF

chmod +x scripts/generate-security-report.py
Task 8: Final Testing and Validation
Subtask 8.1: Run Complete Security Pipeline
# Navigate to project root
cd ~/devsecops-lab

# Run the complete security pipeline locally
echo "Running complete DevSecOps security pipeline..."

# Step 1: Terraform validation
echo "1. Validating Terraform configuration..."
cd terraform
terraform init -backend=false
terraform validate
terraform fmt -check

# Step 2: Security scanning
echo "2. Running security scans..."
tfsec . --format json > tfsec-results.json
tfsec . --format table

# Step 3: Generate security report
echo "3. Generating security report..."
cd ..
python3 scripts/generate-security-report.py terraform/tfsec-results.json

# Step 4: Security dashboard
echo "4. Displaying security dashboard..."
./scripts/security-dashboard.sh
Subtask 8.2: Commit Final Changes
# Add all new files
git add .

# Commit final changes
git commit -m "Add advanced security reporting and dashboard

- Added security dashboard script
- Created automated security report generator
- Enhanced pipeline with comprehensive security checks
- Added final testing and validation procedures"

# Push to GitLab
git push origin main
Subtask 8.3: Verify GitLab Pipeline Execution
Go to GitLab web interface
Navigate to CI/CD > Pipelines
Verify the latest pipeline runs successfully
Check each stage completion:
✅ Terraform validation
✅ Security scanning
✅ Quality checks
⏸️ Manual deployment (as expected)
Troubleshooting Common Issues
Issue 1: GitLab Container Not Starting
# Check GitLab container status
docker ps -a | grep gitlab

# Check GitLab logs
docker logs gitlab

# Restart GitLab if needed
docker-compose -f docker-compose-gitlab.yml restart
Issue 2: SonarQube Connection Issues
# Check SonarQube container status
docker ps -a | grep sonarqube

# Verify SonarQube is accessible
curl -I http://localhost:9000

# Check database connection
docker logs sonarqube-db
Issue 3: TFsec Not Finding Issues
# Verify TFsec installation
tfsec --version

# Run TFsec with verbose output
tfsec . --verbose

# Check if Terraform files are valid
terraform validate
Issue 4: Pipeline Failing on Security Checks
# Review TFsec results
cat terraform/tfsec-results.json | jq '.results[]'

# Check specific security issues
