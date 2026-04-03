# GKE Cluster Autoscaler (Open Source)

At KubeCon EU 2026, Google open-sourced the GKE Cluster Autoscaler. Same code that's been running in production, now public.

Two parts:

- **Part A**: GKE's built-in autoscaler. Trigger scale-up and scale-down, see how it decides.
- **Part B**: Deploy the OSS version yourself as a Deployment (optional, standalone).

## What this covers

- How the autoscaler decides when to add or remove nodes
- Reading the status configmap and autoscaler logs
- How `safe-to-evict` blocks node removal
- Running the OSS autoscaler as a Deployment with Workload Identity

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

Both `ScaleUp` and `ScaleDown` sections should show no action needed at this point.

### Step 3 — Trigger scale-up

5 pods, 500m CPU each. Your single `e2-standard-2` fits about 3 after DaemonSet overhead — the rest sit `Pending` until a new node is ready.

```bash
kubectl apply -f manifests/inflate.yaml
kubectl get pods -w
```

In a second terminal:

```bash
kubectl get nodes -w
```

New nodes appear in 2–3 minutes, then the pending pods get scheduled.

### Step 4 — Read the autoscaler decision

```bash
kubectl describe configmap cluster-autoscaler-status -n kube-system
kubectl get events -n kube-system --sort-by='.lastTimestamp' | grep -i scale
```

### Step 5 — Trigger scale-down

```bash
kubectl delete -f manifests/inflate.yaml
```

The autoscaler waits 10 minutes of sustained underuse before removing a node. Track it:

```bash
kubectl get nodes -w
```

You can track the countdown in the status configmap:

```bash
watch kubectl describe configmap cluster-autoscaler-status -n kube-system
```

### Step 6 — Test safe-to-evict

After Step 5 finishes, check the cluster:

```bash
kubectl get nodes
```

You'll see 2 nodes, not 1. The autoscaler wants to reach min=1 but `konnectivity-agent` runs with anti-affinity — both replicas can't land on one node — so one node stays blocked. Expected.

Deploy the non-evictable pod:

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

Re-apply inflate to push the cluster to add a node:

```bash
kubectl apply -f manifests/inflate.yaml
kubectl get pods -w
```

Once all inflate pods are `Running`, check placement:

```bash
kubectl get pods -o wide
```

Then delete inflate:

```bash
kubectl delete -f manifests/inflate.yaml
```

After ~10 minutes:

```bash
kubectl get nodes
kubectl get pods -o wide
```

Scale-down goes from 3 to 2. The node with `non-evictable-pod` never entered the candidate list.

Check the status:

```bash
kubectl get configmap cluster-autoscaler-status -n kube-system \
  -o jsonpath='{.data.status}'
```

`scaleDown.status: NoCandidates` — neither node qualifies. Remove the annotation and the autoscaler would be free to evict the pod and consolidate further.

```bash
kubectl delete -f manifests/non-evictable-pod.yaml
```

## Part B — Deploy the open-source autoscaler

OSS codebase: https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler

GKE hides the autoscaler as a control-plane component. With the OSS release you can run it as a regular Deployment — handy for self-managed Kubernetes on GCP, auditing, or extending it.

Disable GKE's managed autoscaler:

```bash
gcloud container clusters update $CLUSTER_NAME \
  --zone $ZONE \
  --no-enable-autoscaling \
  --node-pool default-pool
```

GKE's addon reconciler keeps its own `cluster-autoscaler` ClusterRoleBinding alive — delete it and it just recreates with the wrong subject. The OSS manifest uses `cluster-autoscaler-oss` as the binding name to sidestep that entirely.

The OSS autoscaler needs GCP credentials to call the Compute API. Set up Workload Identity:

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

One thing to watch: GKE truncates cluster names in MIG names. `lab-cluster-autoscaler` becomes `lab-cluster-autoscal`. Derive the real prefix from the node names:

```bash
MIG_PREFIX=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}' \
  | sed 's/-[a-z0-9]\{8\}-[a-z0-9]\{4\}$//')
echo $MIG_PREFIX
# e.g. gke-lab-cluster-autoscal-default-pool
```

Substitute your project ID and MIG prefix, then deploy:

```bash
sed -e "s/YOUR_PROJECT_ID/$PROJECT_ID/g" \
    -e "s/YOUR_MIG_PREFIX/$MIG_PREFIX/g" \
    manifests/cluster-autoscaler-oss.yaml | kubectl apply -f -
```

Check the Workload Identity annotation landed on the Kubernetes ServiceAccount:

```bash
kubectl get serviceaccount cluster-autoscaler -n kube-system \
  -o jsonpath='{.metadata.annotations}'
```

Output should include `iam.gke.io/gcp-service-account`.

Check the binding subject:

```bash
kubectl get clusterrolebinding cluster-autoscaler-oss -o jsonpath='{.subjects}'
```

Expected: `[{"kind":"ServiceAccount","name":"cluster-autoscaler","namespace":"kube-system"}]`

Check it's running:

```bash
kubectl get pods -n kube-system -l app=cluster-autoscaler
kubectl logs -n kube-system -l app=cluster-autoscaler --tail=50
```

Run inflate again to confirm scaling still works:

```bash
kubectl apply -f manifests/inflate.yaml
```

## Step 7 — Clean up

```bash
kubectl delete -f manifests/ --ignore-not-found
gcloud container clusters delete $CLUSTER_NAME --zone $ZONE --quiet
```