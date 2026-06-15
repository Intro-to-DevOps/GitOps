# 🛒 I. YAS (Yet Another Shop) - GitOps Repository

This repository contains the complete infrastructure configuration and deployment state for the YAS microservices project, managed according to the GitOps standard using ArgoCD and a Helm Umbrella Chart.

The goal of this repository is to provide a Single Source of Truth for the Kubernetes environments (Dev and Staging), making deployments, rollbacks, and system monitoring fully automated and transparent.

# 🏛️ II. Deployment Architecture (Umbrella Chart Pattern)

Instead of managing dozens of isolated Applications on ArgoCD for each microservice (such as cart, customer, backoffice-bff, etc.), this project adopts the Umbrella Chart pattern.

In this model:

- A "parent" Helm Chart (Umbrella Chart) is created.

- All YAS microservices act as dependencies of this parent chart.

- The ArgoCD UI is streamlined, displaying a single Application cluster for each environment (e.g., yas-devand yas-staging), which contains a tree view of all the nested services.

# III. 📂 Directory Structure
```
.
|---charts/                      # Contains the tgz files that are build from helm
│   
│
|---dev/                         # Dev environment
│   ├── application.yaml         # Dev Application for argocd 
│   ├── values.yaml              # Value for ingress
| 
|---templates/
|   ├── _helpers.tpl      
|
|---Chart.lock
|---Chart.yaml
|---Jenkinsfile
|---values.yaml      
```

# IV. 🛠️ Initialization & Deployment Guide

## 1. System Requirements
- A running Kubernetes Cluster (AKS, Minikube, etc.).
- ArgoCD installed on the cluster (namespace: argocd).
- NGINX Ingress Controller (or equivalent) installed for domain resolution.

## 2. Installing YAS into the Cluster
You do not need to run the helm install command manually. Simply apply the root Application files into ArgoCD using the following commands:

For the DEV environment:
```bash
kubectl apply -f dev/application.yaml
```

For the STAGING environment:
```bash
kubectl apply -f staging/application.yaml
```

## 3. Deploy services

Go to [Chart.yaml](https://github.com/Intro-to-DevOps/GitOps/blob/main/Chart.yaml) in the root directory, uncomment services (each service contains 3 lines: name, version, repository), then run the command:
```bash
helm dependency update
```
This will update chart build, then commit and push changes. Argocd will detect and deploy new services that are uncommented

Note: You should uncomment 1-2 service(s) and commit before go to the next one, since deploy so many services at the same time will cause cpu overload. After that, check log to fix erros if any


