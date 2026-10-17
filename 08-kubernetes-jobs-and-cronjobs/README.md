# Automate Scheduled Tasks and Batch Processing on Azure AKS with Kubernetes Jobs and CronJobs

Welcome to the hands-on cloud operations lab by [Stack Layer](https://stacklayer.blogspot.com/).

In cloud-native production environments, enterprise workloads are not limited to continuous HTTP microservices and web servers. Critical operational tasks—such as asynchronous database migrations, periodic analytics aggregation, data backup pipelines, and compliance sanitation—require dedicated run-to-completion controllers. This repository provides production-tested Kubernetes manifests and hands-on commands for orchestrating batch Jobs and time-scheduled CronJobs on Azure Kubernetes Service (AKS).

---

## 📖 Companion Blog Guide

For in-depth architectural diagrams, CoreDNS resolution packet walks, comparison tables, and Azure Portal visual verification, refer to our companion publication:

* Official Guide: [Automate Scheduled Tasks and Batch Processing on Azure AKS with Kubernetes Jobs and CronJobs](https://stacklayer.blogspot.com/2026/10/azure-aks-kubernetes-jobs-cronjobs-scheduled-tasks-batch-processing.html)

---

## 📁 Directory Structure

```text
08-kubernetes-jobs-and-cronjobs/
├── README.md
└── kube-manifests/
    ├── 01-stacklayer-batch-job.yaml           # One-off database migration batch job (run-to-completion)
    ├── 02-stacklayer-parallel-worker-job.yaml  # Parallel data processing queue with retry backoff limits
    └── 03-stacklayer-db-backup-cronjob.yaml    # Scheduled 5-minute automated database backup CronJob
```

---

## 🎯 Learning Objectives

By executing this hands-on lab, you will achieve the following production capabilities:
1. **Differentiate Workload Controllers:** Understand when to deploy a continuous Deployment versus a run-to-completion Job or scheduled CronJob.
2. **Eliminate Pod Restart Loops:** Correctly configure `restartPolicy: Never` and `restartPolicy: OnFailure` for transient containers.
3. **Control Batch Parallelism:** Orchestrate parallel worker pools utilizing `completions` and `parallelism` directives.
4. **Enforce Fault Tolerance:** Configure exponential retry thresholds with `backoffLimit` and hard execution timeouts with `activeDeadlineSeconds`.
5. **Schedule Crontab Workloads:** Construct automated time-triggered tasks using standardized UNIX crontab expressions.
6. **Prevent Race Conditions & Deadlocks:** Implement enterprise `concurrencyPolicy: Forbid` rules to block overlapping job runs.
7. **Execute On-Demand Testing & Cleanup:** Trigger ad-hoc job executions directly from CronJob templates and safely manage Azure compute consumption.

---

## 📋 Prerequisites

* An active **Azure Subscription** (Free Trial, Student Credits, or Pay-As-You-Go).
* A running **Azure AKS Cluster** (provisioned as shown in [Deploy Your First Containerized App on Azure Kubernetes Service (AKS)](https://stacklayer.blogspot.com/)).
* **Azure CLI** (`az`) and **Kubectl** installed and authenticated.

Connect your local terminal to your running cluster:

```bash
# Connect Azure CLI to your running AKS cluster
az aks get-credentials --resource-group stacklayer-aks-rg --name stacklayer-aks-cluster --overwrite-existing

# Verify cluster connectivity and healthy nodes
kubectl cluster-info
kubectl get nodes
```

---

## 🚀 Step-by-Step Hands-On Guide

### Step 1: Deploy a One-Off Batch Job (Run-to-Completion)

#### kube-manifests/01-stacklayer-batch-job.yaml
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: stacklayer-batch-job
  labels:
    app: stacklayer-batch-processor
    stacklayer.io/tier: automation
spec:
  template:
    metadata:
      labels:
        app: stacklayer-batch-processor
    spec:
      restartPolicy: Never
      containers:
      - name: batch-worker
        image: stacklayer/batch-runner:v1.0
        env:
        - name: TASK_NAME
          value: "database-migration"
        - name: BATCH_MODE
          value: "sequential"
```

Apply and monitor the execution:

```bash
# Apply the batch job manifest
kubectl apply -f kube-manifests/01-stacklayer-batch-job.yaml

# Monitor Job status until completion
kubectl get jobs -l app=stacklayer-batch-processor

# Verify the pod reached Completed state
kubectl get pods -l app=stacklayer-batch-processor

# Inspect standard output logs
kubectl logs -l app=stacklayer-batch-processor
```

---

### Step 2: Implement Parallelism, Retry Backoffs & Timeouts

#### kube-manifests/02-stacklayer-parallel-worker-job.yaml
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: stacklayer-parallel-worker-job
  labels:
    app: stacklayer-parallel-worker
spec:
  completions: 4
  parallelism: 2
  backoffLimit: 3
  activeDeadlineSeconds: 120
  template:
    metadata:
      labels:
        app: stacklayer-parallel-worker
    spec:
      restartPolicy: OnFailure
      containers:
      - name: data-aggregator
        image: stacklayer/batch-runner:v1.0
        env:
        - name: TASK_NAME
          value: "partition-processing"
        - name: BATCH_MODE
          value: "parallel"
```

Apply the parallel job and observe worker scheduling in real-time:

```bash
# Apply the parallel batch job manifest
kubectl apply -f kube-manifests/02-stacklayer-parallel-worker-job.yaml

# Watch completions transition from 0/4 to 4/4
kubectl get jobs -w
```

---

### Step 3: Schedule Recurring Tasks with a Kubernetes CronJob

#### kube-manifests/03-stacklayer-db-backup-cronjob.yaml
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: stacklayer-db-backup-cronjob
  labels:
    app: stacklayer-backup-service
    stacklayer.io/tier: scheduled-task
spec:
  schedule: "*/5 * * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 2
  startingDeadlineSeconds: 100
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: stacklayer-backup-service
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup-agent
            image: stacklayer/batch-runner:v1.0
            env:
            - name: TASK_NAME
              value: "database-backup"
            - name: RETENTION_DAYS
              value: "7"
```

Apply the CronJob and inspect scheduled registration:

```bash
# Register the CronJob in the cluster
kubectl apply -f kube-manifests/03-stacklayer-db-backup-cronjob.yaml

# Verify schedule and last run status
kubectl get cronjobs
```

---

### Step 4: Ad-Hoc Testing & History Verification

Trigger an immediate execution from the CronJob template without waiting for the 5-minute schedule:

```bash
# Trigger an immediate on-demand job run
kubectl create job \
  --from=cronjob/stacklayer-db-backup-cronjob \
  stacklayer-manual-test-run

# Stream the container execution output
kubectl logs -l app=stacklayer-backup-service --tail=20

# Confirm automatic cleanup adheres to history limits
kubectl get jobs --sort-by=.metadata.creationTimestamp
```

---

## 🧹 Cost Safeguards & Cleanup

To eliminate ongoing Azure cloud compute charges after completing your testing, execute one of the two teardown options:

### Option A: Pause Cluster Nodes (Recommended for Ongoing Practice)
Preserves control plane metadata and persistent storage while turning off VM compute billing:

```bash
az aks stop \
  --resource-group stacklayer-aks-rg \
  --name stacklayer-aks-cluster
```

*To resume later, run `az aks start --resource-group stacklayer-aks-rg --name stacklayer-aks-cluster`.*

### Option B: Complete Deletion (Zero Cost)
Permanently deletes all provisioned resources inside the resource group:

```bash
az group delete \
  --name stacklayer-aks-rg \
  --yes \
  --no-wait
```

---

## 🤝 Community & Support

* 💬 Have questions or run into an issue? Join the conversation in [GitHub Discussions](https://github.com/StackLayerOfficially/azure-aks-hands-on-labs/discussions).
* 🌐 Explore more hands-on cloud operations guides on [Stack Layer](https://stacklayer.blogspot.com/).

---

<div align="center">

### 🌐 Automate the Mundane. Architect for Scale.

“Continuous services keep the cluster alive, but deterministic batch automation ensures enterprise data integrity thrives. True cloud resilience lives in run-to-completion precision.”

<br>

Built with ❤️ by the **Stack Layer Engineering Team**

</div>
