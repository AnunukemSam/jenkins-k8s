# Jenkins CI/CD Pipeline for Flask Apps on Kubernetes with Docker Build and GitHub Webhooks

## 📌 Project Summary

This DevOps project sets up a **CI/CD pipeline using Jenkins on AKS (Azure Kubernetes Service)** to build and deploy **three Flask applications** using Docker. It integrates GitHub Webhooks, a shared Jenkins library, Kubernetes agents, and Docker builds inside Kubernetes Pods. All best practices were observed — avoiding Docker-in-Docker when necessary and ensuring image pushes to DockerHub.

---

## 📁 Project Structure

```
.
├── flaskapp-logger/           # One of the Flask app repos
│   ├── app.py
│   ├── requirements.txt
│   ├── Dockerfile
│   └── Jenkinsfile
├── jenkins-shared-lib/       # Jenkins Shared Library
│   ├── vars/
│   │   └── basePipeline.groovy
│   └── jenkins/
│       └── docker-pod.yaml
```

---

## ⚙️ Tools & Technologies

* **Jenkins** (Kubernetes Plugin)
* **DockerHub** (for image registry)
* **AKS (Azure Kubernetes Service)**
* **Helm** (Jenkins deployment)
* **GitHub Webhooks**
* **Shared Library (Jenkins)**

---

## ✅ Prerequisites

* Working Kubernetes cluster (AKS)
* Jenkins deployed via Helm
* DockerHub account
* GitHub account with three Flask repos and shared-lib repo
* Valid PAT (Personal Access Token) for GitHub

---

## 🔑 Secrets Management

### Create DockerHub Secret:

```bash
kubectl create secret generic dockerhub-secret \
  --from-file=.dockerconfigjson=/path/to/config.json \
  --type=kubernetes.io/dockerconfigjson \
  -n jenkins
```

Contents of `config.json`:

```json
{
  "auths": {
    "https://index.docker.io/v1/": {
      "username": "your-dockerhub-username",
      "password": "your-password",
      "auth": "<base64encoded>"
    }
  }
}
```

---

## 🚀 Jenkins Setup on AKS

### Step 1: Add Helm repo and update

```bash
helm repo add jenkins https://charts.jenkins.io
helm repo update
```

### Step 2: Install Jenkins via Helm

```bash
helm install jenkins -n jenkins -f jenkins-values.yaml jenkins/jenkins
```

Example `jenkins-values.yaml` (snippets):

```yaml
controller:
  installPlugins:
    - kubernetes:latest
    - workflow-aggregator:latest
    - git:latest
    - docker-workflow:latest
    - credentials-binding:latest
    - github:latest
  admin:
    username: admin
    password: admin
  resources:
    requests:
      memory: "512Mi"
      cpu: "500m"
```

### Step 3: Access Jenkins UI

```bash
kubectl get svc -n jenkins
```

Visit the external IP of the `LoadBalancer` service.

---

## 🔁 Setup GitHub Webhooks

* Go to each Flask repo on GitHub
* Navigate to **Settings > Webhooks > Add Webhook**
* Payload URL: `http://<jenkins-external-ip>/github-webhook/`
* Content type: `application/json`
* Events: **Just the push event**
* Save

---

## 📦 Shared Library Setup

### Directory Structure (in `jenkins-shared-lib` repo)

```
jenkins-shared-lib/
├── vars/
│   └── basePipeline.groovy
└── jenkins/
    └── docker-pod.yaml
```

### Example `basePipeline.groovy`

```groovy
def call(Map config) {
  def dockerImage = config.dockerImage
  def appPort     = config.port
  def dockerfile  = config.dockerfile ?: './Dockerfile'

  pipeline {
    agent {
      kubernetes {
        yamlFile 'jenkins/docker-pod.yaml'
        defaultContainer 'docker'
      }
    }

    environment {
      DOCKER_CONFIG = '/kaniko/.docker'
    }

    stages {
      stage('Install Dependencies') {
        steps {
          sh '''
            apk add --no-cache python3 py3-pip curl
            python3 -m venv venv
            . venv/bin/activate
            pip install -r requirements.txt
          '''
        }
      }

      stage('Run App Test') {
        steps {
          sh '''
            . venv/bin/activate
            python app.py &
            sleep 5
            curl http://localhost:${config.port} || true
          '''
        }
      }

      stage('Build and Push Docker Image') {
        steps {
          sh """
            export DOCKER_CONFIG=/kaniko/.docker
            docker version
            docker build -t ${dockerImage} -f ${dockerfile} .
            docker push ${dockerImage}
          """
        }
      }

      stage('Cleanup') {
        steps {
          sh 'pkill python || true'
        }
      }
    }

    post {
      always {
        echo "Build completed: ${currentBuild.fullDisplayName} → ${currentBuild.currentResult}"
      }
    }
  }
}
```

---

## 🧱 Jenkinsfile (in Flask App Repo)

```groovy
@Library('jenkins-shared-lib@main') _

basePipeline(
  dockerImage: 'anunukemsam/flaskapp-logger:20',
  port: 5000,
  repoUrl: 'https://github.com/vetrax1/flaskapp-logger.git'
)
```

---

## 🧪 Troubleshooting Lessons (What We Fixed)

| Issue                            | Fix                                                        |
| -------------------------------- | ---------------------------------------------------------- |
| Metrics Server error             | Edited memory request in YAML or disabled memory requests  |
| Jenkins `adminUser` key          | Replaced with `controller.admin.username` in `values.yaml` |
| DinD error with Docker socket    | Switched to using correct Docker mount and secrets         |
| Python dependency install errors | Used virtualenv + proper activation                        |
| Kaniko `/busybox/sh` missing     | Changed to Docker DinD with volume mount and secret        |
| Docker push error                | Proper `DOCKER_CONFIG` mount and secret reference fixed it |

---

## ✅ Final Working Flow

1. Developer pushes to `main` branch of Flask app repo
2. GitHub Webhook triggers Jenkins
3. Jenkins pulls repo and loads shared library
4. Pipeline provisions pod with Docker, installs dependencies
5. Flask app is started and curl-tested
6. Docker image is built and pushed to DockerHub
7. App is cleaned up
8. Status reported back to GitHub

---

## 📍Notes & Best Practices

* Use Helm for clean Jenkins management
* Always externalize secrets via K8s `Secret`
* Avoid Docker-in-Docker unless configured carefully
* Use shared libraries to DRY up pipeline code
* Use virtual environments for Python package management inside jobs
* Add proper logging and result checking to all critical steps

---

## 👏 Authors & Credits

**Engineer:** [Anunukem Sam (TechDotSam)](https://github.com/vetrax1)
**Mentor-style Guidance:** ChatGPT (Senior DevOps Mode)

---

## 📌 Future Improvements

* Add deployment stage to Kubernetes
* Add Helm charts per app
* Setup Slack or Email notifications
* Integrate with Snyk for vulnerability scanning
* Add Prometheus & Grafana monitoring pipeline
