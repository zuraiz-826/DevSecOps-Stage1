Lab 5. CICD Fundamentals - P8
Lab Objectives
By the end of this lab, students will be able to:

• Understand the fundamentals of CI/CD pipeline implementation using open-source tools • Configure GitLab for source code management and CI/CD automation • Integrate SonarQube for code quality analysis and security scanning • Implement OWASP Dependency Check for vulnerability assessment • Configure ZAP (Zed Attack Proxy) for dynamic application security testing • Set up security gates in SonarQube to enforce quality standards • Deploy applications to staging and production environments using automated pipelines • Understand DevSecOps principles and their practical implementation

Prerequisites
Before starting this lab, students should have:

• Basic understanding of Git version control system • Familiarity with Linux command line operations • Basic knowledge of web application development • Understanding of containerization concepts (Docker basics) • Basic networking concepts • Familiarity with YAML syntax

Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines for this lab. Simply click Start Lab to access your environment. No need to build your own VM or install additional software - everything is ready to use!

Your lab environment includes: • Ubuntu 20.04 LTS with Docker pre-installed • GitLab Community Edition • SonarQube Community Edition • OWASP ZAP • Sample web application for testing

Lab Tasks
Task 1: Environment Preparation and Initial Setup
Subtask 1.1: Verify Lab Environment
First, let's verify that all required services are running in your lab environment.

# Check Docker status
sudo systemctl status docker

# Verify GitLab is running
sudo docker ps | grep gitlab

# Check SonarQube container
sudo docker ps | grep sonarqube

# Verify ZAP installation
which zap.sh
Subtask 1.2: Access GitLab Interface
Open your web browser and navigate to GitLab:
http://localhost:8080
Use the default credentials:

Username: root
Password: password123
Create a new project:

Click New Project
Select Create blank project
Project name: devsecops-demo
Visibility: Private
Click Create project
Subtask 1.3: Clone Sample Application
# Navigate to your workspace
cd /home/student/workspace

# Clone the sample application
git clone https://github.com/OWASP/NodeGoat.git devsecops-demo
cd devsecops-demo

# Initialize as GitLab repository
git remote remove origin
git remote add origin http://localhost:8080/root/devsecops-demo.git

# Push to GitLab
git push -u origin main
Task 2: Configure SonarQube Integration
Subtask 2.1: Access and Configure SonarQube
Access SonarQube web interface:
http://localhost:9000
Default login credentials:

Username: admin
Password: admin
Change the default password when prompted.

Subtask 2.2: Create SonarQube Project
Click Create Project > Manually
Project settings:
Project key: devsecops-demo
Display name: DevSecOps Demo
Click Set Up
Choose Use existing token or create a new one
Save the generated token: squ_1234567890abcdef (example)
Subtask 2.3: Configure Quality Gates
Navigate to Quality Gates in SonarQube
Click Create
Name: DevSecOps-Gate
Add conditions:
Coverage: Less than 80%
Duplicated Lines: Greater than 3%
Maintainability Rating: Worse than A
Reliability Rating: Worse than A
Security Rating: Worse than A
# Create sonar-project.properties file
cat > sonar-project.properties << EOF
sonar.projectKey=devsecops-demo
sonar.projectName=DevSecOps Demo
sonar.projectVersion=1.0
sonar.sources=.
sonar.exclusions=node_modules/**,coverage/**
sonar.javascript.lcov.reportPaths=coverage/lcov.info
EOF
Task 3: Configure OWASP Dependency Check
Subtask 3.1: Install OWASP Dependency Check
# Download OWASP Dependency Check
cd /opt
sudo wget https://github.com/jeremylong/DependencyCheck/releases/download/v8.4.0/dependency-check-8.4.0-release.zip

# Extract the tool
sudo unzip dependency-check-8.4.0-release.zip
sudo chmod +x dependency-check/bin/dependency-check.sh

# Create symbolic link
sudo ln -s /opt/dependency-check/bin/dependency-check.sh /usr/local/bin/dependency-check
Subtask 3.2: Test Dependency Check
# Navigate to project directory
cd /home/student/workspace/devsecops-demo

# Run dependency check
dependency-check --project "DevSecOps Demo" --scan . --format HTML --format JSON --out ./dependency-check-report
Task 4: Configure ZAP (Zed Attack Proxy)
Subtask 4.1: Install and Configure ZAP
# Install ZAP if not already installed
sudo apt update
sudo apt install zaproxy -y

# Create ZAP configuration directory
mkdir -p /home/student/.ZAP
Subtask 4.2: Create ZAP Automation Script
# Create ZAP automation script
cat > zap-automation.py << 'EOF'
#!/usr/bin/env python3
import time
import subprocess
import sys

def run_zap_scan(target_url):
    """Run ZAP baseline scan"""
    cmd = [
        'zap-baseline.py',
        '-t', target_url,
        '-J', 'zap-report.json',
        '-r', 'zap-report.html'
    ]
    
    try:
        result = subprocess.run(cmd, capture_output=True, text=True, timeout=300)
        print("ZAP scan completed")
        return result.returncode == 0
    except subprocess.TimeoutExpired:
        print("ZAP scan timed out")
        return False
    except Exception as e:
        print(f"Error running ZAP scan: {e}")
        return False

if __name__ == "__main__":
    if len(sys.argv) != 2:
        print("Usage: python3 zap-automation.py <target_url>")
        sys.exit(1)
    
    target_url = sys.argv[1]
    success = run_zap_scan(target_url)
    sys.exit(0 if success else 1)
EOF

chmod +x zap-automation.py
Task 5: Create GitLab CI/CD Pipeline
Subtask 5.1: Configure GitLab Runner
# Install GitLab Runner
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt-get install gitlab-runner

# Register runner
sudo gitlab-runner register \
  --url http://localhost:8080 \
  --registration-token <your-registration-token> \
  --executor docker \
  --docker-image docker:latest \
  --description "DevSecOps Runner"
Subtask 5.2: Create CI/CD Pipeline Configuration
Create the .gitlab-ci.yml file in your project root:

# .gitlab-ci.yml
stages:
  - build
  - test
  - security-scan
  - quality-gate
  - deploy-staging
  - security-test
  - deploy-production

variables:
  DOCKER_DRIVER: overlay2
  SONAR_USER_HOME: "${CI_PROJECT_DIR}/.sonar"
  GIT_DEPTH: "0"

# Build Stage
build:
  stage: build
  image: node:16
  script:
    - npm install
    - npm run build
  artifacts:
    paths:
      - node_modules/
      - dist/
    expire_in: 1 hour
  only:
    - main
    - develop

# Unit Tests
unit-tests:
  stage: test
  image: node:16
  script:
    - npm install
    - npm test -- --coverage
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml
    paths:
      - coverage/
  only:
    - main
    - develop

# OWASP Dependency Check
dependency-check:
  stage: security-scan
  image: owasp/dependency-check:latest
  script:
    - /usr/share/dependency-check/bin/dependency-check.sh 
      --project "$CI_PROJECT_NAME" 
      --scan . 
      --format HTML 
      --format JSON 
      --out ./dependency-check-report
  artifacts:
    paths:
      - dependency-check-report/
    reports:
      junit: dependency-check-report/dependency-check-junit.xml
    expire_in: 1 week
  allow_failure: true
  only:
    - main
    - develop

# SonarQube Analysis
sonarqube-check:
  stage: security-scan
  image: 
    name: sonarsource/sonar-scanner-cli:latest
    entrypoint: [""]
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
      -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info
  allow_failure: true
  only:
    - main
    - develop

# Quality Gate Check
quality-gate:
  stage: quality-gate
  image: curlimages/curl:latest
  script:
    - |
      # Wait for SonarQube analysis to complete
      sleep 30
      
      # Check quality gate status
      QUALITY_GATE_STATUS=$(curl -s -u $SONAR_TOKEN: \
        "http://sonarqube:9000/api/qualitygates/project_status?projectKey=devsecops-demo" \
        | grep -o '"status":"[^"]*"' | cut -d'"' -f4)
      
      echo "Quality Gate Status: $QUALITY_GATE_STATUS"
      
      if [ "$QUALITY_GATE_STATUS" != "OK" ]; then
        echo "Quality gate failed!"
        exit 1
      fi
      
      echo "Quality gate passed!"
  dependencies:
    - sonarqube-check
  only:
    - main
    - develop

# Deploy to Staging
deploy-staging:
  stage: deploy-staging
  image: docker:latest
  services:
    - docker:dind
  script:
    - echo "Building Docker image for staging..."
    - docker build -t $CI_PROJECT_NAME:staging .
    - echo "Deploying to staging environment..."
    - docker run -d --name staging-app -p 3001:3000 $CI_PROJECT_NAME:staging
    - echo "Application deployed to staging: http://localhost:3001"
  environment:
    name: staging
    url: http://localhost:3001
  dependencies:
    - build
    - quality-gate
  only:
    - main
    - develop

# ZAP Security Testing
zap-security-test:
  stage: security-test
  image: owasp/zap2docker-stable:latest
  script:
    - mkdir -p /zap/wrk
    - |
      # Wait for staging deployment
      sleep 60
      
      # Run ZAP baseline scan
      zap-baseline.py -t http://staging-app:3000 \
        -J zap-report.json \
        -r zap-report.html \
        || true
  artifacts:
    paths:
      - zap-report.html
      - zap-report.json
    expire_in: 1 week
  dependencies:
    - deploy-staging
  allow_failure: true
  only:
    - main

# Deploy to Production
deploy-production:
  stage: deploy-production
  image: docker:latest
  services:
    - docker:dind
  script:
    - echo "Building Docker image for production..."
    - docker build -t $CI_PROJECT_NAME:production .
    - echo "Deploying to production environment..."
    - docker run -d --name production-app -p 3000:3000 $CI_PROJECT_NAME:production
    - echo "Application deployed to production: http://localhost:3000"
  environment:
    name: production
    url: http://localhost:3000
  dependencies:
    - zap-security-test
  when: manual
  only:
    - main
Subtask 5.3: Create Dockerfile
# Dockerfile
FROM node:16-alpine

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy application code
COPY . .

# Create non-root user
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nodeuser -u 1001

# Change ownership
RUN chown -R nodeuser:nodejs /app
USER nodeuser

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

# Start application
CMD ["npm", "start"]
Task 6: Configure Environment Variables and Secrets
Subtask 6.1: Set GitLab CI/CD Variables
Navigate to your GitLab project
Go to Settings > CI/CD > Variables
Add the following variables:
SONAR_TOKEN: <your-sonarqube-token>
SONAR_HOST_URL: http://sonarqube:9000
DOCKER_REGISTRY: localhost:5000
Subtask 6.2: Create Environment-Specific Configurations
# Create staging configuration
cat > config/staging.json << EOF
{
  "environment": "staging",
  "database": {
    "host": "staging-db",
    "port": 5432,
    "name": "app_staging"
  },
  "security": {
    "enableHttps": false,
    "corsOrigins": ["http://localhost:3001"]
  }
}
EOF

# Create production configuration
cat > config/production.json << EOF
{
  "environment": "production",
  "database": {
    "host": "production-db",
    "port": 5432,
    "name": "app_production"
  },
  "security": {
    "enableHttps": true,
    "corsOrigins": ["https://yourdomain.com"]
  }
}
EOF
Task 7: Testing and Validation
Subtask 7.1: Trigger Pipeline Execution
# Make a change to trigger the pipeline
echo "# DevSecOps Pipeline Demo" > README.md
git add README.md
git commit -m "feat: trigger DevSecOps pipeline"
git push origin main
Subtask 7.2: Monitor Pipeline Execution
Navigate to GitLab project
Go to CI/CD > Pipelines
Click on the running pipeline to monitor progress
Check each stage for successful completion
Subtask 7.3: Review Security Reports
SonarQube Report:

Access SonarQube dashboard
Review code quality metrics
Check security vulnerabilities
OWASP Dependency Check:

Download artifacts from GitLab pipeline
Review dependency-check-report.html
ZAP Security Test:

Download ZAP report artifacts
Review security findings
Task 8: Troubleshooting Common Issues
Issue 1: GitLab Runner Not Connecting
# Check runner status
sudo gitlab-runner status

# Restart runner if needed
sudo gitlab-runner restart

# Verify runner registration
sudo gitlab-runner list
Issue 2: SonarQube Connection Issues
# Check SonarQube container logs
docker logs sonarqube

# Verify network connectivity
curl -I http://localhost:9000

# Test SonarQube API
curl -u admin:admin http://localhost:9000/api/system/status
Issue 3: Docker Build Failures
# Check Docker daemon status
sudo systemctl status docker

# Clean up Docker resources
docker system prune -f

# Check available disk space
df -h
Issue 4: Pipeline Permissions
# Fix file permissions
chmod +x scripts/*.sh

# Update GitLab Runner permissions
sudo usermod -aG docker gitlab-runner
sudo systemctl restart gitlab-runner
Lab Validation and Testing
Validation Checklist
 GitLab project created and configured
 SonarQube integration working
 OWASP Dependency Check running
 ZAP security testing configured
 Quality gates enforced
 Staging deployment successful
 Production deployment manual approval working
 All security reports generated
Testing Commands
# Test staging deployment
curl -I http://localhost:3001

# Test production deployment
curl -I http://localhost:3000

# Verify security reports exist
ls -la dependency-check-report/
ls -la zap-report.*
Conclusion
Congratulations! You have successfully completed Lab 5: CICD Fundamentals - P8. In this comprehensive lab, you have:

Key Accomplishments:

• Implemented a complete DevSecOps pipeline using open-source tools including GitLab, SonarQube, OWASP Dependency Check, and ZAP • Configured automated security scanning at multiple stages of the development lifecycle • Set up quality gates in SonarQube to enforce code quality and security standards • Created automated deployment pipelines for both staging and production environments • Integrated security testing into the CI/CD process using industry-standard tools • Implemented proper environment separation with different configurations for staging and production

Why This Matters:

This lab demonstrates real-world DevSecOps practices that are essential in modern software development. By integrating security throughout the development pipeline, organizations can:

Detect vulnerabilities early in the development process, reducing costs and risks
Maintain consistent code quality through automated analysis and quality gates
Ensure secure deployments through comprehensive security testing
Accelerate delivery while maintaining security and quality standards
Implement compliance requirements through automated security controls
Next Steps:

You now have the foundation to implement DevSecOps practices in your own projects. Consider exploring:

Advanced security testing techniques
Infrastructure as Code (IaC) security scanning
Container security scanning
Compliance automation
Advanced monitoring and alerting
The skills you've learned in this lab are directly applicable to enterprise environments and will help you build more secure, reliable, and maintainable applications.
