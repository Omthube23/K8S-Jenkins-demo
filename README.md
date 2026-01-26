---

```markdown
# 🚀 Jenkins CI/CD Pipeline on Kubernetes (Minikube)

This project demonstrates a **real-world CI/CD pipeline** using:

- GitHub (Source Code)
- Jenkins (CI/CD Engine)
- Kubernetes (Runtime Platform)
- Docker (Build & Packaging)
- Minikube (Local Kubernetes on EC2)

The pipeline automatically:

1. Pulls code from GitHub  
2. Creates a Jenkins agent pod in Kubernetes  
3. Builds a Docker image  
4. Deploys the app to Kubernetes using `kubectl`  
5. Updates the running application  
6. Rebuilds automatically on Git changes (SCM Polling)

This is a **production-style DevOps workflow**, not just a demo.

---

## 📐 Architecture

```

Developer (Git Push)
|
v
GitHub Repository
|
v
Jenkins (Controller)
|
v
Kubernetes Jenkins Agent Pod
|
|---> Docker Build
|---> kubectl apply
|
v
Kubernetes Cluster (Minikube)
|
v
Running Application

```

Jenkins itself runs inside Kubernetes.  
Each build spins up a **temporary agent pod**, runs the pipeline, and is destroyed.

---

## 📂 Repository Structure

```

.
├── app.py
├── requirements.txt
├── Dockerfile
├── Jenkinsfile
└── k8s/
├── deployment.yaml
└── service.yaml

````

---

## 🧰 Prerequisites

- EC2 Instance (t2.medium or above recommended)
- Docker installed
- Minikube installed
- kubectl installed
- Helm installed
- GitHub account + Personal Access Token

---

## ⚙️ Setup Steps

### 1️⃣ Start Minikube

```bash
minikube start --driver=docker --memory=1900
alias k="minikube kubectl --"
````

Verify:

```bash
k get nodes
```

---

### 2️⃣ Install Jenkins on Kubernetes

```bash
helm repo add jenkins https://charts.jenkins.io
helm repo update

k create namespace jenkins

helm install jenkins jenkins/jenkins -n jenkins \
  --set controller.resources.requests.cpu="200m" \
  --set controller.resources.requests.memory="512Mi" \
  --set controller.resources.limits.cpu="500m" \
  --set controller.resources.limits.memory="1Gi"
```

Wait for Jenkins:

```bash
k get pods -n jenkins -w
```

When it shows:

```
jenkins-0   2/2   Running
```

Get password:

```bash
k exec -n jenkins jenkins-0 -c jenkins -- cat /run/secrets/additional/chart-admin-password
```

Expose Jenkins:

```bash
k port-forward -n jenkins svc/jenkins 18080:8080
```

Then access Jenkins via SSH tunnel:

```bash
ssh -i key.pem -L 18080:localhost:18080 ec2-user@<EC2_IP>
```

Open in browser:

```
http://localhost:18080
```

---

## 🔐 Kubernetes RBAC (Critical Step)

Jenkins agents need permission to deploy into Kubernetes.

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

This fixes:

```
User "system:serviceaccount:jenkins:default" cannot get resource "deployments"
```

---

## 🧪 Jenkinsfile (Core of the Project)

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

---

## 🔁 Auto Trigger (CI)

Since Jenkins is behind SSH + port-forward, GitHub webhooks can’t reach it.
So we use **Poll SCM**:

In Jenkins Job → Configure → Build Triggers:

```
Poll SCM
*/1 * * * *
```

Now every `git push` triggers the pipeline automatically.

---

## 🧪 Testing the Pipeline

Edit the app:

```bash
nano app.py
```

Change the message:

```python
return "Hello from Jenkins + Kubernetes! Auto Trigger Working!"
```

Commit and push:

```bash
git add app.py
git commit -m "Test CI trigger"
git push
```

Within 1 minute Jenkins runs automatically.

Verify:

```bash
k get pods
k get svc
curl http://localhost:30007
```

You’ll see the updated message.

---

## 🧠 What I Learned

* Jenkins running inside Kubernetes
* Dynamic Jenkins agents as Kubernetes Pods
* Docker builds inside CI pipelines
* Deploying to Kubernetes using `kubectl` from Jenkins
* Kubernetes RBAC & ServiceAccounts
* Debugging real-world CI/CD failures:

  * Pod startup issues
  * Shell execution failures
  * RBAC permission errors
* Designing a secure, production-style pipeline
* Auto-triggering builds with SCM Polling

This project is a **complete DevOps workflow**, not a toy example.

---

## 🖼️ Screenshots

> (Add your screenshots here)

* Jenkins Dashboard
* Pipeline Stages
* Agent Pod Creation
* Kubernetes Pods
* Application Output

---

## 🏁 Final Flow

```
git push
   ↓
Jenkins detects change
   ↓
Agent Pod is created
   ↓
Docker image is built
   ↓
kubectl apply runs
   ↓
App is updated in Kubernetes
```

This is a real CI/CD pipeline.

````

---
