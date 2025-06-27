# Security Documentation

## Security Overview

This document outlines the security measures, best practices, and configurations implemented in the Secure EKS Automation project.

## Security Architecture

### Defense in Depth

The security model implements multiple layers of protection:

1. **Network Security**: VPC isolation, private subnets, security groups
2. **Identity & Access Management**: IAM roles, EKS access entries, RBAC
3. **Cluster Security**: EKS managed control plane, encrypted communication
4. **Application Security**: Namespace isolation, pod security standards
5. **Data Security**: Encryption at rest and in transit

## Network Security

### VPC Configuration

```hcl
# Private subnets for worker nodes
private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]

# Public subnets only for load balancers
public_subnets = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

# NAT Gateway for outbound internet access
enable_nat_gateway = true
single_nat_gateway = true  # Cost optimization - use multiple for HA
```

### Security Groups

EKS automatically manages security groups with these principles:
- Minimum required ports open
- Ingress restricted to necessary sources
- Egress controlled for worker nodes

### Network Policies (Recommended Addition)

```yaml
# Deny all traffic by default
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: online-boutique
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

---
# Allow specific ingress traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
  namespace: online-boutique
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080
```

## Identity and Access Management

### IAM Role Structure

#### External Admin Role

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "eks:DescribeCluster",
        "eks:ListClusters",
        "eks:AccessKubernetesApi"
      ],
      "Resource": "*"
    }
  ]
}
```

**Capabilities:**
- Full cluster read access
- Port forwarding for ArgoCD
- View all namespaces and resources

#### External Developer Role

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "eks:DescribeCluster",
        "eks:AccessKubernetesApi"
      ],
      "Resource": "*"
    }
  ]
}
```

**Capabilities:**
- Limited to `online-boutique` namespace
- Read-only access to specific resources
- Cannot access cluster-wide resources

### EKS Access Entries

```hcl
access_entries = {
  admin = {
    principal_arn = aws_iam_role.external-admin.arn
    username      = "admin"
    type          = "STANDARD"
    access_scope = {
      type = "cluster"  # Cluster-wide access
    }
  }

  developer = {
    principal_arn = aws_iam_role.external-developer.arn
    username      = "developer"
    type          = "STANDARD"
    access_scope = {
      type       = "namespace"
      namespaces = ["online-boutique"]  # Namespace-scoped
    }
  }
}
```

### Kubernetes RBAC

#### Cluster-Level Permissions

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-viewer
rules:
- apiGroups: [""]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods", "pods/portforward"]
  verbs: ["get", "list", "create"]  # For ArgoCD access
- apiGroups: ["apiextensions.k8s.io"]
  resources: ["customresourcedefinitions"]
  verbs: ["get", "list"]
```

#### Namespace-Level Permissions

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: online-boutique
  name: namespace-viewer
rules:
- apiGroups: [""]
  resources: ["pods", "services", "secrets", "configmaps", "persistentvolumeclaims"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "daemonsets", "statefulsets"]
  verbs: ["get", "list", "watch"]
```

## Cluster Security

### EKS Control Plane

- **Managed Service**: AWS manages control plane security
- **API Server**: Public endpoint with IP restrictions (configurable)
- **etcd**: Encrypted at rest by default
- **Audit Logging**: Available via CloudWatch (optional)

### Node Security

```hcl
# EKS Optimized AMI with latest security patches
eks_managed_node_groups = {
  dev = {
    ami_type       = "AL2_x86_64"  # Amazon Linux 2
    instance_types = ["m5.xlarge"]
    
    # Security configurations
    block_device_mappings = {
      xvda = {
        device_name = "/dev/xvda"
        ebs = {
          volume_size           = 50
          volume_type          = "gp3"
          encrypted            = true  # Encrypt EBS volumes
          delete_on_termination = true
        }
      }
    }
  }
}
```

### Container Runtime Security

- **containerd**: Secure container runtime
- **Image Scanning**: Integrate with ECR image scanning
- **Runtime Security**: Consider Falco for runtime threat detection

## Pod Security

### Pod Security Standards

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: online-boutique
  labels:
    # Enforce restricted pod security standard
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### Security Context Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: app
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1000
          capabilities:
            drop:
            - ALL
```

## Data Security

### Encryption at Rest

- **EBS Volumes**: Encrypted by default
- **S3 Backend**: AES256 encryption enabled
- **Secrets**: Kubernetes secrets (consider AWS Secrets Manager)

### Encryption in Transit

- **API Server**: TLS 1.2+ for all communications
- **Node Communication**: Encrypted via VPC CNI
- **Application Traffic**: Implement service mesh (Istio) for mTLS

### Secrets Management

#### Current Implementation
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: online-boutique
type: Opaque
data:
  password: <base64-encoded-value>
```

#### Recommended: AWS Secrets Manager Integration
```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-manager
  namespace: online-boutique
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        secretRef:
          accessKeyID:
            name: awssm-secret
            key: access-key
          secretAccessKey:
            name: awssm-secret
            key: secret-access-key
```

## Monitoring and Auditing

### Security Monitoring

#### CloudWatch Integration
```bash
# Enable EKS audit logs
aws eks update-cluster-config \
  --name my-cluster \
  --logging '{"enable":["api","audit","authenticator","controllerManager","scheduler"]}'
```

#### Security Scanning

```yaml
# GitHub Actions security scan
- name: Run tfsec
  uses: aquasecurity/tfsec-action@v1.0.3
  with:
    soft_fail: true

- name: Run Trivy vulnerability scanner
  uses: aquasecurity/trivy-action@master
  with:
    scan-type: 'fs'
    scan-ref: '.'
```

### Compliance Monitoring

#### CIS Kubernetes Benchmark
```bash
# Install kube-bench
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job-eks.yaml

# Check results
kubectl logs job/kube-bench
```

#### Falco Runtime Security
```bash
# Install Falco
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco falcosecurity/falco \
  --set falco.grpc.enabled=true \
  --set falco.grpcOutput.enabled=true
```

## Incident Response

### Security Incident Playbook

1. **Detection**
   - Monitor CloudWatch logs
   - Review Falco alerts
   - Check unusual API activity

2. **Containment**
   - Isolate affected nodes
   - Revoke compromised credentials
   - Apply network policies

3. **Investigation**
   - Analyze audit logs
   - Review access patterns
   - Check for lateral movement

4. **Recovery**
   - Patch vulnerabilities
   - Rotate credentials
   - Update security policies

5. **Lessons Learned**
   - Document incident
   - Update procedures
   - Improve monitoring

### Emergency Procedures

#### Revoke Access
```bash
# Remove user from access entries
aws eks delete-access-entry \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::123456789012:user/compromised-user

# Delete role binding
kubectl delete rolebinding namespace-viewer -n online-boutique
```

#### Isolate Workload
```yaml
# Emergency network policy - deny all
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: emergency-isolation
  namespace: online-boutique
spec:
  podSelector:
    matchLabels:
      app: compromised-app
  policyTypes:
  - Ingress
  - Egress
```

## Security Best Practices

### Development Practices

1. **Least Privilege**: Grant minimum required permissions
2. **Regular Updates**: Keep all components updated
3. **Image Security**: Scan container images for vulnerabilities
4. **Secrets Management**: Never hardcode secrets in code
5. **Network Segmentation**: Use namespaces and network policies

### Operational Practices

1. **Regular Audits**: Review access logs and permissions
2. **Backup Strategy**: Regular backups of critical data
3. **Incident Response**: Maintain updated incident response plan
4. **Security Training**: Regular security awareness training
5. **Compliance**: Regular compliance assessments

### Configuration Management

1. **Infrastructure as Code**: All infrastructure in version control
2. **Configuration Drift**: Monitor for unauthorized changes
3. **Change Management**: Proper approval process for changes
4. **Documentation**: Keep security documentation updated

## Security Checklist

### Pre-Deployment
- [ ] Review IAM permissions
- [ ] Validate network configuration
- [ ] Check encryption settings
- [ ] Review RBAC policies
- [ ] Scan Terraform code with tfsec

### Post-Deployment
- [ ] Verify access controls
- [ ] Test RBAC permissions
- [ ] Enable audit logging
- [ ] Configure monitoring
- [ ] Run security benchmarks

### Ongoing Maintenance
- [ ] Regular security updates
- [ ] Monitor security alerts
- [ ] Review access logs
- [ ] Update documentation
- [ ] Conduct security assessments

## Compliance Frameworks

### SOC 2 Type II
- Access controls implemented
- Audit logging enabled
- Encryption at rest and in transit
- Regular security monitoring

### PCI DSS (if applicable)
- Network segmentation
- Access controls
- Encryption requirements
- Regular security testing

### GDPR (if applicable)
- Data encryption
- Access controls
- Audit trails
- Data retention policies

## Security Tools Integration

### Recommended Security Stack

1. **Vulnerability Scanning**: Trivy, Clair
2. **Runtime Security**: Falco, Sysdig
3. **Policy Enforcement**: OPA Gatekeeper
4. **Secrets Management**: External Secrets Operator
5. **Service Mesh**: Istio for mTLS
6. **Monitoring**: Prometheus, Grafana
7. **SIEM Integration**: Splunk, ELK Stack

### Implementation Examples

#### OPA Gatekeeper Policy
```yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8srequiredsecuritycontext
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredSecurityContext
      validation:
        properties:
          runAsNonRoot:
            type: boolean
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredsecuritycontext
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.securityContext.runAsNonRoot
          msg := "Container must run as non-root user"
        }
```

This security documentation provides comprehensive coverage of security measures, best practices, and implementation guidelines for the secure EKS infrastructure.