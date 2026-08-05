# Local Setup Guide — AI Platform Engineering
## Windows + WSL2 + Docker Desktop · 16GB RAM

---

## Your Environment Summary

| Component | Status |
|---|---|
| OS | Windows + WSL2 |
| RAM | 16 GB |
| GPU | None (Intel i5) |
| Docker Desktop | Running |
| GPU Labs | Google Colab (Path A) |

---

## WSL2 Resource Limit (Do This First)

By default WSL2 can consume all 16GB. Cap it so Windows stays stable.

Create `C:\Users\<YourUsername>\.wslconfig`:

```ini
[wsl2]
memory=10GB
processors=4
swap=4GB
localhostForwarding=true
```

Restart WSL2:
```powershell
wsl --shutdown
wsl
```

---

## Tool Installation Order

Install everything inside WSL2 Ubuntu terminal, not Windows PowerShell.

---

### 1. Core CLI Tools

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git unzip jq make build-essential

# Verify
git --version
curl --version
jq --version
```

---

### 2. kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl

# Verify
kubectl version --client
```

---

### 3. Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify
helm version
```

---

### 4. Kind (already installed ✓)

Kind is already installed. Verify it works with Docker Desktop:

```bash
# Verify
kind version

# Quick smoke test
kind create cluster --name test
kubectl get nodes
kind delete cluster --name test
```

You should see one node in Ready state before deleting.

---

### 5. k9s (Kubernetes TUI — you will use this constantly)

```bash
curl -sS https://webinstall.dev/k9s | bash
source ~/.config/envman/PATH.env

# Verify
k9s version
```

---

### 6. Terraform

```bash
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install -y terraform

# Verify
terraform version
```

---

### 7. ArgoCD CLI

```bash
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64

# Verify
argocd version --client
```

---

### 8. Trivy (container scanning)

```bash
sudo apt install -y apt-transport-https gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo "deb https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt update && sudo apt install -y trivy

# Verify
trivy --version
```

---

### 9. Cosign (image signing)

```bash
curl -O -L "https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64"
sudo mv cosign-linux-amd64 /usr/local/bin/cosign
sudo chmod +x /usr/local/bin/cosign

# Verify
cosign version
```

---

### 10. Gitleaks (secret scanning)

```bash
GITLEAKS_VERSION=$(curl -s https://api.github.com/repos/gitleaks/gitleaks/releases/latest | jq -r .tag_name)
curl -LO "https://github.com/gitleaks/gitleaks/releases/download/${GITLEAKS_VERSION}/gitleaks_${GITLEAKS_VERSION#v}_linux_x64.tar.gz"
tar -xzf gitleaks_*.tar.gz gitleaks
sudo mv gitleaks /usr/local/bin/
rm gitleaks_*.tar.gz

# Verify
gitleaks version
```

---

### 11. Syft (SBOM generation)

```bash
curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b /usr/local/bin

# Verify
syft --version
```

---

### 12. Ollama (local LLMs — CPU mode)

```bash
curl -fsSL https://ollama.com/install.sh | sh

# Pull a small model to test (1.3B — fast on CPU)
ollama pull llama3.2:1b

# Verify
ollama run llama3.2:1b "say hello in one sentence"
```

---

### 13. Python + pip

```bash
sudo apt install -y python3 python3-pip python3-venv

# Verify
python3 --version
pip3 --version
```

---

### 14. MLflow

```bash
pip3 install mlflow --break-system-packages

# Verify
mlflow --version
```

---

### 15. GitHub CLI

```bash
type -p curl >/dev/null || sudo apt install curl -y
curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | sudo dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null
sudo apt update && sudo apt install -y gh

# Authenticate
gh auth login

# Verify
gh --version
```

---

## Kind Cluster Strategy (RAM Management)

Never run everything at once. Create a named cluster per project, delete when done.
Kind has no "stop" — you either keep it or delete it.

```bash
# Create a cluster for a lab (single node default)
kind create cluster --name ai-platform

# Create a multi-node cluster (used from Week 6+)
# Kind config files are provided per lab
kind create cluster --name ai-platform --config kind-config.yaml

# List running clusters
kind get clusters

# Switch kubectl context between clusters
kubectl cluster-info --context kind-ai-platform

# Delete when moving to next project (frees all RAM)
kind delete cluster --name ai-platform
```

### Example multi-node Kind config (provided per lab)

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
    labels:
      node-type: gpu-worker   # simulated GPU node
  - role: worker
    labels:
      node-type: cpu-worker
```

---

## Lab-by-Lab Resource Guide

| Week | Project | Kind RAM | Nodes | Extra Services | Notes |
|---|---|---|---|---|---|
| 1 | ai-platform-foundation | 4GB | 1 control + 2 workers | None | Safe to run alongside Docker Desktop |
| 2 | gitops-ai-platform | 5GB | 1 control + 2 workers | ArgoCD | Delete week 1 cluster first |
| 3 | policy-as-code-ai | 4GB | 1 control + 1 worker | Kyverno / Gatekeeper | Run one policy engine at a time |
| 4 | secure-ai-supply-chain | 3GB | 1 control | None | Mostly CLI tools, lightest week |
| 5 | mlops-platform | 6GB | 1 control + 2 workers | MLflow + MinIO | Heaviest so far. Close all browsers. |
| 6 | model-serving-platform | 6GB | 1 control + 2 workers | KServe | **PATH A — GPU parts on Colab** |
| 7 | ai-observability-stack | 6GB | 1 control + 2 workers | Prometheus + Grafana + Loki | Run in evening, close other apps |
| 8 | distributed-training-platform | 5GB | 1 control + 3 workers | Ray | **PATH A — multi-GPU on Colab** |
| 9 | ai-security-platform | 5GB | 1 control + 2 workers | Falco + Tetragon | eBPF needs kernel 5.15+ (WSL2 has it) |
| 10 | ai-agent-platform | 4GB | 1 control + 1 worker | Ollama | LLM runs locally on CPU (slow but works) |
| 11 | internal-ai-developer-platform | 6GB | 1 control + 2 workers | Backstage + Crossplane | Heaviest week. Close everything else. |
| 12 | ai-model-governance | 4GB | 1 control + 1 worker | MLflow | Wraps up all tools |

---

## GPU Labs — Path A (Google Colab)

The following labs have GPU components. Use Google Colab free tier (T4 GPU).

| Week | Lab | What Runs on Colab |
|---|---|---|
| 6 | model-serving-platform | Triton Inference Server, vLLM serving a 7B model |
| 7 | ai-observability-stack | GPU metrics collection with DCGM exporter |
| 8 | distributed-training-platform | Ray multi-GPU training job |
| 12 | ai-platform-benchmarking | vLLM vs Triton throughput benchmark |

### Colab Setup (do once)

1. Go to [colab.research.google.com](https://colab.research.google.com)
2. Runtime → Change runtime type → T4 GPU
3. Each GPU lab will include a ready-to-run Colab notebook

---

## Verify Everything Is Working

Run this after all installs:

```bash
echo "=== kubectl ===" && kubectl version --client
echo "=== helm ===" && helm version --short
echo "=== kind ===" && kind version
echo "=== terraform ===" && terraform version
echo "=== argocd ===" && argocd version --client
echo "=== trivy ===" && trivy --version
echo "=== cosign ===" && cosign version
echo "=== gitleaks ===" && gitleaks version
echo "=== syft ===" && syft --version
echo "=== ollama ===" && ollama --version
echo "=== mlflow ===" && mlflow --version
echo "=== gh ===" && gh --version
echo "=== k9s ===" && k9s version
echo "=== python ===" && python3 --version
echo "=== docker ===" && docker --version
echo ""
echo "All tools verified."
```

---

## VS Code Setup (Recommended)

Install VS Code on Windows, then install these extensions:

```
- Remote - WSL (connect VS Code to your WSL2 environment)
- Kubernetes (browse clusters, view pods/logs)
- YAML (schema validation for K8s manifests)
- HashiCorp Terraform
- GitLens
- Docker
- GitHub Actions
```

Open your WSL2 workspace from VS Code:
```bash
# Inside WSL2
cd ~
mkdir ai-platform-engineering
code .   # opens VS Code connected to WSL2
```

---

## GitHub Setup

Create one GitHub organisation or use your personal account.
Name your repos exactly as listed in the project list.

```bash
# Configure git inside WSL2
git config --global user.name "Your Name"
git config --global user.email "your@email.com"
git config --global init.defaultBranch main

# Authenticate GitHub CLI
gh auth login
```

---

## You Are Ready When

- [ ] All tools print a version without errors
- [ ] `kind create cluster --name test` succeeds
- [ ] `kubectl get nodes` shows one node Ready
- [ ] `kind delete cluster --name test` cleans up
- [ ] `helm list` returns without error
- [ ] VS Code opens with Remote WSL connected
- [ ] GitHub CLI authenticated
- [ ] Ollama returns a response from llama3.2:1b

Once all boxes are checked — say **"setup done"** and we start Week 1 Lesson 1.