# GKE Image Streaming

When a pod lands on a node without the image cached, the node downloads the full image before the container starts. For large images (5–20 GB ML images), this takes several minutes.

Image Streaming changes this — the container starts immediately and the node fetches layer data on demand, only the blocks the process actually reads. Most containers only touch a small fraction of their image at startup, so the rest is never downloaded.

Hard requirement: images must be in **Artifact Registry**. Docker Hub, GCR, ECR are not supported.

## What this covers

- Enabling Image Streaming on a GKE node pool
- Measuring container startup time with and without streaming
- Testing on cold nodes where the difference is largest

## Prerequisites

- `gcloud` CLI authenticated
- `kubectl` installed
- GCP project with billing enabled
- APIs enabled:

```bash
gcloud services enable \
  container.googleapis.com \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com
```

## Set variables

```bash
export PROJECT_ID=$(gcloud config get-value project)
export REGION=us-central1
export ZONE=us-central1-a
export REPO=lab-streaming
export CLUSTER_NAME=lab-image-streaming
```

## Clone the repo

```bash
git clone https://github.com/misskecupbung/gke-image-streaming.git
cd gke-image-streaming
```

## Step 1 — Create an Artifact Registry repository

```bash
gcloud artifacts repositories create $REPO \
  --repository-format=docker \
  --location=$REGION \
  --description="Lab: image streaming"
gcloud artifacts repositories list

gcloud auth configure-docker ${REGION}-docker.pkg.dev
```

## Step 2 — Push a large test image

We use Cloud Build to pull from Docker Hub (~6 GB) and push to AR. First, grant the Cloud Build service account access to Storage:

```bash
PROJECT_NUMBER=$(gcloud projects describe ${PROJECT_ID} --format='value(projectNumber)')

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
  --role="roles/storage.objectAdmin"
```

Then submit the build. The config is already in `manifests/cloudbuild-push-image.yaml`:

```bash
gcloud builds submit --no-source \
  --substitutions=_REGION=${REGION},_PROJECT_ID=${PROJECT_ID},_REPO=${REPO} \
  --config manifests/cloudbuild-push-image.yaml
```

Then update the `image` field in both manifest files using `sed`:

```bash
sed -i "s|REPLACE_WITH_YOUR_AR_IMAGE|${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO}/pytorch-test:latest|g" \
  manifests/test-pods-no-streaming.yaml \
  manifests/test-pods-streaming.yaml
```

## Step 3 — Create a cluster without streaming (baseline)

```bash
gcloud container clusters create $CLUSTER_NAME \
  --zone $ZONE \
  --num-nodes 2 \
  --machine-type e2-standard-4 \
  --image-type COS_CONTAINERD \
  --release-channel regular

gcloud container clusters get-credentials $CLUSTER_NAME --zone $ZONE
```

No `--enable-image-streaming`. This is the baseline.

## Step 4 — Measure baseline startup time

```bash
kubectl apply -f manifests/test-pods-no-streaming.yaml

time kubectl wait --for=condition=Ready pod \
  -l app=test-no-streaming \
  --timeout=600s
```

Check the pull duration:

```bash
kubectl describe pod -l app=test-no-streaming | grep -A3 "Pulling\|Pulled"
```

```bash
kubectl delete -f manifests/test-pods-no-streaming.yaml
```

## Step 5 — Enable streaming on the node pool

```bash
gcloud container node-pools update default-pool \
  --cluster $CLUSTER_NAME \
  --zone $ZONE \
  --enable-image-streaming
```

Wait for the rolling restart:

```bash
kubectl get nodes -w
```

Verify streaming is active:

```bash
kubectl get nodes \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.labels.cloud\.google\.com/gke-image-streaming}{"\n"}{end}'
```

## Step 6 — Measure startup time with streaming

```bash
kubectl apply -f manifests/test-pods-streaming.yaml

time kubectl wait --for=condition=Ready pod \
  -l app=test-streaming \
  --timeout=600s
```

```bash
kubectl describe pod -l app=test-streaming | grep -A3 "Pulling\|Pulled\|Started"
```

With streaming, `Pulled image` appears within seconds — metadata registered, not full download.

## Step 7 — Compare the results

| Mode | Image pull time | Container start |
|------|----------------|-----------------|
| No streaming | Full download (~minutes) | After pull completes |
| Streaming | Near instant | During pull |

## Step 8 — Test on a cold node

```bash
kubectl cordon $(kubectl get nodes -o name | head -2 | sed 's|node/||')

gcloud container clusters resize $CLUSTER_NAME \
  --zone $ZONE \
  --num-nodes 3 \
  --node-pool default-pool

kubectl get nodes -w
```

```bash
NEW_NODE=$(kubectl get nodes --sort-by='.metadata.creationTimestamp' \
  -o jsonpath='{.items[-1].metadata.name}')

kubectl run cold-test \
  --image=${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO}/pytorch-test:latest \
  --overrides='{"spec":{"nodeName":"'$NEW_NODE'"}}' \
  --restart=Never

time kubectl wait --for=condition=Ready pod cold-test --timeout=300s
```

## Step 9 — Clean up

```bash
kubectl delete -f manifests/ --ignore-not-found
kubectl delete pod cold-test --ignore-not-found
gcloud container clusters delete $CLUSTER_NAME --zone $ZONE --quiet
gcloud artifacts repositories delete $REPO --location=$REGION --quiet
```

## When streaming helps

- Cold nodes during scale-out (autoscaler adds nodes, all cold)
- CI/CD where each job runs on a fresh node
- GPU nodes with 5–20 GB ML images

## When streaming does not help

- Warm nodes with cached layers
- Small images (<500 MB)
- Images outside Artifact Registry

## Resources

- [GKE Image Streaming documentation](https://cloud.google.com/kubernetes-engine/docs/how-to/image-streaming)
