# CI/CD Pipeline for 2-Tier Flask Application (Flask + MySQL) on Azure

## Project Overview

This project implements a fully automated CI/CD pipeline for deploying a 2-tier web application consisting of a Flask backend and a MySQL database. The solution uses Docker for containerization, Jenkins for automation, GitHub for source control, and Microsoft Azure as the cloud hosting platform.

The pipeline enables continuous integration and continuous deployment, ensuring that every code change pushed to the repository is automatically built, tested, and deployed to a live environment hosted on Azure.

---

## Architecture
```
+-----------------+      +----------------------+      +-----------------------------+
|   Developer     |----->|     GitHub Repo      |----->|        Jenkins Server       |
| (pushes code)   |      |    (Source Code )    |      |  (on  Azure VM)               |
+-----------------+      +----------------------+      |                             |
                                                       | 1. Clones Repo              |
                                                       | 2. Builds Docker Image      |
                                                       | 3. Runs Docker Compose      |
                                                       +--------------+--------------+
                                                                      |
                                                                      | Deploys
                                                                      v
                                                       +-----------------------------+
                                                       |      Application Server     |
                                                       |      (Same Azure VM)         |
                                                       |                             |
                                                       | +-------------------------+ |
                                                       | | Docker Container: Flask | |
                                                       | +-------------------------+ |
                                                       |              |              |
                                                       |              v              |
                                                       | +-------------------------+ |
                                                       | | Docker Container: MySQL | |
                                                       | +-------------------------+ |
                                                       +-----------------------------+

```

---

## Azure Cloud Deployment

The application is deployed on a Microsoft Azure Virtual Machine running Ubuntu 22.04 LTS.

### Azure Configuration

- Azure Virtual Machine created with Ubuntu 22.04 LTS
- Network Security Group configured to allow required ports:
  - SSH (22)
  - Flask Application (5000)
  - Jenkins (8080)
  - HTTP (80)
<img src="diagrams/06.png">

### Role of Azure

Microsoft Azure serves as the production environment where:
- Jenkins runs as the CI/CD automation server
- Docker containers are deployed and managed
- The application is hosted and publicly accessible via a static public IP address

---

## Live Application Access

- Flask Application: `http://<azure-public-ip>:5000`
- Jenkins Dashboard: `http://<azure-public-ip>:8080`

Any changes pushed to the GitHub repository are automatically deployed to the live Azure environment.

---

## CI/CD Workflow

1. Developer pushes code changes to GitHub repository
2. Jenkins running on Azure VM detects changes every 5 minutes using SCM polling
3. Jenkins pulls the latest source code
4. Docker image is built for the Flask application
5. Docker Compose is used to start and manage containers
6. Updated application is deployed and made live on Azure

---

## Step 1: Azure Virtual Machine Setup

- Ubuntu 22.04 LTS virtual machine created on Azure
- Inbound security rules configured for required services (22, 5000, 8080, 80)
### Connect to Azure VM using SSH
Connect to the VM using its public IP:

```bash
ssh azureuser@<azure-public-ip>
```
If using a key file:
```
ssh -i key.pem azureuser@<azure-public-ip>
```
---

## Step 2: Install System Dependencies

```bash
sudo apt update
sudo apt install git docker.io docker-compose-v2 -y
```


## Step 3: Enable and configure Docker:

→ Starts the Docker service immediately so you can use Docker commands right away.
```bash
sudo systemctl start docker
```
→ Makes Docker start automatically every time your system boots.
```bash
sudo systemctl enable docker
```
→ Adds the current user to the docker group (so Docker commands can be run without sudo).
```bash
sudo usermod -aG docker $USER
```
→ Reloads the current shell with the docker group permissions immediately (no logout needed).
```bash
newgrp docker
```
## Step 3: Java and Jenkins Installation
Install Java
```bash
sudo apt update
sudo apt install fontconfig openjdk-21-jre -y
java -version
```
Install Jenkins (LTS Version)
```bash
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key

echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] \
https://pkg.jenkins.io/debian-stable binary/" | sudo tee \
/etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install jenkins -y
```

→ Starts the Jenkins service immediately 
```bash
sudo systemctl start jenkins
```
→ Makes Jenkins start automatically every time your system boots.
```bash
sudo systemctl enable jenkins
```

### Jenkins Initial Setup

Access Jenkins:
```
http://<azure-public-ip>:8080
```
<img src="diagrams/06.png">

Get initial admin password:
```
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
- Paste password into Jenkins UI
- Install suggested plugins
- Create admin user
- Access Jenkins at:


##  Dockerfile
```dockerfile
FROM python:3.9-slim

WORKDIR /app

RUN apt-get update && apt-get install -y gcc default-libmysqlclient-dev pkg-config

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 5000

CMD ["python", "app.py"]
```

## docker-compose.yml
```yaml
version: "3.8"

services:
  mysql:
    image: mysql
    container_name: mysql
    environment:
      MYSQL_DATABASE: devops
      MYSQL_ROOT_PASSWORD: root
    ports:
      - "3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql
    networks:
      - two-tier
    restart: always

  flask:
    build: .
    container_name: flask-app
    ports:
      - "5000:5000"
    environment:
      MYSQL_HOST: mysql
      MYSQL_USER: root
      MYSQL_PASSWORD: root
      MYSQL_DB: devops
    depends_on:
      - mysql
    networks:
      - two-tier
    restart: always

volumes:
  mysql-data:

networks:
  two-tier:
  ```

## Jenkinsfile (CI/CD Pipeline)
```groovy
pipeline {
    agent any

    triggers {
        pollSCM('H/5 * * * *')
    }

    stages {
        stage('Clone Repository') {
            steps {
                git branch: 'main', url: 'https://github.com/your-username/your-repo.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t flask-app:latest .'
            }
        }

        stage('Deploy Application') {
            steps {
                sh 'docker compose down || true'
                sh 'docker compose up -d --build'
            }
        }
    }
}
```

###  Jenkins Pipeline Creation and Execution**

1.  **Create a New Pipeline Job in Jenkins:**
    * From the Jenkins dashboard, select **New Item**.
    * Name the project, choose **Multibranch Pipeline**, and click **OK**.

2.  **Configure the Pipeline:**
    * Configure the Multibranch Pipeline
    * Enter your GitHub repository URL.
    * Save the configuration.

<img src="diagrams/04.png">

3.  **Run the Pipeline:**
    * Click **Build Now** to trigger the pipeline manually for the first time.
    * Monitor the execution through the **Stage View** or **Console Output**.

<img src="diagrams/05.png">
<img src="diagrams/06.png">

4.  **Verify Deployment:**
    * After a successful build, your Flask application will be accessible at `http://<your-ec2-public-ip>:5000`.
    * Confirm the containers are running on the EC2 instance with `docker ps`.

---

## Key Features
- End-to-end CI/CD automation
- Cloud deployment using Microsoft Azure
- Containerized microservices architecture
- Jenkins-based pipeline automation
- Docker and Docker Compose orchestration
- Automated deployment on every code commit
- Live production deployment environment

## Conclusion
This project demonstrates a complete DevOps CI/CD pipeline implemented on Microsoft Azure. It integrates GitHub, Jenkins, Docker, and Azure Virtual Machines to achieve continuous integration and continuous deployment
