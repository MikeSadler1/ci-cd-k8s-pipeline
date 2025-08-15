# CI/CD Pipeline with Jenkins, SonarQube, Docker, Kubernetes, and Terraform

## What this does
- Builds a Docker image for a sample app
- Runs code quality scan with SonarQube
- Pushes the image to Docker Hub
- Deploys to Kubernetes (Minikube by default)
- (Optional) Provisions Jenkins EC2 + CloudWatch alarm via Terraform

## Tech
Jenkins • SonarQube • Docker • Kubernetes • Minikube • Terraform • Node.js

## Quickstart
1) Start Docker Desktop
2) `minikube start`
3) Run SonarQube: docker run -d --name sonarqube -p 9000:9000 sonarqube:lts-community
4) Run Jenkins: see Jenkins section in this README
5) Add Jenkins credentials: `DOCKERHUB_CREDS`, `SONAR_TOKEN`
6) Create Pipeline in Jenkins pointing to this repo, then **Build Now**

## Kubernetes
Apply: `kubectl apply -f k8s/`  
Check: `kubectl get pods -w`  
Service URL: `minikube service cicd-pipeline --url`

## Security Notes
- No secrets in repo
- Docker Hub creds & Sonar token stored in Jenkins Credentials
- Resource requests/limits & probes set in Deployment

## Troubleshooting
- Docker not running? Start Docker Desktop
- Pods pending? `kubectl describe pod <pod>`
- Kubeconfig for Jenkins: copy `~/.kube/config` into Jenkins home
- Sonar token invalid? Regenerate in SonarQube and update Jenkins credential
