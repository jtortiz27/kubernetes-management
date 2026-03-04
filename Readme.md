# Private “ChatGPT-like” AI Gateway (OpenClaw + Ollama) — Helm / k3s

This repo is a **plug-and-play kit** for running an AI chatbot “brain” **inside your own infrastructure** (Kubernetes), instead of sending prompts to OpenAI or another external service.

It deploys:
- **Ollama** (local model runtime) to run LLMs on your own nodes
- **OpenClaw Gateway** as a single **ChatGPT-like API front door** that apps can call
- Optional **basic observability** (OpenTelemetry Collector + Prometheus-style metrics endpoint)

---

## What this is trying to accomplish

### Goals
- **Private AI service**: answer questions using a model running on *your* servers
- **Repeatable installs**: the same deployment can be turned on in new environments with minimal manual work
- **Auto model download (first start)**: optionally **pre-pull** the model when the pod starts so it’s ready quickly
- **Persist model files**: keep downloaded model data on disk so restarts don’t re-download
- **One API endpoint**: expose a single gateway service other apps integrate with (like a ChatGPT/OpenAI-style base URL)
- **Health + performance visibility**: basic monitoring hooks and metrics

**In one sentence:** this repo provides a reliable, repeatable way to run a “ChatGPT-like” AI service privately on your own infrastructure, with minimal setup hassle.

---

## High-level architecture

- **OpenClaw Gateway**
  - Exposes an HTTP API (your “front door”)
  - Talks to Ollama using an OpenAI-compatible base URL (`/v1`)
  - Uses a shared token for basic request gating (configure via env)

- **Ollama**
  - Runs locally in-cluster
  - Stores models on a mounted volume (PVC) so they persist

- **Observability (optional)**
  - OpenTelemetry Collector for traces/metrics export
  - Metrics endpoint enabled for scraping

---

## Key configuration knobs (typical)

You can configure (via Helm values):
- Which model to use (example: `qwen2.5:7b`)
- Whether to **pre-pull** models on start
- Persistence sizes for:
  - Ollama model storage (often large)
  - OpenClaw config storage (small)
- Service exposure (NodePort by default in examples)
- OpenTelemetry endpoints / toggles

---

## Deploying to a k3s cluster (with dependency setup)

### 0) Prereqs on your machine (client tools)

Install `kubectl` and `helm`:
```zsh
# kubectl (example for Linux; use your OS package manager as preferred)
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```
### 1) Install k3s (server)

On the machine that will host k3s:
```zsh
curl -sfL https://get.k3s.io | sh -
sudo systemctl status k3s
```
Copy kubeconfig so your user (or remote machine) can access the cluster:
```zsh
sudo cat /etc/rancher/k3s/k3s.yaml
```
If you’re using `kubectl` locally on the k3s server:
```zsh
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown "$(id -u)":"$(id -g)" ~/.kube/config
kubectl get nodes
```
### 2) Confirm default storage class (k3s local-path)

k3s typically installs the `local-path` StorageClass automatically.
```zsh
kubectl get storageclass
```
You should see something like `local-path`. If you don’t have a default StorageClass, set one (adjust name as needed):
```zsh
kubectl patch storageclass local-path -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```
### 3) Create a namespace
```zsh
kubectl create namespace openclaw
```
### 4) Install the chart into k3s

From the repo root (where the Helm chart lives), install with Helm. Example:
```zsh
helm install openclaw-ollama . \
--namespace openclaw \
--create-namespace
```
If you have an environment-specific values file (recommended), apply it like:
```zsh
helm upgrade --install openclaw-ollama . \
--namespace openclaw \
-f values.yaml
```
For a “local/dev” style setup (example values file), you can do:
```zsh
helm upgrade --install openclaw-ollama . \
--namespace openclaw \
-f values-local-macmini.yaml
```
### 5) Watch the rollout
```zsh
kubectl -n openclaw get pods -w
kubectl -n openclaw get svc
kubectl -n openclaw get pvc
```
### 6) Access the Gateway API

If the service is exposed via **NodePort**, find the NodePort and node IP:
```zsh
kubectl -n openclaw get svc
kubectl get nodes -o wide
```
Then call the gateway (replace `<NODE_IP>` and `<NODE_PORT>`):
```zsh
curl http://<NODE_IP>:<NODE_PORT>/
```
If your gateway expects a token, pass it as your deployment requires (commonly via an `Authorization` header or similar pattern in your client).

---

## Model download + persistence behavior

- **Pre-pull enabled**: the deployment can pull one or more models during pod startup so it’s ready immediately.
- **Persistence enabled**: model data is stored in a PVC so restarts don’t require re-downloading.

Typical operational notes:
- Model storage can be large (tens of GB depending on model(s)).
- If you change models frequently, plan storage accordingly.

---

## Observability / metrics

If enabled:
- The stack exposes a Prometheus-style metrics endpoint (path typically `/metrics`)
- An OpenTelemetry Collector can run alongside to receive OTLP traffic

You can validate endpoints by checking services and pod logs:
```zsh
kubectl -n openclaw logs deploy/openclaw-ollama --all-containers=true --tail=200
kubectl -n openclaw get svc
```
---

## Common operations

### Upgrade with new values
```zsh
helm upgrade --install openclaw-ollama . \
--namespace openclaw \
-f values.yaml
```
### Uninstall
```zsh
helm uninstall openclaw-ollama -n openclaw
```
If you want to delete persisted model/config data too (irreversible), delete PVCs:
```zsh
kubectl -n openclaw delete pvc --all
```
---

## Security notes (baseline)

This kit is meant to run privately, but you should still:
- Put the Gateway behind your preferred ingress / auth layer for production
- Treat tokens/secrets as Kubernetes Secrets (not plaintext values) for real deployments
- Apply network policies if your cluster supports them
```