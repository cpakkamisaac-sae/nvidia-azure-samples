# VSS LVS Blueprint on AKS — Hands-On Workshop

This workshop guides you through deploying the **NVIDIA Video Search and Summarization (VSS) Long Video Summarization (LVS)** blueprint on Azure Kubernetes Service (AKS) using a single `Standard_NC96ads_A100_v4` node (4× A100 80 GB GPUs).

Services are exposed through one Azure public IP using the **HAProxy Kubernetes Ingress** controller and [`nip.io`](https://nip.io/) hostnames. This update targets VSS `v3.3.0rc0`: deployment, summarization, and report generation were validated on an existing cluster with two NC48ads nodes (four A100 80 GB GPUs total). The AKS driver plus NVIDIA device-plugin path was validated on a separate two-NC24 A100 cluster. On that cluster, a generated report's Markdown and PDF downloads survived both agent and storage-pod restarts. Creating a fresh four-A100 cluster and the browser upload control remain to be validated.

---

## Prerequisites

- Azure CLI (`az`) installed and logged in
- `kubectl` and `helm` (3.x) installed
- NGC API key — see the [NGC User Guide](https://docs.nvidia.com/ngc/latest/ngc-user-guide.html) for key generation instructions
- Azure subscription with quota for `Standard_NC96ads_A100_v4` (96 vCPUs of `Standard NCADS_A100_v4` family)
- A short MP4 you can use for the guided UI exercise

If you already have an AKS cluster with four available A100 80 GB GPUs, set the
environment variables for that cluster, obtain its credentials, and continue
at Task 3. Check the current context and GPU allocation before installing on a
shared cluster.

---

## Task 1: Environment Configuration

### 1. Install AKS preview extension

```bash
az extension add --name aks-preview
az extension update --name aks-preview
```

### 2. Set environment variables

```bash
export NGC_API_KEY="<YOUR_NGC_API_KEY>"
export SUBSCRIPTION="xxxxx"
export LOCATION="WestEurope"
export RESOURCE_GROUP="rg-vss-build-nc96"
export CLUSTER_NAME="aks-vss-az"
export GPU_NODEPOOL="gpupool"
export GPU_NUM_NODES=1
export GPU_NODE_SIZE="Standard_NC96ads_A100_v4"
export SYSTEM_NODE_SIZE="Standard_D8s_v5"

export NAMESPACE="vss-lvs"
export RELEASE="vss-lvs"
```

### 3. Set active subscription

```bash
az account set --subscription "$SUBSCRIPTION"
```

### 4. Check A100 quota

```bash
az vm list-usage \
  --location "$LOCATION" \
  --query "[?name.value=='StandardNCADSA100v4Family'].{Current:currentValue, Limit:limit}" \
  --output table
```

Ensure `Limit - Current >= 96` (vCPUs for one NC96ads node).

---

## Task 2: Create AKS Cluster

### 1. Create resource group

```bash
az group create \
  --name "$RESOURCE_GROUP" \
  --location "$LOCATION"
```

### 2. Create AKS cluster (system node pool)

```bash
az aks create \
  --resource-group "$RESOURCE_GROUP" \
  --name "$CLUSTER_NAME" \
  --location "$LOCATION" \
  --node-count 1 \
  --node-vm-size "$SYSTEM_NODE_SIZE" \
  --generate-ssh-keys \
  --network-plugin azure \
  --enable-managed-identity
```

### 3. Add GPU node pool

```bash
az aks nodepool add \
  --resource-group "$RESOURCE_GROUP" \
  --cluster-name "$CLUSTER_NAME" \
  --name "$GPU_NODEPOOL" \
  --node-count "$GPU_NUM_NODES" \
  --node-vm-size "$GPU_NODE_SIZE" \
  --node-osdisk-size 512 \
  --labels hardware=gpu gpu-sku=a100 \
  --max-pods 110
```

AKS installs the NVIDIA driver on GPU nodes by default. Task 3 installs the
device plugin that advertises `nvidia.com/gpu` to Kubernetes. Do not specify
`--gpu-driver none` for this workshop path.

### 4. Get cluster credentials

```bash
az aks get-credentials \
  --resource-group "$RESOURCE_GROUP" \
  --name "$CLUSTER_NAME" \
  --overwrite-existing
```

### 5. Verify nodes

```bash
kubectl get nodes -o wide
```

Wait until all nodes reach `Ready` status before proceeding.

---

## Task 3: Install Cluster Prerequisites

### 1. Add NVIDIA Helm repository

```bash
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia --force-update
helm repo update nvidia
```

### 2. Install the NVIDIA device plugin

The fresh AKS node pool from Task 2 has an AKS-installed NVIDIA driver but does
not advertise GPUs until a device plugin is installed. On a dedicated workshop
cluster, install the same pinned device plugin used in the two-NC24 validation:

```bash
sed "s/<GPU_NODEPOOL>/${GPU_NODEPOOL}/g" aks/gpu/nvidia-device-plugin.yaml \
  | kubectl apply -f -
kubectl rollout status daemonset/nvidia-device-plugin-daemonset -n kube-system --timeout=5m
```

The manifest targets the configured `GPU_NODEPOOL`. If you reuse a cluster whose
GPU nodes already advertise `nvidia.com/gpu`, keep its existing device plugin and
skip this installation. Do not run a second plugin on those nodes. GPU Operator
`v26.7.1` made a fresh AKS 1.35.8 / Ubuntu 24.04 GPU node `NotReady` in two
tests, so it is not part of this workshop path.

### 3. Validate GPU allocation

Confirm the GPU node is `Ready` and reports allocatable GPUs:

```bash
kubectl get nodes -l kubernetes.azure.com/agentpool="$GPU_NODEPOOL" \
  -o jsonpath='{.items[0].status.allocatable.nvidia\.com/gpu}'
```

Expected output: `4` (four A100 GPUs on NC96ads).

### 4. Install NVIDIA NIM Operator

The NIM Operator manages NIM model caching and serving (`NIMCache` / `NIMService` CRDs).

#### Install the NIM Operator on any Kubernetes platform

Use the following steps to install the NVIDIA NIM Operator on your Kubernetes cluster.

Ensure the NVIDIA Helm repo is available (already done in **Task 3.1** for this workshop; run again on a fresh machine or if charts are stale):

```bash
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia --force-update
helm repo update nvidia
```

Create the operator namespace (ignore `AlreadyExists` if you run this twice):

```bash
kubectl create namespace nim-operator
```

Install the operator (`--create-namespace` covers a missing namespace if you skipped the step above):

```bash
helm upgrade --install nim-operator nvidia/k8s-nim-operator \
  -n nim-operator \
  --create-namespace \
  --version=3.1.2
```

### 5. Verify NIM Operator

Optionally, confirm the controller pod is running:

```bash
kubectl get pods -n nim-operator
```

Wait until all pods are `Running` before proceeding.

---

## Task 4: Install HAProxy ingress

Install the controller before VSS so its public IP can be used in browser URLs.
The controller is cluster-wide; if one already exists, reuse its IngressClass
and LoadBalancer Service instead of installing a second one.

From this workshop directory:

```bash
helm repo add haproxytech https://haproxytech.github.io/helm-charts --force-update
helm repo update haproxytech

helm upgrade --install vss-haproxy haproxytech/kubernetes-ingress \
  --version 1.54.2 \
  -n vss-ingress --create-namespace \
  -f aks/ingress/haproxy-values.yaml \
  --wait --timeout 5m

kubectl get ingressclass haproxy
kubectl -n vss-ingress get svc vss-haproxy-kubernetes-ingress -w
```

Once the Service has an `EXTERNAL-IP`, press Ctrl-C to stop watching and capture it:

```bash
export EXTERNAL_HOST="$(kubectl -n vss-ingress get svc \
  vss-haproxy-kubernetes-ingress \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')"
test -n "$EXTERNAL_HOST"
echo "Azure LoadBalancer IP: $EXTERNAL_HOST"
```

The supplied values use `externalTrafficPolicy: Local`, as tested on AKS.

---

## Task 5: Deploy VSS LVS Blueprint

### 1. Prepare persistent report storage

The VSS agent otherwise keeps generated Markdown and PDF files in memory, so
their download links stop working after an agent restart. The workshop uses a
single-node S3-compatible store backed by an AKS persistent disk. Create its
credentials once per namespace; retain the Secret and disk while reports must
remain available:

```bash
kubectl create namespace "$NAMESPACE" --dry-run=client -o yaml | kubectl apply -f -

if ! kubectl get secret vss-report-s3 -n "$NAMESPACE" >/dev/null 2>&1; then
  export REPORT_S3_ACCESS_KEY="$(openssl rand -hex 16)"
  export REPORT_S3_SECRET_KEY="$(openssl rand -hex 32)"
  kubectl create secret generic vss-report-s3 -n "$NAMESPACE" \
    --from-literal=AWS_ACCESS_KEY_ID="$REPORT_S3_ACCESS_KEY" \
    --from-literal=AWS_SECRET_ACCESS_KEY="$REPORT_S3_SECRET_KEY" \
    --dry-run=client -o yaml | kubectl apply -f -
  unset REPORT_S3_ACCESS_KEY REPORT_S3_SECRET_KEY
fi

sed "s/<NAMESPACE>/${NAMESPACE}/g" aks/storage/report-s3.yaml \
  | kubectl apply -f -
kubectl rollout status statefulset/vss-report-s3 -n "$NAMESPACE" --timeout=5m
kubectl get pvc data-vss-report-s3-0 -n "$NAMESPACE"
```

The PVC must be `Bound`. The storage manifest uses `managed-csi-premium`;
change its StorageClass if your cluster uses another class. Keep the Secret;
rotating its credentials without updating the store interrupts report access.
This single-node store is sized for a workshop; plan separate redundancy and
retention for longer-lived deployments.

### 2. Check out the tested release and apply the A100 and report-store patches

Run from this workshop directory. Keep `WORKSHOP_DIR` for the later commands.

```bash
export WORKSHOP_DIR="$PWD"
git clone --branch v3.3.0rc0 --single-branch \
  https://github.com/NVIDIA-AI-Blueprints/video-search-and-summarization.git
cd video-search-and-summarization

git apply "$WORKSHOP_DIR/aks/patches/nemotron-35-a100-cache-profile.patch"
git apply "$WORKSHOP_DIR/aks/patches/persistent-report-store.patch"
helm dependency build deploy/helm/services/nims --skip-refresh
helm dependency build deploy/helm/developer-profiles/dev-profile-lvs --skip-refresh
```

The release moved the LVS chart to `deploy/helm/developer-profiles/dev-profile-lvs`.
It uses **Nemotron 3.5 Lightning 30B-A3B** for the LLM and the integrated
**Cosmos 3 Nano Reasoner** checkpoint in RT-VLM. The old `llmNameSlug`,
`vlmNameSlug`, and Cosmos Reason2 A100 KV-cache overrides do not apply.

On A100, the unmodified Nemotron `NIMCache` selects multiple model profiles,
including NVFP4 profiles the hardware cannot serve, into its 100 GiB PVC.
The cache patch adds an optional `cacheModelProfile` selector; the workshop
values pin it to the INT4 profile also used by the NIMService. The report patch
switches the agent from in-memory storage to its included S3 object-store
provider. Both patches target `v3.3.0rc0` and are needed until equivalent
changes are included upstream.

The source tag pins the charts. Some RC images use `develop-latest`, so their
contents can change between deployments even with the source tag pinned.

### 3. Create NGC secrets outside Helm values

```bash
kubectl create namespace "$NAMESPACE" --dry-run=client -o yaml \
  | kubectl apply -f -

kubectl create secret generic ngc-api -n "$NAMESPACE" \
  --from-literal=NGC_API_KEY="$NGC_API_KEY" \
  --from-literal=NGC_CLI_API_KEY="$NGC_API_KEY" \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl create secret docker-registry ngc-secret -n "$NAMESPACE" \
  --docker-server=nvcr.io \
  --docker-username='$oauthtoken' \
  --docker-password="$NGC_API_KEY" \
  --dry-run=client -o yaml | kubectl apply -f -
```

The A100 values set `ngc.createSecrets: false`, keeping the key out of Helm
release values. Do not commit keys or rendered Secret manifests.

### 4. Install the LVS chart

```bash
cd deploy/helm/developer-profiles

helm upgrade --install "$RELEASE" ./dev-profile-lvs \
  -f ./dev-profile-lvs/values-lvs.yaml \
  -f "$WORKSHOP_DIR/aks/values/values-aks-a100.yaml" \
  --set global.externalHost="vss.${EXTERNAL_HOST}.nip.io" \
  --set global.kibanaPublicUrl="http://kibana.vss.${EXTERNAL_HOST}.nip.io" \
  -n "$NAMESPACE" --timeout 10m
```

`managed-csi-premium` is the tested Azure StorageClass. Check `kubectl get sc`
and change the workshop value if your cluster uses another class.

### 5. Wait for model cache and workloads

```bash
kubectl get nimcache,nimservice,pvc -n "$NAMESPACE"
kubectl get pods -n "$NAMESPACE"
```

Wait for Nemotron `NIMCache` and `NIMService` to report `Ready`, all PVCs to be
`Bound`, and long-running pods to be `Running` and Ready. First model download
and initialization can take several minutes. The four GPU workloads are
Nemotron NIM, RT-VLM, video summarization, and VIOS stream processing.

---

## Task 6: Apply the workshop ingress

The [workshop ingress](aks/ingress/vss-ingress.yaml) routes the 3.3 UI, agent,
WebSocket, VST, report artifacts, and Kibana to their current Service names.
It uses HAProxy's `haproxy` IngressClass.

```bash
cd "$WORKSHOP_DIR"
sed -e "s/<RELEASE_NAME>/${RELEASE}/g" \
    -e "s/<NAMESPACE>/${NAMESPACE}/g" \
    -e "s/<EXTERNAL_HOST>/${EXTERNAL_HOST}/g" \
    aks/ingress/vss-ingress.yaml \
  | kubectl apply -n "$NAMESPACE" -f -

kubectl get ingress -n "$NAMESPACE"
```

The UI hostname should be `vss.<EXTERNAL_HOST>.nip.io`; Kibana uses
`kibana.vss.<EXTERNAL_HOST>.nip.io`.

---

## Task 7: Verify and try guided prompts

```bash
echo "Open http://vss.${EXTERNAL_HOST}.nip.io/"
curl -sS -o /dev/null -w 'UI HTTP %{http_code}\n' \
  "http://vss.${EXTERNAL_HOST}.nip.io/"
kubectl get pods,nimcache,nimservice -n "$NAMESPACE"
```

Use a short, non-sensitive MP4 for the exercise:

1. In the UI, upload the clip and wait for its ingestion to finish. Confirm it
   appears in the video list and plays. Record the **sensor name** shown by VSS.
2. Start a new chat scoped to that sensor. Ask: **“Summarize this video and cite
   the timestamps of the main events.”** Compare the response with playback.
3. Ask: **“What changes between the beginning, middle, and end? Mention only
   things visible in this video.”** Check that the answer does not invent people,
   vehicles, or actions absent from the clip.
4. Ask: **“Generate a report for `<sensor name>` using video summarization.”**
   Use the sensor name exactly as VSS displays it, generally without `.mp4`.
   Accept or refine the report prompt, then open the Markdown/PDF and playback
   links. Confirm the report names the intended sensor and timestamps match.

If a report references a different sensor, start a new chat with the correct
sensor selected and repeat the request. Record the UI action, response, and
relevant pod logs if a step fails. The report generation and artifact links
were validated on the A100 deployment; browser upload should be confirmed in
each workshop run.

### Instructor readiness check: report persistence

Before attendees use a **dedicated** lab deployment, save both URLs from a
newly generated report, restart the agent, and confirm the **same URLs** still
download the Markdown and PDF. Do not restart the shared agent during an
attendee session.

```bash
kubectl rollout restart deployment/vss-agent -n "$NAMESPACE"
kubectl rollout status deployment/vss-agent -n "$NAMESPACE" --timeout=5m
curl -fsS -o /dev/null "<SAVED_MARKDOWN_URL>"
curl -fsS -o /dev/null "<SAVED_PDF_URL>"
```

On the isolated two-NC24 validation cluster, a report generated for a synthetic
12-second video returned HTTP 200 for both formats. The same URLs returned
identical files after restarting the agent and after restarting the storage
pod. This validates report persistence through those restarts, not a fresh
four-A100 deployment.

---

## Cleanup

Only clean up a dedicated workshop deployment after it is no longer needed.
Do not run these commands against a shared cluster or resource group.

```bash
helm uninstall "$RELEASE" -n "$NAMESPACE"
kubectl delete nimcache --all -n "$NAMESPACE"
kubectl delete statefulset,service vss-report-s3 -n "$NAMESPACE"
kubectl delete pvc --all -n "$NAMESPACE"
helm uninstall vss-haproxy -n vss-ingress
```

The chart marks NIMCache resources to survive Helm uninstall; their PVCs
remain until explicitly deleted. Removing the HAProxy Service also releases
its dynamic public IP. If you created a dedicated AKS cluster in Task 2,
remove that cluster separately after checking it contains no other workloads.
