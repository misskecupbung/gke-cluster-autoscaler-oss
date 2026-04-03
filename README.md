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
export ZONE=us-central1-f
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
  --disk-size=50 \
  --disk-type=pd-standard \
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

First deploy the non-evictable pod:

```bash
kubectl apply -f manifests/non-evictable-pod.yaml
kubectl get pods
```

Confirm the annotation is set:

```bash
kubectl get pod non-evictable-pod \
  -o jsonpath='{.metadata.annotations.cluster-autoscaler\.kubernetes\.io/safe-to-evict}'
```

Expected output:
```
false
```

Now re-apply inflate to force a second node to be added:

```bash
kubectl apply -f manifests/inflate.yaml
kubectl get pods -w
```

Wait until all inflate pods are `Running`. Check which pods are on which nodes:

```bash
kubectl get pods -o wide
```

Delete inflate to trigger scale-down:

```bash
kubectl delete -f manifests/inflate.yaml
```

After ~10 minutes, check nodes and pods:

```bash
kubectl get nodes
kubectl get pods -o wide
```

The cluster scaled down from 4 nodes to 2. One node is kept for the `min=1` requirement, the other is kept because `non-evictable-pod` blocks eviction.

Check the autoscaler status:

```bash
kubectl get configmap cluster-autoscaler-status -n kube-system \
  -o jsonpath='{.data.status}'
```

You'll see `scaleDown.status: NoCandidates` — both remaining nodes are blocked from removal. Without the annotation, the cluster would have scaled down to 1 node.

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

GKE's addon reconciler keeps its own `cluster-autoscaler` ClusterRoleBinding alive. The OSS manifest uses the name `cluster-autoscaler-oss` for its ClusterRoleBinding to avoid that conflict. No cleanup needed.

The OSS autoscaler runs as a pod, so it needs GCP API access to manage MIGs. Set up Workload Identity:

```bash
# Enable Workload Identity on the cluster
gcloud container clusters update $CLUSTER_NAME \
  --zone $ZONE \
  --workload-pool=$PROJECT_ID.svc.id.goog

# Update the node pool to use GKE metadata server
gcloud container node-pools update default-pool \
  --cluster=$CLUSTER_NAME \
  --zone=$ZONE \
  --workload-metadata=GKE_METADATA

# Create a GCP service account
gcloud iam service-accounts create cluster-autoscaler \
  --display-name "Cluster Autoscaler OSS"

# Grant it permission to manage compute resources
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member "serviceAccount:cluster-autoscaler@$PROJECT_ID.iam.gserviceaccount.com" \
  --role "roles/compute.admin"

# Allow the Kubernetes ServiceAccount to impersonate it
gcloud iam service-accounts add-iam-policy-binding \
  cluster-autoscaler@$PROJECT_ID.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:$PROJECT_ID.svc.id.goog[kube-system/cluster-autoscaler]"
```

GKE truncates cluster names in MIG names (e.g. `lab-cluster-autoscaler` → `lab-cluster-autoscal`). Derive the exact prefix from existing node names:

```bash
MIG_PREFIX=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}' \
  | sed 's/-[a-z0-9]\{8\}-[a-z0-9]\{4\}$//')
echo $MIG_PREFIX
# e.g. gke-lab-cluster-autoscal-default-pool
```

Deploy:

```bash
sed -e "s/YOUR_PROJECT_ID/$PROJECT_ID/g" \
    -e "s/YOUR_MIG_PREFIX/$MIG_PREFIX/g" \
    manifests/cluster-autoscaler-oss.yaml | kubectl apply -f -
```

Verify the Workload Identity annotation is on the Kubernetes ServiceAccount:

```bash
kubectl get serviceaccount cluster-autoscaler -n kube-system \
  -o jsonpath='{.metadata.annotations}'
```

Expected output includes `iam.gke.io/gcp-service-account`.

Verify the binding has the correct subject:

```bash
kubectl get clusterrolebinding cluster-autoscaler-oss -o jsonpath='{.subjects}'
```

Expected: `[{"kind":"ServiceAccount","name":"cluster-autoscaler","namespace":"kube-system"}]`

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

## Key autoscaler settings

| Setting | Default |
|---------|---------|
| Scale-up check interval | 10 seconds |
| Scale-down delay after add | 10 minutes |
| Underutilization threshold | 50% |
| Scale-down unneeded time | 10 minutes |

All configurable via autoscaler flags.
