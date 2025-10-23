# `vps.jmsola.dev` – Personal Project Hosting Environment

- [`vps.jmsola.dev` – Personal Project Hosting Environment](#vpsjmsoladev--personal-project-hosting-environment)
  - [Overview](#overview)
  - [📦 Prerequisites](#-prerequisites)
  - [🚀 Installation](#-installation)
    - [1. Connect to the VPS](#1-connect-to-the-vps)
    - [2. Install K3s (Lightweight Kubernetes)](#2-install-k3s-lightweight-kubernetes)
    - [3. Configure Remote `kubectl` Access](#3-configure-remote-kubectl-access)
      - [3.1. Export K3s Config File from VPS](#31-export-k3s-config-file-from-vps)
      - [3.2. Use KUBECONFIG](#32-use-kubeconfig)
  - [📥 ArgoCD Installation](#-argocd-installation)
    - [1. Install ArgoCD on the Cluster](#1-install-argocd-on-the-cluster)
    - [2. Access the ArgoCD UI](#2-access-the-argocd-ui)
    - [3. Configure ArgoCD CLI and Deploy a Project](#3-configure-argocd-cli-and-deploy-a-project)
    - [4. Install `kubeseal` for Managing Sealed Secrets](#4-install-kubeseal-for-managing-sealed-secrets)
    - [5. Give to `argocd-image-updater` our GitHub credentials](#5-give-to-argocd-image-updater-our-github-credentials)
    - [6. Install `Crypto Stop Loss` application](#6-install-crypto-stop-loss-application)
  - [🛡️ License](#️-license)


## Overview

This repository documents the setup process for provisioning and managing a personal VPS (`vps.jmsola.dev`) using [K3s](https://k3s.io/) and [ArgoCD](https://argo-cd.readthedocs.io/en/stable/). The goal is to maintain a lightweight, production-ready Kubernetes environment for hosting and deploying personal side projects.

---

## 📦 Prerequisites

- A VPS instance with root access (e.g., via SSH).
- [kubectl](https://kubernetes.io/docs/tasks/tools/) installed locally.
- [Homebrew](https://brew.sh) for installing CLI tools (on macOS).
- A GitHub Personal Access Token with repo access (for ArgoCD integration).
- Local `~/.kube` directory for storing Kubernetes config files.

---

## 🚀 Installation

### 1. Connect to the VPS

```bash
ssh root@vps.jmsola.dev
```

---

### 2. Install K3s (Lightweight Kubernetes)

Install K3s with custom TLS SANs to support domain and IP access:
```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="\
--tls-san vps.jmsola.dev \
--tls-san 207.180.239.230 \
--write-kubeconfig-mode=644 \
--disable metrics-server \
--kube-apiserver-arg event-ttl=1h \
--kube-scheduler-arg leader-elect=false \
--kubelet-arg container-log-max-size=10Mi \
--kubelet-arg container-log-max-files=3" sh -
```

Open necessary ports for Kubernetes API and HTTP/HTTPS traffic:

```bash
apt update && apt install -y netfilter-persistent

iptables -I INPUT -p tcp --dport 80 -m state --state NEW -j ACCEPT
iptables -I INPUT -p tcp --dport 443 -m state --state NEW -j ACCEPT
iptables -I INPUT -p tcp --dport 6443 -m state --state NEW -j ACCEPT

netfilter-persistent save
```

---

### 3. Configure Remote `kubectl` Access

#### 3.1. Export K3s Config File from VPS

On the VPS:

```bash
sudo cat /etc/rancher/k3s/k3s.yaml
```

Copy the output to your local machine and save it as:

```bash
~/.kube/vps.jmsola.dev_k3s-config.yaml
```

#### 3.2. Use KUBECONFIG

On your local machine:

```bash
export KUBECONFIG=$HOME/.kube/vps.jmsola.dev_k3s-config.yaml
```

You can now run `kubectl` commands against your K3s cluster.

---

## 📥 ArgoCD Installation

### 1. Install ArgoCD on the Cluster

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Once ArgoCD has been installed, it is important to apply some tuning stuff in order to avoid heavy CPU usage that is mainly provoked by `ArgoCD` itself, since it uses Kubernetes API so much, which is traduced into a heavy `k3s-server` heavy CPU usage.

To avoid it, exec

```bash
kubectl edit configmap argocd-cm -n argocd
```

Then, add under `data:` the following values: 

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  [...]
  timeout.reconciliation: 15m
  timeout.reconciliation.jitter: 60s
```

You should see at bash terminal something like: 

```bash
configmap/argocd-cm edited...
```

---

### 2. Access the ArgoCD UI

Retrieve the admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d | pbcopy
```

Port-forward the ArgoCD dashboard locally:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Visit [https://localhost:8080](https://localhost:8080) and log in as `admin` using the password above.

---

### 3. Configure ArgoCD CLI and Deploy a Project

Install and configure the ArgoCD CLI:

```bash
brew install argocd
export ARGOCD_OPTS='--insecure --port-forward-namespace argocd'
argocd login vps.jmsola.dev
```

Add your GitHub repository:

```bash
export GITHUB_USERNAME=jsoladur
export GITHUB_PERSONAL_ACCESS_TOKEN=<your_personal_access_token>

argocd repo add https://github.com/jsoladur/contabo-vps-gitops-argocd.git \
  --username $GITHUB_USERNAME \
  --password $GITHUB_PERSONAL_ACCESS_TOKEN

kubectl apply -f argocd/project.yaml
```

> ℹ️ You can create a GitHub Personal Access Token [here](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens).

---

### 4. Install `kubeseal` for Managing Sealed Secrets

To support encrypted Kubernetes secrets via the Sealed Secrets controller:

```bash
brew install kubeseal
```

Once you have installed kubeseal, we have to download the private key locally for being able to encrypt/decrypt secrets. 

```bash
kubeseal \
  --controller-name=sealed-secrets \
  --controller-namespace=toolbox \
  --fetch-cert > $HOME/.kube/vps-jmsola-dev-sealed-secrets.cert
```


### 5. Give to `argocd-image-updater` our GitHub credentials

```bash
export GITHUB_USERNAME=jsoladur
export GITHUB_PERSONAL_ACCESS_TOKEN=<your_personal_access_token>

kubectl create secret generic github-creds \
  --from-literal=username=${GITHUB_USERNAME} \
  --from-literal=password=${GITHUB_PERSONAL_ACCESS_TOKEN} \
  --namespace=argocd-image-updater \
  --dry-run=client -o yaml |
kubeseal \
  --format=yaml \
  --cert=$HOME/.kube/vps-jmsola-dev-sealed-secrets.cert > ./argocd/manifests/argocd-image-updater/base/github-creds-secret.yaml
```

### 6. Install `Crypto Stop Loss` application

To install Crypto Stop Loss application, we have to encrypt the `SealedSecret` properly. Therefore, it's needed to execute the following commands: 

```bash
export GOOGLE_OAUTH_CLIENT_ID=<value>
export GOOGLE_OAUTH_CLIENT_SECRET=<value>
export TELEGRAM_BOT_TOKEN=<value>
export BIT2ME_API_KEY=<value>
export BIT2ME_API_SECRET=<value>
export GEMINI_PRO_API_KEY=<value>

kubectl create secret generic crypto-stop-loss-bot \
  --from-literal=google.oauth.client.id=${GOOGLE_OAUTH_CLIENT_ID} \
  --from-literal=google.oauth.client.secret=${GOOGLE_OAUTH_CLIENT_SECRET} \
  --from-literal=telegram.bot.token=${TELEGRAM_BOT_TOKEN} \
  --from-literal=bit2me.api.key=${BIT2ME_API_KEY} \
  --from-literal=bit2me.api.secret=${BIT2ME_API_SECRET} \
  --from-literal=gemini.pro.api.key=${GEMINI_PRO_API_KEY} \
  --namespace=crypto-stop-loss-bot \
  --dry-run=client -o yaml |
kubeseal \
  --format=yaml \
  --cert=$HOME/.kube/vps-jmsola-dev-sealed-secrets.cert > ./argocd/manifests/crypto-stop-loss-bot/base/secret.yaml
```

For MEXC:

```bash
export GOOGLE_OAUTH_CLIENT_ID=<value>
export GOOGLE_OAUTH_CLIENT_SECRET=<value>
export TELEGRAM_BOT_TOKEN=<value>
export MEXC_API_KEY=<value>
export MEXC_API_SECRET=<value>

kubectl create secret generic mexc-crypto-bot \
  --from-literal=google.oauth.client.id=${GOOGLE_OAUTH_CLIENT_ID} \
  --from-literal=google.oauth.client.secret=${GOOGLE_OAUTH_CLIENT_SECRET} \
  --from-literal=telegram.bot.token=${TELEGRAM_BOT_TOKEN} \
  --from-literal=mexc.api.key=${MEXC_API_KEY} \
  --from-literal=mexc.api.secret=${MEXC_API_SECRET} \
  --namespace=mexc-crypto-bot \
  --dry-run=client -o yaml |
kubeseal \
  --format=yaml \
  --cert=$HOME/.kube/vps-jmsola-dev-sealed-secrets.cert > ./argocd/manifests/mexc-crypto-bot/base/secret.yaml
```

---

## 🛡️ License

This setup is intended for personal use. Feel free to fork and adapt it to suit your own deployment needs.
