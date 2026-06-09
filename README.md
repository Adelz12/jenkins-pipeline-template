# CI/CD Pipeline with Jenkins and Docker Compose

## Overview

This project demonstrates a complete CI/CD workflow using Jenkins and Docker Compose.

The pipeline automatically:

1. Pulls source code from GitHub.
2. Packages the application.
3. Transfers files to a remote Linux server.
4. Deploys the application using Docker Compose.
5. Verifies that containers are running successfully.

---

## Architecture

```text
GitHub Repository
        │
        ▼
     Jenkins
        │
        ├── Checkout Source Code
        ├── Package Application
        ├── Transfer Files (SCP)
        ▼
   Remote Server
        │
        ▼
 Docker Compose
        │
        ├── Application Container
        ├── Database Container
        └── Supporting Services
```

---

## Project Structure

```text
.
├── Jenkinsfile
├── docker-compose.yml
├── Dockerfile
├── app/
├── requirements.txt
└── README.md
```

---

## Prerequisites

### Jenkins Server

Install:

* Jenkins
* Git
* OpenSSH Client

Required Jenkins Plugins:

* Git Plugin
* Credentials Plugin
* SSH Agent Plugin
* Pipeline Plugin

Verify installation:

```bash
jenkins --version
git --version
```

---

### Target Server

Verify Docker installation:

```bash
docker --version
docker compose version
```

Add the deployment user to the Docker group:

```bash
sudo usermod -aG docker $USER
```

Log out and log back in after running the command.

---

## Jenkins Credentials Setup

Navigate to:

```text
Manage Jenkins
└── Credentials
```

Add a new credential:

### Recommended Method

```text
Kind:
SSH Username with private key
```

Example:

```text
ID: server-ssh-key
Username: deploy
Private Key: <your-private-key>
```

The credential ID will be referenced inside the Jenkinsfile.

---

## Jenkins Job Configuration

Create a new Jenkins Pipeline Job:

```text
Dashboard
└── New Item
    └── Pipeline
```

Configuration:

```text
Pipeline Definition:
Pipeline script from SCM
```

SCM:

```text
Git
```

Repository URL:

```text
https://github.com/your-username/your-project.git
```

Branch:

```text
main
```

Script Path:

```text
Jenkinsfile
```

Save the job.

---

## Jenkinsfile Example

```groovy
pipeline {
    agent any

    environment {
        SERVER_USER = "deploy"
        SERVER_IP   = "192.168.1.100"
        TARGET_DIR  = "/home/deploy/application"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                url: 'https://github.com/your-username/project.git'
            }
        }

        stage('Package') {
            steps {
                sh '''
                    tar -czf project.tar.gz --exclude=.git .
                '''
            }
        }

        stage('Deploy') {
            steps {

                sshagent(credentials: ['server-ssh-key']) {

                    sh '''
                        ssh ${SERVER_USER}@${SERVER_IP} \
                        "mkdir -p ${TARGET_DIR}"

                        scp project.tar.gz \
                        ${SERVER_USER}@${SERVER_IP}:${TARGET_DIR}/

                        ssh ${SERVER_USER}@${SERVER_IP} "
                            cd ${TARGET_DIR}

                            tar -xzf project.tar.gz
                            rm -f project.tar.gz

                            docker compose up -d --build

                            docker ps
                        "
                    '''
                }
            }
        }
    }

    post {

        success {
            echo 'Deployment completed successfully.'
        }

        failure {
            echo 'Deployment failed.'
        }

        always {
            sh 'rm -f project.tar.gz || true'
        }
    }
}
```

---

## Docker Compose Example

```yaml
services:
  app:
    build: .
    ports:
      - "9999:8000"
    restart: unless-stopped
```

Start services:

```bash
docker compose up -d
```

Stop services:

```bash
docker compose down
```

View running containers:

```bash
docker ps
```

---

## Deployment Workflow

### Manual Deployment

From Jenkins:

```text
Dashboard
└── Pipeline Job
    └── Build Now
```

---

### Automatic Deployment

Configure a GitHub Webhook:

```text
GitHub Repository
└── Settings
    └── Webhooks
```

Webhook URL:

```text
http://your-jenkins-server:8080/github-webhook/
```

Each push to the repository automatically triggers the pipeline.

---

## Verification

Verify deployed containers:

```bash
docker ps
```

Check logs:

```bash
docker compose logs
```

Check a specific service:

```bash
docker compose logs app
```

---

## Troubleshooting

### SSH Authentication Failed

Test manually:

```bash
ssh deploy@server-ip
```

Verify:

* SSH service is running.
* Correct username is used.
* Correct private key is configured.
* Jenkins credential ID is correct.

---

### Docker Permission Denied

Verify user groups:

```bash
groups
```

Expected output should include:

```text
docker
```

If not:

```bash
sudo usermod -aG docker $USER
```

Log in again afterward.

---

### Application Not Reachable

Check containers:

```bash
docker ps
```

Verify:

* Port mappings
* Firewall configuration
* Application listening address
* Docker Compose configuration

---

### Build Failed

Review:

```text
Jenkins Dashboard
└── Build History
    └── Console Output
```

Look for:

* Git errors
* SSH errors
* Docker build failures
* Missing environment variables

---

## Security Recommendations

* Use SSH keys instead of passwords.
* Never hardcode credentials.
* Store secrets in Jenkins Credentials Manager.
* Restrict SSH access to authorized users.
* Keep Jenkins and Docker updated.
* Enable regular backups for Jenkins.

---

## Useful Commands

```bash
docker ps
docker compose up -d
docker compose down
docker compose restart
docker compose logs
docker compose logs -f
docker image ls
docker container ls
```

---

## License

This project is intended for educational and DevOps learning purposes.
