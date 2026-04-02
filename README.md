# GKE Cluster Autoscaler (Open Source)

GKE Cluster Autoscaler adds nodes when pods can't be scheduled and removes nodes when they're underused. At KubeCon EU 2026, Google open-sourced the GKE implementation — same code that runs GKE at scale, now available to run on any Kubernetes cluster.

This lab has two parts:

- **Part A**: Use GKE's built-in autoscaler. Trigger scale-up/down and observe behavior.
- **Part B**: Deploy the open-source autoscaler yourself (optional, independent).

## What this covers

- How the autoscaler decides when to add or remove nodes
- How to read autoscaler logs and the status configmap
- How `safe-to-evict` annotation blocks node removal
- How to deploy the OSS autoscaler as a Deployment

## Prerequisites

- `gcloud` CLI authenticated
- `kubectl` installed
- GCP project with billing enabled
- GKE API enabled:

```bash
gcloud services enable container.googleapis.com
```

## Set variables

```bash
export PROJECT_ID=$(gcloud config get-value project)
export CLUSTER_NAME=lab-cluster-autoscaler
export ZONE=us-central1-a
```

## Clone the repo

```bash
git clone https://github.com/misskecupbung/gke-cluster-autoscaler-oss.git
cd gke-cluster-autoscaler-oss
```

## Part A — GKE managed autoscaler

### Step 1 — Create a cluster with autoscaling

```bash
gcloud container clusters create $CLUSTER_NAME \
  --zone $ZONE \
  --num-nodes 1 \
  --machine-type e2-standard-2 \
  --enable-autoscaling \
  --min-nodes 1 \
  --max-nodes 5
```

```bash
gcloud container clusters get-credentials $CLUSTER_NAME --zone $ZONE
kubectl get nodes
```

### Step 2 — Check autoscaler status

```bash
kubectl describe configmap cluster-autoscaler-status -n kube-system
```

The `ScaleUp` and `ScaleDown` sections show current state. Right now they should say no action needed.

### Step 3 — Trigger scale-up

Deploy 10 pods each requesting 500m CPU. Your single `e2-standard-2` fits about 3 after DaemonSet overhead. The rest stay `Pending`.

```bash
kubectl apply -f manifests/inflate.yaml
kubectl get pods -w
```

In a second terminal:

```bash
kubectl get nodes -w
```

New nodes appear in 2–3 minutes, then pending pods get scheduled.

### Step 4 — Read the autoscaler decision

```bash
kubectl describe configmap cluster-autoscaler-status -n kube-system
kubectl get events -n kube-system --sort-by='.lastTimestamp' | grep -i scale
```

### Step 5 — Trigger scale-down

```bash
kubectl delete -f manifests/inflate.yaml
```

The autoscaler waits 10 minutes before removing underused nodes. Watch:

```bash
kubectl get nodes -w
```

You can track the countdown in the status configmap:

```bash
watch kubectl describe configmap cluster-autoscaler-status -n kube-system
```

### Step 6 — Test safe-to-evict

```bash
kubectl apply -f manifests/non-evictable-pod.yaml
kubectl apply -f manifests/inflate.yaml
```

Wait for 2 nodes, then delete inflate:

```bash
kubectl delete -f manifests/inflate.yaml
```

The node with the non-evictable pod stays — the autoscaler respects the annotation.

```bash
kubectl describe configmap cluster-autoscaler-status -n kube-system | grep -A5 "ScaleDown"
```

You should see one node listed as `NotSafeToEvict`.

```bash
kubectl delete -f manifests/non-evictable-pod.yaml
```

## Part B — Deploy the open-source autoscaler

The OSS codebase: https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler

GKE runs the autoscaler as a control-plane component. With the OSS release, you can run the same code as a Deployment in your cluster — useful for self-managed K8s on GCP, auditing, or custom plugins.

Disable GKE's managed autoscaler:

```bash
gcloud container clusters update $CLUSTER_NAME \
  --zone $ZONE \
  --no-enable-autoscaling \
  --node-pool default-pool
```

Deploy the OSS version:

```bash
kubectl apply -f manifests/cluster-autoscaler-oss.yaml
```

Check it's running:

```bash
kubectl get pods -n kube-system -l app=cluster-autoscaler
kubectl logs -n kube-system -l app=cluster-autoscaler --tail=50
```

Test with inflate again:

```bash
kubectl apply -f manifests/inflate.yaml
```

Same behavior — nodes added when pods are pending.

## Step 7 — Clean up

```bash
kubectl delete -f manifests/ --ignore-not-found
gcloud container clusters delete $CLUSTER_NAME --zone $ZONE --quiet
```

---

## Key autoscaler settings

| Setting | Default |
|---------|---------|
| Scale-up check interval | 10 seconds |
| Scale-down delay after add | 10 minutes |
| Underutilization threshold | 50% |
| Scale-down unneeded time | 10 minutes |

All configurable via autoscaler flags.
