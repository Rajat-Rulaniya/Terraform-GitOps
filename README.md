# Terraform GitOps — AWS EKS Infrastructure with GitHub Actions

GitOps-driven infrastructure provisioning using **Terraform** and **GitHub Actions**. Automatically provisions an **AWS VPC** and **EKS cluster** with a branch-based workflow — `stage` branch validates and plans, `main` branch applies and deploys the **NGINX Ingress Controller**.

![Terraform](https://img.shields.io/badge/IaC-Terraform-7B42BC?style=flat&logo=terraform&logoColor=white)
![AWS EKS](https://img.shields.io/badge/Orchestration-AWS_EKS-FF9900?style=flat&logo=amazoneks&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitOps-GitHub_Actions-2088FF?style=flat&logo=githubactions&logoColor=white)
![AWS VPC](https://img.shields.io/badge/Network-AWS_VPC-FF9900?style=flat&logo=amazonwebservices&logoColor=white)

---

## GitOps Workflow — Branch-Based Strategy

Infrastructure changes follow a **GitOps promotion model** via two GitHub Actions workflows that trigger only when files under `terraform/` are modified:

### `stage` Branch → Validate & Plan Only

Workflow: [`.github/workflows/stage.yml`](.github/workflows/stage.yml)

| Step | What It Does |
|------|-------------|
| **Terraform Init** | Initializes with S3 backend (bucket name from GitHub Secrets) |
| **Validate** | Checks Terraform configuration syntax |
| **Format Check** | Enforces consistent code formatting (`terraform fmt -check`) |
| **Plan** | Generates an execution plan — **does NOT apply** |
| **Plan Status Check** | Fails the workflow if the plan has errors |

Push to `stage` → review the plan output → merge to `main` when satisfied.

### `main` Branch → Apply & Configure Cluster

Workflow: [`.github/workflows/main.yml`](.github/workflows/main.yml)

| Step | What It Does |
|------|-------------|
| **Terraform Init** | Initializes with S3 backend |
| **Validate** | Checks configuration syntax |
| **Plan** | Generates execution plan and saves to `planfile` |
| **Apply** | Applies infrastructure changes with `parallelism=1` (sequential resource creation to avoid dependency issues) |
| **Configure AWS Credentials** | Authenticates to AWS for kubectl/eksctl operations |
| **Install kubectl & eksctl** | Installs Kubernetes CLI tools on the runner |
| **Update Kubeconfig** | Connects to the newly created EKS cluster |
| **Install NGINX Ingress Controller** | Deploys the Ingress Controller (v1.12.0) to the cluster |

---

## Infrastructure — What Gets Provisioned

### VPC ([`vpc.tf`](terraform/vpc.tf))

Uses the [`terraform-aws-modules/vpc`](https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws) module (v5.1.2):

| Resource | Details |
|----------|---------|
| **VPC CIDR** | `172.20.0.0/16` |
| **Availability Zones** | 3 AZs (dynamically selected) |
| **Public Subnets** | `172.20.4.0/24`, `172.20.5.0/24`, `172.20.6.0/24` |
| **Private Subnets** | `172.20.1.0/24`, `172.20.2.0/24`, `172.20.3.0/24` |
| **NAT Gateway** | Single NAT Gateway (cost-optimized) |
| **DNS Hostnames** | Enabled |
| **Subnet Tags** | Kubernetes-aware tags for ELB/internal-ELB discovery |

### EKS Cluster ([`eks-cluster.tf`](terraform/eks-cluster.tf))

Uses the [`terraform-aws-modules/eks`](https://registry.terraform.io/modules/terraform-aws-modules/eks/aws) module (v20.31):

| Setting | Value |
|---------|-------|
| **Cluster Name** | `vprofile-eks` |
| **Kubernetes Version** | 1.30 |
| **Endpoint Access** | Public |
| **Node AMI** | Amazon Linux 2 (x86_64) |
| **Network** | Private subnets (worker nodes in private subnets) |

**Managed Node Groups:**

| Node Group | Instance Type | Min | Max | Desired |
|------------|--------------|-----|-----|---------|
| `node-group-1` | `t3.small` | 1 | 3 | 1 |
| `node-group-2` | `t3.small` | 1 | 2 | 1 |

### Post-Provisioning

After EKS is up, the `main` workflow automatically:
- Updates the local kubeconfig to connect to the cluster
- Installs the **NGINX Ingress Controller** (v1.12.0) for external traffic routing

---

## Terraform State Management

| Setting | Value |
|---------|-------|
| **Backend** | AWS S3 |
| **Bucket** | `rajat-tf-state` (overridable via `BUCKET_TF_STATE` secret) |
| **State Key** | `terraform/cloud-tf.state` |
| **Region** | `us-east-1` |

The S3 backend ensures **remote state locking** and enables team collaboration without state conflicts.

---

## Secrets Management

All sensitive values are stored as **GitHub Secrets**:

| Secret | Purpose |
|--------|---------|
| `AWS_USER_ACCESS_ID` | AWS IAM access key ID |
| `AWS_USER_SECRET_KEY` | AWS IAM secret access key |
| `BUCKET_TF_STATE` | S3 bucket name for Terraform state (overrides default in `terraform.tf`) |

---

## Project Structure

```
├── .github/workflows/
│   ├── main.yml              # Apply workflow (main branch → provision + configure)
│   └── stage.yml             # Plan-only workflow (stage branch → validate + plan)
├── terraform/
│   ├── terraform.tf          # Backend config (S3) + required providers
│   ├── main.tf               # AWS & Kubernetes provider config
│   ├── vpc.tf                # VPC module (3-AZ, public/private subnets, NAT GW)
│   ├── eks-cluster.tf        # EKS module (cluster + 2 managed node groups)
│   ├── variables.tf          # Input variables (region, cluster name)
│   └── outputs.tf            # Outputs (cluster name, endpoint, region, SG ID)
└── .gitignore
```

---

## Tools & Technologies

| Category | Tools |
|----------|-------|
| **Infrastructure as Code** | Terraform (>= 1.10.5) |
| **GitOps / CI/CD** | GitHub Actions (branch-based promotion) |
| **Container Orchestration** | AWS EKS (Kubernetes 1.30) |
| **Networking** | AWS VPC (3-AZ, NAT Gateway, public/private subnets) |
| **Ingress** | NGINX Ingress Controller (v1.12.0) |
| **State Management** | AWS S3 (remote backend) |
| **Cloud Provider** | AWS (EKS, VPC, EC2, S3, IAM) |
| **Terraform Modules** | `terraform-aws-modules/vpc` (v5.1.2), `terraform-aws-modules/eks` (v20.31) |

---

## How It All Comes Together

1. Developer creates infrastructure changes in the `terraform/` directory
2. Pushes to the **`stage`** branch → GitHub Actions runs **validate, format check, and plan**
3. Reviews the plan output in the GitHub Actions logs
4. Merges to **`main`** → GitHub Actions runs **plan + apply**
5. Terraform provisions the **VPC** (3-AZ, subnets, NAT GW) and **EKS cluster** (2 node groups)
6. The workflow **configures kubeconfig** and installs **kubectl/eksctl** on the runner
7. **NGINX Ingress Controller** is deployed to the EKS cluster — ready to route external traffic
8. All state is persisted to **S3** for team collaboration and drift detection

---
