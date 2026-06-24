Complete CI/CD Pipeline Setup: Jenkins + Tomcat on AWS EC2
This guide documents the complete setup of a CI/CD pipeline using Jenkins and Apache Tomcat on an AWS EC2 instance, including the deployment of a Java web application.
________________________________________
Table of Contents
1.	Infrastructure Setup
2.	User Creation
3.	Tomcat Installation & Configuration
4.	Jenkins Installation & Configuration
5.	System Fixes
6.	Jenkins Plugin Installation
7.	Jenkins Tools Configuration
8.	Credentials Setup
9.	Pipeline Creation
10.	Troubleshooting & Fixes
11.	Successful Deployment
________________________________________
1. Infrastructure Setup
1.1 AWS EC2 Instance Creation
We started by creating an AWS EC2 instance with the following specifications:
Setting	Value
Instance Name	Student
AMI	Ubuntu Server 26.04 LTS
Instance Type	t3.micro (Free Tier Eligible)
Storage	20 GB gp3
Security Group	Custom with ports 22, 8080, 9090
EC2 Setup - Instance Configuration
EC2 Setup - Instance Configuration
1.2 Security Group Configuration
We configured the security group to allow inbound traffic on: - Port 22 (SSH) - for remote access - Port 8080 (Jenkins) - for Jenkins web interface - Port 9090 (Tomcat) - for Tomcat web server (changed from default 8080 to avoid conflict)
EC2 Setup - Security Group Rules EC2 Setup - Security Group Rules Continued
Note: Port 9090 was chosen for Tomcat because Jenkins uses the default port 8080.
________________________________________
2. User Creation
We created dedicated users for both Tomcat and Jenkins for security and isolation.
2.1 Creating the Tomcat User
# Create the tomcat user
sudo useradd tomcat

# Set password for tomcat user
sudo passwd tomcat

# Assign bash shell to tomcat user
sudo usermod -s /bin/bash tomcat
2.2 Creating the Jenkins User
# Create the jenkins user
sudo useradd jenkins

# Set password for jenkins user
sudo passwd jenkins

# Assign bash shell to jenkins user
sudo usermod -s /bin/bash jenkins
Why separate users? Running services under dedicated users follows the principle of least privilege and improves system security.
________________________________________
3. Tomcat Installation & Configuration
3.1 Downloading Tomcat
We downloaded Apache Tomcat 9.0.119 from the official website: - Source: https://tomcat.apache.org/download-90.cgi
Tomcat Download Page
Tomcat Download Page
# Download Tomcat using wget
wget https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.119/bin/apache-tomcat-9.0.119.tar.gz

# Extract to /opt directory
sudo tar -xzf apache-tomcat-9.0.119.tar.gz -C /opt/

# Rename for simplicity
sudo mv /opt/apache-tomcat-9.0.119 /opt/apache-tomcat-9.0.119
Why /opt? The /opt directory is used for optional/add-on software packages that are not part of the default system. Tomcat is a static application server, making /opt the appropriate location.
3.2 Changing Ownership
# Change ownership of Tomcat directory to tomcat user
sudo chown -R tomcat:tomcat /opt/apache-tomcat-9.0.119
3.3 Creating the systemd Service File
We created a systemd service file to manage Tomcat as a system service:
sudo vi /etc/systemd/system/tomcat.service
File contents:
[Unit]
Description=Tomcat 9.0.104 container
After=network.target

[Service]
Type=forking
User=tomcat
Group=tomcat
ExecStart=/opt/apache-tomcat-9.0.119/bin/startup.sh
ExecStop=/opt/apache-tomcat-9.0.119/bin/shutdown.sh
Restart=on-failure

[Install]
WantedBy=multi-user.target
Tomcat Service File
Tomcat Service File
3.4 Starting and Enabling Tomcat
# Reload systemd daemon
sudo systemctl daemon-reload

# Enable Tomcat to start on boot
sudo systemctl enable tomcat

# Start Tomcat service
sudo systemctl start tomcat

# Check Tomcat status
sudo systemctl status tomcat
Tomcat Service Started
Tomcat Service Started
3.5 Changing Tomcat Port (8080 → 9090)
Since Jenkins uses port 8080 by default, we changed Tomcat’s port to 9090:
sudo vi /opt/apache-tomcat-9.0.119/conf/server.xml
Change the Connector port:
<Connector port="9090" protocol="HTTP/1.1"
           connectionTimeout="20000"
           redirectPort="8443"
           maxParameterCount="1000" />
Server.xml Port Edit
Server.xml Port Edit
3.6 Configuring Tomcat Users
We configured tomcat-users.xml to enable manager access for deployment:
sudo vi /opt/apache-tomcat-9.0.119/conf/tomcat-users.xml
File contents:
<tomcat-users>
  <role rolename="manager-gui" />
  <role rolename="manager-status" />
  <role rolename="manager-script" />
  <role rolename="manager-jmx" />
  <role rolename="admin-gui" />
  <role rolename="admin-script" />

  <user username="tomcat" password="tomcat" 
        roles="manager-gui,admin-script,admin-gui,manager-status,manager-script,manager-jmx"/>
</tomcat-users>
Tomcat Users Configuration
Tomcat Users Configuration
3.7 Configuring Manager and Host-Manager Access
We edited the Manager application’s context file to allow remote access:
sudo vi /opt/apache-tomcat-9.0.119/webapps/manager/META-INF/context.xml
sudo vi /opt/apache-tomcat-9.0.119/webapps/host-manager/META-INF/context.xml
Commented out the RemoteAddrValve restriction:
<!--
  <Valve className="org.apache.catalina.valves.RemoteAddrValve"
         allow="127\.0\.0\.1|::1" />
-->
Manager Context Edit Host-Manager Context Edit
3.8 Restarting Tomcat
sudo systemctl restart tomcat
________________________________________
4. Jenkins Installation & Configuration
4.1 Downloading and Installing Jenkins
We followed the official Jenkins installation guide for Linux: - Source: https://www.jenkins.io/doc/book/installing/linux/
Jenkins Installation Guide
Jenkins Installation Guide
# Update package index
sudo apt update

# Install Java (required for Jenkins)
sudo apt install fontconfig openjdk-21-jre

# Verify Java installation
java -version

# Add Jenkins repository key
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
    https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key

# Add Jenkins repository
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
    https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
    /etc/apt/sources.list.d/jenkins.list > /dev/null

# Update and install Jenkins
sudo apt update
sudo apt install jenkins
Why /var? Jenkins stores variable data (builds, workspaces, logs) that changes frequently. The /var directory is designed for variable data files, making it the appropriate location.
4.2 Changing Jenkins Directory Ownership
# Change ownership of Jenkins home directory
sudo chown -R jenkins:jenkins /var/lib/jenkins
4.3 Starting Jenkins
# Start Jenkins service
sudo systemctl start jenkins

# Enable Jenkins to start on boot
sudo systemctl enable jenkins

# Check Jenkins status
sudo systemctl status jenkins
4.4 Initial Jenkins Setup
1.	Access Jenkins at http://<EC2-IP>:8080
2.	Unlock Jenkins using the initial admin password:
 	sudo cat /var/lib/jenkins/secrets/initialAdminPassword
3.	Install suggested plugins
Jenkins Getting Started
Jenkins Getting Started
________________________________________
5. System Fixes
5.1 Fixing /tmp Directory Size (Built-in Node Offline Issue)
Jenkins built-in node went offline due to insufficient temp space:
Built-in Node Offline
Built-in Node Offline
Diagnosis:
# Check disk space
df -h
Fix - Resize /tmp to 3GB:
# Remount /tmp with larger size
sudo mount -o remount,size=3G /tmp

# Verify the change
df -h
Resize Temp Directory
Resize Temp Directory
Note: The /tmp directory was mounted as tmpfs with limited size. Increasing it to 3GB resolved the issue.
5.2 Adding Swap Space (Performance Fix)
The system was running low on memory (only 908MB RAM). We added 2GB swap:
# Create 2GB swap file
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Make permanent
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
________________________________________
6. Jenkins Plugin Installation
We installed the following required plugins:
6.1 Maven Integration Plugin
Maven Integration Plugin
Maven Integration Plugin
6.2 Pipeline Maven Integration Plugin
Pipeline Maven Integration
Pipeline Maven Integration
6.3 Deploy to Container Plugin
Deploy to Container Plugin
Deploy to Container Plugin
Installation Path: Manage Jenkins → Plugins → Available Plugins → Search and Install
________________________________________
7. Jenkins Tools Configuration
7.1 Adding Maven as a Tool
We configured Maven under Jenkins tools for use in pipelines:
Path: Manage Jenkins → Tools → Maven Installations
Adding Maven as Tool
Adding Maven as Tool
Configuration: - Name: maven - Install automatically: Checked - Version: 3.9.16 - Installer: Install from Apache
________________________________________
8. Credentials Setup
8.1 Adding GitHub Credentials
For accessing the private GitHub repository:
Path: Manage Jenkins → Credentials → System → Global Credentials → Add Credentials
GitHub Credentials
GitHub Credentials
Configuration: - Kind: Username with password - Username: AbhishekSingh070693 - Password: GitHub Personal Access Token (PAT) - ID: Git-hub - Description: Git-hub
Important: GitHub no longer accepts account passwords for Git operations. A Personal Access Token (PAT) must be used.
8.2 Adding Tomcat Credentials
For deploying WAR files to Tomcat:
Tomcat Credentials
Tomcat Credentials
Configuration: - Kind: Username with password - Username: tomcat - Password: tomcat - ID: tomcat - Description: tomcat-credentials
________________________________________
9. Pipeline Creation
9.1 Creating a New Pipeline Job
Create New Pipeline
Create New Pipeline
Steps: 1. Click “New Item” 2. Enter item name: student 3. Select “Pipeline” 4. Click OK
9.2 Pipeline Script
We used the Pipeline Syntax generator to build the script:
Pipeline Syntax Generator
Pipeline Syntax Generator
Final Pipeline Script:

pipeline {
    agent any
    stages {
        stage('checkout') {
            steps {
                checkout scmGit(
                    branches: [[name: '*/main']], 
                    extensions: [], 
                    userRemoteConfigs: [[
                        credentialsId: 'Git-hub', 
                        url: 'https://github.com/AbhishekSingh070693/Student2.git'
                    ]]
                )
            }
        }
        stage('test') {
            steps {
                echo 'Hello World'
            }
        }
        stage('build') {
            steps {
                withMaven(maven: 'maven', traceability: true) {
                    dir('studentapp') {
                    sh 'mvn clean package'
                    }
                }
            }
        }
        stage('deploy') {
            steps {
                deploy adapters: [tomcat9(alternativeDeploymentContext: '', credentialsId: 'tomcat', path: '', url: 'http://localhost:9090')], contextPath: 'student', war: 'studentapp/target/*.war'
            }
        }
    }
}

Pipeline Script
Pipeline Script
9.3 Generating Checkout Syntax
Generate Checkout Script
Generate Checkout Script
9.4 Generating Build Syntax (withMaven)
Build Syntax - Part 1 Build Syntax - Part 2
9.5 Generating Deploy Syntax
Deploy Syntax
Deploy Syntax
________________________________________
10. Troubleshooting & Fixes
10.1 Error: Maven Local Repository Not Accessible
Error Message:
[ERROR] Could not create local repository at /home/jenkins/.m2/repository
Error 1
Error 1
Fix:
# Create Maven repository directory
sudo mkdir -p /home/jenkins/.m2/repository

# Change ownership to jenkins user
sudo chown -R jenkins:jenkins /home/jenkins
Error 1 Resolution
Error 1 Resolution
10.2 Error: No POM in Directory
Error Message:
[ERROR] The goal you specified requires a project to execute but there is no POM in this directory
Error 2
Error 2
Fix: Added dir('studentapp') to navigate to the correct directory before running Maven:
stage('build') {
    steps {
        withMaven(maven: 'maven', traceability: true) {
            dir('studentapp') {  // Navigate to subdirectory containing pom.xml
                sh 'mvn clean package'
            }
        }
    }
}
Error 2 Resolution - Directory Fix
Error 2 Resolution - Directory Fix
Also updated WAR path:
war: 'studentapp/target/*.war'  // Changed from 'studentapp/*.war'
Error 2 Resolution - WAR Path Fix
Error 2 Resolution - WAR Path Fix
10.3 Pipeline Structure Reference
We referred to the official Jenkins documentation for pipeline syntax:
Pipeline Documentation Reference
Pipeline Documentation Reference
________________________________________
11. Successful Deployment
11.1 Pipeline Execution Success
The pipeline executed successfully with all stages passing:
Deployment Successful
Deployment Successful
Pipeline Stages: 1. ✅ Checkout - Code pulled from GitHub (0.61s) 2. ✅ Test - Echo test passed (59ms) 3. ✅ Build - Maven build completed (7s) 4. ✅ Deploy - WAR deployed to Tomcat (3s)
Deploy Log:
[DeployPublisher][INFO] Attempting to deploy 1 war file(s)
[DeployPublisher][INFO] Deploying /var/lib/jenkins/workspace/student/studentapp/target/studentapp-1.0.war to container Tomcat 9.x Remote with context student
[/var/lib/jenkins/workspace/student/studentapp/target/studentapp-1.0.war] is not deployed. Doing a fresh deployment.
Deploying [/var/lib/jenkins/workspace/student/studentapp/target/studentapp-1.0.war]
11.2 Application Access
The application is successfully deployed and accessible at:
URL: http://54.160.252.48:9090/student/
Student Application Running
Student Application Running
________________________________________
Summary
This setup demonstrates a complete CI/CD pipeline with the following workflow:
1.	Developer pushes code to GitHub repository
2.	Jenkins triggers build automatically (or manually)
3.	Maven compiles the Java web application into a WAR file
4.	Jenkins deploys the WAR file to Tomcat using the Deploy to Container plugin
5.	Application is live and accessible via the configured context path
Key Configurations Summary
Component	Setting
EC2 Instance	t3.micro, Ubuntu 26.04, 20GB storage
Jenkins	Port 8080, Location /var/lib/jenkins
Tomcat	Port 9090, Location /opt/apache-tomcat-9.0.119
Tomcat User	tomcat/tomcat with manager-script role
GitHub Repo	https://github.com/AbhishekSingh070693/Student2.git
Application Context	/student
________________________________________
Commands Reference
User Management
useradd <username>              # Create user
passwd <username>               # Set password
usermod -s /bin/bash <username> # Change shell
Tomcat Management
systemctl start tomcat          # Start Tomcat
systemctl stop tomcat           # Stop Tomcat
systemctl restart tomcat        # Restart Tomcat
systemctl status tomcat         # Check status
systemctl enable tomcat         # Enable auto-start
Jenkins Management
systemctl start jenkins         # Start Jenkins
systemctl stop jenkins          # Stop Jenkins
systemctl restart jenkins       # Restart Jenkins
systemctl status jenkins        # Check status
System Fixes
# Resize /tmp
mount -o remount,size=3G /tmp

# Add swap
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile

# Check disk space
df -h

# Check memory
free -h
________________________________________
End of Document
