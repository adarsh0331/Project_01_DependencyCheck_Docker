<img width="940" height="385" alt="image" src="https://github.com/user-attachments/assets/28010ee4-0ff2-4ce5-8f14-b59d09f523be" />

# Spring Boot Java Web Application CI/CD Pipeline

## Overview

This repository contains a simple Spring Boot-based Java web application following the MVC (Model-View-Controller) architecture. The application is built using Maven and can be deployed as a Docker container. The controller returns a page with title and message attributes to the view layer.

This setup includes a comprehensive CI/CD pipeline using Jenkins, SonarQube for code quality analysis, OWASP Dependency-Check for security vulnerability scanning, Trivy for Docker image scanning, and Docker for building, pushing, and deploying the application.

The pipeline automates:
- Code compilation and testing
- Code quality checks with SonarQube
- Dependency security scanning
- Building the JAR artifact
- Docker image creation and vulnerability scanning
- Pushing the image to Docker Hub
- Deployment to a Docker container


## Prerequisites

- AWS EC2 instance with **t2.large** instance type
- Attached EBS volume: **50 GB**
- Amazon Linux 2 AMI (or compatible)
- User: `ec2-user` (with sudo privileges)
- Internet access for package installations

## Initial Server Setup

1. Launch an EC2 instance with the specified configuration.
2. SSH into the instance as `ec2-user`.
3. Update the system:
   ```
   sudo yum update -y
   ```

## Installing Jenkins

Install Jenkins on the EC2 instance using the following commands:

```
sudo yum install wget -y
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
sudo yum upgrade -y
sudo dnf install java-17-amazon-corretto -y
sudo yum install jenkins -y
sudo systemctl start jenkins
sudo systemctl enable jenkins
```

- Access Jenkins at `http://<EC2-Public-IP>:8080`.
- Unlock Jenkins using the initial admin password from `/var/lib/jenkins/secrets/initialAdminPassword`.
- Install suggested plugins during setup.
- Create an admin user.

## Installing Docker

Install and configure Docker on the Jenkins server:

```
sudo yum update -y
sudo yum install -y docker
sudo service docker start
sudo usermod -aG docker ec2-user
sudo usermod -a -G docker jenkins
```

- Log out and log back in as `ec2-user` to apply group changes.
- Verify Docker: `docker --version`.

## Jenkins Plugins Installation

Install the following plugins via **Manage Jenkins > Manage Plugins > Available**:

| Plugin Name              | Purpose                          |
|--------------------------|----------------------------------|
| SonarQube Scanner        | Integrate SonarQube analysis     |
| Docker Pipeline          | Docker integration in pipelines  |
| Pipeline: Stage View     | Visualize pipeline stages        |
| OWASP Dependency-Check   | Scan dependencies for vulnerabilities |

Restart Jenkins after installation.

## Credentials Setup in Jenkins

Configure global credentials in **Manage Jenkins > Manage Credentials > System > Global credentials (unrestricted)**:

### Docker Hub Credentials
- **Kind**: Secret text
- **Secret**: Your Docker Hub password
- **ID**: `dockerhub`
- **Description**: Docker Hub password for pushing images

### SonarQube Token
1. Start SonarQube container (see below).
2. Access SonarQube at `http://<EC2-Public-IP>:9000`.
3. Login with default credentials: `admin` / `admin`.
4. Go to **My Account > Security > Generate Tokens**.
5. Create a token (e.g., name: "Jenkins Token").
6. In Jenkins:
   - **Kind**: Secret text
   - **Secret**: Paste the generated token
   - **ID**: `sonar-token`
   - **Description**: SonarQube authentication token

## SonarQube Setup

### Create SonarQube Container
Run the following command to start SonarQube using Docker:

```
sudo docker run -itd --name sonar -p 9000:9000 sonarqube
```

- Wait for SonarQube to start (check logs: `sudo docker logs sonar`).
- Access at `http://<EC2-Public-IP>:9000`.
- Change default password on first login.

### Integrate SonarQube with Jenkins

1. **Install Plugins**:
   - SonarQube Scanner for Jenkins
   - (Optional) Quality Gates Plugin

2. **Configure SonarQube Server**:
   - Go to **Manage Jenkins > System Configuration > SonarQube Servers**.
   - Add new server:
     - **Name**: `MySonarQube`
     - **Server URL**: `http://localhost:9000` (or EC2 IP if remote)
     - **Credentials**: Select the `sonar-token` credential created earlier.
   - Save.

3. **Configure SonarQube Scanner**:
   - Go to **Manage Jenkins > Tools > SonarQube Scanner**.
   - Add installation:
     - **Name**: `SonarScanner`
     - **Install automatically**: Check this box (or provide manual path).
   - Save.

## OWASP Dependency-Check Setup

### Install OWASP Dependency-Check on Server

```
sudo yum update -y
cd /opt
sudo curl -L -O https://github.com/jeremylong/DependencyCheck/releases/download/v8.4.2/dependency-check-8.4.2-release.zip
sudo yum install unzip -y
sudo unzip dependency-check-8.4.2-release.zip
sudo mv dependency-check /opt/dependency-check
echo 'export PATH=$PATH:/opt/dependency-check/bin' | sudo tee -a /etc/profile.d/dependency-check.sh
source /etc/profile.d/dependency-check.sh
```

### Integrate with Jenkins

1. **Install Plugin**:
   - Search and install "OWASP Dependency-Check" plugin.

2. **Configure Tool**:
   - Go to **Manage Jenkins > Tools > Dependency-Check installations**.
   - Add:
     - **Name**: `dependency-check`
     - **Installation directory**: `/opt/dependency-check`
   - Save.

## Trivy Installation for Image Scanning

Install Trivy on the Jenkins server:

```
sudo rpm -ivh https://github.com/aquasecurity/trivy/releases/download/v0.18.3/trivy_0.18.3_Linux-64bit.rpm
```

- Verify: `trivy --version`.

## Jenkins Pipeline Configuration

Create a new Pipeline job in Jenkins (e.g., name: `spring-boot-cicd`):
- **Source Code Management**: Git (repository URL: your repo).
- **Pipeline**: Pipeline script from SCM or inline script.

Use the following Jenkinsfile (or inline pipeline script). Place the repository code in `java-cicd-project/spring-boot-app`.

```groovy
pipeline {
    agent any
    tools {
        maven 'Maven'  // Configure Maven tool in Manage Jenkins > Tools if needed
    }
    stages {
        stage('Compile and Test') {
            steps {
                echo 'Compiling and testing the code'
                sh 'ls -ltr'
                sh 'cd java-cicd-project/spring-boot-app && mvn compile && mvn test'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                echo 'Scanning project with SonarQube'
                sh 'ls -ltr'
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                    sh '''cd java-cicd-project/spring-boot-app && mvn sonar:sonar \\
                          -Dsonar.host.url=http://localhost:9000 \\
                          -Dsonar.login=$SONAR_TOKEN'''
                }
            }
        }
        stage('Dependency Check') {
            steps {
                sh '''
                /opt/dependency-check/bin/dependency-check.sh \
                --project "SpringBootApp" \
                --scan java-cicd-project/spring-boot-app \
                --format HTML \
                --data /var/lib/jenkins/odc-data \
                --out dependency-check-report
                '''
                // Archive the report
                archiveArtifacts 'dependency-check-report/**'
            }
        }
        stage('Building the code') {
            steps {
                sh 'ls -ltr'
                // build the project and create a JAR file
                sh 'cd java-cicd-project/spring-boot-app && mvn clean package'
            }
        }
        stage('Build docker image') {
            steps {
                script {
                    echo 'docker image build'
                    sh 'cd java-cicd-project/spring-boot-app && docker build -t adarshbarkunta/java:${BUILD_NUMBER} .'
                }
            }
        }
        stage('docker image scan') {
            steps {
                sh "trivy image adarshbarkunta/java:${BUILD_NUMBER}"
            }
        }
        stage('Push image to Hub') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'dockerhub', variable: 'dockerhub')]) {
                        sh 'docker login -u adarshbarkunta -p ${dockerhub}'
                    }
                    sh 'docker push adarshbarkunta/java:${BUILD_NUMBER}'
                }
            }
        }
        stage('Deploying image to docker container') {
            steps {
                script {
                    sh 'docker run -itd --name java-app -p 8000:8080 adarshbarkunta/java:${BUILD_NUMBER}'
                }
            }
        }
    }
    post {
        always {
            // Clean up if needed
            sh 'docker stop java-app || true'
            sh 'docker rm java-app || true'
        }
    }
}
```

### Pipeline Stages Explanation

| Stage                  | Description                                                                 |
|------------------------|-----------------------------------------------------------------------------|
| Compile and Test      | Runs `mvn compile && mvn test` to build and unit test the code.             |
| SonarQube Analysis    | Performs static code analysis using SonarQube with authentication token.    |
| Dependency Check      | Scans project dependencies for known vulnerabilities using OWASP tool. Outputs HTML report. |
| Building the code     | Builds the JAR artifact with `mvn clean package`.                           |
| Build docker image    | Creates Docker image from Dockerfile in the app directory, tagged with build number. |
| docker image scan     | Scans the Docker image for vulnerabilities using Trivy.                     |
| Push image to Hub     | Logs into Docker Hub and pushes the image.                                  |
| Deploying image       | Runs the image as a container, mapping port 8000 (host) to 8080 (container). |

- Access the deployed app at `http://<EC2-Public-IP>:8000`.
- View pipeline visualization using Stage View plugin.
- Reports: SonarQube dashboard, Dependency-Check HTML in Jenkins artifacts.

## Application Structure

- **pom.xml**: Root-level Maven configuration with Spring Boot dependencies.
- **java-cicd-project/spring-boot-app/**: Core application source code.
  - Controllers handle MVC requests.
  - Views rendered with Thymeleaf (or similar).
- **Dockerfile**: In `spring-boot-app/` for building the image (ensure it exists; example below if needed).

### Sample Dockerfile (if not present)
```
FROM openjdk:17-jdk-slim
COPY target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

## Troubleshooting

- **Jenkins not starting**: Check `sudo systemctl status jenkins`.
- **Docker permission issues**: Ensure `jenkins` user is in `docker` group; restart Jenkins.
- **SonarQube connection**: Verify container is running and URL is accessible.
- **Maven failures**: Ensure `JAVA_HOME` is set to Java 17; configure Maven tool in Jenkins.
- **Pipeline errors**: Check console output; ensure repo path is correct.
- **Security groups**: Allow inbound traffic on ports 8080 (Jenkins), 9000 (SonarQube), 8000 (App).

## Next Steps

- Add more tests and quality gates in SonarQube.
- Integrate notifications (e.g., email on failure).
- Scale with Kubernetes for production deployment.
- Monitor with Prometheus/Grafana.












