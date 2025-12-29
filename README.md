# iMeetPro Infrastructure & CI/CD Setup

## 📋 Overview

Complete infrastructure setup for iMeetPro video conferencing platform using:
- **Infrastructure as Code**: Terraform
- **Container Orchestration**: AWS EKS (Kubernetes)
- **CI/CD**: Jenkins
- **Container Registry**: AWS ECR
- **Monitoring**: Prometheus + Grafana + Loki

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              AWS Cloud (ap-south-1)                          │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │                           VPC (10.0.0.0/16)                             │ │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐         │ │
│  │  │ Public Subnet   │  │ Private Subnet  │  │ Database Subnet │         │ │
│  │  │ 10.0.101.0/24   │  │ 10.0.1.0/24     │  │ 10.0.201.0/24   │         │ │
│  │  │                 │  │                 │  │                 │         │ │
│  │  │ ┌─────────────┐ │  │ ┌─────────────┐ │  │ ┌─────────────┐ │         │ │
│  │  │ │     ALB     │ │  │ │  EKS Nodes  │ │  │ │  RDS MySQL  │ │         │ │
│  │  │ └─────────────┘ │  │ └─────────────┘ │  │ └─────────────┘ │         │ │
│  │  │ ┌─────────────┐ │  │ ┌─────────────┐ │  │                 │         │ │
│  │  │ │ NAT Gateway │ │  │ │  GPU Nodes  │ │  │                 │         │ │
│  │  │ └─────────────┘ │  │ └─────────────┘ │  │                 │         │ │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘         │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │     ECR      │  │      S3      │  │  CloudFront  │  │   Route 53   │     │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘     │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                            EKS Cluster                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │                        Namespace: imeetpro                              │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐                │ │
│  │  │ Frontend │  │ Backend  │  │  Celery  │  │   GPU    │                │ │
│  │  │ (React)  │  │ (Django) │  │ Workers  │  │ Workers  │                │ │
│  │  │  x3 pods │  │  x3 pods │  │  x2 pods │  │  x2 pods │                │ │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘                │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │                       Namespace: databases                              │ │
│  │  ┌──────────────────────┐  ┌──────────────────────┐                    │ │
│  │  │      MongoDB         │  │        Redis         │                    │ │
│  │  │   (StatefulSet x3)   │  │   (StatefulSet x3)   │                    │ │
│  │  └──────────────────────┘  └──────────────────────┘                    │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │                       Namespace: monitoring                             │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐            │ │
│  │  │Prometheus│  │ Grafana  │  │   Loki   │  │ AlertManager │            │ │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────────┘            │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 📁 Project Structure

```
imeetpro-infra/
├── terraform/                    # Infrastructure as Code
│   ├── modules/
│   │   ├── vpc/                 # VPC, Subnets, NAT
│   │   ├── eks/                 # EKS Cluster
│   │   ├── ecr/                 # Container Registry
│   │   ├── rds/                 # MySQL Database
│   │   ├── s3/                  # S3 Buckets
│   │   ├── cloudfront/          # CDN
│   │   └── security/            # Security Groups, WAF
│   ├── environments/
│   │   ├── dev/
│   │   ├── staging/
│   │   └── prod/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── providers.tf
│
├── jenkins/                      # CI/CD Configuration
│   ├── pipelines/
│   │   └── Jenkinsfile          # Main CI/CD Pipeline
│   ├── scripts/
│   │   ├── install-jenkins.sh   # Jenkins Installation
│   │   └── setup-credentials.sh # Credentials Setup
│   └── docker/
│       ├── Dockerfile.frontend
│       ├── Dockerfile.backend
│       ├── Dockerfile.celery
│       ├── Dockerfile.gpu-worker
│       └── nginx/
│
├── kubernetes/                   # Kubernetes Manifests
│   ├── namespaces/
│   ├── apps/
│   │   ├── frontend/
│   │   ├── backend/
│   │   ├── celery/
│   │   └── gpu-workers/
│   ├── databases/
│   │   ├── mongodb/
│   │   └── redis/
│   ├── monitoring/
│   ├── ingress/
│   └── secrets/
│
└── scripts/
    └── deploy.sh                # Master Deployment Script
```

---

## 🚀 Quick Start

### Prerequisites

```bash
# Install required tools
# AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install

# Terraform
wget https://releases.hashicorp.com/terraform/1.6.0/terraform_1.6.0_linux_amd64.zip
unzip terraform_1.6.0_linux_amd64.zip && sudo mv terraform /usr/local/bin/

# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### Configure AWS

```bash
aws configure
# Enter your AWS Access Key ID
# Enter your AWS Secret Access Key
# Default region: ap-south-1
# Default output format: json
```

### Deploy Infrastructure

```bash
# Clone the repository
git clone https://github.com/your-org/imeetpro-infra.git
cd imeetpro-infra

# Set environment variables
export AWS_REGION=ap-south-1
export ENVIRONMENT=prod

# Run deployment
chmod +x scripts/deploy.sh
./scripts/deploy.sh all
```

---

## 🔧 Terraform Commands

```bash
cd terraform

# Initialize
terraform init

# Plan
terraform plan -var-file="environments/prod/terraform.tfvars"

# Apply
terraform apply -var-file="environments/prod/terraform.tfvars"

# Destroy (careful!)
terraform destroy -var-file="environments/prod/terraform.tfvars"

# Show outputs
terraform output
```

---

## 🔄 CI/CD Pipeline Stages

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   Checkout   │───▶│  Code Scan   │───▶│    Build     │───▶│Security Scan │
│   (GitHub)   │    │ (SonarQube)  │    │  (Docker)    │    │   (Trivy)    │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
                                                                    │
                                                                    ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ Smoke Tests  │◀───│   Deploy     │◀───│  Push ECR    │◀───│   Unit Test  │
│              │    │   (EKS)      │    │              │    │              │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
```

### Pipeline Features

| Stage | Tool | Description |
|-------|------|-------------|
| Checkout | Git | Pull code from GitHub |
| Code Quality | SonarQube | Static code analysis |
| Unit Tests | Jest/Pytest | Run unit tests |
| Build | Docker | Build container images |
| Security Scan | Trivy | Vulnerability scanning |
| Push | ECR | Push images to registry |
| Deploy | kubectl | Deploy to EKS |
| Smoke Tests | curl | Verify deployment |

---

## 🖥️ Jenkins Setup

### Install Jenkins

```bash
# On EC2 instance (t3.large recommended)
chmod +x jenkins/scripts/install-jenkins.sh
./jenkins/scripts/install-jenkins.sh
```

### Required Plugins

- Docker Pipeline
- Kubernetes
- AWS Steps
- Pipeline: AWS Steps
- SonarQube Scanner
- Slack Notification
- Blue Ocean
- Git
- Credentials Binding

### Configure Credentials

1. Go to Jenkins → Manage Jenkins → Credentials
2. Add the following credentials:

| Credential ID | Type | Description |
|--------------|------|-------------|
| `aws-credentials` | Username/Password | AWS Access Key |
| `aws-account-id` | Secret Text | AWS Account ID |
| `github-credentials` | Username/Password | GitHub Token |
| `sonar-token` | Secret Text | SonarQube Token |
| `slack-token` | Secret Text | Slack Webhook |

---

## 📊 Monitoring

### Access Grafana

```bash
# Port forward
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring

# Access at http://localhost:3000
# Username: admin
# Password: kubectl get secret -n monitoring monitoring-grafana -o jsonpath="{.data.admin-password}" | base64 --decode
```

### Pre-configured Dashboards

| Dashboard ID | Name |
|-------------|------|
| 1860 | Node Exporter |
| 2583 | MongoDB |
| 763 | Redis |
| 9528 | Django |
| 6417 | Kubernetes |

### Alert Rules

- High CPU (>80%)
- Low Memory (<20% available)
- Pod Crashes
- High Error Rate (>5%)
- Slow API Response (>2s)
- Database Connections

---

## 💰 Cost Estimation

| Component | Instance/Type | Monthly Cost |
|-----------|--------------|--------------|
| EKS Control Plane | - | $73 |
| EKS Nodes (3x t3.large) | t3.large | $180 |
| GPU Nodes (2x g4dn.xlarge) | g4dn.xlarge | $780 |
| RDS MySQL | db.t3.medium | $50 |
| NAT Gateway | - | $100 |
| S3 + CloudFront | - | $62 |
| Data Transfer | 10TB | $180 |
| LiveKit Cloud | - | $600 |
| **Total** | | **~$2,025/month** |

---

## 🔐 Security Features

- ✅ Private subnets for all workloads
- ✅ WAF protection
- ✅ SSL/TLS everywhere
- ✅ Network policies
- ✅ RBAC enabled
- ✅ Secrets encryption (KMS)
- ✅ Image vulnerability scanning
- ✅ Pod security standards

---

## 📞 Support

- **Documentation**: [docs.imeetpro.com](https://docs.imeetpro.com)
- **Issues**: [GitHub Issues](https://github.com/your-org/imeetpro-infra/issues)
- **Slack**: #imeetpro-devops

---

## 📝 License

Copyright © 2024 iMeetPro. All rights reserved.
