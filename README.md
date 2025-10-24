# DevOps Engineer Assignment – Sample App (Flask + EKS + ECR + Helm)

This repository implements the full assignment:
- Simple Flask API (`/health`)
- Dockerized app
- CI/CD via GitHub Actions: build → push (ECR) → deploy (EKS) with Helm
- Kubernetes HPA auto-scaling (2–10 pods, 50% CPU target)
- Monitoring-ready with Prometheus scrape annotations (use kube-prometheus-stack)

## 1) Prerequisites

- AWS account with ECR and EKS
- `aws` CLI authenticated (`aws configure`)
- EKS cluster available (e.g., via `eksctl` or Terraform)
- `kubectl` and `helm` installed locally (for manual ops)
- GitHub repo with the following **Secrets** configured:
  - `AWS_ACCESS_KEY_ID`
  - `AWS_SECRET_ACCESS_KEY`
  - `AWS_REGION` (e.g., `ap-south-1`)
  - `ECR_REPOSITORY` (e.g., `sample-app`)
  - `EKS_CLUSTER_NAME` (your EKS cluster name)

### Create an ECR repository
```bash
aws ecr create-repository --repository-name sample-app
```

## 2) Run locally (optional)

```bash
pip install -r requirements.txt
gunicorn -b 0.0.0.0:8080 app:app
# open http://localhost:8080/health
```

## 3) CI/CD Pipeline (GitHub Actions)

- On push to `main`:
  1. Lint + test
  2. Build container
  3. Push to ECR
  4. Deploy to EKS using Helm

Workflow file: `.github/workflows/pipeline.yml`

## 4) Kubernetes Deployment (Helm)

Values defaults:
- Service: `LoadBalancer` (port 80 → 8080)
- Resources: small limits/requests
- Autoscaling:
  - `minReplicas: 2`
  - `maxReplicas: 10`
  - `targetCPUUtilizationPercentage: 50`

Install/upgrade manually:
```bash
# Set your ECR repo and image tag
export AWS_REGION=ap-south-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export ECR_REPOSITORY=sample-app
export IMAGE_TAG=latest
helm upgrade --install sample-app ./helm/sample-app   --set image.repository=$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPOSITORY   --set image.tag=$IMAGE_TAG
```

## 5) Enable Metrics & HPA

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
# HPA is installed by the chart when autoscaling.enabled=true
kubectl get hpa
```

Generate load to demonstrate scaling:
```bash
kubectl run -i --tty load-generator --image=busybox --restart=Never -- /bin/sh -c   'while true; do wget -q -O- http://sample-app.default.svc.cluster.local:8080/health > /dev/null; done'
```

## 6) Monitoring & Alerting

Install kube-prometheus-stack:
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install monitoring prometheus-community/kube-prometheus-stack
```

Access Grafana:
```bash
kubectl port-forward svc/monitoring-grafana 3000:80
# Open http://localhost:3000 (user: admin / pass: prom-operator)
```

Dashboards:
- Nodes / Pods / Namespace
- Add Prometheus target discovery (the app pods expose `/health` and are annotated)

## 7) Deliverables Checklist

- [x] App code + Dockerfile
- [x] Helm chart (Service, Deployment, HPA)
- [x] CI/CD workflow
- [x] README setup steps
- [x] Commands to test HPA
- [x] Monitoring instructions

## 8) Notes

- The workflow uses `yq` to update the Helm values at deploy time. If it is not found on the GitHub runner, add an installation step:
```yaml
- name: Install yq
  run: sudo snap install yq
```
- For cost control, consider EKS Fargate or a single small node group and shut down resources after the demo.