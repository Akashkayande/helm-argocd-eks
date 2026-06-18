# Velero Backup & Disaster Recovery for Amazon EKS

## Overview

This document explains how Velero is used in our Amazon EKS cluster to perform backup and disaster recovery operations for Kubernetes resources and persistent data.

---

# What is Velero?

Velero is an open-source Kubernetes backup and disaster recovery tool.

Velero enables:

* Backup of Kubernetes resources
* Backup of Persistent Volume data
* Restore of deleted resources
* Namespace recovery
* Cluster migration
* Disaster Recovery (DR)
* Scheduled automated backups

Velero stores:

| Component                   | Storage Location     |
| --------------------------- | -------------------- |
| Kubernetes Resources        | Amazon S3            |
| Backup Metadata             | Amazon S3            |
| Persistent Volume Snapshots | Amazon EBS Snapshots |

---
<p align="center">
  <img src="/images/velero-architecture.webp" width="700"/>
</p>
---

# Why Do We Use Velero?

In production environments, accidental deletion, cluster failure, application corruption, or infrastructure issues can cause data loss.

Velero helps us:

* Recover deleted namespaces
* Restore applications after failures
* Recover MongoDB/PostgreSQL data
* Migrate workloads between clusters
* Meet disaster recovery requirements
* Improve system reliability and availability

---

# Components Used

## Amazon S3

Stores:

* Backup metadata
* Kubernetes manifests
* Backup archives

Example:

```text
s3://eks-velero-backup/
```

---

## EBS CSI Driver

Responsible for:

* Dynamic volume provisioning
* Volume snapshots
* PVC backup support

---

## Snapshot Controller

Responsible for:

* Managing Kubernetes VolumeSnapshots
* Creating and restoring EBS snapshots

---

## Velero Server

Runs inside Kubernetes cluster.

Responsibilities:

* Create backups
* Manage schedules
* Restore resources
* Communicate with S3
* Communicate with AWS APIs

---

# Backup Scope

Velero can back up:

## Kubernetes Resources

* Namespace
* Deployment
* StatefulSet
* DaemonSet
* Service
* Ingress
* Secret
* ConfigMap
* ServiceAccount
* HPA
* NetworkPolicy

---

## Persistent Volumes

* PVC
* PV
* EBS Volumes

Example:

```text
MongoDB Data
PostgreSQL Data
Redis Data
```

---
# Step by Step Implementation

## Installing Velero

### Step 1: Create S3 Bucket

* Velero stores backup metadata in S3.

```bash
aws s3api create-bucket \
  --bucket velero-backups \
  --region $AWS_REGION
```

Verify:
```bash
aws s3 ls
```
---
### Step 2 : Install EBS CSI Driver

- Velero snapshot backups require CSI support.

* Check:
```bash
kubectl get pods -n kube-system | grep ebs
```

* Install if missing:
```bash
eksctl create addon \
  --name aws-ebs-csi-driver \
  --cluster my-cluster \
  --force
```

* CSI snapshots require both the EBS CSI driver and the CSI snapshot controller.
---
### Step 4: Install Snapshot Controller
```bash
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name snapshot-controller
```
---
### Step 5: Create IAM Policy

- Download policy:

```bash
curl -LO https://raw.githubusercontent.com/vmware-tanzu/velero/main/examples/aws/velero-policy.json
```

* Create policy:
```bash
aws iam create-policy \
  --policy-name VeleroPolicy \
  --policy-document file://velero-policy.json
```

* Copy ARN.

---
### Step 6: Create IAM Role for Service Account (IRSA)

* Create service account:
```bash
eksctl create iamserviceaccount \
  --cluster=pern-eks \
  --namespace=velero \
  --name=velero \
  --attach-policy-arn=arn:aws:iam::407622020962:policy/VeleroPolicy\
  --approve
```
---

### Step 7: Install Velero via Helm

* Add repo:
```bash
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
helm repo update
```
```bash
helm install velero vmware-tanzu/velero \
  -n velero \
  --create-namespace \
  -f values.yaml
```
* add s3 bucket =>

### [values.yaml](/velero/values.yaml)
---

### Step 8: Verify Velero
```bash
kubectl get pods -n velero
```
---
<p align="center">
  <img src="/images/velero-pods.png" width="700"/>
</p>

---
### Step 9: Create Backup

* Full namespace backup:
```bash
velero backup create pern-prod-backup \
  --include-namespaces pern-prod
```
---  
### Step 10: Schedule Automatic Backups

* Daily backup:
```bash
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --include-namespaces pern-prod
```
---
## How Restore Backup Data ?

### Restore Namespace

- Restore:

```bash
velero restore create \
  --from-backup pern-prod-backup
```

- Verify:

```bash
velero restore get
```

### Restore Into Different Namespace

* Useful for testing DR.
```bash
velero restore create \
  --from-backup pern-prod-backup \
  --namespace-mappings pern-prod:restore-test
```

* Velero supports namespace remapping during restore operations.

### Cross-Cluster Disaster Recovery

* Cluster A

- Backup:

```bash
velero backup create prod-backup
```

- Data goes to:

* S3
* EBS Snapshots

* Cluster B

- Install Velero with same S3 bucket.

* velero backup get

- Restore:

```bash
velero restore create \
  --from-backup prod-backup
```

- Applications come back on new cluster.

# Summary

Velero is the primary backup and disaster recovery solution for our EKS platform.

Key capabilities:

* Namespace Backup
* PVC Backup
* EBS Snapshot Backup
* Disaster Recovery
* Scheduled Backups
* Cross-Cluster Migration
* Full Application Restore

Using Velero with Amazon S3 and EBS snapshots ensures that Kubernetes resources and application data can be recovered quickly during failures, accidental deletions, or disaster recovery scenarios.
