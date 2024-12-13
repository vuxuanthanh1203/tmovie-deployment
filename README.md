# Deploying tmovie on Kubernetes using DevSecOps Methodology

In this project, we will deploy the **tmovie** application on an EKS cluster using the DevSecOps methodology. We will utilize security tools such as **SonarQube**, **OWASP Dependency Check**, and **Trivy**. Monitoring will be handled using **Prometheus** and **Grafana**, and **ArgoCD** will be used for deployments.

---

## Table of Contents
1. [Launch EC2 Instance and Install Tools](#step-1-launch-ec2-instance-and-install-tools)
2. [Configure Jenkins Plugins](#step-2-configure-jenkins-plugins)
3. [Set Up SonarQube](#step-3-set-up-sonarqube)
4. [Set Up OWASP Dependency Check](#step-4-set-up-owasp-dependency-check)
5. [Set Up Docker for Jenkins](#step-5-set-up-docker-for-jenkins)
6. [Create Jenkins Pipeline](#step-6-create-jenkins-pipeline)
7. [Create an EKS Cluster](#step-7-create-an-eks-cluster)
8. [Deploy Prometheus and Grafana](#step-8-deploy-prometheus-and-grafana)
9. [Deploy ArgoCD](#step-9-deploy-argocd)

---

## Step 1: Launch EC2 Instance and Install Tools

We will use **Terraform** to launch an EC2 instance and add a script as `userdata` to install the following tools:
- **Jenkins**
- **SonarQube**
- **Trivy**
- **Docker**

Ensure that Terraform is correctly configured, and the required version is installed on your machine.

---

## Step 2: Configure Jenkins Plugins

Access Jenkins at port `8080` and install the following plugins:

1. **NodeJS**
2. **Eclipse Temurin Installer**
3. **SonarQube Scanner**
4. **OWASP Dependency Check**
5. **Docker**
6. **Docker Commons**
7. **Docker Pipeline**
8. **Docker API**
9. **docker-build-step**

---

## Step 3: Set Up SonarQube

1. Access the SonarQube dashboard via `http://<elastic_ip>:9000`.
2. Generate a token:
   - Navigate to `Administration -> Security -> Users -> Create a Token`.
3. Add the token as a credential in Jenkins:
   - `Manage Jenkins -> Credentials -> System -> Global Credentials`.
4. Configure SonarQube in Jenkins:
   - `Manage Jenkins -> System -> SonarQube installation`.
   - Add the URL and select the token from step 3.
5. Install SonarQube Scanner:
   - `Manage Jenkins -> Tools -> SonarQube Scanner Installations` -> Install automatically.

---

## Step 4: Set Up OWASP Dependency Check

1. Install OWASP Dependency Check in Jenkins:
   - `Manage Jenkins -> Tools -> Dependency-Check Installations` -> Install automatically.

---

## Step 5: Set Up Docker for Jenkins

1. Install Docker in Jenkins:
   - `Manage Jenkins -> Tools -> Docker Installations` -> Install automatically.
2. Add Docker credentials in Jenkins:
   - Navigate to `Manage Jenkins -> Credentials -> System -> Global Credentials`.
   - Add your DockerHub username and password.

---

## Step 6: Create Jenkins Pipeline

Create a pipeline in Jenkins to build and push the Dockerized image securely using multiple security tools.

### Pipeline Code:
```groovy
pipeline {
    agent any
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/vuxuanthanh1203/tmovie.git'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=tmovie \
                    -Dsonar.projectKey=tmovie'''
                }
            }
        }
        stage('OWASP FS Scan') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'OWASP DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('Trivy FS Scan') {
            steps {
                script {
                    try {
                        sh "trivy fs . > trivyfs.txt"
                    } catch(Exception e) {
                        input(message: "Are you sure to proceed?", ok: "Proceed")
                    }
                }
            }
        }
        stage('Docker Build') {
            steps {
                sh "docker build -t tmovie ."
            }
        }
        stage('Trivy Image Scan') {
            steps {
                sh "trivy image tmovie > trivyimage.txt"
                script {
                    input(message: "Are you sure to proceed?", ok: "Proceed")
                }
            }
        }
        stage('Docker Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {   
                        sh "docker tag tmovie xthanh12/tmovie:latest"
                        sh "docker push xthanh12/tmovie:latest"
                    }
                }
            }
        }
    }
}
```

---

## Step 7: Create an EKS Cluster

Use **Terraform** to create an EKS cluster. Ensure `kubectl` and `helm` are installed before proceeding. Example Terraform configuration can include VPC, IAM roles, and EKS node groups.

---

## Step 8: Deploy Prometheus and Grafana

### Steps:
1. Update kubeconfig:
   ```bash
   aws eks update-kubeconfig --name "Cluster-Name" --region "Region"
   ```
2. Add Helm repositories:
   ```bash
   helm repo add stable https://charts.helm.sh/stable
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   ```
3. Create Prometheus namespace:
   ```bash
   kubectl create namespace prometheus
   ```
4. Install Prometheus:
   ```bash
   helm install stable prometheus-community/kube-prometheus-stack -n prometheus
   ```
5. Update Prometheus and Grafana services to LoadBalancer:
   ```bash
   kubectl edit svc stable-kube-prometheus-sta-prometheus -n prometheus
   kubectl edit svc stable-grafana -n prometheus
   ```

---

## Step 9: Deploy ArgoCD

### Steps:
1. Create namespace:
   ```bash
   kubectl create namespace argocd
   ```
2. Install ArgoCD:
   ```bash
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.4.7/manifests/install.yaml
   ```
3. Expose ArgoCD server via LoadBalancer:
   ```bash
   kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
   ```
4. Get LoadBalancer hostname:
   ```bash
   kubectl get svc argocd-server -n argocd -o json
   ```
5. Retrieve default admin password:
   ```bash
   kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
   ```

Access the ArgoCD dashboard and log in with the credentials above.
