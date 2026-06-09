# CI/CD Pipeline with Jenkins and Docker Compose

This project demonstrates how to automate application deployment using **Jenkins Declarative Pipeline** and **Docker Compose** on a remote Linux server.

---

## Architecture

```text
GitHub Repository
        │
        ▼
     Jenkins
        │
        ├── Build
        ├── Package (tar.gz)
        ├── Transfer (scp)
        ▼
   Remote Server
        │
        ▼
 Docker Compose
        │
        ├── Application
        ├── Database
        └── Other Services
```

---

## Prerequisites

### Jenkins Server

* Jenkins installed and running
* Git installed
* SSH access to the target server
* Required Jenkins plugins:

  * Git Plugin
  * Credentials Plugin
  * SSH Agent Plugin

### Target Server

Verify Docker and Docker Compose are installed:

```bash
docker --version
docker compose version
```

Add the deployment user to the Docker group:

```bash
sudo usermod -aG docker $USER
```

Log out and log back in for the changes to take effect.

---

## Jenkins Credentials Configuration

Store sensitive information securely in Jenkins.

Navigate to:

```text
Manage Jenkins
└── Credentials
    └── Global
```

Add one of the following credential types:

### Option 1: SSH Private Key (Recommended)

* Kind: SSH Username with private key
* ID: server-ssh-key

### Option 2: Username and Password

* Kind: Username with password

SSH keys are recommended because they are more secure and easier to automate.

---

## Jenkinsfile

Create a file named `Jenkinsfile` in the root of your repository.

```groovy
pipeline {
    agent any

    environment {
        SERVER_USER = "your-user"
        SERVER_IP   = "your-server-ip"
        TARGET_DIR  = "/home/your-user/app"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/your-username/your-repository.git'
            }
        }

        stage('Package Application') {
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

Example `docker-compose.yml`:

```yaml
services:
  web:
    build: .
    ports:
      - "8000:8000"
    restart: unless-stopped
```

Start services manually:

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

1. Developer pushes code to GitHub.
2. Jenkins pulls the latest source code.
3. Jenkins packages the application.
4. Jenkins transfers files to the target server.
5. Docker Compose rebuilds and starts containers.
6. Application becomes available on the configured port.

---

## Troubleshooting

### SSH Connection Failed

Verify:

```bash
ssh user@server-ip
```

Check:

* Server IP address
* SSH service status
* Firewall rules
* Jenkins credentials

---

### Docker Permission Denied

Verify the deployment user belongs to the Docker group:

```bash
groups
```

If Docker is missing:

```bash
sudo usermod -aG docker $USER
```

Then log in again.

---

### Container Starts but Application Is Not Reachable

Check container status:

```bash
docker ps
```

Check logs:

```bash
docker logs <container_id>
```

Verify:

* Port mappings
* Firewall rules
* Application listening address
* Docker Compose configuration

---

### Deployment Fails During Build

Inspect Jenkins console output:

```text
Jenkins Dashboard
└── Build History
    └── Console Output
```

Look for:

* Git authentication errors
* SSH authentication failures
* Docker build errors
* Missing environment variables

---

## Security Recommendations

* Use SSH keys instead of passwords.
* Avoid hardcoding secrets in source code.
* Store credentials in Jenkins Credentials Manager.
* Limit SSH access to trusted users.
* Regularly update Docker and Jenkins.

---

## Useful Commands

```bash
docker ps
docker compose up -d
docker compose down
docker compose logs
docker compose restart
```

---

## License

This project is provided for educational and DevOps learning purposes.
