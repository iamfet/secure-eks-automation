# Deployment Guide

## Prerequisites Setup

### 1. AWS Account Setup

Ensure you have:
- AWS account with appropriate permissions
- AWS CLI installed and configured
- IAM user with programmatic access

Required AWS permissions:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "eks:*",
        "ec2:*",
        "iam:*",
        "s3:*",
        "autoscaling:*",
        "elasticloadbalancing:*"
      ],
      "Resource": "*"
    }
  ]
}
```

### 2. Local Environment Setup

Install required tools:

```bash
# Terraform
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt-get update && sudo apt-get install terraform

# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Helm
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update && sudo apt-get install helm

# AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

### 3. GitHub Setup

Configure GitHub repository:

1. Fork or clone this repository
2. Set up GitHub Actions secrets:
   - `AWS_ROLE_ARN`: IAM role for GitHub Actions
   - `ADMIN_USER_ARN`: Admin user ARN
   - `DEV_USER_ARN`: Developer user ARN

3. Set up GitHub Actions variables:
   - `AWS_REGION`: Target AWS region

## Step-by-Step Deployment

### Step 1: Bootstrap Backend

The backend must be created first to store Terraform state.

#### Option A: Using GitHub Actions

1. Go to your repository's Actions tab
2. Select "Bootstrap Backend" workflow
3. Click "Run workflow"
4. Monitor the execution

#### Option B: Manual Bootstrap

```bash
cd backend
terraform init
terraform plan
terraform apply
```

Verify backend creation:
```bash
aws s3 ls s3://state-secure-eks-automation
```

### Step 2: Configure Variables

Create `terraform.tfvars` file:

```hcl
# Project Configuration
project_name = "my-secure-eks"
aws_region = "us-east-1"

# Network Configuration
vpc_cidr_block = "10.0.0.0/16"
private_subnets_cidr = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
public_subnets_cidr = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

# EKS Configuration
cluster_version = "1.32"

# IAM Configuration (Required)
user_for_admin_role = "arn:aws:iam::123456789012:user/admin-user"
user_for_dev_role = "arn:aws:iam::123456789012:user/dev-user"
```

### Step 3: Deploy Infrastructure

#### Option A: Using GitHub Actions (Recommended)

1. Commit and push your changes to the main branch
2. GitHub Actions will automatically:
   - Validate Terraform code
   - Run security scans
   - Generate execution plan
   - Apply changes (if on main branch)

#### Option B: Manual Deployment

```bash
# Initialize Terraform
terraform init

# Review the plan
terraform plan -var-file="terraform.tfvars"

# Apply the configuration
terraform apply -var-file="terraform.tfvars"
```

### Step 4: Configure kubectl

After successful deployment:

```bash
# Get cluster name from output
CLUSTER_NAME=$(terraform output -raw cluster_name)

# Update kubeconfig
aws eks update-kubeconfig --name $CLUSTER_NAME --region us-east-1

# Verify connection
kubectl get nodes
kubectl get pods --all-namespaces
```

### Step 5: Verify Deployment

Check all components are running:

```bash
# Check nodes
kubectl get nodes -o wide

# Check system pods
kubectl get pods -n kube-system

# Check ArgoCD
kubectl get pods -n argocd

# Check add-ons
kubectl get pods -n kube-system | grep -E "(aws-load-balancer|cluster-autoscaler|metrics-server)"
```

## Post-Deployment Configuration

### 1. Access ArgoCD

Get ArgoCD admin password:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Access ArgoCD UI:
```bash
kubectl port-forward svc/argo-cd-argocd-server -n argocd 8080:443
```

Open browser to `https://localhost:8080`
- Username: `admin`
- Password: (from previous command)

### 2. Configure Role-Based Access

#### For Admin Users

```bash
# Assume admin role
aws sts assume-role --role-arn arn:aws:iam::ACCOUNT:role/external-admin --role-session-name AdminSession

# Export credentials
export AWS_ACCESS_KEY_ID=<access-key>
export AWS_SECRET_ACCESS_KEY=<secret-key>
export AWS_SESSION_TOKEN=<session-token>

# Update kubeconfig
aws eks update-kubeconfig --name $CLUSTER_NAME --region us-east-1
```

#### For Developer Users

```bash
# Assume developer role
aws sts assume-role --role-arn arn:aws:iam::ACCOUNT:role/external-developer --role-session-name DevSession

# Export credentials and update kubeconfig
# (same as admin, but with limited access)
```

### 3. Deploy Sample Application

Create a sample application in the `online-boutique` namespace:

```bash
# Create deployment
kubectl create deployment nginx --image=nginx -n online-boutique

# Expose as service
kubectl expose deployment nginx --port=80 --type=ClusterIP -n online-boutique

# Verify deployment
kubectl get pods -n online-boutique
```

## Validation Checklist

- [ ] EKS cluster is running and accessible
- [ ] Worker nodes are in Ready state
- [ ] All system pods are running
- [ ] ArgoCD is accessible
- [ ] Load Balancer Controller is running
- [ ] Cluster Autoscaler is running
- [ ] Metrics Server is running
- [ ] RBAC permissions are working
- [ ] Sample application deploys successfully

## Troubleshooting Common Issues

### Backend Issues

**Error**: Backend bucket doesn't exist
```bash
# Solution: Run bootstrap first
cd backend && terraform apply
```

**Error**: Access denied to S3 bucket
```bash
# Solution: Check AWS credentials and permissions
aws sts get-caller-identity
aws s3 ls s3://state-secure-eks-automation
```

### EKS Cluster Issues

**Error**: Cluster creation timeout
```bash
# Check CloudFormation stacks
aws cloudformation list-stacks --stack-status-filter CREATE_IN_PROGRESS

# Check EKS cluster status
aws eks describe-cluster --name <cluster-name>
```

**Error**: Nodes not joining cluster
```bash
# Check node group status
aws eks describe-nodegroup --cluster-name <cluster-name> --nodegroup-name <nodegroup-name>

# Check Auto Scaling Group
aws autoscaling describe-auto-scaling-groups
```

### kubectl Access Issues

**Error**: Unable to connect to cluster
```bash
# Verify kubeconfig
kubectl config current-context

# Update kubeconfig
aws eks update-kubeconfig --name <cluster-name> --region <region>

# Check IAM permissions
aws eks describe-cluster --name <cluster-name>
```

**Error**: Forbidden access
```bash
# Check access entries
aws eks list-access-entries --cluster-name <cluster-name>

# Verify IAM role assumption
aws sts get-caller-identity
```

### Add-on Issues

**Error**: ArgoCD pods not starting
```bash
# Check ArgoCD namespace
kubectl get pods -n argocd

# Check events
kubectl get events -n argocd --sort-by='.lastTimestamp'

# Check Helm release
helm list -n argocd
```

**Error**: Load Balancer Controller not working
```bash
# Check controller pods
kubectl get pods -n kube-system | grep aws-load-balancer

# Check logs
kubectl logs -n kube-system deployment/aws-load-balancer-controller
```

## Performance Tuning

### Node Group Optimization

```hcl
# In main.tf, modify node group configuration
eks_managed_node_groups = {
  dev = {
    instance_types = ["m5.xlarge", "m5.large"]  # Mixed instance types
    capacity_type  = "SPOT"                     # Use spot instances
    min_size       = 1
    max_size       = 10                         # Increase max size
    desired_size   = 3                          # Increase desired size
    
    # Add taints for specific workloads
    taints = [
      {
        key    = "dedicated"
        value  = "gpu-workload"
        effect = "NO_SCHEDULE"
      }
    ]
  }
}
```

### Cluster Autoscaler Tuning

```hcl
# In main.tf, modify cluster autoscaler settings
cluster_autoscaler = {
  set = [
    {
      name  = "extraArgs.scale-down-delay-after-add"
      value = "10m"
    },
    {
      name  = "extraArgs.scale-down-unneeded-time"
      value = "10m"
    },
    {
      name  = "extraArgs.max-node-provision-time"
      value = "15m"
    }
  ]
}
```

## Security Hardening

### Network Policies

```yaml
# network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: online-boutique
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

### Pod Security Standards

```yaml
# pod-security-policy.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: online-boutique
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

## Monitoring Setup

### CloudWatch Container Insights

```bash
# Install CloudWatch agent
kubectl apply -f https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/cloudwatch-namespace.yaml

kubectl apply -f https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/cwagent/cwagent-daemonset.yaml
```

### Prometheus and Grafana

```bash
# Add Prometheus Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install Prometheus
helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
```

## Backup and Recovery

### Velero Setup

```bash
# Install Velero CLI
wget https://github.com/vmware-tanzu/velero/releases/download/v1.12.0/velero-v1.12.0-linux-amd64.tar.gz
tar -xzf velero-v1.12.0-linux-amd64.tar.gz
sudo mv velero-v1.12.0-linux-amd64/velero /usr/local/bin/

# Install Velero in cluster
velero install \
    --provider aws \
    --plugins velero/velero-plugin-for-aws:v1.8.0 \
    --bucket my-backup-bucket \
    --backup-location-config region=us-east-1 \
    --snapshot-location-config region=us-east-1
```

This deployment guide provides comprehensive instructions for setting up and configuring the secure EKS infrastructure. Follow each step carefully and refer to the troubleshooting section if you encounter any issues.