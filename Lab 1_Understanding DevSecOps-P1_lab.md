Lab 1. Understanding DevSecOps-P1
Objectives
By the end of this lab, students will be able to:

• Understand the fundamental concepts of DevSecOps and its importance in modern software development • Install and configure Jenkins as a CI/CD automation server • Set up SonarQube for Static Application Security Testing (SAST) • Deploy and configure OWASP ZAP for Dynamic Application Security Testing (DAST) • Create a basic CI/CD pipeline that integrates security testing tools • Demonstrate how security can be embedded into the development lifecycle • Identify security vulnerabilities using automated scanning tools

Prerequisites
Before starting this lab, students should have:

• Basic understanding of Linux command line operations • Familiarity with software development concepts • Basic knowledge of web applications and HTTP protocols • Understanding of what CI/CD means in software development • No prior experience with Jenkins, SonarQube, or ZAP is required

Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines for this lab. Simply click Start Lab to access your dedicated environment. No need to build your own virtual machine or install any software locally.

Your cloud machine includes: • Ubuntu 20.04 LTS with 4GB RAM and 20GB storage • Docker and Docker Compose pre-installed • All necessary network ports configured • Internet access for downloading required tools

Task 1: Setting Up Jenkins as CI/CD Tool
Subtask 1.1: Install Jenkins
First, we'll install Jenkins using Docker to ensure a clean and consistent setup.

Connect to your cloud machine and open a terminal

Create a directory for our DevSecOps lab:

mkdir ~/devsecops-lab
cd ~/devsecops-lab
Create a Docker Compose file for Jenkins:
nano docker-compose-jenkins.yml
Add the following content to the file:
version: '3.8'
services:
  jenkins:
    image: jenkins/jenkins:lts
    container_name: jenkins-server
    restart: unless-stopped
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - JAVA_OPTS=-Djenkins.install.runSetupWizard=false
    user: root

volumes:
  jenkins_home:
Start Jenkins container:
docker-compose -f docker-compose-jenkins.yml up -d
Verify Jenkins is running:
docker ps
Subtask 1.2: Configure Jenkins Initial Setup
Wait for Jenkins to start (approximately 2-3 minutes), then access Jenkins web interface:
http://your-cloud-machine-ip:8080
Get the initial admin password:
docker exec jenkins-server cat /var/jenkins_home/secrets/initialAdminPassword
Complete Jenkins setup: • Enter the admin password when prompted • Select Install suggested plugins • Create your first admin user with these details:

Username: admin
Password: admin123
Full name: DevSecOps Admin
Email: admin@devsecops.local • Keep the default Jenkins URL
Install additional required plugins: • Go to Manage Jenkins → Manage Plugins • Click on Available tab • Search and install these plugins:

SonarQube Scanner
OWASP ZAP Official Jenkins Plugin
Pipeline
Git • Restart Jenkins when prompted
Task 2: Setting Up SonarQube as SAST Tool
Subtask 2.1: Install SonarQube
Create a Docker Compose file for SonarQube:
nano docker-compose-sonarqube.yml
Add the following content:
version: '3.8'
services:
  sonarqube:
    image: sonarqube:community
    container_name: sonarqube-server
    restart: unless-stopped
    ports:
      - "9000:9000"
    environment:
      - SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true
    volumes:
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_logs:/opt/sonarqube/logs
      - sonarqube_extensions:/opt/sonarqube/extensions

volumes:
  sonarqube_data:
  sonarqube_logs:
  sonarqube_extensions:
Start SonarQube container:
docker-compose -f docker-compose-sonarqube.yml up -d
Verify SonarQube is running:
docker ps
docker logs sonarqube-server
Subtask 2.2: Configure SonarQube
Access SonarQube web interface (wait 3-4 minutes for startup):
http://your-cloud-machine-ip:9000
Login with default credentials: • Username: admin • Password: admin

Change the default password: • You'll be prompted to change the password • Set new password: admin123

Create a new project: • Click Create Project → Manually • Project key: devsecops-demo • Display name: DevSecOps Demo Project • Click Set Up

Generate authentication token: • Select Generate a token • Token name: jenkins-integration • Click Generate • Important: Copy and save this token for later use

Subtask 2.3: Integrate SonarQube with Jenkins
Configure SonarQube server in Jenkins: • Go to Manage Jenkins → Configure System • Scroll to SonarQube servers section • Click Add SonarQube • Name: SonarQube-Server • Server URL: http://your-cloud-machine-ip:9000 • Server authentication token: Click Add → Jenkins
Kind: Secret text
Secret: (paste the token you generated)
ID: sonarqube-token
Description: SonarQube Authentication Token • Select the credential you just created • Click Save
Task 3: Setting Up OWASP ZAP as DAST Tool
Subtask 3.1: Install OWASP ZAP
Create a Docker Compose file for ZAP:
nano docker-compose-zap.yml
Add the following content:
version: '3.8'
services:
  zap:
    image: owasp/zap2docker-stable
    container_name: zap-server
    restart: unless-stopped
    ports:
      - "8090:8080"
      - "8091:8090"
    command: zap-webswing.sh
    volumes:
      - zap_data:/zap/wrk

volumes:
  zap_data:
Start ZAP container:
docker-compose -f docker-compose-zap.yml up -d
Verify ZAP is running:
docker ps
Subtask 3.2: Configure OWASP ZAP
Access ZAP web interface (wait 2-3 minutes for startup):
http://your-cloud-machine-ip:8090
Configure ZAP for API access: • The web interface should load showing ZAP desktop • ZAP is now ready to perform security scans
Subtask 3.3: Create a Test Web Application
To demonstrate DAST scanning, we'll deploy a vulnerable web application:

Create a Docker Compose file for a test application:
nano docker-compose-webapp.yml
Add the following content:
version: '3.8'
services:
  dvwa:
    image: vulnerables/web-dvwa
    container_name: test-webapp
    restart: unless-stopped
    ports:
      - "8081:80"
    environment:
      - MYSQL_HOSTNAME=db
      - MYSQL_DATABASE=dvwa
      - MYSQL_USERNAME=dvwa
      - MYSQL_PASSWORD=p@ssw0rd
    depends_on:
      - db

  db:
    image: mysql:5.7
    container_name: test-db
    restart: unless-stopped
    environment:
      - MYSQL_ROOT_PASSWORD=p@ssw0rd
      - MYSQL_DATABASE=dvwa
      - MYSQL_USER=dvwa
      - MYSQL_PASSWORD=p@ssw0rd
    volumes:
      - db_data:/var/lib/mysql

volumes:
  db_data:
Start the test application:
docker-compose -f docker-compose-webapp.yml up -d
Verify the application is running:
docker ps
Access the test application:
http://your-cloud-machine-ip:8081
Task 4: Creating an Integrated DevSecOps Pipeline
Subtask 4.1: Create a Sample Application for Testing
Create a simple web application directory:
mkdir ~/devsecops-lab/sample-app
cd ~/devsecops-lab/sample-app
Create a simple HTML file:
nano index.html
Add the following content:
<!DOCTYPE html>
<html>
<head>
    <title>DevSecOps Demo App</title>
</head>
<body>
    <h1>Welcome to DevSecOps Demo</h1>
    <p>This is a sample application for security testing.</p>
    <form action="/login" method="post">
        <input type="text" name="username" placeholder="Username">
        <input type="password" name="password" placeholder="Password">
        <input type="submit" value="Login">
    </form>
</body>
</html>
Create a simple JavaScript file with potential security issues:
nano app.js
Add the following content:
// Sample application with security issues for demonstration
const express = require('express');
const app = express();

// Security issue: Missing input validation
app.post('/login', (req, res) => {
    const username = req.body.username;
    const password = req.body.password;
    
    // Security issue: SQL injection vulnerability
    const query = "SELECT * FROM users WHERE username='" + username + "' AND password='" + password + "'";
    
    // Security issue: Hardcoded credentials
    if (username === 'admin' && password === 'password123') {
        res.send('Login successful');
    } else {
        res.send('Login failed');
    }
});

app.listen(3000, () => {
    console.log('App running on port 3000');
});
Create a package.json file:
nano package.json
Add the following content:
{
  "name": "devsecops-demo-app",
  "version": "1.0.0",
  "description": "Demo application for DevSecOps pipeline",
  "main": "app.js",
  "scripts": {
    "start": "node app.js",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "dependencies": {
    "express": "^4.18.0"
  }
}
Create a SonarQube configuration file:
nano sonar-project.properties
Add the following content:
sonar.projectKey=devsecops-demo
sonar.projectName=DevSecOps Demo Project
sonar.projectVersion=1.0
sonar.sources=.
sonar.language=js
sonar.sourceEncoding=UTF-8
Subtask 4.2: Create Jenkins Pipeline
In Jenkins, create a new pipeline job: • Go to Jenkins dashboard • Click New Item • Enter name: DevSecOps-Pipeline • Select Pipeline • Click OK

Configure the pipeline: • Scroll to Pipeline section • Select Pipeline script from Definition dropdown • Add the following pipeline script:

pipeline {
    agent any
    
    environment {
        SONAR_TOKEN = credentials('sonarqube-token')
        ZAP_PORT = '8090'
        TARGET_URL = 'http://your-cloud-machine-ip:8081'
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                // In a real scenario, this would checkout from Git
                sh 'echo "Source code checked out"'
            }
        }
        
        stage('Build') {
            steps {
                echo 'Building application...'
                sh 'echo "Application built successfully"'
            }
        }
        
        stage('SAST - SonarQube Analysis') {
            steps {
                echo 'Running Static Application Security Testing...'
                script {
                    def scannerHome = tool 'SonarScanner'
                    withSonarQubeEnv('SonarQube-Server') {
                        sh """
                            cd /home/ubuntu/devsecops-lab/sample-app
                            ${scannerHome}/bin/sonar-scanner \
                                -Dsonar.projectKey=devsecops-demo \
                                -Dsonar.sources=. \
                                -Dsonar.host.url=http://your-cloud-machine-ip:9000 \
                                -Dsonar.login=${SONAR_TOKEN}
                        """
                    }
                }
            }
        }
        
        stage('Deploy to Test') {
            steps {
                echo 'Deploying to test environment...'
                sh 'echo "Application deployed to test environment"'
            }
        }
        
        stage('DAST - ZAP Security Scan') {
            steps {
                echo 'Running Dynamic Application Security Testing...'
                script {
                    sh """
                        docker run --rm -v \$(pwd):/zap/wrk/:rw \
                        -t owasp/zap2docker-weekly zap-baseline.py \
                        -t ${TARGET_URL} -J zap-report.json || true
                    """
                }
            }
        }
        
        stage('Security Report') {
            steps {
                echo 'Generating security reports...'
                sh 'echo "Security scan completed. Check SonarQube dashboard for SAST results."'
                sh 'echo "ZAP scan completed. Check workspace for DAST results."'
            }
        }
    }
    
    post {
        always {
            echo 'Pipeline completed!'
            // Archive security reports
            archiveArtifacts artifacts: '*.json', allowEmptyArchive: true
        }
        success {
            echo 'Pipeline succeeded!'
        }
        failure {
            echo 'Pipeline failed! Check security issues.'
        }
    }
}
Note: Replace your-cloud-machine-ip with your actual cloud machine IP address in the pipeline script.

Install SonarQube Scanner tool: • Go to Manage Jenkins → Global Tool Configuration • Scroll to SonarQube Scanner • Click Add SonarQube Scanner • Name: SonarScanner • Check Install automatically • Select latest version • Click Save
Subtask 4.3: Run the DevSecOps Pipeline
Execute the pipeline: • Go to your DevSecOps-Pipeline job • Click Build Now • Monitor the build progress in Console Output

Review SAST results in SonarQube: • Go to SonarQube dashboard: http://your-cloud-machine-ip:9000 • Click on your project devsecops-demo • Review security vulnerabilities, code smells, and bugs identified

Review DAST results: • Check Jenkins workspace for ZAP report files • Review security vulnerabilities found in the web application

Task 5: Understanding the Results
Subtask 5.1: Analyze SAST Results
In SonarQube dashboard, examine: • Security Hotspots: Potential security vulnerabilities • Bugs: Code issues that could lead to security problems • Code Smells: Maintainability issues that could indirectly affect security • Coverage: How much of your code is tested

Common SAST findings you might see: • Hardcoded credentials • SQL injection vulnerabilities • Cross-site scripting (XSS) potential • Input validation issues

Subtask 5.2: Analyze DAST Results
Review ZAP scan results: • Look for common web application vulnerabilities • Check for OWASP Top 10 security risks • Review risk levels (High, Medium, Low, Informational)

Common DAST findings include: • Missing security headers • Insecure HTTP methods • Directory traversal vulnerabilities • Authentication bypass issues

Troubleshooting Common Issues
Issue 1: Jenkins Container Won't Start
Solution:

# Check if port 8080 is already in use
sudo netstat -tlnp | grep 8080
# If in use, stop the conflicting service or change Jenkins port
Issue 2: SonarQube Takes Too Long to Start
Solution:

# Check SonarQube logs
docker logs sonarqube-server
# Increase memory if needed
docker-compose down
# Edit docker-compose file to add memory limits
Issue 3: ZAP Scan Fails
Solution:

# Ensure target application is accessible
curl http://your-cloud-machine-ip:8081
# Check ZAP container logs
docker logs zap-server
Issue 4: Pipeline Fails at SonarQube Stage
Solution:

# Verify SonarQube token is correctly configured
# Check SonarQube server connectivity from Jenkins
# Ensure SonarQube Scanner tool is properly installed
Verification Steps
To verify your lab setup is working correctly:

Check all containers are running:
docker ps
You should see containers for Jenkins, SonarQube, ZAP, and the test web application.

Verify web interfaces are accessible: • Jenkins: http://your-cloud-machine-ip:8080 • SonarQube: http://your-cloud-machine-ip:9000 • ZAP: http://your-cloud-machine-ip:8090 • Test App: http://your-cloud-machine-ip:8081

Run a test pipeline build and ensure it completes successfully

Check that security reports are generated in both SonarQube and Jenkins workspace

Conclusion
Congratulations! You have successfully completed Lab 1 of Understanding DevSecOps. In this lab, you have accomplished the following:

Key Achievements: • Implemented a complete DevSecOps toolchain using three essential open-source tools • Set up Jenkins as your CI/CD automation server to orchestrate the entire pipeline • Configured SonarQube for Static Application Security Testing (SAST) to identify security vulnerabilities in source code • Deployed OWASP ZAP for Dynamic Application Security Testing (DAST) to find runtime security issues • Created an integrated pipeline that automatically runs security tests as part of the development process • Learned to interpret security scan results from both static and dynamic analysis tools

Why This Matters: This lab demonstrates the core principle of DevSecOps: shifting security left in the development lifecycle. Instead of treating security as an afterthought, you've learned to integrate security testing directly into your CI/CD pipeline. This approach helps teams:

• Identify security issues early when they're cheaper and easier to fix • Automate security testing to ensure consistent coverage • Reduce security debt by catching vulnerabilities before they reach production • Build security awareness among development teams through immediate feedback

Real-World Impact: The skills you've developed in this lab are directly applicable to enterprise environments where security is paramount. Organizations worldwide are adopting DevSecOps practices to build more secure software faster, and the tools you've learned to use are industry standards.

Next Steps: In future labs, you'll expand on this foundation by exploring advanced DevSecOps concepts such as: • Container security scanning • Infrastructure as Code (IaC) security • Compliance automation • Advanced threat modeling • Security metrics and reporting

You now have a solid foundation in DevSecOps tooling and can confidently explain how security integration improves software development practices. Keep practicing with different types of applications and security scenarios to deepen your expertise!
