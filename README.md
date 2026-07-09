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

# V. 🔐 Service Mesh (Istio) Setup Guide

This section covers enabling Istio service mesh (mTLS, AuthorizationPolicy, Retry policy) on top of the YAS deployment described above.

## 1. Install Istio control plane
```bash
istioctl x precheck
istioctl install --set profile=default -y
```
Verify `istiod`, `istio-ingressgateway`, `istio-egressgateway` are `Running`:
```bash
kubectl get pods -n istio-system
```

## 2. Enable sidecar injection
Do this **before** deploying/restarting workloads to avoid unnecessary rollout churn:
```bash
kubectl label namespace yas-dev istio-injection=enabled
kubectl label namespace yas-staging istio-injection=enabled
```

## 3. Install Kiali + Prometheus addon
Kiali requires its own lightweight Prometheus instance to compute the traffic graph (separate from the full observability stack, which is not required for this project):
```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.30/samples/addons/prometheus.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.30/samples/addons/kiali.yaml
```
Access the dashboard:
```bash
kubectl port-forward -n istio-system svc/kiali 20001:20001
```
Open `http://localhost:20001` → Graph → namespace `yas-dev` → App graph → enable Display > Security to see mTLS lock icons on edges.

## 4. Apply the Istio policies (mTLS, AuthorizationPolicy, Retry)
These manifests live in `dev/istio/` and `staging/istio/`, and can be applied directly or via the ArgoCD Applications below:
```bash
kubectl apply -f dev/istio/peer-authentication.yaml
kubectl apply -f dev/istio/destination-rule.yaml
kubectl apply -f dev/istio/authorization-policy.yaml
kubectl apply -f dev/istio/virtual-service.yaml
```
Or through ArgoCD (self-heal + auto-sync from Git):
```bash
kubectl apply -f dev/istio/application.yaml
kubectl apply -f staging/istio/application.yaml
```

## 5. Restart existing workloads to pick up the sidecar
If services were already running before injection was enabled, restart them **in small batches** (not all at once — restarting every Deployment simultaneously has been observed to overload small clusters' CPU/RAM and cause cascading crash loops):
```bash
kubectl rollout restart deployment/<name> -n yas-dev
```
Confirm injection succeeded — the `READY` column should read `2/2` (app container + `istio-proxy`), not `1/1`:
```bash
kubectl get pods -n yas-dev
```

## 6. Verify mTLS / AuthorizationPolicy / Retry behavior
```bash
# mTLS: request from a pod without a sidecar should be rejected (connection reset)
kubectl run no-sidecar-test -n yas-dev --image=nicolaka/netshoot \
  --overrides='{"metadata":{"annotations":{"sidecar.istio.io/inject":"false"}}}' \
  -it --rm --restart=Never -- curl -sv --max-time 5 http://cart.yas-dev.svc.cluster.local

# AuthorizationPolicy: identity not in the allow-internal whitelist should get 403
kubectl create serviceaccount test-outsider -n yas-dev
kubectl run curl-test-outsider -n yas-dev --image=nicolaka/netshoot \
  --overrides='{"spec":{"serviceAccountName":"test-outsider"}}' \
  -it --rm --restart=Never -- curl -sv --max-time 5 http://cart.yas-dev.svc.cluster.local
```
