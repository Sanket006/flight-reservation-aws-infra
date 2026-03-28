# 🏗️ Three-Tier Infrastructure — Flight Reservation App

<div align="center">

![Terraform](https://img.shields.io/badge/Terraform-7B42BC?style=for-the-badge&logo=terraform&logoColor=white)
![AWS EKS](https://img.shields.io/badge/AWS_EKS-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)
![AWS RDS](https://img.shields.io/badge/AWS_RDS-527FFF?style=for-the-badge&logo=amazonrds&logoColor=white)
![AWS S3](https://img.shields.io/badge/AWS_S3-569A31?style=for-the-badge&logo=amazons3&logoColor=white)
![HCL](https://img.shields.io/badge/HCL-blueviolet?style=for-the-badge)

*Terraform IaC to provision the complete three-tier AWS infrastructure for the Flight Reservation System — EKS (backend), RDS MySQL (database), S3 static website (frontend)*

</div>

---

## 📌 Overview

This repository contains the **Terraform Infrastructure as Code** that provisions all AWS resources required to run the [Flight Reservation System](https://github.com/Sanket006/flight-reservation-app) across a three-tier architecture on AWS, region `ap-south-1`.

| Tier | Technology | AWS Service |
|---|---|---|
| **Tier 1 — Frontend** | React + Vite (static build) | S3 Static Website Hosting |
| **Tier 2 — Backend** | Spring Boot REST API | AWS EKS (Kubernetes) |
| **Tier 3 — Database** | MySQL 8.0 | AWS RDS |

---

## 🏗️ Architecture

```
  Browser
     │
     ▼
┌──────────────────────────────────────┐
│  AWS S3 — Static Website             │  ← Tier 1: React frontend
│  flight-reservation-s3-              │    Served via S3 website endpoint
│  frontend-bucket                     │    Public read via bucket policy
└──────────────────┬───────────────────┘
                   │ API calls (VITE_API_URL)
                   ▼
┌──────────────────────────────────────┐
│  AWS EKS Cluster                     │  ← Tier 2: Spring Boot backend
│  flight-reservation-app-cluster      │    2 × c7i-flex.large worker nodes
│  Region: ap-south-1                  │    IAM roles: cluster + node group
│  Default VPC                         │
└──────────────────┬───────────────────┘
                   │ JDBC :3306
                   ▼
┌──────────────────────────────────────┐
│  AWS RDS — MySQL 8.0                 │  ← Tier 3: Managed database
│  db.t3.micro                         │    20 GB storage (auto-scales to 100 GB)
│  Security group: inbound :3306       │    Default VPC subnets
└──────────────────────────────────────┘
```

---

## 📁 Repository Structure

```
flight-reservation-aws-infra/
│
├── main.tf                    # Root — calls eks, rds, and s3 modules
│
└── modules/
    ├── eks/
    │   ├── main.tf            # EKS cluster, managed node group, IAM roles & policies
    │   ├── variable.tf        # project, desired/max/min_nodes, node_instance_type
    │   └── output.tf
    │
    ├── rds/
    │   ├── main.tf            # RDS MySQL 8.0, security group, DB subnet group
    │   ├── variable.tf
    │   └── output.tf
    │
    └── s3/
        └── main.tf            # S3 bucket, static website config, public access policy
```

---

## ☁️ Resources Provisioned

### EKS Module (`modules/eks`)

| Resource | Details |
|---|---|
| `aws_iam_role` — cluster | `eks-cluster-role-1` with `AmazonEKSClusterPolicy` + `AmazonEKSVPCResourceController` |
| `aws_iam_role` — nodes | `eks-node-role-1` with `AmazonEKSWorkerNodePolicy` + `AmazonEKS_CNI_Policy` + `AmazonEC2ContainerRegistryReadOnly` |
| `aws_eks_cluster` | `flight-reservation-app-cluster` — uses default VPC subnets |
| `aws_eks_node_group` | `flight-reservation-app-node-group` — 2 × `c7i-flex.large` nodes |

### RDS Module (`modules/rds`)

| Resource | Details |
|---|---|
| `aws_db_instance` | MySQL 8.0, `db.t3.micro`, 20 GB storage (auto-scales to 100 GB), `skip_final_snapshot = true` |
| `aws_security_group` | `rds-sg` — allows inbound TCP `:3306` |
| `aws_db_subnet_group` | `default-db-subnet-group-1` — uses all default VPC subnets |

### S3 Module (`modules/s3`)

| Resource | Details |
|---|---|
| `aws_s3_bucket` | `flight-reservation-s3-frontend-bucket` — static website with `index.html` |
| `aws_s3_bucket_public_access_block` | Block public access disabled — required for website hosting |
| `aws_s3_bucket_policy` | Public `s3:GetObject` on all objects in the bucket |
| `output.website_endpoint` | Outputs the public S3 website URL |

---

## ⚙️ Root Module Configuration (`main.tf`)

```hcl
provider "aws" {
  region = "ap-south-1"
}

module "rds" {
  source = "./modules/rds"
}

module "eks" {
  source             = "./modules/eks"
  project            = "flight-reservation-app"
  desired_nodes      = 2
  max_nodes          = 2
  min_nodes          = 2
  node_instance_type = "c7i-flex.large"
}

module "s3" {
  source = "./modules/s3"
}
```

---

## 🚀 Getting Started

### Prerequisites

- [Terraform CLI](https://developer.hashicorp.com/terraform/install) (v1.0+)
- AWS CLI configured (`aws configure`)
- IAM permissions: `eks:*`, `rds:*`, `s3:*`, `iam:*`, `ec2:Describe*`

### Deploy

```bash
# 1. Clone the repo
git clone https://github.com/Sanket006/flight-reservation-aws-infra.git
cd flight-reservation-aws-infra

# 2. Initialise providers and modules
terraform init

# 3. Preview all resources that will be created
terraform plan

# 4. Apply (provisions EKS + RDS + S3 in one command)
terraform apply
```

### Connect kubectl to the EKS Cluster

```bash
aws eks update-kubeconfig \
  --region ap-south-1 \
  --name flight-reservation-app-cluster

# Verify nodes are ready
kubectl get nodes
```

### Upload React Frontend to S3

```bash
# Build the React app
cd ../flight-reservation-app/frontend
export VITE_API_URL=http://<your-eks-load-balancer-url>
npm install && npm run build

# Sync build output to S3
aws s3 sync dist/ s3://flight-reservation-s3-frontend-bucket --delete
```

### Get the S3 Website URL

```bash
terraform output website_endpoint
```

### Destroy All Resources

```bash
terraform destroy
```

---

## ⚠️ Security Improvements for Production

The current setup is designed for a **dev/learning environment**. Before using in production, address the following:

| Issue | File | Fix |
|---|---|---|
| Hardcoded DB password `Redhat123` | `modules/rds/main.tf` | Move to `var.db_password` — pass via `TF_VAR_db_password` env variable or AWS Secrets Manager |
| `publicly_accessible = true` on RDS | `modules/rds/main.tf` | Set to `false` — restrict access to EKS node security group only |
| RDS SG open to `0.0.0.0/0` on port 3306 | `modules/rds/main.tf` | Restrict `cidr_blocks` to EKS node group security group ID |
| S3 direct public access | `modules/s3/main.tf` | Use AWS CloudFront + Origin Access Control (OAC) instead of direct public bucket |
| No Terraform remote state | `main.tf` | Add S3 backend + DynamoDB state locking for team-safe usage |
| Default VPC used for all resources | `modules/eks`, `modules/rds` | Create a dedicated VPC with separate public/private subnets |

---

## 🔗 Related Repositories

| Repo | Description |
|---|---|
| [flight-reservation-app](https://github.com/Sanket006/flight-reservation-app) | Application source code — Spring Boot API + React frontend |
| [jenkins-cicd-pipelines](https://github.com/Sanket006/jenkins-cicd-pipelines) | Jenkins pipeline that builds and deploys to this infrastructure |
| [terraform-aws-iac](https://github.com/Sanket006/terraform-aws-iac) | Other reusable Terraform IaC modules and patterns |

---

## 👨‍💻 Author

**Sanket Ajay Chopade** — DevOps Engineer

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=flat&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/sanketchopade07)
[![GitHub](https://img.shields.io/badge/GitHub-181717?style=flat&logo=github&logoColor=white)](https://github.com/Sanket006)
