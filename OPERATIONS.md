# Operations Guide

## Overview

This document provides operational procedures, maintenance tasks, and day-to-day management guidelines for the Secure EKS Automation infrastructure.

## Daily Operations

### Health Checks

#### Cluster Health
```bash
# Check cluster status
aws eks describe-cluster --name $(terraform output -raw cluster_name) --query 'cluster.status'

# Check node status
kubectl get nodes -o wide

# Check system pods
kubectl get pods -n kube-system --field-selector=status.phase!=Running

# Check critical add-ons
kubectl get pods -n kube-system | grep -E "(aws-load-balancer|cluster-autoscaler|metrics-server)"
```

#### Application Health
```bash
# Check ArgoCD
kubectl get pods -n argocd
kubectl get applications -n argocd

# Check OPA Gatekeeper
kubectl get pods -n gatekeeper-system
kubectl get constraints

# Check application namespaces
kubectl get pods -n online-boutique

# Check services and endpoints
kubectl get svc,endpoints -n online-boutique
```

#### Resource Utilization
```bash
# Node resource usage
kubectl top nodes

# Pod resource usage
kubectl top pods --all-namespaces --sort-by=cpu
kubectl top pods --all-namespaces --sort-by=memory

# Check for resource constraints
kubectl describe nodes | grep -A 5 "Allocated resources"
```

### Monitoring Dashboard

#### Key Metrics to Monitor

1. **Cluster Metrics**
   - Node count and status
   - Pod count per namespace
   - CPU and memory utilization
   - Network traffic

2. **Application Metrics**
   - Pod restart count
   - Service response times
   - Error rates
   - Resource consumption

3. **Infrastructure Metrics**
   - EKS control plane status
   - Auto Scaling Group health
   - Load balancer health
   - VPC flow logs

#### CloudWatch Queries

```bash
# Get cluster insights
aws logs start-query \
  --log-group-name "/aws/eks/$(terraform output -raw cluster_name)/cluster" \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, @message | filter @message like /ERROR/'
```

## Maintenance Procedures

### Regular Maintenance Tasks

#### Weekly Tasks

1. **Security Updates**
```bash
# Check for EKS cluster updates
aws eks describe-cluster --name $(terraform output -raw cluster_name) --query 'cluster.version'

# Check for node group AMI updates
aws eks describe-nodegroup \
  --cluster-name $(terraform output -raw cluster_name) \
  --nodegroup-name dev \
  --query 'nodegroup.amiType'

# Update Helm charts
helm repo update
helm list --all-namespaces
```

2. **Resource Cleanup**
```bash
# Clean up completed jobs
kubectl delete jobs --field-selector=status.successful=1 --all-namespaces

# Clean up failed pods
kubectl delete pods --field-selector=status.phase=Failed --all-namespaces

# Clean up unused persistent volumes
kubectl get pv | grep Released
```

3. **Backup Verification**
```bash
# Verify Terraform state backup
aws s3 ls s3://state-secure-eks-automation/terraform.tfstate

# Check state file integrity
terraform state list
```

#### Monthly Tasks

1. **Capacity Planning**
```bash
# Analyze resource trends
kubectl top nodes --sort-by=cpu
kubectl top nodes --sort-by=memory

# Check Auto Scaling Group metrics
aws autoscaling describe-auto-scaling-groups \
  --query 'AutoScalingGroups[?contains(Tags[?Key==`kubernetes.io/cluster/$(terraform output -raw cluster_name)`].Value, `owned`)].[AutoScalingGroupName,DesiredCapacity,MinSize,MaxSize]'
```

2. **Security Review**
```bash
# Review access entries
aws eks list-access-entries --cluster-name $(terraform output -raw cluster_name)

# Check RBAC permissions
kubectl auth can-i --list --as=system:serviceaccount:online-boutique:default

# Review network policies
kubectl get networkpolicies --all-namespaces
```

3. **Cost Optimization**
```bash
# Analyze resource usage
kubectl resource-capacity --sort cpu.request
kubectl resource-capacity --sort memory.request

# Check for unused resources
kubectl get pvc --all-namespaces | grep -v Bound
kubectl get services --all-namespaces --field-selector=spec.type=LoadBalancer
```

### Upgrade Procedures

#### EKS Cluster Upgrade

1. **Pre-upgrade Checklist**
```bash
# Check current version
kubectl version --short

# Review upgrade path
aws eks describe-addon-versions --kubernetes-version 1.32

# Backup critical data
kubectl get all --all-namespaces -o yaml > cluster-backup.yaml
```

2. **Upgrade Process**
```bash
# Update Terraform configuration
# In variables.tf, update cluster_version = "1.33"

# Plan the upgrade
terraform plan -var-file="terraform.tfvars"

# Apply the upgrade
terraform apply -var-file="terraform.tfvars"
```

3. **Post-upgrade Validation**
```bash
# Verify cluster version
kubectl version --short

# Check node status
kubectl get nodes

# Verify add-ons
kubectl get pods -n kube-system

# Test application functionality
kubectl get pods -n online-boutique
```

#### Node Group Updates

```bash
# Update node group AMI
aws eks update-nodegroup-version \
  --cluster-name $(terraform output -raw cluster_name) \
  --nodegroup-name dev \
  --force

# Monitor update progress
aws eks describe-nodegroup \
  --cluster-name $(terraform output -raw cluster_name) \
  --nodegroup-name dev \
  --query 'nodegroup.status'
```

## Scaling Operations

### Manual Scaling

#### Scale Node Groups
```bash
# Scale up
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name <asg-name> \
  --desired-capacity 5

# Scale down
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name <asg-name> \
  --desired-capacity 2
```

#### Scale Applications
```bash
# Scale deployment
kubectl scale deployment nginx --replicas=5 -n online-boutique

# Scale with HPA
kubectl autoscale deployment nginx --cpu-percent=50 --min=1 --max=10 -n online-boutique
```

### Auto Scaling Configuration

#### Cluster Autoscaler Tuning
```bash
# Check current configuration
kubectl get configmap cluster-autoscaler-status -n kube-system -o yaml

# View autoscaler logs
kubectl logs -n kube-system deployment/cluster-autoscaler

# Modify scaling parameters
kubectl patch deployment cluster-autoscaler -n kube-system -p '
{
  "spec": {
    "template": {
      "spec": {
        "containers": [
          {
            "name": "cluster-autoscaler",
            "command": [
              "./cluster-autoscaler",
              "--v=4",
              "--stderrthreshold=info",
              "--cloud-provider=aws",
              "--skip-nodes-with-local-storage=false",
              "--expander=least-waste",
              "--node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/$(terraform output -raw cluster_name)",
              "--scale-down-delay-after-add=2m",
              "--scale-down-unneeded-time=2m"
            ]
          }
        ]
      }
    }
  }
}'
```

## Backup and Recovery

### Backup Procedures

#### Automated Backups
```bash
# Velero backup (if installed)
velero backup create daily-backup --include-namespaces online-boutique

# Manual backup
kubectl get all --all-namespaces -o yaml > full-cluster-backup-$(date +%Y%m%d).yaml
```

#### Critical Data Backup
```bash
# Backup secrets
kubectl get secrets --all-namespaces -o yaml > secrets-backup-$(date +%Y%m%d).yaml

# Backup configmaps
kubectl get configmaps --all-namespaces -o yaml > configmaps-backup-$(date +%Y%m%d).yaml

# Backup persistent volumes
kubectl get pv,pvc --all-namespaces -o yaml > volumes-backup-$(date +%Y%m%d).yaml
```

### Recovery Procedures

#### Disaster Recovery
```bash
# Recreate cluster from Terraform
terraform destroy -auto-approve
terraform apply -auto-approve

# Restore applications
kubectl apply -f full-cluster-backup-$(date +%Y%m%d).yaml

# Verify recovery
kubectl get pods --all-namespaces
```

#### Partial Recovery
```bash
# Restore specific namespace
kubectl create namespace online-boutique
kubectl apply -f namespace-backup.yaml -n online-boutique

# Restore secrets
kubectl apply -f secrets-backup.yaml

# Verify restoration
kubectl get all -n online-boutique
```

## Troubleshooting

### Common Issues and Solutions

#### Pod Issues

**Pods in Pending State**
```bash
# Check node resources
kubectl describe nodes | grep -A 5 "Allocated resources"

# Check pod events
kubectl describe pod <pod-name> -n <namespace>

# Check for taints and tolerations
kubectl describe nodes | grep Taints
```

**Pods in CrashLoopBackOff**
```bash
# Check pod logs
kubectl logs <pod-name> -n <namespace> --previous

# Check resource limits
kubectl describe pod <pod-name> -n <namespace> | grep -A 10 "Limits"

# Check liveness/readiness probes
kubectl describe pod <pod-name> -n <namespace> | grep -A 5 "Liveness\|Readiness"
```

#### Network Issues

**Service Not Accessible**
```bash
# Check service endpoints
kubectl get endpoints <service-name> -n <namespace>

# Check service selector
kubectl describe service <service-name> -n <namespace>

# Test connectivity
kubectl run test-pod --image=busybox --rm -it -- wget -qO- <service-name>.<namespace>.svc.cluster.local
```

**Load Balancer Issues**
```bash
# Check AWS Load Balancer Controller
kubectl logs -n kube-system deployment/aws-load-balancer-controller

# Check ingress status
kubectl describe ingress <ingress-name> -n <namespace>

# Verify security groups
aws ec2 describe-security-groups --filters "Name=tag:kubernetes.io/cluster/$(terraform output -raw cluster_name),Values=owned"
```

#### Storage Issues

**PVC Pending**
```bash
# Check storage class
kubectl get storageclass

# Check PVC events
kubectl describe pvc <pvc-name> -n <namespace>

# Check available storage
kubectl get pv
```

### Performance Troubleshooting

#### High CPU Usage
```bash
# Identify high CPU pods
kubectl top pods --all-namespaces --sort-by=cpu

# Check node CPU usage
kubectl top nodes --sort-by=cpu

# Analyze resource requests vs limits
kubectl describe pod <pod-name> -n <namespace> | grep -A 10 "Requests\|Limits"
```

#### Memory Issues
```bash
# Check memory usage
kubectl top pods --all-namespaces --sort-by=memory

# Check for OOMKilled pods
kubectl get pods --all-namespaces --field-selector=status.phase=Failed
kubectl describe pod <failed-pod> -n <namespace> | grep -i "oom\|memory"

# Check node memory pressure
kubectl describe nodes | grep -A 5 "MemoryPressure"
```

## Alerting and Notifications

### Critical Alerts

1. **Cluster Down**: EKS API server unreachable
2. **Node Failure**: Worker node becomes NotReady
3. **Pod Failures**: Critical pods in CrashLoopBackOff
4. **Resource Exhaustion**: CPU/Memory usage > 80%
5. **Storage Issues**: PVC provisioning failures

### Alert Configuration

#### CloudWatch Alarms
```bash
# Create CPU utilization alarm
aws cloudwatch put-metric-alarm \
  --alarm-name "EKS-HighCPU" \
  --alarm-description "EKS cluster high CPU utilization" \
  --metric-name CPUUtilization \
  --namespace AWS/EKS \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2
```

#### Prometheus Alerts (if using Prometheus)
```yaml
groups:
- name: kubernetes-alerts
  rules:
  - alert: PodCrashLooping
    expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Pod {{ $labels.pod }} is crash looping"
      
  - alert: NodeNotReady
    expr: kube_node_status_condition{condition="Ready",status="true"} == 0
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Node {{ $labels.node }} is not ready"
```

## Documentation Maintenance

### Runbook Updates

1. **Incident Documentation**: Record all incidents and resolutions
2. **Procedure Updates**: Update procedures based on lessons learned
3. **Configuration Changes**: Document all configuration modifications
4. **Version Updates**: Track all component version changes

### Knowledge Base

Maintain documentation for:
- Common troubleshooting scenarios
- Emergency procedures
- Contact information
- Escalation procedures
- Service dependencies

## Compliance and Auditing

### Audit Procedures

#### Access Audit
```bash
# Review access entries
aws eks list-access-entries --cluster-name $(terraform output -raw cluster_name)

# Check RBAC bindings
kubectl get rolebindings,clusterrolebindings --all-namespaces

# Review service accounts
kubectl get serviceaccounts --all-namespaces
```

#### Security Audit
```bash
# Run CIS benchmark
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job-eks.yaml
kubectl logs job/kube-bench

# Check pod security policies
kubectl get psp

# Review network policies
kubectl get networkpolicies --all-namespaces
```

### Compliance Reporting

Generate regular reports for:
- Security compliance status
- Resource utilization trends
- Incident response metrics
- Change management logs
- Access review results

This operations guide provides comprehensive procedures for managing the EKS infrastructure effectively. Regular execution of these procedures ensures optimal performance, security, and reliability of the cluster.