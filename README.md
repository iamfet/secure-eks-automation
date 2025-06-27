# Secure EKS Automation

A comprehensive Infrastructure as Code (IaC) solution for deploying a secure Amazon EKS cluster with automated CI/CD pipelines, role-based access control (RBAC), and GitOps capabilities using ArgoCD.

## 🏗️ Architecture Overview

This project provisions:
- **Amazon EKS Cluster** with managed node groups
- **VPC with public/private subnets** across multiple AZs
- **IAM roles and policies** for secure access control
- **Kubernetes RBAC** for namespace-level permissions
- **ArgoCD** for GitOps deployment workflows
- **OPA Gatekeeper** for policy enforcement
- **AWS Load Balancer Controller** for ingress management
- **Cluster Autoscaler** for automatic node scaling
- **Metrics Server** for resource monitoring

## 📋 Prerequisites

- AWS CLI configured with appropriate permissions
- Terraform >= 1.12.0
- kubectl
- Helm (for ArgoCD deployment)
- GitHub repository with Actions enabled

## 🚀 Quick Start

### 1. Bootstrap Backend

First, create the S3 backend for Terraform state management:

```bash
cd backend
terraform init
terraform plan
terraform apply
```

### 2. Configure Variables

Create a `terraform.tfvars` file or set environment variables:

```hcl
project_name = "your-project-name"
aws_region = "us-east-1"
user_for_admin_role = "arn:aws:iam::ACCOUNT-ID:user/admin-user"
user_for_dev_role = "arn:aws:iam::ACCOUNT-ID:user/dev-user"
```

### 3. Deploy Infrastructure

```bash
terraform init
terraform plan
terraform apply
```

### 4. Configure kubectl

```bash
aws eks update-kubeconfig --name <cluster-name> --region <aws-region>
```

## 📁 Project Structure

```
├── .github/workflows/              # GitHub Actions CI/CD pipelines
│   ├── bootstrap-backend.yaml      # Bootstrap S3 backend
│   ├── deploy-infrastructure.yaml  # Main deployment pipeline
│   └── destroy-infrastructure.yaml # Infrastructure cleanup
├── backend/                        # Terraform backend configuration
│   ├── main.tf                     # S3 bucket for state storage
│   └── outputs.tf                  # Backend outputs
├── main.tf                         # Main infrastructure resources
├── variables.tf                    # Input variables
├── outputs.tf                      # Output values
├── providers.tf                    # Provider configurations
├── iam-roles.tf                    # IAM roles and policies
├── kube-RBAC.tf                    # Kubernetes RBAC configuration
├── argocd.tf                       # ArgoCD Helm deployment
└── opa-gatekeeper.tf               # OPA Gatekeeper policy engine
```

## 🔧 Configuration

### Configuration Variables

#### Required Variables
- **`user_for_admin_role`** - ARN of AWS user for admin role
- **`user_for_dev_role`** - ARN of AWS user for developer role

#### Optional Variables (with defaults)
- **`project_name`** - Name prefix for resources (default: `fiifi`)
- **`aws_region`** - AWS region (default: `us-east-1`)
- **`cluster_version`** - EKS cluster version (default: `1.32`)
- **`vpc_cidr_block`** - VPC CIDR block (default: `10.0.0.0/16`)
- **`private_subnets_cidr`** - Private subnet CIDRs (default: `["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]`)
- **`public_subnets_cidr`** - Public subnet CIDRs (default: `["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]`)

### GitHub Configuration

#### Required Secrets
- **`AWS_ROLE_ARN`** - IAM role ARN for GitHub Actions
- **`ADMIN_USER_ARN`** - ARN of the admin user
- **`DEV_USER_ARN`** - ARN of the developer user

#### Required Variables
- **`AWS_REGION`** - AWS region for deployment

## 🔐 Security Features

### IAM Roles
- **external-admin**: Full cluster access with read-only EKS permissions
- **external-developer**: Namespace-scoped access to `online-boutique` namespace

### Kubernetes RBAC
- **cluster-viewer**: Cluster-wide read access for admins
- **namespace-viewer**: Namespace-scoped read access for developers

### Network Security
- Private subnets for worker nodes
- Public subnets for load balancers
- NAT Gateway for outbound internet access
- Security groups managed by EKS

## 🔄 CI/CD Workflows

### Bootstrap Backend
- **Trigger**: Manual workflow dispatch
- **Purpose**: Create S3 backend for Terraform state
- **Steps**: Init → Plan → Apply

### Deploy Infrastructure
- **Trigger**: Push to main branch or PR with `.tf` file changes
- **Steps**:
  1. **Validate**: Format check and validation
  2. **Security**: tfsec security scanning
  3. **Plan**: Generate and review execution plan
  4. **Apply**: Deploy changes (main branch only)
  5. **Verify**: Validate cluster deployment

### Destroy Infrastructure
- **Trigger**: Manual workflow dispatch with confirmation
- **Safety**: Requires typing "destroy" to confirm
- **Purpose**: Clean up all infrastructure resources

## 🛠️ Installed Add-ons

- **AWS Load Balancer Controller**: Manages ALB/NLB for services
- **Cluster Autoscaler**: Automatically scales worker nodes
- **Metrics Server**: Provides resource usage metrics
- **ArgoCD**: GitOps continuous deployment
- **OPA Gatekeeper**: Policy enforcement and admission control
- **CoreDNS**: Cluster DNS resolution
- **VPC CNI**: Pod networking
- **kube-proxy**: Network proxy

## 📊 Monitoring and Observability

### Metrics Server
Provides CPU and memory metrics for pods and nodes:
```bash
kubectl top nodes
kubectl top pods
```

### ArgoCD Access
Access ArgoCD UI via port-forward:
```bash
kubectl port-forward svc/argo-cd-argocd-server -n argocd 8080:443
```

### OPA Gatekeeper
Check policy enforcement status:
```bash
kubectl get pods -n gatekeeper-system
kubectl get constraints
```

## 🔍 Troubleshooting

### Common Issues

1. **Backend Access Denied**
   - Ensure AWS credentials have S3 access
   - Verify bucket exists and is accessible

2. **EKS Access Denied**
   - Check IAM roles and policies
   - Verify cluster access entries
   - Update kubeconfig

3. **Node Group Issues**
   - Check subnet availability
   - Verify instance type availability in AZ
   - Review security group rules

### Useful Commands

```bash
# Check cluster status
aws eks describe-cluster --name <cluster-name>

# Verify nodes
kubectl get nodes

# Check all pods
kubectl get pods --all-namespaces

# View ArgoCD pods
kubectl get pods -n argocd

# Check cluster autoscaler logs
kubectl logs -n kube-system deployment/cluster-autoscaler
```

## 🧹 Cleanup

### Automated Cleanup
Use the GitHub Actions destroy workflow:
1. Go to Actions → Terraform Destroy
2. Click "Run workflow"
3. Type "destroy" to confirm
4. Run the workflow

### Manual Cleanup
```bash
# Destroy main infrastructure
terraform destroy -var="user_for_admin_role=<arn>" -var="user_for_dev_role=<arn>"

# Destroy backend (optional)
cd backend
terraform destroy
```

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test thoroughly
5. Submit a pull request

## 📄 License

This project is licensed under the terms specified in the LICENSE file.

## 🆘 Support

For issues and questions:
1. Check the troubleshooting section
2. Review GitHub Issues
3. Create a new issue with detailed information

---

**Note**: This infrastructure creates AWS resources that incur costs. Monitor your AWS billing and clean up resources when not needed.