# Trend App DevOps Deployment

This repository contains the production-ready `dist/` build for the Trend frontend application. The project is deployed as a static web application using Nginx, Docker, DockerHub, Jenkins, AWS EKS, Kubernetes, Terraform, Helm, Prometheus, and Grafana.

The application is exposed on port `3000` through a Kubernetes `LoadBalancer` service.

## Architecture

```text
GitHub
  v
Jenkins Pipeline
  v
Docker Build
  v
DockerHub
  v
AWS EKS
  v
Kubernetes Deployment and Service
  v
AWS Load Balancer
  v
Browser
```

## Project Repository

```text
https://github.com/YuvarajRajadurai/Trend.git
```

## Application URL

```text
http://ada147c0a5a624b059fae83f2c0fc6a3-1533842508.ap-south-1.elb.amazonaws.com:3000/
```

## Technology Stack

- GitHub for source code management
- Docker for containerization
- DockerHub for image registry
- Terraform for Jenkins EC2 infrastructure
- Jenkins for CI/CD automation
- AWS EKS for Kubernetes
- kubectl for Kubernetes deployment
- Helm for package installation
- Prometheus and Grafana for monitoring
- Nginx for serving the static frontend

## Prerequisites

Install and verify the following tools before starting:

```bash
git --version
docker --version
aws --version
terraform version
kubectl version --client
eksctl version
helm version --short
java -version
```

Configure AWS CLI:

```bash
aws configure
```

Use the following AWS region:

```text
ap-south-1
```

Confirm the configured region:

```bash
aws configure get region
```

If needed, update it:

```bash
aws configure set region ap-south-1
```

## Repository Setup

Clone the repository:

```bash
git clone https://github.com/YuvarajRajadurai/Trend.git
cd Trend
```

Expected structure:

```text
Trend/
|-- dist/
|-- Dockerfile
|-- nginx.conf
|-- Jenkinsfile
|-- k8s/
|   |-- deployment.yaml
|   `-- service.yaml
`-- terraform/
    `-- main.tf
```

This repository contains a pre-built `dist/` folder. There is no `package.json`, so no Node.js build step is required.

## Docker Setup

Create `Dockerfile`:

```dockerfile
FROM nginx:1.27-alpine

COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY dist/ /usr/share/nginx/html/

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

Create `nginx.conf`:

```nginx
server {
    listen 80;
    server_name _;

    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
}
```

Create `.dockerignore`:

```dockerignore
.git
.gitignore
README.md
terraform
k8s
Jenkinsfile
*.log
node_modules
```

Build the Docker image:

```bash
docker build -t trend-app:1.0 .
```

Run the application locally on port `3000`:

```bash
docker run -d --name trend-app -p 3000:80 trend-app:1.0
```

Verify:

```bash
docker ps
curl http://localhost:3000
curl http://localhost:3000/health
```

Open in browser:

```text
http://localhost:3000
```

Stop and remove the local container:

```bash
docker stop trend-app
docker rm trend-app
```

## DockerHub Setup

DockerHub image used:

```text
yuvarajjr/trend-app
```

Login to DockerHub:

```bash
docker login
```

Tag the image:

```bash
docker tag trend-app:1.0 yuvarajjr/trend-app:1.0
docker tag trend-app:1.0 yuvarajjr/trend-app:latest
```

Push the image:

```bash
docker push yuvarajjr/trend-app:1.0
docker push yuvarajjr/trend-app:latest
```

## AWS EKS Cluster Setup

Cluster details:

```text
Cluster name: trend-cluster
Region: ap-south-1
Node group: trend-nodes
Node type: t3.medium
Nodes: 2
```

Create the EKS cluster:

```bash
eksctl create cluster \
  --name trend-cluster \
  --region ap-south-1 \
  --nodegroup-name trend-nodes \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed
```

Verify the cluster:

```bash
aws eks list-clusters --region ap-south-1
aws eks describe-cluster --name trend-cluster --region ap-south-1 --query "cluster.status"
```

Update kubeconfig:

```bash
aws eks update-kubeconfig --region ap-south-1 --name trend-cluster
```

Verify Kubernetes nodes:

```bash
kubectl get nodes
kubectl get pods -A
```

## Kubernetes Deployment

Create `k8s/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: trend-app
  labels:
    app: trend-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: trend-app
  template:
    metadata:
      labels:
        app: trend-app
    spec:
      containers:
        - name: trend-app
          image: yuvarajjr/trend-app:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
          readinessProbe:
            httpGet:
              path: /health
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 20
```

Create `k8s/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: trend-app-service
spec:
  type: LoadBalancer
  selector:
    app: trend-app
  ports:
    - protocol: TCP
      port: 3000
      targetPort: 80
```

Deploy to Kubernetes:

```bash
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
```

Check deployment:

```bash
kubectl get deployments
kubectl get pods
kubectl get svc
```

Check rollout status:

```bash
kubectl rollout status deployment/trend-app
```

## Application Health Check

Check service endpoint:

```bash
kubectl get svc trend-app-service
```

Application URL:

```text
http://ada147c0a5a624b059fae83f2c0fc6a3-1533842508.ap-south-1.elb.amazonaws.com:3000/
```

Health endpoint:

```bash
curl http://ada147c0a5a624b059fae83f2c0fc6a3-1533842508.ap-south-1.elb.amazonaws.com:3000/health
```

Expected output:

```text
healthy
```

Check pod status:

```bash
kubectl get pods -o wide
```

Check application logs:

```bash
kubectl logs -l app=trend-app
```

Describe deployment:

```bash
kubectl describe deployment trend-app
```

## Terraform Jenkins Infrastructure

Terraform is used to create the Jenkins EC2 instance and related AWS infrastructure.

Key pair used:

```text
trend-key
```

Check AWS key pair:

```bash
aws ec2 describe-key-pairs --region ap-south-1 --query "KeyPairs[*].KeyName" --output table
```

Move into the Terraform folder:

```bash
cd terraform
```

Initialize Terraform:

```bash
terraform init
```

Format and validate:

```bash
terraform fmt
terraform validate
```

Plan:

```bash
terraform plan -var="key_name=trend-key"
```

Apply:

```bash
terraform apply -var="key_name=trend-key"
```

Terraform output:

```text
jenkins_public_ip = "13.234.111.233"
jenkins_url       = "http://13.234.111.233:8080"
```

## Jenkins Setup

Open Jenkins:

```text
http://13.234.111.233:8080
```

Get the initial Jenkins password from the Jenkins EC2 server:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Install the following Jenkins plugins:

- Docker
- Docker Pipeline
- Git
- GitHub
- Kubernetes
- Pipeline

Add DockerHub credentials in Jenkins:

```text
Manage Jenkins
  -> Credentials
  -> System
  -> Global credentials
  -> Add Credentials
```

Credential values:

```text
Kind: Username with password
ID: dockerhub-creds
Username: DockerHub username
Password: DockerHub password or access token
```

## Jenkins Pipeline

Create `Jenkinsfile`:

```groovy
pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "yuvarajjr/trend-app"
        AWS_REGION = "ap-south-1"
        EKS_CLUSTER = "trend-cluster"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/YuvarajRajadurai/Trend.git'
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                docker build -t $DOCKER_IMAGE:$BUILD_NUMBER -t $DOCKER_IMAGE:latest .
                '''
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    docker push $DOCKER_IMAGE:$BUILD_NUMBER
                    docker push $DOCKER_IMAGE:latest
                    '''
                }
            }
        }

        stage('Deploy To EKS') {
            steps {
                sh '''
                aws eks update-kubeconfig --region $AWS_REGION --name $EKS_CLUSTER
                kubectl set image deployment/trend-app trend-app=$DOCKER_IMAGE:$BUILD_NUMBER
                kubectl rollout status deployment/trend-app
                kubectl get pods
                kubectl get svc trend-app-service
                '''
            }
        }
    }
}
```

## GitHub Webhook

In Jenkins pipeline job, enable:

```text
GitHub hook trigger for GITScm polling
```

In GitHub:

```text
Repository
  -> Settings
  -> Webhooks
  -> Add webhook
```

Webhook configuration:

```text
Payload URL: http://13.234.111.233:8080/github-webhook/
Content type: application/json
Events: Just the push event
```

After every push to GitHub, Jenkins will automatically trigger the pipeline.

## CI/CD Flow

The Jenkins pipeline performs the following steps:

1. Pulls the latest code from GitHub.
2. Builds the Docker image.
3. Tags the image with the Jenkins build number and `latest`.
4. Pushes the image to DockerHub.
5. Updates the EKS kubeconfig.
6. Updates the Kubernetes deployment image.
7. Waits for rollout completion.
8. Displays pod and service status.

Successful deployment output should include:

```text
deployment "trend-app" successfully rolled out
```

Example pod output:

```text
NAME                         READY   STATUS    RESTARTS   AGE
trend-app-675875dff6-fh6t4   1/1     Running   0          15s
trend-app-675875dff6-nsn28   1/1     Running   0          30s
```

## Monitoring Setup

Prometheus and Grafana are installed using Helm.

Add the Prometheus Helm repository:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

Install monitoring stack:

```bash
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace
```

Check monitoring pods:

```bash
kubectl get pods -n monitoring
```

Check monitoring services:

```bash
kubectl get svc -n monitoring
```

Access Grafana locally:

```bash
kubectl port-forward svc/monitoring-grafana 3001:80 -n monitoring
```

Open Grafana:

```text
http://localhost:3001
```

Default username:

```text
admin
```

Get Grafana password:

```bash
kubectl get secret monitoring-grafana -n monitoring \
  -o jsonpath="{.data.admin-password}" | base64 --decode
```

Useful Grafana dashboards:

- Kubernetes / Compute Resources / Cluster
- Kubernetes / Compute Resources / Namespace
- Kubernetes / Compute Resources / Pod
- Kubernetes / API server
- CoreDNS
- Grafana Overview

## Useful Commands

Check current Kubernetes context:

```bash
kubectl config current-context
```

List contexts:

```bash
kubectl config get-contexts
```

Update kubeconfig:

```bash
aws eks update-kubeconfig --region ap-south-1 --name trend-cluster
```

Check pods:

```bash
kubectl get pods
```

Check services:

```bash
kubectl get svc
```

Check deployment:

```bash
kubectl get deployment trend-app
```

Check rollout:

```bash
kubectl rollout status deployment/trend-app
```

Check logs:

```bash
kubectl logs -l app=trend-app
```

Restart deployment:

```bash
kubectl rollout restart deployment/trend-app
```

## Troubleshooting

### kubectl points to the wrong region

If `kubectl` tries to connect to an old cluster, update kubeconfig:

```bash
aws eks update-kubeconfig --region ap-south-1 --name trend-cluster
```

Then verify:

```bash
kubectl config current-context
```

### LoadBalancer external IP is pending

Wait a few minutes and check again:

```bash
kubectl get svc trend-app-service
```

### Pods are not running

Check pod details:

```bash
kubectl describe pods
kubectl logs -l app=trend-app
```

### Jenkins cannot run Docker

Add Jenkins user to the Docker group:

```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

### Jenkins cannot access EKS

Update kubeconfig on the Jenkins server:

```bash
aws eks update-kubeconfig --region ap-south-1 --name trend-cluster
kubectl get nodes
```

### DockerHub push fails

Check Jenkins credential ID:

```text
dockerhub-creds
```

Make sure the credential is a valid DockerHub username and access token.

## Cleanup

Delete Kubernetes resources:

```bash
kubectl delete -f k8s/service.yaml
kubectl delete -f k8s/deployment.yaml
```

Delete monitoring:

```bash
helm uninstall monitoring -n monitoring
kubectl delete namespace monitoring
```

Delete EKS cluster:

```bash
eksctl delete cluster --name trend-cluster --region ap-south-1
```

Destroy Terraform resources:

```bash
cd terraform
terraform destroy -var="key_name=trend-key"
```

## Final Result

The Trend application is containerized, pushed to DockerHub, deployed to AWS EKS through Jenkins CI/CD, exposed through a Kubernetes LoadBalancer on port `3000`, and monitored using Prometheus and Grafana.


