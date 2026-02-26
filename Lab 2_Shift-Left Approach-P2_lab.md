Lab 2: Shift-Left Approach Implementation with CI/CD Pipeline
Lab Objectives
By the end of this lab, students will be able to:

• Understand and implement the Shift-Left security approach in DevOps pipelines • Design and visualize a complete CI/CD pipeline with integrated security tools • Configure Jenkins as a CI/CD server for automated builds and deployments • Integrate SonarQube for Static Application Security Testing (SAST) • Implement ZAP (OWASP Zed Attack Proxy) for Dynamic Application Security Testing (DAST) • Demonstrate automated security scanning throughout the development lifecycle • Analyze security scan results and understand their impact on code quality

Prerequisites
Before starting this lab, students should have:

• Basic understanding of DevOps concepts and CI/CD pipelines • Familiarity with Linux command line operations • Basic knowledge of web application security concepts • Understanding of version control systems (Git) • Basic knowledge of containerization concepts

Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines for this lab. Simply click Start Lab to access your environment. No need to build your own VM or install additional software - everything is ready to use!

Your lab environment includes: • Ubuntu 20.04 LTS with Docker pre-installed • Jenkins server ready for configuration • SonarQube server ready for setup • OWASP ZAP proxy tool • Sample web application for testing • Draw.io access for pipeline visualization

Lab Tasks
Task 1: Pipeline Visualization and Shift-Left Approach Design
Subtask 1.1: Understanding Shift-Left Security
The Shift-Left approach means integrating security testing early in the development process rather than waiting until the end. This approach helps identify and fix security issues when they are less expensive and easier to resolve.

Key benefits of Shift-Left security: • Early Detection: Find vulnerabilities during development • Cost Reduction: Fix issues before they reach production • Faster Delivery: Reduce delays caused by late-stage security findings • Better Quality: Improve overall code quality and security posture

Subtask 1.2: Creating Pipeline Visualization with Draw.io
Access Draw.io

# Open your web browser and navigate to:
https://app.diagrams.net/
Create New Diagram

Click "Create New Diagram"
Select "Blank Diagram"
Name it "Shift-Left-Security-Pipeline"
Design the Complete Pipeline

Create the following pipeline stages from left to right:

Stage 1: Development Phase (Shift-Left Security)

Developer commits code to Git repository
Pre-commit hooks run basic security checks
Code review with security focus
Stage 2: Build Phase

Jenkins triggers automated build
Unit tests execution
Static code analysis with SonarQube (SAST)
Stage 3: Testing Phase

Integration tests
Security unit tests
Container security scanning
Stage 4: Deployment Phase

Deploy to staging environment
Dynamic security testing with OWASP ZAP (DAST)
Performance and load testing
Stage 5: Production Phase

Production deployment
Runtime security monitoring
Continuous security assessment
Add Security Tools Integration Points

Mark where SonarQube integrates (Build phase)
Mark where OWASP ZAP integrates (Testing phase)
Show feedback loops back to developers
Save Your Diagram

File → Export as → PNG
Save as "shift-left-pipeline.png"
Task 2: Tools Integration and Code Scanning Implementation
Subtask 2.1: Jenkins CI/CD Server Setup
Start Jenkins Service

# Check if Jenkins is running
sudo systemctl status jenkins

# If not running, start Jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins
Access Jenkins Web Interface

# Get the initial admin password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
Open browser and navigate to: http://localhost:8080

Complete Jenkins Initial Setup

Enter the initial admin password
Install suggested plugins
Create admin user:
Username: admin
Password: admin123
Full name: Lab Administrator
Email: admin@lab.local
Install Required Plugins

Go to "Manage Jenkins" → "Manage Plugins"
Install the following plugins:
SonarQube Scanner
OWASP ZAP Official Jenkins Plugin
Git Plugin
Pipeline Plugin
Docker Pipeline Plugin
Subtask 2.2: SonarQube SAST Tool Integration
Start SonarQube Server

# Navigate to SonarQube directory
cd /opt/sonarqube/bin/linux-x86-64/

# Start SonarQube
sudo ./sonar.sh start

# Check status
sudo ./sonar.sh status
Access SonarQube Web Interface

Open browser: http://localhost:9000
Default credentials:
Username: admin
Password: admin
Change password when prompted to: admin123
Create SonarQube Project

Click "Create Project" → "Manually"
Project key: secure-web-app
Display name: Secure Web Application
Click "Set Up"
Generate SonarQube Token

Go to "My Account" → "Security"
Generate token named: jenkins-integration
Copy and save the token: squ_xxxxxxxxxxxxxxxxxxxxxxxxx
Configure SonarQube in Jenkins

In Jenkins: "Manage Jenkins" → "Configure System"
Find "SonarQube servers" section
Add SonarQube server:
Name: SonarQube-Local
Server URL: http://localhost:9000
Server authentication token: (paste the token generated above)
Subtask 2.3: OWASP ZAP DAST Tool Integration
Start OWASP ZAP

# Start ZAP in daemon mode
/opt/zaproxy/zap.sh -daemon -host 0.0.0.0 -port 8090 -config api.addrs.addr.name=.* -config api.addrs.addr.regex=true
Verify ZAP is Running

# Check if ZAP is listening
netstat -tlnp | grep 8090

# Test ZAP API
curl http://localhost:8090/JSON/core/view/version/
Configure ZAP in Jenkins

In Jenkins: "Manage Jenkins" → "Configure System"
Find "ZAP" section
Add ZAP installation:
Name: ZAP-Local
Host: localhost
Port: 8090
Subtask 2.4: Sample Application Setup
Create Sample Web Application

# Create project directory
mkdir -p /home/ubuntu/secure-web-app
cd /home/ubuntu/secure-web-app

# Create a simple vulnerable web application
cat > app.py << 'EOF'
from flask import Flask, request, render_template_string
import sqlite3
import os

app = Flask(__name__)

# Vulnerable SQL query (for demonstration)
@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        
        # Vulnerable SQL injection point
        query = f"SELECT * FROM users WHERE username='{username}' AND password='{password}'"
        # This is intentionally vulnerable for testing purposes
        
        return f"Query executed: {query}"
    
    return '''
    <form method="post">
        Username: <input type="text" name="username"><br>
        Password: <input type="password" name="password"><br>
        <input type="submit" value="Login">
    </form>
    '''

@app.route('/')
def home():
    return '<h1>Secure Web App Demo</h1><a href="/login">Login</a>'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
EOF
Create Requirements File

cat > requirements.txt << 'EOF'
Flask==2.3.3
Werkzeug==2.3.7
EOF
Create Dockerfile

cat > Dockerfile << 'EOF'
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY app.py .

EXPOSE 5000

CMD ["python", "app.py"]
EOF
Initialize Git Repository

git init
git add .
git commit -m "Initial commit - Sample web application"
Subtask 2.5: Create Jenkins Pipeline with Security Integration
Create Jenkins Pipeline Job

In Jenkins dashboard, click "New Item"
Enter name: Secure-Web-App-Pipeline
Select "Pipeline" and click "OK"
Configure Pipeline Script In the Pipeline section, add the following script:

pipeline {
    agent any
    
    environment {
        SONAR_PROJECT_KEY = 'secure-web-app'
        ZAP_PORT = '8090'
        APP_PORT = '5000'
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out code from repository...'
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                echo 'Building the application...'
                sh '''
                    cd /home/ubuntu/secure-web-app
                    docker build -t secure-web-app:latest .
                '''
            }
        }
        
        stage('SAST - SonarQube Analysis') {
            steps {
                echo 'Running Static Application Security Testing...'
                script {
                    def scannerHome = tool 'SonarQubeScanner'
                    withSonarQubeEnv('SonarQube-Local') {
                        sh """
                            cd /home/ubuntu/secure-web-app
                            ${scannerHome}/bin/sonar-scanner \
                                -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                                -Dsonar.sources=. \
                                -Dsonar.host.url=http://localhost:9000 \
                                -Dsonar.login=${SONAR_AUTH_TOKEN}
                        """
                    }
                }
            }
        }
        
        stage('Deploy to Staging') {
            steps {
                echo 'Deploying application to staging environment...'
                sh '''
                    # Stop any existing container
                    docker stop secure-web-app || true
                    docker rm secure-web-app || true
                    
                    # Run the application
                    docker run -d --name secure-web-app -p 5000:5000 secure-web-app:latest
                    
                    # Wait for application to start
                    sleep 10
                '''
            }
        }
        
        stage('DAST - ZAP Security Scan') {
            steps {
                echo 'Running Dynamic Application Security Testing...'
                script {
                    sh '''
                        # Basic ZAP scan
                        curl "http://localhost:8090/JSON/spider/action/scan/?url=http://localhost:5000"
                        
                        # Wait for spider to complete
                        sleep 30
                        
                        # Run active scan
                        curl "http://localhost:8090/JSON/ascan/action/scan/?url=http://localhost:5000"
                        
                        # Wait for scan to complete
                        sleep 60
                        
                        # Generate report
                        curl "http://localhost:8090/OTHER/core/other/htmlreport/" > zap-report.html
                    '''
                }
            }
        }
        
        stage('Security Report') {
            steps {
                echo 'Generating security reports...'
                sh '''
                    echo "=== SECURITY SCAN SUMMARY ==="
                    echo "SonarQube SAST scan completed"
                    echo "ZAP DAST scan completed"
                    echo "Reports available in Jenkins workspace"
                '''
            }
        }
    }
    
    post {
        always {
            echo 'Pipeline completed'
            sh 'docker stop secure-web-app || true'
        }
        success {
            echo 'All security scans passed successfully!'
        }
        failure {
            echo 'Security scans detected issues. Please review the reports.'
        }
    }
}
Configure SonarQube Scanner Tool

Go to "Manage Jenkins" → "Global Tool Configuration"
Find "SonarQube Scanner" section
Add SonarQube Scanner:
Name: SonarQubeScanner
Install automatically: Check
Version: Latest
Subtask 2.6: Execute Pipeline and Analyze Results
Run the Pipeline

Go to your pipeline job
Click "Build Now"
Monitor the build progress in "Console Output"
Analyze SonarQube Results

Navigate to SonarQube: http://localhost:9000
Click on your project "Secure Web Application"
Review the security findings:
Security Hotspots: Potential security issues
Vulnerabilities: Confirmed security problems
Code Smells: Maintainability issues
Coverage: Test coverage metrics
Analyze ZAP DAST Results

Check the ZAP report generated in Jenkins workspace
Review findings such as:
High Risk: Critical security vulnerabilities
Medium Risk: Important security issues
Low Risk: Minor security concerns
Informational: General security information
Understanding Scan Results

Common SonarQube Findings:

Security Hotspots Found:
- SQL Injection vulnerability in login function
- Hardcoded credentials
- Insecure random number generation
- Cross-site scripting (XSS) potential
Common ZAP Findings:

DAST Scan Results:
- Missing security headers
- Unencrypted communications
- Session management issues
- Input validation problems
Subtask 2.7: Demonstrate Shift-Left Benefits
Show Early Detection

# View SonarQube quality gate status
curl -u admin:admin123 "http://localhost:9000/api/qualitygates/project_status?projectKey=secure-web-app"
Create Security-Fixed Version

cd /home/ubuntu/secure-web-app

# Create improved version of the application
cat > app_secure.py << 'EOF'
from flask import Flask, request, render_template_string
import sqlite3
import hashlib
import secrets

app = Flask(__name__)
app.secret_key = secrets.token_hex(16)

# Secure SQL query using parameterized statements
@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        
        # Secure parameterized query
        conn = sqlite3.connect(':memory:')
        cursor = conn.cursor()
        cursor.execute("SELECT * FROM users WHERE username=? AND password=?", (username, password))
        result = cursor.fetchone()
        conn.close()
        
        if result:
            return "Login successful!"
        else:
            return "Invalid credentials"
    
    return '''
    <form method="post">
        Username: <input type="text" name="username" required><br>
        Password: <input type="password" name="password" required><br>
        <input type="submit" value="Login">
    </form>
    '''

@app.route('/')
def home():
    return '<h1>Secure Web App Demo</h1><a href="/login">Login</a>'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF
Run Pipeline Again with Secure Code

Replace app.py with app_secure.py
Commit changes and trigger pipeline
Compare security scan results
Troubleshooting Common Issues
Jenkins Issues
# If Jenkins fails to start
sudo systemctl restart jenkins
sudo journalctl -u jenkins -f

# If plugins fail to install
sudo chown -R jenkins:jenkins /var/lib/jenkins/
sudo systemctl restart jenkins
SonarQube Issues
# If SonarQube fails to start
sudo sysctl -w vm.max_map_count=262144
echo 'vm.max_map_count=262144' | sudo tee -a /etc/sysctl.conf

# Check SonarQube logs
tail -f /opt/sonarqube/logs/sonar.log
ZAP Issues
# If ZAP API is not accessible
sudo ufw allow 8090
netstat -tlnp | grep 8090

# Restart ZAP if needed
pkill -f zap
/opt/zaproxy/zap.sh -daemon -host 0.0.0.0 -port 8090 -config api.addrs.addr.name=.* -config api.addrs.addr.regex=true
Key Concepts Summary
Shift-Left Security Benefits
• Early Detection: Security issues found during development phase • Cost Efficiency: Cheaper to fix issues early in the lifecycle • Faster Delivery: Reduced delays from late-stage security findings • Quality Improvement: Better overall code quality and security posture

SAST vs DAST
• SAST (Static): Analyzes source code without executing it • DAST (Dynamic): Tests running applications for vulnerabilities • Complementary: Both approaches needed for comprehensive security

Tool Integration Points
• Jenkins: Orchestrates the entire CI/CD pipeline • SonarQube: Provides static code analysis and security scanning • OWASP ZAP: Performs dynamic security testing of web applications

Conclusion
In this lab, you have successfully:

• Designed and visualized a complete CI/CD pipeline implementing the Shift-Left security approach • Configured Jenkins as a central CI/CD server to orchestrate automated builds and deployments • Integrated SonarQube for Static Application Security Testing (SAST) to catch security issues in source code • Implemented OWASP ZAP for Dynamic Application Security Testing (DAST) to find runtime vulnerabilities • Demonstrated automated security scanning throughout the development lifecycle • Analyzed security scan results and understood their impact on application security

The Shift-Left approach you've implemented represents a fundamental shift in how organizations approach security. By integrating security testing early and continuously throughout the development process, teams can:

Identify vulnerabilities when they're easier and cheaper to fix
Maintain faster development cycles without compromising security
Build security awareness and expertise within development teams
Achieve better overall security posture for applications
This hands-on experience with Jenkins, SonarQube, and OWASP ZAP provides you with practical skills in implementing DevSecOps practices that are highly valued in modern software development environments. The pipeline you've created serves as a foundation that can be extended with additional security tools and practices as your security program matures.

Remember that security is not a one-time activity but an ongoing process that requires continuous monitoring, updating, and improvement of your security practices and tools.

