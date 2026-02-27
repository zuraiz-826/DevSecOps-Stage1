Lab 3: DevSecOps Series - Compliance and Testing
Learning Objectives
By the end of this lab, students will be able to:

• Create secure infrastructure using Terraform on AWS with compliance as code principles • Implement security baseline testing using OpenSCAP for vulnerability scanning • Perform compliance testing using InSpec for infrastructure validation • Execute Infrastructure as Code (IaC) security testing using Checkov • Integrate compliance and security testing tools into CI/CD pipelines using Jenkins • Generate and interpret compliance reports for security decision-making • Configure automated build failures based on critical security findings

Prerequisites
Before starting this lab, students should have:

• Basic understanding of cloud computing concepts and AWS services • Familiarity with Linux command line operations • Basic knowledge of Infrastructure as Code (IaC) concepts • Understanding of CI/CD pipeline fundamentals • Basic knowledge of YAML and JSON file formats • Familiarity with Git version control system

Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines for this lab. Simply click Start Lab to access your environment - no need to build your own VM or install additional software.

Your lab environment includes: • Ubuntu 20.04 LTS with all required tools pre-installed • AWS CLI configured with appropriate permissions • Terraform, OpenSCAP, InSpec, and Checkov tools ready to use • Jenkins server pre-configured for CI/CD integration • Sample code repositories and configuration files

Task 1: Infrastructure Setup with Terraform
Subtask 1.1: Create Terraform Configuration Files
First, let's create a directory structure for our Terraform project and set up the basic configuration files.

# Create project directory
mkdir devsecops-compliance-lab
cd devsecops-compliance-lab

# Create subdirectories for organization
mkdir terraform
mkdir compliance
mkdir scripts
mkdir reports
Create the main Terraform configuration file:

nano terraform/main.tf
Add the following Terraform configuration:

# Provider configuration
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

# Variables
variable "aws_region" {
  description = "AWS region for resources"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "devsecops-lab"
}

# VPC Configuration
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "${var.environment}-vpc"
    Environment = var.environment
    Compliance  = "required"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name        = "${var.environment}-igw"
    Environment = var.environment
  }
}

# Public Subnet
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "${var.aws_region}a"
  map_public_ip_on_launch = true

  tags = {
    Name        = "${var.environment}-public-subnet"
    Environment = var.environment
    Type        = "public"
  }
}

# Route Table
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name        = "${var.environment}-public-rt"
    Environment = var.environment
  }
}

# Route Table Association
resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

# Security Group
resource "aws_security_group" "web" {
  name_prefix = "${var.environment}-web-"
  vpc_id      = aws_vpc.main.id

  # HTTP access
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # HTTPS access
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # SSH access (restricted)
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/16"]
  }

  # Outbound traffic
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "${var.environment}-web-sg"
    Environment = var.environment
  }
}

# EC2 Instance
resource "aws_instance" "web" {
  ami                    = "ami-0c02fb55956c7d316"  # Amazon Linux 2
  instance_type          = "t3.micro"
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.web.id]
  
  # Enable detailed monitoring for compliance
  monitoring = true
  
  # EBS encryption for compliance
  root_block_device {
    encrypted   = true
    volume_type = "gp3"
    volume_size = 20
  }

  user_data = <<-EOF
    #!/bin/bash
    yum update -y
    yum install -y httpd
    systemctl start httpd
    systemctl enable httpd
    echo "<h1>DevSecOps Compliance Lab Server</h1>" > /var/www/html/index.html
  EOF

  tags = {
    Name        = "${var.environment}-web-server"
    Environment = var.environment
    Compliance  = "required"
    Backup      = "daily"
  }
}

# S3 Bucket for compliance reports
resource "aws_s3_bucket" "compliance_reports" {
  bucket = "${var.environment}-compliance-reports-${random_id.bucket_suffix.hex}"

  tags = {
    Name        = "${var.environment}-compliance-reports"
    Environment = var.environment
    Purpose     = "compliance-storage"
  }
}

# Random ID for unique bucket naming
resource "random_id" "bucket_suffix" {
  byte_length = 4
}

# S3 Bucket encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "compliance_reports" {
  bucket = aws_s3_bucket.compliance_reports.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# S3 Bucket versioning
resource "aws_s3_bucket_versioning" "compliance_reports" {
  bucket = aws_s3_bucket.compliance_reports.id
  versioning_configuration {
    status = "Enabled"
  }
}

# Outputs
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.main.id
}

output "instance_id" {
  description = "ID of the EC2 instance"
  value       = aws_instance.web.id
}

output "instance_public_ip" {
  description = "Public IP of the EC2 instance"
  value       = aws_instance.web.public_ip
}

output "s3_bucket_name" {
  description = "Name of the S3 bucket for compliance reports"
  value       = aws_s3_bucket.compliance_reports.bucket
}
Subtask 1.2: Initialize and Deploy Terraform Infrastructure
Initialize Terraform and deploy the infrastructure:

# Navigate to terraform directory
cd terraform

# Initialize Terraform
terraform init

# Validate the configuration
terraform validate

# Plan the deployment
terraform plan

# Apply the configuration
terraform apply -auto-approve
Save the output values for later use:

# Save outputs to a file for reference
terraform output > ../outputs.txt
Task 2: Security Baseline Testing with OpenSCAP
Subtask 2.1: Install and Configure OpenSCAP
OpenSCAP is already pre-installed in your lab environment. Let's verify the installation and explore available security profiles:

# Verify OpenSCAP installation
oscap --version

# List available security profiles
oscap info /usr/share/xml/scap/ssg/content/ssg-rhel8-ds.xml

# Download additional security content if needed
sudo yum install -y scap-security-guide
Subtask 2.2: Create OpenSCAP Compliance Scripts
Create a script to run OpenSCAP security scans:

nano scripts/openscap_scan.sh
Add the following script content:

#!/bin/bash

# OpenSCAP Security Baseline Testing Script
# This script performs security baseline testing using OpenSCAP

set -e

# Configuration
SCAN_DATE=$(date +%Y%m%d_%H%M%S)
REPORT_DIR="../reports/openscap"
PROFILE="xccdf_org.ssgproject.content_profile_cis"
CONTENT_FILE="/usr/share/xml/scap/ssg/content/ssg-rhel8-ds.xml"

# Create report directory
mkdir -p $REPORT_DIR

echo "Starting OpenSCAP Security Baseline Scan..."
echo "Scan Date: $(date)"
echo "Profile: $PROFILE"

# Run OpenSCAP evaluation
oscap xccdf eval \
    --profile $PROFILE \
    --results $REPORT_DIR/openscap-results-$SCAN_DATE.xml \
    --report $REPORT_DIR/openscap-report-$SCAN_DATE.html \
    --cpe /usr/share/xml/scap/ssg/content/ssg-rhel8-cpe-dictionary.xml \
    $CONTENT_FILE || true

# Generate summary report
echo "Generating summary report..."
cat > $REPORT_DIR/scan-summary-$SCAN_DATE.txt << EOF
OpenSCAP Security Baseline Scan Summary
=======================================
Scan Date: $(date)
Profile: $PROFILE
Content File: $CONTENT_FILE

Results File: openscap-results-$SCAN_DATE.xml
HTML Report: openscap-report-$SCAN_DATE.html

Scan Status: Completed
EOF

# Extract key metrics from results
if [ -f "$REPORT_DIR/openscap-results-$SCAN_DATE.xml" ]; then
    PASS_COUNT=$(xmllint --xpath "count(//rule-result[@result='pass'])" $REPORT_DIR/openscap-results-$SCAN_DATE.xml)
    FAIL_COUNT=$(xmllint --xpath "count(//rule-result[@result='fail'])" $REPORT_DIR/openscap-results-$SCAN_DATE.xml)
    
    echo "Passed Rules: $PASS_COUNT" >> $REPORT_DIR/scan-summary-$SCAN_DATE.txt
    echo "Failed Rules: $FAIL_COUNT" >> $REPORT_DIR/scan-summary-$SCAN_DATE.txt
    
    # Check for critical failures
    if [ "$FAIL_COUNT" -gt 10 ]; then
        echo "WARNING: High number of failed security rules detected!"
        echo "Critical Security Issues: YES" >> $REPORT_DIR/scan-summary-$SCAN_DATE.txt
        exit 1
    else
        echo "Critical Security Issues: NO" >> $REPORT_DIR/scan-summary-$SCAN_DATE.txt
    fi
fi

echo "OpenSCAP scan completed successfully!"
echo "Reports saved in: $REPORT_DIR"
Make the script executable and run it:

chmod +x scripts/openscap_scan.sh
cd scripts
./openscap_scan.sh
Task 3: Compliance as Code Testing with InSpec
Subtask 3.1: Install and Configure InSpec
InSpec is pre-installed in your environment. Let's verify and create compliance profiles:

# Verify InSpec installation
inspec version

# Create InSpec profile directory
mkdir -p compliance/inspec-profiles/aws-compliance
cd compliance/inspec-profiles/aws-compliance
Subtask 3.2: Create InSpec Compliance Profiles
Initialize a new InSpec profile:

inspec init profile aws-compliance --overwrite
Edit the main controls file:

nano controls/aws_compliance.rb
Add the following InSpec controls:

# AWS Infrastructure Compliance Controls
# This profile tests AWS infrastructure for security and compliance

title 'AWS Infrastructure Compliance'
maintainer 'DevSecOps Team'
copyright 'DevSecOps Team'
license 'Apache-2.0'
summary 'Compliance tests for AWS infrastructure'
version '1.0.0'

# Control 1: VPC Configuration
control 'aws-vpc-001' do
  impact 1.0
  title 'VPC should have DNS resolution enabled'
  desc 'Ensure VPC has DNS resolution and DNS hostnames enabled'
  
  describe aws_vpc(vpc_id: input('vpc_id')) do
    it { should exist }
    its('dns_resolution') { should eq 'enabled' }
    its('dns_hostnames') { should eq 'enabled' }
  end
end

# Control 2: Security Group Configuration
control 'aws-sg-001' do
  impact 1.0
  title 'Security groups should not allow unrestricted SSH access'
  desc 'Security groups should not have SSH (port 22) open to 0.0.0.0/0'
  
  aws_security_groups.group_ids.each do |group_id|
    describe aws_security_group(group_id: group_id) do
      it { should_not allow_in(port: 22, ipv4_range: '0.0.0.0/0') }
    end
  end
end

# Control 3: EC2 Instance Configuration
control 'aws-ec2-001' do
  impact 0.8
  title 'EC2 instances should have detailed monitoring enabled'
  desc 'EC2 instances should have detailed monitoring enabled for better observability'
  
  describe aws_ec2_instance(instance_id: input('instance_id')) do
    it { should exist }
    its('monitoring_state') { should eq 'enabled' }
  end
end

# Control 4: EBS Encryption
control 'aws-ebs-001' do
  impact 1.0
  title 'EBS volumes should be encrypted'
  desc 'All EBS volumes should be encrypted to protect data at rest'
  
  aws_ebs_volumes.volume_ids.each do |volume_id|
    describe aws_ebs_volume(volume_id: volume_id) do
      it { should be_encrypted }
    end
  end
end

# Control 5: S3 Bucket Security
control 'aws-s3-001' do
  impact 1.0
  title 'S3 buckets should have server-side encryption enabled'
  desc 'S3 buckets should have server-side encryption configured'
  
  describe aws_s3_bucket(bucket_name: input('s3_bucket_name')) do
    it { should have_server_side_encryption }
  end
end

# Control 6: S3 Bucket Versioning
control 'aws-s3-002' do
  impact 0.7
  title 'S3 buckets should have versioning enabled'
  desc 'S3 buckets should have versioning enabled for data protection'
  
  describe aws_s3_bucket(bucket_name: input('s3_bucket_name')) do
    it { should have_versioning_enabled }
  end
end

# Control 7: Resource Tagging
control 'aws-tag-001' do
  impact 0.5
  title 'Resources should be properly tagged'
  desc 'AWS resources should have required tags for governance'
  
  describe aws_ec2_instance(instance_id: input('instance_id')) do
    its('tags') { should include('Environment') }
    its('tags') { should include('Name') }
  end
end
Create an inputs file for the profile:

nano inspec.yml
Add the following configuration:

name: aws-compliance
title: AWS Infrastructure Compliance
maintainer: DevSecOps Team
copyright: DevSecOps Team
license: Apache-2.0
summary: Compliance tests for AWS infrastructure
version: 1.0.0

inputs:
  - name: vpc_id
    description: VPC ID to test
    type: string
    required: true
  
  - name: instance_id
    description: EC2 instance ID to test
    type: string
    required: true
  
  - name: s3_bucket_name
    description: S3 bucket name to test
    type: string
    required: true

depends:
  - name: inspec-aws
    git: https://github.com/inspec/inspec-aws.git
    tag: v1.81.0
Subtask 3.3: Run InSpec Compliance Tests
Create a script to run InSpec tests:

cd ../../../scripts
nano inspec_compliance.sh
Add the following script:

#!/bin/bash

# InSpec Compliance Testing Script
# This script runs InSpec compliance tests against AWS infrastructure

set -e

# Configuration
SCAN_DATE=$(date +%Y%m%d_%H%M%S)
REPORT_DIR="../reports/inspec"
PROFILE_PATH="../compliance/inspec-profiles/aws-compliance"

# Create report directory
mkdir -p $REPORT_DIR

# Read Terraform outputs
VPC_ID=$(grep 'vpc_id' ../outputs.txt | cut -d'"' -f2)
INSTANCE_ID=$(grep 'instance_id' ../outputs.txt | cut -d'"' -f2)
S3_BUCKET=$(grep 's3_bucket_name' ../outputs.txt | cut -d'"' -f2)

echo "Starting InSpec Compliance Testing..."
echo "Scan Date: $(date)"
echo "Profile: $PROFILE_PATH"

# Run InSpec tests
inspec exec $PROFILE_PATH \
    --input vpc_id=$VPC_ID \
    --input instance_id=$INSTANCE_ID \
    --input s3_bucket_name=$S3_BUCKET \
    --reporter cli json:$REPORT_DIR/inspec-results-$SCAN_DATE.json \
    --reporter html:$REPORT_DIR/inspec-report-$SCAN_DATE.html

# Generate summary
echo "Generating InSpec summary report..."
cat > $REPORT_DIR/inspec-summary-$SCAN_DATE.txt << EOF
InSpec Compliance Testing Summary
================================
Scan Date: $(date)
Profile: AWS Infrastructure Compliance
Version: 1.0.0

Test Results:
- VPC ID: $VPC_ID
- Instance ID: $INSTANCE_ID
- S3 Bucket: $S3_BUCKET

Reports Generated:
- JSON Results: inspec-results-$SCAN_DATE.json
- HTML Report: inspec-report-$SCAN_DATE.html

Status: Completed
EOF

echo "InSpec compliance testing completed!"
echo "Reports saved in: $REPORT_DIR"
Make the script executable and run it:

chmod +x inspec_compliance.sh
./inspec_compliance.sh
Task 4: Infrastructure as Code Security Testing with Checkov
Subtask 4.1: Install and Configure Checkov
Checkov is pre-installed in your environment. Let's verify and configure it:

# Verify Checkov installation
checkov --version

# Create Checkov configuration
nano ../compliance/checkov-config.yaml
Add the following Checkov configuration:

# Checkov Configuration for IaC Security Testing
framework:
  - terraform
  - dockerfile
  - kubernetes

check:
  - CKV_AWS_79  # Ensure Instance Metadata Service Version 1 is not enabled
  - CKV_AWS_8   # Ensure all data stored in the Launch configuration EBS is securely encrypted at rest
  - CKV_AWS_3   # Ensure all data stored in the EBS is securely encrypted at rest
  - CKV_AWS_21  # Ensure S3 bucket has versioning enabled
  - CKV_AWS_144 # Ensure that S3 bucket has server-side-encryption enabled
  - CKV_AWS_18  # Ensure S3 bucket has access logging configured
  - CKV_AWS_23  # Ensure every security groups rule has a description
  - CKV_AWS_24  # Ensure no security groups allow ingress from 0.0.0.0:0 to port 22
  - CKV_AWS_25  # Ensure no security groups allow ingress from 0.0.0.0:0 to port 3389

skip-check:
  - CKV_AWS_126 # Skip IAM policy checks for lab environment

output: cli
quiet: false
compact: false
Subtask 4.2: Create Checkov Security Testing Script
Create a comprehensive Checkov testing script:

nano checkov_security.sh
Add the following script:

#!/bin/bash

# Checkov IaC Security Testing Script
# This script performs security testing on Infrastructure as Code using Checkov

set -e

# Configuration
SCAN_DATE=$(date +%Y%m%d_%H%M%S)
REPORT_DIR="../reports/checkov"
TERRAFORM_DIR="../terraform"
CONFIG_FILE="../compliance/checkov-config.yaml"

# Create report directory
mkdir -p $REPORT_DIR

echo "Starting Checkov IaC Security Testing..."
echo "Scan Date: $(date)"
echo "Terraform Directory: $TERRAFORM_DIR"

# Run Checkov scan on Terraform files
checkov -d $TERRAFORM_DIR \
    --config-file $CONFIG_FILE \
    --output cli \
    --output json \
    --output-file-path $REPORT_DIR/checkov-results-$SCAN_DATE.json \
    --soft-fail || CHECKOV_EXIT_CODE=$?

# Generate HTML report
checkov -d $TERRAFORM_DIR \
    --config-file $CONFIG_FILE \
    --output html \
    --output-file-path $REPORT_DIR/checkov-report-$SCAN_DATE.html \
    --soft-fail

# Parse results and generate summary
if [ -f "$REPORT_DIR/checkov-results-$SCAN_DATE.json" ]; then
    PASSED_CHECKS=$(jq '.summary.passed' $REPORT_DIR/checkov-results-$SCAN_DATE.json)
    FAILED_CHECKS=$(jq '.summary.failed' $REPORT_DIR/checkov-results-$SCAN_DATE.json)
    SKIPPED_CHECKS=$(jq '.summary.skipped' $REPORT_DIR/checkov-results-$SCAN_DATE.json)
    
    # Generate summary report
    cat > $REPORT_DIR/checkov-summary-$SCAN_DATE.txt << EOF
Checkov IaC Security Testing Summary
===================================
Scan Date: $(date)
Terraform Directory: $TERRAFORM_DIR
Configuration: $CONFIG_FILE

Test Results:
- Passed Checks: $PASSED_CHECKS
- Failed Checks: $FAILED_CHECKS
- Skipped Checks: $SKIPPED_CHECKS

Reports Generated:
- JSON Results: checkov-results-$SCAN_DATE.json
- HTML Report: checkov-report-$SCAN_DATE.html

Status: Completed
EOF

    # Check for critical failures
    if [ "$FAILED_CHECKS" -gt 5 ]; then
        echo "WARNING: High number of failed security checks detected!"
        echo "Critical Security Issues: YES" >> $REPORT_DIR/checkov-summary-$SCAN_DATE.txt
        echo "Build should FAIL due to critical security issues"
        exit 1
    else
        echo "Critical Security Issues: NO" >> $REPORT_DIR/checkov-summary-$SCAN_DATE.txt
        echo "Build can PROCEED - security checks passed"
    fi
fi

echo "Checkov IaC security testing completed!"
echo "Reports saved in: $REPORT_DIR"
Make the script executable and run it:

chmod +x checkov_security.sh
./checkov_security.sh
Task 5: CI/CD Pipeline Integration with Jenkins
Subtask 5.1: Configure Jenkins Pipeline
Jenkins is pre-configured in your lab environment. Let's create a comprehensive pipeline that integrates all our compliance and security testing tools.

Create a Jenkins pipeline file:

nano ../compliance/Jenkinsfile
Add the following Jenkins pipeline configuration:

pipeline {
    agent any
    
    environment {
        AWS_DEFAULT_REGION = 'us-east-1'
        TERRAFORM_VERSION = '1.5.0'
        SCAN_DATE = sh(script: 'date +%Y%m%d_%H%M%S', returnStdout: true).trim()
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                checkout scm
            }
        }
        
        stage('Terraform Validate') {
            steps {
                dir('terraform') {
                    script {
                        echo 'Validating Terraform configuration...'
                        sh 'terraform init'
                        sh 'terraform validate'
                        sh 'terraform plan -out=tfplan'
                    }
                }
            }
        }
        
        stage('IaC Security Testing - Checkov') {
            steps {
                script {
                    echo 'Running Checkov IaC security tests...'
                    dir('scripts') {
                        sh './checkov_security.sh'
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'reports/checkov/**/*', fingerprint: true
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'reports/checkov',
                        reportFiles: "checkov-report-${env.SCAN_DATE}.html",
                        reportName: 'Checkov Security Report'
                    ])
                }
            }
        }
        
        stage('Deploy Infrastructure') {
            steps {
                dir('terraform') {
                    script {
                        echo 'Deploying infrastructure...'
                        sh 'terraform apply -auto-approve tfplan'
                        sh 'terraform output > ../outputs.txt'
                    }
                }
            }
        }
        
        stage('Security Baseline Testing - OpenSCAP') {
            steps {
                script {
                    echo 'Running OpenSCAP security baseline tests...'
                    dir('scripts') {
                        sh './openscap_scan.sh'
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'reports/openscap/**/*', fingerprint: true
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'reports/openscap',
                        reportFiles: "openscap-report-${env.SCAN_DATE}.html",
                        reportName: 'OpenSCAP Security Report'
                    ])
                }
            }
        }
        
        stage('Compliance Testing - InSpec') {
            steps {
                script {
                    echo 'Running InSpec compliance tests...'
                    dir('scripts') {
                        sh './inspec_compliance.sh'
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'reports/inspec/**/*', fingerprint: true
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'reports/inspec',
                        reportFiles: "inspec-report-${env.SCAN_DATE}.html",
                        reportName: 'InSpec Compliance Report'
                    ])
                }
            }
        }
        
        stage('Generate Consolidated Report') {
            steps {
                script {
                    echo 'Generating consolidated compliance report...'
                    sh '''
                        mkdir -p reports/consolidated
                        cat > reports/consolidated/compliance-dashboard-${SCAN_DATE}.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>DevSecOps Compliance Dashboard</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 20px; }
        .header { background-color: #2c3e50; color: white; padding: 20px; text-align: center; }
        .section { margin: 20px 0; padding: 15px; border: 1px solid #ddd; border-radius: 5px; }
        .pass { background-color: #d4edda; border-color: #c3e6cb; }
        .fail { background-color: #f8d7da; border-color: #f5c6cb; }
        .warning { background-color: #fff3cd; border-color: #ffeaa7; }
        .metric { display: inline-block; margin: 10px; padding: 10px; background-color: #f8f9fa; border-radius: 3px; }
    </style>
</head>
<body>
    <div class="header">
        <h1>DevSecOps Compliance Dashboard</h1>
        <p>Generated on: $(date)</p>
    </div>
    
    <div class="section">
        <h2>Security Testing Summary</h2>
        <div class="metric">
            <strong>Checkov IaC Security:</strong> 
            <span id="checkov-status">Completed</span>
        </div>
        <div class="metric">
            <strong>OpenSCAP Baseline:</strong> 
            <span id="openscap-status">Completed</span>
        </div>
        <div class="metric">
            <strong>InSpec Compliance:</strong> 
            <span id="inspec-status">Completed</span>
        </div>
    </div>
    
    <div class="section">
        <h2>Infrastructure Details</h2>
        <p><strong>Environment:</strong> DevSecOps Lab</p>
        <p><strong>Region:</strong> ${AWS_DEFAULT_REGION}</p>
        <p><strong>Scan Date:</strong> ${SCAN_DATE}</p>
    </div>
    
    <div class="section">
        <h2>Report Links</h2>
        <ul>
            <li><a href="checkov-report-${SCAN_DATE}.html">Checkov IaC Security Report</a></li>
            <li><a href="openscap-report-${SCAN_DATE}.html">OpenSCAP Security Baseline Report</a></li>
            <li><a href="inspec-report-${SCAN_DATE}.html">InSpec Compliance Report</a></li>
        </ul>
    </div>
</body>
</html>
EOF
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'reports/consolidated/**/*', fingerprint: true
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'reports/consolidated',
                        reportFiles: "compliance-dashboard-${env.SCAN_DATE}.html",
                        reportName: 'Compliance Dashboard'
                    ])
                }
            }
        }
        
