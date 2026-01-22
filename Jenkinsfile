pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: docker
    image: docker:24.0
    command: ["cat"]
    tty: true
    volumeMounts:
    - name: dockersock
      mountPath: /var/run/docker.sock

  - name: kubectl
    image: bitnami/kubectl:latest
    command: ["cat"]
    tty: true
    volumeMounts:
    - name: kubeconfig
      mountPath: /root/.kube

  volumes:
  - name: dockersock
    hostPath:
      path: /var/run/docker.sock

  - name: kubeconfig
    hostPath:
      path: /home/ec2-user/.kube
"""
        }
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
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
                container('kubectl') {
                    sh 'kubectl apply -f k8s/'
                }
            }
        }
    }
}

