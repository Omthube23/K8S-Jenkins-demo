# Jenkins CI/CD Pipeline on Kubernetes

A production-ready CI/CD pipeline implementation using Jenkins on Kubernetes with automated deployments, dynamic agent provisioning, and GitHub integration.

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Usage](#usage)
- [Pipeline Stages](#pipeline-stages)
- [Troubleshooting](#troubleshooting)
- [Learning Outcomes](#learning-outcomes)
- [Contributing](#contributing)
- [License](#license)

## Overview

This project demonstrates a complete DevOps workflow using industry-standard tools and practices. The pipeline automatically builds, tests, and deploys applications to Kubernetes whenever changes are pushed to GitHub.

### Key Components

- **Source Control:** GitHub
- **CI/CD Engine:** Jenkins
- **Container Runtime:** Docker
- **Orchestration Platform:** Kubernetes (Minikube)
- **Package Manager:** Helm

## Features

- ✅ Automated CI/CD pipeline triggered by Git commits
- ✅ Dynamic Jenkins agent pods created on-demand
- ✅ Docker image building within the pipeline
- ✅ Automated Kubernetes deployments using kubectl
- ✅ RBAC-secured service accounts
- ✅ SCM polling for automatic build triggers
- ✅ Production-ready architecture

## Architecture

```
┌─────────────────┐
│   Developer     │
│   (Git Push)    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│     GitHub      │
│   Repository    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│     Jenkins     │
│   (Controller)  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  K8s Agent Pod  │
│  ┌───────────┐  │
│  │Docker Build│  │
│  └───────────┘  │
│  ┌───────────┐  │
│  │kubectl    │  │
│  │apply      │  │
│  └───────────┘  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Kubernetes    │
│     Cluster     │
│   (Minikube)    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│    Running      │
│  Application    │
└─────────────────┘
```

## Prerequisites

### System Requirements

- EC2 Instance (t2.medium or higher recommended)
- Minimum 4GB RAM
- 20GB disk space

### Required Software

- Docker (latest version)
- Minikube
- kubectl
- Helm 3.x
- Git

### Access Requirements

- GitHub account
- GitHub Personal Access Token
- SSH access to EC2 instance

## Installation

### Step 1: Start Minikube

```bash
# Start Minikube with Docker driver
minikube start --driver=docker --memory=1900

# Create kubectl alias for convenience
alias k="minikube kubectl --"

# Verify cluster is running
k get nodes
```

### Step 2: Install Jenkins on Kubernetes

```bash
# Add Jenkins Helm repository
helm repo add jenkins https://charts.jenkins.io
helm repo update

# Create Jenkins namespace
k create namespace jenkins

# Install Jenkins with resource limits
helm install jenkins jenkins/jenkins -n jenkins \
  --set controller.resources.requests.cpu="200m" \
  --set controller.resources.requests.memory="512Mi" \
  --set controller.resources.limits.cpu="500m" \
  --set controller.resources.limits.memory="1Gi"
```

### Step 3: Wait for Jenkins to Start

```bash
# Watch pod status
k get pods -n jenkins -w
```

Wait until you see:
```
jenkins-0   2/2   Running
```

### Step 4: Retrieve Jenkins Admin Password

```bash
k exec -n jenkins jenkins-0 -c jenkins -- \
  cat /run/secrets/additional/chart-admin-password
```

### Step 5: Expose Jenkins UI

```bash
# Port forward Jenkins service
k port-forward -n jenkins svc/jenkins 18080:8080
```

In a separate terminal, create SSH tunnel:

```bash
ssh -i key.pem -L 18080:localhost:18080 ec2-user@<EC2_IP>
```

Access Jenkins at: `http://localhost:18080`

## Configuration

### RBAC Setup

Create service account and permissions for Jenkins agents:

```bash
cat <<EOF | k apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins-admin
  namespace: jenkins
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: jenkins-admin-role
  namespace: jenkins
rules:
- apiGroups: ["", "apps", "extensions"]
  resources: ["deployments", "services", "pods", "replicasets", "configmaps", "secrets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jenkins-admin-rolebinding
  namespace: jenkins
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: jenkins-admin-role
subjects:
- kind: ServiceAccount
  name: jenkins-admin
  namespace: jenkins
EOF
```

### Enable Automatic Triggers

Since Jenkins runs behind SSH tunnel, configure SCM polling:

1. Go to Jenkins Dashboard
2. Select your job → Configure
3. Under "Build Triggers", enable "Poll SCM"
4. Set schedule: `*/1 * * * *` (polls every minute)

## Usage

### Project Structure

```
.
├── app.py                  # Application code
├── requirements.txt        # Python dependencies
├── Dockerfile             # Container image definition
├── Jenkinsfile            # Pipeline configuration
└── k8s/
    ├── deployment.yaml    # Kubernetes deployment
    └── service.yaml       # Kubernetes service
```

### Jenkinsfile Configuration

```groovy
pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: jenkins-admin
  containers:
  - name: docker
    image: docker:24.0
    command: ["cat"]
    tty: true
    volumeMounts:
    - name: dockersock
      mountPath: /var/run/docker.sock
  volumes:
  - name: dockersock
    hostPath:
      path: /var/run/docker.sock
"""
        }
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install kubectl') {
            steps {
                container('docker') {
                    sh '''
                      apk add --no-cache curl
                      curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
                      chmod +x kubectl
                      mv kubectl /usr/local/bin/kubectl
                      kubectl version --client
                    '''
                }
            }
        }

        stage('Build Image') {
            steps {
                container('docker') {
                    sh 'docker build -t demo-app:latest .'
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                container('docker') {
                    sh 'kubectl apply -f k8s/'
                }
            }
        }
    }
}
```

### Testing the Pipeline

1. Make changes to your application:
   ```bash
   nano app.py
   ```

2. Commit and push changes:
   ```bash
   git add app.py
   git commit -m "Update application message"
   git push
   ```

3. Jenkins will automatically detect changes and trigger build within 1 minute

4. Verify deployment:
   ```bash
   k get pods
   k get svc
   curl http://localhost:30007
   ```

## Pipeline Stages

### 1. Checkout
Clones the repository from GitHub to the Jenkins agent pod.

### 2. Install kubectl
Downloads and installs kubectl CLI tool in the agent container.

### 3. Build Image
Builds Docker image from Dockerfile using application code.

### 4. Deploy to Kubernetes
Applies Kubernetes manifests using kubectl to deploy/update the application.

## Troubleshooting

### Common Issues

**Problem:** Pod fails to start
```bash
# Check pod logs
k logs -n jenkins <pod-name>

# Describe pod for events
k describe pod -n jenkins <pod-name>
```

**Problem:** RBAC permission denied
```bash
# Verify service account exists
k get sa -n jenkins

# Check role bindings
k get rolebinding -n jenkins
```

**Problem:** Docker build fails
```bash
# Verify Docker socket mount
k describe pod <agent-pod-name> -n jenkins

# Check Docker daemon status on host
systemctl status docker
```

**Problem:** kubectl command not found
- Ensure kubectl installation stage completes successfully
- Check agent pod has internet access

## Learning Outcomes

This project provides hands-on experience with:

- Jenkins installation and configuration on Kubernetes
- Dynamic Jenkins agent provisioning using Kubernetes pods
- Docker image building within CI/CD pipelines
- Kubernetes deployments using kubectl
- RBAC configuration and service account management
- SCM polling for automated builds
- Debugging production-level CI/CD issues
- Designing secure, scalable DevOps workflows

## Contributing

Contributions are welcome! Please follow these steps:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## License

This project is licensed under the MIT License - see the LICENSE file for details.

---

**Built with** ❤️ **for learning DevOps practices**

For questions or issues, please open an issue in the repository.
