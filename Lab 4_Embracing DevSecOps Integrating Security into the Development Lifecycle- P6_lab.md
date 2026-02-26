Lab 4: Embracing DevSecOps - Integrating Security into the Development Lifecycle
Lab Objectives
By the end of this lab, students will be able to:

• Understand the core principles of DevSecOps and its importance in modern software development • Set up and configure Jenkins for continuous integration with security integration • Install and configure SonarQube for static code analysis and security vulnerability detection • Implement OWASP Dependency Check to identify vulnerable dependencies in projects • Configure OWASP ZAP for automated security testing • Create a complete DevSecOps pipeline that integrates security at every stage of development • Analyze security reports and understand how to remediate common vulnerabilities • Implement security gates in CI/CD pipelines to prevent insecure code deployment

Prerequisites
Before starting this lab, students should have:

• Basic understanding of software development lifecycle (SDLC) • Familiarity with Linux command line operations • Basic knowledge of version control systems (Git) • Understanding of web applications and common security vulnerabilities • Basic knowledge of continuous integration concepts

Note: Al Nafi provides ready-to-use Linux-based cloud machines for this lab. Simply click "Start Lab" to access your pre-configured environment - no need to build your own VM.

Lab Environment Setup
Your cloud machine comes pre-installed with: • Ubuntu 20.04 LTS • Docker and Docker Compose • Java 11 • Git • Maven • Basic development tools

Task 1: Understanding DevSecOps Fundamentals
Subtask 1.1: DevSecOps Overview
DevSecOps integrates security practices within the DevOps process. Instead of treating security as a separate phase, it embeds security throughout the entire development lifecycle.

Key Principles: • Shift Left Security: Implement security early in the development process • Automation: Automate security testing and compliance checks • Continuous Monitoring: Monitor applications and infrastructure continuously • Collaboration: Foster collaboration between development, operations, and security teams

Subtask 1.2: Security Integration Points
In a DevSecOps pipeline, security is integrated at multiple stages:

Code Development: Static code analysis, secure coding practices
Build Phase: Dependency scanning, container security
Testing Phase: Dynamic application security testing (DAST)
Deployment: Infrastructure security, configuration management
Runtime: Continuous monitoring, incident response
Task 2: Setting Up Jenkins for DevSecOps
Subtask 2.1: Installing Jenkins
First, let's install Jenkins on your cloud machine:

# Update system packages
sudo apt update

# Install Java 11 (required for Jenkins)
sudo apt install openjdk-11-jdk -y

# Add Jenkins repository key
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -

# Add Jenkins repository
sudo sh -c 'echo deb https://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'

# Update package list
sudo apt update

# Install Jenkins
sudo apt install jenkins -y

# Start Jenkins service
sudo systemctl start jenkins
sudo systemctl enable jenkins

# Check Jenkins status
sudo systemctl status jenkins
Subtask 2.2: Initial Jenkins Configuration
# Get the initial admin password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
Open your web browser and navigate to http://your-server-ip:8080
Enter the initial admin password
Click "Install suggested plugins"
Create your first admin user
Configure Jenkins URL (use your server IP)
Subtask 2.3: Installing Required Jenkins Plugins
Navigate to Manage Jenkins > Manage Plugins > Available and install:

• SonarQube Scanner • OWASP Dependency-Check Plugin • Pipeline • Git Plugin • Maven Integration Plugin • Build Pipeline Plugin

Task 3: Setting Up SonarQube for Code Quality and Security Analysis
Subtask 3.1: Installing SonarQube with Docker
# Create a directory for SonarQube
mkdir ~/sonarqube
cd ~/sonarqube

# Create docker-compose.yml file
cat > docker-compose.yml << 'EOF'
version: '3.7'

services:
  sonarqube:
    image: sonarqube:9.9-community
    container_name: sonarqube
    ports:
      - "9000:9000"
    environment:
      - SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true
    volumes:
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_extensions:/opt/sonarqube/extensions
      - sonarqube_logs:/opt/sonarqube/logs

volumes:
  sonarqube_data:
  sonarqube_extensions:
  sonarqube_logs:
EOF

# Start SonarQube
docker-compose up -d

# Check if SonarQube is running
docker-compose ps
Subtask 3.2: Configuring SonarQube
Wait for SonarQube to start (it may take 2-3 minutes)
Access SonarQube at http://your-server-ip:9000
Login with default credentials: admin/admin
Change the default password when prompted
Generate a token for Jenkins integration:
Go to User > My Account > Security
Generate a new token named "jenkins-integration"
Save this token - you'll need it for Jenkins configuration
Subtask 3.3: Integrating SonarQube with Jenkins
In Jenkins, go to Manage Jenkins > Configure System
Find the SonarQube servers section
Add SonarQube server:
Name: SonarQube
Server URL: http://localhost:9000
Server authentication token: Add the token you generated
Task 4: Setting Up OWASP Dependency Check
Subtask 4.1: Installing OWASP Dependency Check
# Create directory for OWASP tools
mkdir ~/owasp-tools
cd ~/owasp-tools

# Download OWASP Dependency Check
wget https://github.com/jeremylong/DependencyCheck/releases/download/v8.4.0/dependency-check-8.4.0-release.zip

# Extract the archive
unzip dependency-check-8.4.0-release.zip

# Make the script executable
chmod +x dependency-check/bin/dependency-check.sh

# Add to PATH (optional)
echo 'export PATH=$PATH:~/owasp-tools/dependency-check/bin' >> ~/.bashrc
source ~/.bashrc
Subtask 4.2: Configuring OWASP Dependency Check in Jenkins
Go to Manage Jenkins > Global Tool Configuration
Find OWASP Dependency-Check installations
Add installation:
Name: OWASP-Dependency-Check
Install automatically: Uncheck
DEPENDENCY_CHECK_HOME: /home/ubuntu/owasp-tools/dependency-check
Task 5: Setting Up OWASP ZAP for Dynamic Security Testing
Subtask 5.1: Installing OWASP ZAP
# Install ZAP using Docker
docker pull owasp/zap2docker-stable

# Create a directory for ZAP reports
mkdir ~/zap-reports

# Test ZAP installation
docker run -t owasp/zap2docker-stable zap-baseline.py --help
Subtask 5.2: Creating a ZAP Scanning Script
# Create a ZAP scanning script
cat > ~/owasp-tools/zap-scan.sh << 'EOF'
#!/bin/bash

# ZAP Baseline Scan Script
TARGET_URL=$1
REPORT_DIR=$2

if [ -z "$TARGET_URL" ] || [ -z "$REPORT_DIR" ]; then
    echo "Usage: $0 <target_url> <report_directory>"
    exit 1
fi

echo "Starting ZAP baseline scan for: $TARGET_URL"

# Run ZAP baseline scan
docker run -v $REPORT_DIR:/zap/wrk/:rw \
    -t owasp/zap2docker-stable \
    zap-baseline.py \
    -t $TARGET_URL \
    -J zap-report.json \
    -H zap-report.html \
    -r zap-report.md

echo "ZAP scan completed. Reports saved to: $REPORT_DIR"
EOF

# Make the script executable
chmod +x ~/owasp-tools/zap-scan.sh
Task 6: Creating a Sample Application for Testing
Subtask 6.1: Setting Up a Vulnerable Web Application
# Clone a sample vulnerable application
cd ~
git clone https://github.com/WebGoat/WebGoat.git
cd WebGoat

# Build the application using Maven
mvn clean install -DskipTests

# Create a simple Dockerfile for the application
cat > Dockerfile << 'EOF'
FROM openjdk:11-jre-slim

COPY webgoat-server/target/webgoat-server-*.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "/app.jar"]
EOF
Subtask 6.2: Creating a Sample Java Project for Analysis
# Create a simple Java project structure
mkdir -p ~/sample-project/src/main/java/com/example
cd ~/sample-project

# Create pom.xml with some vulnerable dependencies
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
    
    <properties>
        <maven.compiler.source>11</maven.compiler.source>
        <maven.compiler.target>11</maven.compiler.target>
        <sonar.projectKey>devsecops-demo</sonar.projectKey>
        <sonar.projectName>DevSecOps Demo</sonar.projectName>
    </properties>
    
    <dependencies>
        <!-- Intentionally using older versions with known vulnerabilities -->
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-collections4</artifactId>
            <version>4.0</version>
        </dependency>
        
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>2.9.8</version>
        </dependency>
        
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.13.1</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.sonarsource.scanner.maven</groupId>
                <artifactId>sonar-maven-plugin</artifactId>
                <version>3.9.1.2184</version>
            </plugin>
        </plugins>
    </build>
</project>
EOF

# Create a sample Java class with security issues
cat > src/main/java/com/example/VulnerableApp.java << 'EOF'
package com.example;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.ResultSet;
import java.sql.Statement;
import java.util.Scanner;

public class VulnerableApp {
    
    // Hard-coded credentials (security vulnerability)
    private static final String DB_URL = "jdbc:mysql://localhost:3306/testdb";
    private static final String USERNAME = "admin";
    private static final String PASSWORD = "password123";
    
    public static void main(String[] args) {
        VulnerableApp app = new VulnerableApp();
        app.demonstrateVulnerabilities();
    }
    
    public void demonstrateVulnerabilities() {
        // SQL Injection vulnerability
        Scanner scanner = new Scanner(System.in);
        System.out.print("Enter user ID: ");
        String userId = scanner.nextLine();
        
        try {
            Connection conn = DriverManager.getConnection(DB_URL, USERNAME, PASSWORD);
            Statement stmt = conn.createStatement();
            
            // Vulnerable SQL query - direct string concatenation
            String query = "SELECT * FROM users WHERE id = '" + userId + "'";
            ResultSet rs = stmt.executeQuery(query);
            
            while (rs.next()) {
                System.out.println("User: " + rs.getString("username"));
            }
            
            conn.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
        
        scanner.close();
    }
    
    // Method with potential null pointer exception
    public String processUserInput(String input) {
        return input.toUpperCase(); // No null check
    }
    
    // Unused private method (code smell)
    private void unusedMethod() {
        System.out.println("This method is never called");
    }
}
EOF

# Initialize Git repository
git init
git add .
git commit -m "Initial commit with vulnerable code"
Task 7: Creating a DevSecOps Pipeline
Subtask 7.1: Creating a Jenkins Pipeline Script
# Create Jenkinsfile in the sample project
cd ~/sample-project

cat > Jenkinsfile << 'EOF'
pipeline {
    agent any
    
    tools {
        maven 'Maven'
        jdk 'Java-11'
    }
    
    environment {
        SONAR_TOKEN = credentials('sonar-token')
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                echo 'Building the application...'
                sh 'mvn clean compile'
            }
        }
        
        stage('Unit Tests') {
            steps {
                echo 'Running unit tests...'
                sh 'mvn test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }
        
        stage('Static Code Analysis') {
            steps {
                echo 'Running SonarQube analysis...'
                withSonarQubeEnv('SonarQube') {
                    sh 'mvn sonar:sonar'
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                echo 'Checking SonarQube Quality Gate...'
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        
        stage('Dependency Check') {
            steps {
                echo 'Running OWASP Dependency Check...'
                dependencyCheck additionalArguments: '''
                    -o "./dependency-check-report"
                    -s "."
                    -f "ALL"
                    --prettyPrint
                ''', odcInstallation: 'OWASP-Dependency-Check'
                
                dependencyCheckPublisher pattern: 'dependency-check-report/dependency-check-report.xml'
            }
        }
        
        stage('Package') {
            steps {
                echo 'Packaging the application...'
                sh 'mvn package -DskipTests'
            }
        }
        
        stage('Deploy to Test Environment') {
            steps {
                echo 'Deploying to test environment...'
                // Simulate deployment
                sh 'echo "Application deployed to test environment"'
            }
        }
        
        stage('Dynamic Security Testing') {
            steps {
                echo 'Running OWASP ZAP security scan...'
                script {
                    // Run ZAP scan against deployed application
                    sh '''
                        mkdir -p zap-reports
                        docker run -v $(pwd)/zap-reports:/zap/wrk/:rw \
                            -t owasp/zap2docker-stable \
                            zap-baseline.py \
                            -t http://example.com \
                            -J zap-report.json \
                            -H zap-report.html \
                            -r zap-report.md || true
                    '''
                }
            }
            post {
                always {
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'zap-reports',
                        reportFiles: 'zap-report.html',
                        reportName: 'ZAP Security Report'
                    ])
                }
            }
        }
    }
    
    post {
        always {
            echo 'Pipeline completed!'
            cleanWs()
        }
        success {
            echo 'Pipeline succeeded!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
EOF

# Commit the Jenkinsfile
git add Jenkinsfile
git commit -m "Add DevSecOps pipeline configuration"
Subtask 7.2: Setting Up the Jenkins Job
In Jenkins, click New Item
Enter job name: DevSecOps-Pipeline
Select Pipeline and click OK
In the configuration:
Pipeline Definition: Pipeline script from SCM
SCM: Git
Repository URL: /home/ubuntu/sample-project (local path)
Branch: */master
Script Path: Jenkinsfile
Click Save
Task 8: Running Security Scans and Analysis
Subtask 8.1: Manual OWASP Dependency Check
# Run dependency check manually on the sample project
cd ~/sample-project

~/owasp-tools/dependency-check/bin/dependency-check.sh \
    --project "DevSecOps Demo" \
    --scan . \
    --format ALL \
    --out ./dependency-check-report

# View the HTML report
echo "Dependency check report generated at: $(pwd)/dependency-check-report/dependency-check-report.html"
Subtask 8.2: Manual SonarQube Analysis
# Run SonarQube analysis manually
cd ~/sample-project

mvn sonar:sonar \
    -Dsonar.projectKey=devsecops-demo \
    -Dsonar.host.url=http://localhost:9000 \
    -Dsonar.login=your-sonar-token
Subtask 8.3: Manual ZAP Security Scan
# Create a simple web server for testing
cd ~/sample-project

# Start a simple HTTP server
python3 -m http.server 8000 &
SERVER_PID=$!

# Wait for server to start
sleep 2

# Run ZAP scan
mkdir -p zap-reports
docker run -v $(pwd)/zap-reports:/zap/wrk/:rw \
    -t owasp/zap2docker-stable \
    zap-baseline.py \
    -t http://host.docker.internal:8000 \
    -J zap-report.json \
    -H zap-report.html \
    -r zap-report.md

# Stop the test server
kill $SERVER_PID

echo "ZAP scan completed. Report available at: $(pwd)/zap-reports/zap-report.html"
Task 9: Analyzing Security Reports and Remediation
Subtask 9.1: Understanding SonarQube Security Reports
Access SonarQube at http://your-server-ip:9000
Navigate to your project: devsecops-demo
Review the Security tab to see:
Security Hotspots: Potential security vulnerabilities
Vulnerabilities: Confirmed security issues
Security Rating: Overall security score
Common Issues Found:

Hard-coded credentials
SQL injection vulnerabilities
Potential null pointer exceptions
Subtask 9.2: Understanding OWASP Dependency Check Reports
Open the dependency check HTML report and review:

Summary: Overview of vulnerabilities found
Dependencies: List of all dependencies analyzed
Vulnerabilities: Detailed information about each vulnerability including:
CVE numbers
CVSS scores
Severity levels
Remediation suggestions
Subtask 9.3: Remediation Examples
Create a secure version of the vulnerable code:

# Create a secure version of the Java class
cat > src/main/java/com/example/SecureApp.java << 'EOF'
package com.example;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.util.Properties;
import java.util.Scanner;

public class SecureApp {
    
    // Use configuration file instead of hard-coded credentials
    private Properties config;
    
    public SecureApp() {
        loadConfiguration();
    }
    
    private void loadConfiguration() {
        config = new Properties();
        // In real application, load from secure configuration
        config.setProperty("db.url", "jdbc:mysql://localhost:3306/testdb");
        config.setProperty("db.username", System.getenv("DB_USERNAME"));
        config.setProperty("db.password", System.getenv("DB_PASSWORD"));
    }
    
    public static void main(String[] args) {
        SecureApp app = new SecureApp();
        app.demonstrateSecurePractices();
    }
    
    public void demonstrateSecurePractices() {
        Scanner scanner = new Scanner(System.in);
        System.out.print("Enter user ID: ");
        String userId = scanner.nextLine();
        
        try {
            Connection conn = DriverManager.getConnection(
                config.getProperty("db.url"),
                config.getProperty("db.username"),
                config.getProperty("db.password")
            );
            
            // Use prepared statement to prevent SQL injection
            String query = "SELECT * FROM users WHERE id = ?";
            PreparedStatement pstmt = conn.prepareStatement(query);
            pstmt.setString(1, userId);
            
            ResultSet rs = pstmt.executeQuery();
            
            while (rs.next()) {
                System.out.println("User: " + rs.getString("username"));
            }
            
            conn.close();
        } catch (Exception e) {
            // Log error securely without exposing sensitive information
            System.err.println("Database error occurred");
        }
        
        scanner.close();
    }
    
    // Method with proper null checking
    public String processUserInput(String input) {
        if (input == null) {
            throw new IllegalArgumentException("Input cannot be null");
        }
        return input.toUpperCase();
    }
}
EOF

# Update pom.xml to use secure dependency versions
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
    
    <properties>
        <maven.compiler.source>11</maven.compiler.source>
        <maven.compiler.target>11</maven.compiler.target>
        <sonar.projectKey>devsecops-demo</sonar.projectKey>
        <sonar.projectName>DevSecOps Demo</sonar.projectName>
    </properties>
    
    <dependencies>
        <!-- Updated to secure versions -->
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-collections4</artifactId>
            <version>4.4</version>
        </dependency>
        
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>2.15.2</version>
        </dependency>
        
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.13.2</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.sonarsource.scanner.maven</groupId>
                <artifactId>sonar-maven-plugin</artifactId>
                <version>3.9.1.2184</version>
            </plugin>
        </plugins>
    </build>
</project>
EOF

# Commit the secure version
git add .
git commit -m "Fix security vulnerabilities and update dependencies"
Task 10: Implementing Security Gates and Policies
Subtask 10.1: Configuring SonarQube Quality Gates
In SonarQube, go to Quality Gates
Create a new quality gate: DevSecOps-Security-Gate
Add conditions:
Security Rating: is worse than A
Reliability Rating: is worse than A
Coverage: is less than 80%
Duplicated Lines (%): is greater than 3%
Subtask 10.2: Creating Security Policy Scripts
# Create a security policy checker script
cat > ~/owasp-tools/security-gate.sh << 'EOF'
#!/bin/bash

# Security Gate Script
# This script checks if the build meets security requirements

DEPENDENCY_CHECK_REPORT="dependency-check-report/dependency-check-report.json"
ZAP_REPORT="zap-reports/zap-report.json"

echo "=== DevSecOps Security Gate ==="

# Check if dependency check found high/critical vulnerabilities
if [ -f "$DEPENDENCY_CHECK_REPORT" ]; then
    HIGH_VULNS=$(jq '.dependencies[].vulnerabilities[]? | select(.severity == "HIGH" or .severity == "CRITICAL")' "$DEPENDENCY_CHECK_REPORT" | wc -l)
    
    if [ "$HIGH_VULNS" -gt 0 ]; then
        echo "❌ FAIL: Found $HIGH_VULNS high/critical vulnerabilities in dependencies"
        exit 1
    else
        echo "✅ PASS: No high/critical vulnerabilities found in dependencies"
    fi
else
    echo "⚠️  WARNING: Dependency check report not found"
fi

# Check ZAP scan results
if [ -f "$ZAP_REPORT" ]; then
    HIGH_ALERTS=$(jq '.site[].alerts[]? | select(.riskdesc | contains("High"))' "$ZAP_REPORT" | wc -l)
    
    if [ "$HIGH_ALERTS" -gt 0 ]; then
        echo "❌ FAIL: Found $HIGH_ALERTS high-risk security alerts"
        exit 1
    else
        echo "✅ PASS: No high-risk security alerts found"
    fi
else
    echo "⚠️  WARNING: ZAP scan report not found"
fi

echo "=== Security Gate: PASSED ==="
exit 0
EOF

chmod +x ~/owasp-tools/security-gate.sh
Task 11: Monitoring and Continuous Improvement
Subtask 11.1: Setting Up Security Metrics Dashboard
Create a simple script to collect security metrics:

# Create metrics collection script
cat > ~/owasp-tools/collect-metrics.sh << 'EOF'
#!/bin/bash

# Security Metrics Collection Script
METRICS_FILE="security-metrics.json"
DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

echo "Collecting security metrics..."

# Initialize metrics JSON
cat > "$METRICS_FILE" << EOL
{
  "timestamp": "$DATE",
  "sonarqube": {},
  "dependency_check": {},
  "zap": {}
}
EOL

# Collect SonarQube metrics (if available)
if command -v curl &> /dev/null; then
    SONAR_METRICS=$(curl -s "http://localhost:9000/api/measures/component?component=devsecops-demo&metricKeys=security_rating,reliability_rating,sqale_rating,coverage" || echo "{}")
    echo "$SONAR_METRICS" | jq '.component.measures' > temp_sonar.json 2>/dev/null || echo "[]" > temp_sonar.json
    jq --argjson sonar "$(cat temp_sonar.json)" '.sonarqube = $sonar' "$METRICS_FILE" > temp_metrics.json && mv temp_metrics.json "$METRICS_FILE"
    rm -f temp_sonar.json
fi

echo "Security metrics collected in $METRICS_FILE"
EOF

chmod +x ~/owasp-tools/collect-metrics.sh
Subtask 11.2: Creating Security Incident Response Plan
# Create incident response template
cat > ~/security-incident-response.md << 'EOF'
# Security Incident Response Plan

## Immediate Actions (0-1 hour)
1. **Identify and Isolate**
   - Determine the scope of the security issue
   - Isolate affected systems if necessary
   - Document initial findings

2. **Assess Impact**
   - Evaluate potential data exposure
   - Determine business impact
   - Identify affected users/systems

3. **Notify Stakeholders**
   - Security team
   - Development team
   - Management (if high severity)

## Short-term Actions (1-24 hours)
1. **Contain the Issue**
   - Apply temporary fixes
   - Block malicious traffic
   - Revoke compromised credentials

2. **Investigate**
   - Analyze logs and security reports
   - Determine root cause
   - Document timeline of events

3. **Communicate**
   - Update stakeholders
   - Prepare customer communication (if needed)

## Long-term Actions (1-7 days)
1. **Remediate**
   - Implement permanent fixes
   - Update security policies
   - Enhance monitoring

2. **Review and Improve**
   - Conduct post-incident review
   - Update security procedures
   - Provide additional training

## Prevention Measures
- Regular security assessments
- Automated security testing
- Security awareness training
- Incident response drills
EOF
Task 12: Running the Complete DevSecOps Pipeline
Subtask 12.1: Execute the Jenkins Pipeline
Go to Jenkins dashboard
Click on your DevSecOps-Pipeline job
Click Build Now
Monitor the pipeline execution in the Console Output
Subtask 12.2: Review Pipeline Results
After the pipeline completes, review:

Build Status: Check if all stages passed
SonarQube Report: Review code quality and security issues
Dependency Check Report: Check for vulnerable dependencies
ZAP Security Report: Review dynamic security testing results
Subtask 12.3: Troubleshooting Common Issues
Issue 1: SonarQube Connection Failed

# Check SonarQube status
docker-compose -f ~/sonarqube/docker-compose.yml ps

# Restart SonarQube if needed
docker-compose -f ~/sonarqube/docker-compose.yml restart
**Issue

