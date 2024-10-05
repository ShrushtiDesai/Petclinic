# DevSecOps Experimentation Guide

This guide outlines the steps to download, configure, and deploy a Java Maven project using Jenkins, Apache Tomcat, and Ngrok, along with integrating GitHub webhooks to trigger CI/CD pipelines automatically.

## Step 1: Download and Setup Apache Tomcat

### 1a: Download Apache Tomcat
Switch to the root user and navigate to the `/opt` directory to download Apache Tomcat.

```bash
sudo su
cd /opt
wget https://archive.apache.org/dist/tomcat/tomcat-9/v9.0.80/bin/apache-tomcat-9.0.80.tar.gz
```
### 1b: Extract Apache Tomcat
Extract the downloaded file.
```bash
tar -xvf apache-tomcat-9.0.80.tar.gz
```

### 1c: Configure Apache Tomcat
Edit the tomcat-users.xml file to add an admin user.
```bash
cd /opt/apache-tomcat-9.0.80/conf
nano tomcat-users.xml
```
Add the following line before the last line:
```bash
<user username="admin" password="admin1234" roles="admin-gui,manager-gui"/>
```
Next, change the connector port in server.xml to 8082 (as Jenkins is running on port 8080).
```bash
nano /opt/apache-tomcat-9.0.80/conf/server.xml
```
Edit the port: (only required if your 8080 port is occupied by other application like Jenkins)
```bash
<Connector port="8082" protocol="HTTP/1.1" ... />
```
### 1d: Disable Valve
Comment out the following line in both manager/context.xml and host-manager/context.xml.
```bash
nano /opt/apache-tomcat-9.0.80/webapps/manager/META-INF/context.xml
nano /opt/apache-tomcat-9.0.80/webapps/host-manager/META-INF/context.xml
```
Comment the following:
```bash
<!--
<Valve className="org.apache.catalina.valves.RemoteAddrValve"
allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1" />
-->
```
### 1e: Create Symbolic Links
Create symbolic links to start and stop Tomcat easily.
```bash
ln -s /opt/apache-tomcat-9.0.80/bin/startup.sh /usr/bin/startTomcat
ln -s /opt/apache-tomcat-9.0.80/bin/shutdown.sh /usr/bin/stopTomcat
```
### 1f: Set Permissions
Give write and execute permissions to Apache Tomcat.
```bash
cd /opt
chmod -R 757 apache-tomcat-9.0.80
```

## Step 2: Download and Install Ngrok
```bash
sudo apt update
```
Download Ngrok from [here](https://ngrok.com/download). After installation, update the system again.

## Step 3: Install Plugins for Jenkins
### 3a. Install Jenkins
Follow the installation guide from the [Jenkins docs](https://www.jenkins.io/doc/book/installing/) as per your Operating System.

### 3b: Start Jenkins
Start Jenkins using the following command.
```bash
sudo systemctl start jenkins
```
Access Jenkins at http://localhost:8080 and log in.
### 3c: Install Plugins
In Jenkins, go to Manage Jenkins > System Configuration > Plugins and install the following plugins:
- Eclipse Temurin Installer
- openJDK-native-plugin
- Pipeline Maven Integration Plugin
  
### 3d: Configure Plugins
Go to Manage Jenkins > System Configuration > Tools, and add the following configurations:

- JDK: Name it jdk11, select "Install automatically," and choose "Adoptium.net."
- Maven: Name it maven3, select "Install automatically," and choose the latest version (e.g., 3.9.4).

## Step 4: Fork the GitHub Repository
Fork the following [Petclinic](https://github.com/jaiswaladi246/Petclinic) repository into your GitHub account.

## Step 5: Create a Pipeline Job in Jenkins
### 5a: Create Pipeline
In Jenkins, click New Item, select Pipeline, and name your project.

Under Build Triggers, select GitHub hook trigger for GITScm polling. In the pipeline section, use the following script:
```bash
pipeline {
    agent any
    tools {
        jdk "jdk11"
        maven "maven3"
    }
    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', url: 'your forked GitHub repository link'
            }
        }
        stage('Compile') {
            steps {
                sh "mvn clean compile"
            }
        }
        stage('Build') {
            steps {
                sh "mvn package -DskipTests=true"
            }
        }
        stage('Deploy') {
            steps {
                sh "cp target/*.war /opt/apache-tomcat-9.0.80/webapps"
            }
        }
    }
}
```
## Step 6: Create a GitHub Webhook
### 6a: Generate Ngrok URL
Run Ngrok to create an internet-accessible URL for Jenkins.
```bash
ngrok http 8080
```
### 6b: Set Up Webhook
Go to your GitHub repository, navigate to Settings > Webhooks, and add a new webhook with the following details:
- Payload URL: Ngrok HTTP URL/github-webhook/
- Trigger: Just the push event
Ensure the webhook is configured successfully by checking for the green tick.

## Step 7: Trigger the Jenkins Pipeline
### 7a: Start Apache Tomcat
Start the Tomcat server.
```bash
sudo startTomcat
```
Access Tomcat at http://localhost:8082.

### 7b: Perform Push Event
Make a change in the forked GitHub repository, commit, and push the changes. Jenkins will trigger a build, and after successful deployment, you can access the application at http://localhost:8082/petclinic.
