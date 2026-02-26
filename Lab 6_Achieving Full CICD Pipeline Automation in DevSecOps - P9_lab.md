Lab 6: Achieving Full CI/CD Pipeline Automation in DevSecOps
Lab Objectives
By the end of this lab, students will be able to:

• Implement a complete DevSecOps pipeline using GitLab CI/CD • Integrate security scanning tools including Checkmarx, OWASP Dependency Check, and ZAP • Configure automated deployment to staging and production environments • Set up continuous monitoring using Prometheus and Grafana • Understand the complete DevSecOps workflow from code commit to production monitoring • Troubleshoot common pipeline issues and security scan failures

Prerequisites
Before starting this lab, students should have:

• Basic understanding of Git and version control concepts • Familiarity with Linux command line operations • Basic knowledge of Docker containers • Understanding of web application deployment concepts • Previous experience with CI/CD pipeline concepts

Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines for this lab. Simply click Start Lab to access your environment. No need to build your own VM or install additional software - everything is pre-installed and ready to use.

Your lab environment includes: • GitLab Community Edition server • Docker and Docker Compose • Pre-installed security scanning tools • Prometheus and Grafana monitoring stack • Sample web application for deployment

Task 1: Setting Up the Complete DevSecOps Pipeline
Subtask 1.1: Initialize GitLab Project and Repository
Access GitLab Interface

# Open your browser and navigate to GitLab
http://localhost:8080
Create New Project

Click New Project
Select Create blank project
Project name: devsecops-demo-app
Visibility: Private
Initialize with README: Checked
Clone the Repository

cd /home/student
git clone http://localhost:8080/root/devsecops-demo-app.git
cd devsecops-demo-app
Subtask 1.2: Create Sample Application
Create Application Structure

mkdir -p src/main/java/com/example/demo
mkdir -p src/test/java/com/example/demo
mkdir -p src/main/resources
Create Main Application File

cat > src/main/java/com/example/demo/DemoApplication.java << 'EOF'
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@SpringBootApplication
@RestController
public class DemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }

    @GetMapping("/")
    public String home() {
        return "Hello DevSecOps World!";
    }

    @GetMapping("/health")
    public String health() {
        return "Application is healthy";
    }
}
EOF
Create pom.xml for Maven Build

cat > pom.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    
    <groupId>com.example</groupId>
    <artifactId>devsecops-demo</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>
    
    <name>DevSecOps Demo Application</name>
    
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.7.0</version>
        <relativePath/>
    </parent>
    
    <properties>
        <java.version>11</java.version>
    </properties>
    
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
EOF
Create Dockerfile

cat > Dockerfile << 'EOF'
FROM openjdk:11-jre-slim

WORKDIR /app

COPY target/devsecops-demo-1.0.0.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
EOF
Subtask 1.3: Create GitLab CI/CD Pipeline Configuration
Create .gitlab-ci.yml File
cat > .gitlab-ci.yml << 'EOF'
stages:
  - build
  - test
  - security-scan
  - build-image
  - deploy-staging
  - security-test
  - deploy-production
  - monitor

variables:
  MAVEN_OPTS: "-Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository"
  DOCKER_DRIVER: overlay2
  APP_NAME: "devsecops-demo"
  STAGING_URL: "http://staging.example.com"
  PRODUCTION_URL: "http://production.example.com"

cache:
  paths:
    - .m2/repository/

# Build Stage
build:
  stage: build
  image: maven:3.8.1-openjdk-11
  script:
    - mvn clean compile
  artifacts:
    paths:
      - target/
    expire_in: 1 hour

# Unit Test Stage
unit-test:
  stage: test
  image: maven:3.8.1-openjdk-11
  script:
    - mvn test
  artifacts:
    reports:
      junit:
        - target/surefire-reports/TEST-*.xml
  coverage: '/Total.*?([0-9]{1,3})%/'

# OWASP Dependency Check
dependency-check:
  stage: security-scan
  image: owasp/dependency-check:latest
  script:
    - /usr/share/dependency-check/bin/dependency-check.sh 
      --project "$APP_NAME" 
      --scan . 
      --format XML 
      --format HTML 
      --out dependency-check-report
  artifacts:
    paths:
      - dependency-check-report/
    expire_in: 1 week
  allow_failure: true

# Static Code Analysis (Checkmarx Alternative - SonarQube)
sast-scan:
  stage: security-scan
  image: sonarsource/sonar-scanner-cli:latest
  script:
    - sonar-scanner 
      -Dsonar.projectKey=$APP_NAME 
      -Dsonar.sources=src/main/java 
      -Dsonar.host.url=http://sonarqube:9000 
      -Dsonar.login=$SONAR_TOKEN
  allow_failure: true

# Build Docker Image
build-docker:
  stage: build-image
  image: docker:20.10.16
  services:
    - docker:20.10.16-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - mvn package -DskipTests
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  dependencies:
    - build

# Deploy to Staging
deploy-staging:
  stage: deploy-staging
  image: docker:20.10.16
  services:
    - docker:20.10.16-dind
  script:
    - docker pull $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker stop staging-app || true
    - docker rm staging-app || true
    - docker run -d --name staging-app -p 8081:8080 $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - sleep 30
    - curl -f http://localhost:8081/health || exit 1
  environment:
    name: staging
    url: http://localhost:8081
  dependencies:
    - build-docker

# ZAP Security Testing
zap-security-test:
  stage: security-test
  image: owasp/zap2docker-stable:latest
  script:
    - mkdir -p zap-reports
    - zap-baseline.py -t http://localhost:8081 -J zap-report.json -r zap-report.html
  artifacts:
    paths:
      - zap-reports/
    expire_in: 1 week
  dependencies:
    - deploy-staging
  allow_failure: true

# Deploy to Production
deploy-production:
  stage: deploy-production
  image: docker:20.10.16
  services:
    - docker:20.10.16-dind
  script:
    - docker pull $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker stop production-app || true
    - docker rm production-app || true
    - docker run -d --name production-app -p 8080:8080 $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - sleep 30
    - curl -f http://localhost:8080/health || exit 1
  environment:
    name: production
    url: http://localhost:8080
  when: manual
  dependencies:
    - build-docker
  only:
    - main

# Setup Monitoring
setup-monitoring:
  stage: monitor
  image: alpine:latest
  script:
    - echo "Setting up monitoring for production deployment"
    - echo "Prometheus metrics endpoint: http://localhost:8080/actuator/prometheus"
    - echo "Grafana dashboard: http://localhost:3000"
  dependencies:
    - deploy-production
  only:
    - main
EOF
Task 2: Configure Security Scanning Tools
Subtask 2.1: Set Up OWASP Dependency Check
Create Dependency Check Configuration
mkdir -p .dependency-check
cat > .dependency-check/suppression.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<suppressions xmlns="https://jeremylong.github.io/DependencyCheck/dependency-suppression.1.3.xsd">
    <!-- Example suppression for false positives -->
    <suppress>
        <notes><![CDATA[
        This suppresses false positives related to Spring Boot
        ]]></notes>
        <packageUrl regex="true">^pkg:maven/org\.springframework\.boot/.*$</packageUrl>
        <cve>CVE-2016-1000027</cve>
    </suppress>
</suppressions>
EOF
Subtask 2.2: Configure SonarQube for Static Analysis
Create SonarQube Properties File
cat > sonar-project.properties << 'EOF'
sonar.projectKey=devsecops-demo
sonar.projectName=DevSecOps Demo Application
sonar.projectVersion=1.0.0
sonar.sources=src/main/java
sonar.tests=src/test/java
sonar.java.binaries=target/classes
sonar.java.test.binaries=target/test-classes
sonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
EOF
Subtask 2.3: Configure ZAP Security Testing
Create ZAP Configuration
mkdir -p zap-config
cat > zap-config/zap-baseline.conf << 'EOF'
# ZAP Baseline Configuration
# Ignore specific alerts that are false positives
10021 IGNORE (X-Content-Type-Options Header Missing)
10020 IGNORE (X-Frame-Options Header Not Set)
EOF
Task 3: Set Up Staging and Production Environments
Subtask 3.1: Create Docker Compose for Environments
Create Staging Environment Configuration

mkdir -p environments/staging
cat > environments/staging/docker-compose.yml << 'EOF'
version: '3.8'
services:
  app:
    image: ${CI_REGISTRY_IMAGE}:${CI_COMMIT_SHA}
    ports:
      - "8081:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=staging
      - MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE=health,info,metrics,prometheus
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped
EOF
Create Production Environment Configuration

mkdir -p environments/production
cat > environments/production/docker-compose.yml << 'EOF'
version: '3.8'
services:
  app:
    image: ${CI_REGISTRY_IMAGE}:${CI_COMMIT_SHA}
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=production
      - MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE=health,info,metrics,prometheus
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 512M
        reservations:
          memory: 256M
EOF
Subtask 3.2: Create Environment-Specific Configuration
Create Staging Application Properties

mkdir -p src/main/resources
cat > src/main/resources/application-staging.yml << 'EOF'
server:
  port: 8080

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: always

logging:
  level:
    com.example.demo: DEBUG
EOF
Create Production Application Properties

cat > src/main/resources/application-production.yml << 'EOF'
server:
  port: 8080

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: when-authorized

logging:
  level:
    com.example.demo: INFO
    root: WARN
EOF
Task 4: Configure Monitoring with Prometheus and Grafana
Subtask 4.1: Set Up Prometheus Configuration
Create Prometheus Configuration

mkdir -p monitoring/prometheus
cat > monitoring/prometheus/prometheus.yml << 'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alert_rules.yml"

scrape_configs:
  - job_name: 'devsecops-demo-staging'
    static_configs:
      - targets: ['localhost:8081']
    metrics_path: '/actuator/prometheus'
    scrape_interval: 10s

  - job_name: 'devsecops-demo-production'
    static_configs:
      - targets: ['localhost:8080']
    metrics_path: '/actuator/prometheus'
    scrape_interval: 10s

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093
EOF
Create Alert Rules

cat > monitoring/prometheus/alert_rules.yml << 'EOF'
groups:
- name: devsecops-demo-alerts
  rules:
  - alert: ApplicationDown
    expr: up{job=~"devsecops-demo-.*"} == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Application {{ $labels.job }} is down"
      description: "Application {{ $labels.job }} has been down for more than 1 minute."

  - alert: HighResponseTime
    expr: http_request_duration_seconds{quantile="0.95"} > 1
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "High response time on {{ $labels.job }}"
      description: "95th percentile response time is {{ $value }} seconds."

  - alert: HighErrorRate
    expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.1
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "High error rate on {{ $labels.job }}"
      description: "Error rate is {{ $value }} errors per second."
EOF
Subtask 4.2: Set Up Grafana Dashboard
Create Grafana Configuration

mkdir -p monitoring/grafana/provisioning/datasources
mkdir -p monitoring/grafana/provisioning/dashboards

cat > monitoring/grafana/provisioning/datasources/prometheus.yml << 'EOF'
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
EOF
Create Dashboard Configuration

cat > monitoring/grafana/provisioning/dashboards/dashboard.yml << 'EOF'
apiVersion: 1

providers:
  - name: 'default'
    orgId: 1
    folder: ''
    type: file
    disableDeletion: false
    updateIntervalSeconds: 10
    allowUiUpdates: true
    options:
      path: /etc/grafana/provisioning/dashboards
EOF
Create Application Dashboard JSON

cat > monitoring/grafana/provisioning/dashboards/devsecops-dashboard.json << 'EOF'
{
  "dashboard": {
    "id": null,
    "title": "DevSecOps Demo Application",
    "tags": ["devsecops"],
    "timezone": "browser",
    "panels": [
      {
        "id": 1,
        "title": "Application Status",
        "type": "stat",
        "targets": [
          {
            "expr": "up{job=~\"devsecops-demo-.*\"}",
            "legendFormat": "{{ job }}"
          }
        ],
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 0}
      },
      {
        "id": 2,
        "title": "Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])",
            "legendFormat": "{{ job }} - {{ method }} {{ status }}"
          }
        ],
        "gridPos": {"h": 8, "w": 12, "x": 12, "y": 0}
      },
      {
        "id": 3,
        "title": "Response Time",
        "type": "graph",
        "targets": [
          {
            "expr": "http_request_duration_seconds{quantile=\"0.95\"}",
            "legendFormat": "95th percentile"
          }
        ],
        "gridPos": {"h": 8, "w": 24, "x": 0, "y": 8}
      }
    ],
    "time": {"from": "now-1h", "to": "now"},
    "refresh": "5s"
  }
}
EOF
Subtask 4.3: Create Monitoring Stack Docker Compose
Create Complete Monitoring Stack

cat > monitoring/docker-compose.yml << 'EOF'
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus/alert_rules.yml:/etc/prometheus/alert_rules.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--storage.tsdb.retention.time=200h'
      - '--web.enable-lifecycle'
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin123
      - GF_USERS_ALLOW_SIGN_UP=false
    restart: unless-stopped

  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml
      - alertmanager_data:/alertmanager
    restart: unless-stopped

volumes:
  prometheus_data:
  grafana_data:
  alertmanager_data:

networks:
  default:
    name: monitoring
EOF
Create Alertmanager Configuration

mkdir -p monitoring/alertmanager
cat > monitoring/alertmanager/alertmanager.yml << 'EOF'
global:
  smtp_smarthost: 'localhost:587'
  smtp_from: 'alerts@example.com'

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'web.hook'

receivers:
- name: 'web.hook'
  webhook_configs:
  - url: 'http://localhost:5001/webhook'
    send_resolved: true
EOF
Task 5: Commit and Execute the Pipeline
Subtask 5.1: Commit All Files to Repository
Add All Files to Git

git add .
git config user.email "student@example.com"
git config user.name "DevSecOps Student"
git commit -m "Initial DevSecOps pipeline setup with complete CI/CD automation"
Push to GitLab

git push origin main
Subtask 5.2: Configure GitLab Variables
Set Up CI/CD Variables in GitLab
Navigate to Project Settings > CI/CD > Variables
Add the following variables:
SONAR_TOKEN: your-sonar-token
CI_REGISTRY: localhost:5000
CI_REGISTRY_USER: gitlab-ci-token
CI_REGISTRY_PASSWORD: $CI_JOB_TOKEN
Subtask 5.3: Monitor Pipeline Execution
Watch Pipeline Progress

Go to CI/CD > Pipelines in GitLab
Click on the running pipeline to see detailed logs
Monitor each stage completion
Verify Security Scans

# Check dependency scan results
curl -H "PRIVATE-TOKEN: your-token" \
     "http://localhost:8080/api/v4/projects/1/jobs/artifacts/main/browse/dependency-check-report" \
     --output dependency-report.zip

# Check ZAP scan results
curl -H "PRIVATE-TOKEN: your-token" \
     "http://localhost:8080/api/v4/projects/1/jobs/artifacts/main/browse/zap-reports" \
     --output zap-report.zip
Task 6: Start Monitoring Stack and Verify Deployment
Subtask 6.1: Launch Monitoring Infrastructure
Start Monitoring Stack

cd monitoring
docker-compose up -d
Verify Monitoring Services

# Check Prometheus
curl http://localhost:9090/api/v1/targets

# Check Grafana
curl http://localhost:3000/api/health

# Verify application metrics
curl http://localhost:8080/actuator/prometheus
Subtask 6.2: Access Monitoring Dashboards
Access Grafana Dashboard

Open browser: http://localhost:3000
Login: admin / admin123
Navigate to Dashboards > DevSecOps Demo Application
Access Prometheus

Open browser: http://localhost:9090
Check Status > Targets to verify scraping
Subtask 6.3: Test Complete Pipeline Flow
Make a Code Change

# Edit the application
sed -i 's/Hello DevSecOps World!/Hello Secure DevSecOps World - Updated!/' \
    src/main/java/com/example/demo/DemoApplication.java

# Commit and push
git add .
git commit -m "Update application message - testing complete pipeline"
git push origin main
Monitor Pipeline Execution

Watch the pipeline run through all stages
Verify staging deployment
Manually approve production deployment
Check monitoring dashboards for new deployment
Troubleshooting Common Issues
Pipeline Failures
Issue: Maven build fails

# Solution: Check Java version and dependencies
docker run --rm -v $(pwd):/app -w /app maven:3.8.1-openjdk-11 mvn dependency:tree
Issue: Docker build fails

# Solution: Check Dockerfile syntax and build context
docker build --no-cache -t test-build .
Issue: Security scans fail

# Solution: Check scan configurations and update suppression files
# For OWASP Dependency Check
dependency-check --project test --scan . --format XML --suppression .dependency-check/suppression.xml
Deployment Issues
Issue: Application health check fails

# Solution: Check application logs
docker logs staging-app
docker logs production-app

# Verify health endpoint
curl -v http://localhost:8080/actuator/health
Issue: Monitoring not working

# Solution: Check Prometheus configuration
docker exec prometheus promtool check config /etc/prometheus/prometheus.yml

# Verify network connectivity
docker exec prometheus wget -qO- http://localhost:8080/actuator/prometheus
Verification Steps
Complete Pipeline Verification
Verify All Pipeline Stages

# Check build artifacts
ls -la target/

# Verify security scan reports
ls -la dependency-check-report/
ls -la zap-reports/

# Check deployed applications
curl http://localhost:8081/health  # Staging
curl http://localhost:8080/health  # Production
Verify Monitoring Integration

# Check Prometheus metrics
curl http://localhost:9090/api/v1/query?query=up

# Verify Grafana data source
curl -u admin:admin123 http://localhost:3000/api/datasources
Test Security Integration

# Verify dependency check results
grep -i "vulnerability" dependency-check-report/dependency-check-report.xml

# Check ZAP scan results
grep -i "risk" zap-reports/zap-report.html
Conclusion
Congratulations! You have successfully implemented a complete DevSecOps CI/CD pipeline that includes:

What You Accomplished: • Complete CI/CD Automation: Built a fully automated pipeline from code commit to production deployment • Integrated Security Scanning: Implemented OWASP Dependency Check, static code analysis, and dynamic security testing with ZAP • Multi-Environment Deployment: Set up automated deployment to staging with manual approval for production • Continuous Monitoring: Configured Prometheus and Grafana for real-time application monitoring and alerting • Security-First Approach: Embedded security checks at every stage of the pipeline

Why This Matters: This lab demonstrates industry best practices for DevSecOps implementation. By integrating security tools directly into the CI/CD pipeline, you ensure that security is not an afterthought but a fundamental part of the development process. The monitoring setup provides visibility into application performance and security posture in real-time.

Key Takeaways: • Security scanning should be automated and integrated into every build • Staging environments provide safe testing grounds for security validation • Continuous monitoring is essential for maintaining security and performance in production • Manual approval gates for production deployments provide necessary control while maintaining automation • Open-source tools can provide enterprise-grade DevSecOps capabilities

This complete DevSecOps pipeline serves as a foundation that can be extended with additional security tools, more sophisticated deployment strategies, and enhanced monitoring capabilities as your organization's needs grow.

