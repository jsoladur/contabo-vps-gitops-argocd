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
    - [6. Install `Crypto Spot Bot` application](#6-install-crypto-spot-bot-application)
    - [6. Install `Crypto Futures Bot` application](#6-install-crypto-futures-bot-application)
    - [7. Install `OpenClaw` AI assistant](#7-install-openclaw-ai-assistant)
      - [7.1. Prepare Google Cloud OAuth Credentials](#71-prepare-google-cloud-oauth-credentials)
      - [7.2. Run the OAuth Flow Locally](#72-run-the-oauth-flow-locally)
      - [7.3. Bundle the Keyring](#73-bundle-the-keyring)
      - [7.4. Create the SealedSecret](#74-create-the-sealedsecret)
      - [7.5. Deploy and Verify](#75-deploy-and-verify)
      - [7.6. Approve Device Pairing](#76-approve-device-pairing)
      - [7.7. Re-authorizing (Token Rotation)](#77-re-authorizing-token-rotation)
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


Visit https://localhost:8080 and log in as `admin` using the password above.


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


### 6. Install `Crypto Spot Bot` application


To install Crypto Spot Bot application, we have to encrypt the `SealedSecret` properly. Therefore, it's needed to execute the following commands:


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


### 6. Install `Crypto Futures Bot` application


To install Crypto Futures Bot application, we have to encrypt the `SealedSecret` properly. Therefore, it's needed to execute the following commands:


```bash
export ROOT_USER=<value>
export ROOT_PASSWORD=<value>
export TELEGRAM_BOT_TOKEN=<value>
export MEXC_API_KEY=<value>
export MEXC_API_SECRET=<value>
export MEXC_WEB_AUTH_TOKEN=<value>

kubectl create secret generic mexc-crypto-futures-bot \
  --from-literal=root.user=${ROOT_USER} \
  --from-literal=root.password=${ROOT_PASSWORD} \
  --from-literal=telegram.bot.token=${TELEGRAM_BOT_TOKEN} \
  --from-literal=mexc.api.key=${MEXC_API_KEY} \
  --from-literal=mexc.api.secret=${MEXC_API_SECRET} \
  --from-literal=mexc.web.auth.token=${MEXC_WEB_AUTH_TOKEN} \
  --namespace=mexc-crypto-futures-bot \
  --dry-run=client -o yaml |
kubeseal \
  --format=yaml \
  --cert=$HOME/.kube/vps-jmsola-dev-sealed-secrets.cert > ./argocd/manifests/mexc-crypto-futures-bot/base/secret.yaml
```


### 7. Install `OpenClaw` AI assistant


OpenClaw uses [`gogcli`](https://github.com/steipete/gogcli) to access Gmail and Google Calendar on your behalf. Because the pod runs headlessly in Kubernetes, the full Google OAuth flow must be completed **locally first**, and the resulting token keyring must be baked into the `openclaw` SealedSecret alongside the other credentials.

#### 7.1. Prepare Google Cloud OAuth Credentials

1. Go to [Google Cloud Console](https://console.cloud.google.com) → **APIs & Services → Credentials**.
2. Click **Create Credentials → OAuth client ID**.
3. Set **Application type = Desktop app** — this is required so that `gog` can use its loopback redirect URI (`http://127.0.0.1:<port>`). Using **Web application** will produce a `redirect_uri_mismatch` error.
4. Download the generated JSON file (e.g. `google_client_secret.json`). Its structure must have a root key `"installed"`:
   ```json
   {
     "installed": {
       "client_id": "XXXXXXXX.apps.googleusercontent.com",
       "client_secret": "XXXXXXXX",
       "redirect_uris": ["http://localhost"],
       ...
     }
   }
   ```
5. Go to **APIs & Services → OAuth consent screen → Test users** and add your Gmail address (e.g. `josemaria.sola.duran@gmail.com`). Without this, the OAuth flow will return `Error 403: access_denied`.

> ℹ️ Make sure **Gmail API** and **Google Calendar API** are enabled in your project under **APIs & Services → Enabled APIs**.

#### 7.2. Run the OAuth Flow Locally

Download `gogcli` and complete the interactive OAuth authorization on your local machine. This generates the encrypted token keyring directory that will be injected into the cluster.

> ⚠️ **Critical:** The order of these commands matters. You must set `XDG_CONFIG_HOME`, switch the keyring backend to `file`, and export `GOG_KEYRING_PASSWORD` **before** running `gog auth add`. This password encrypts the token files on disk and must be stored in Kubernetes so the pod can decrypt them at runtime.

```bash
# Download gogcli locally (macOS/Linux)
wget -qO- https://github.com/steipete/gogcli/releases/download/v0.11.0/gogcli_0.11.0_linux_amd64.tar.gz \
  | tar -xz -C /usr/local/bin gog && chmod +x /usr/local/bin/gog

# 1. Use an isolated config directory to avoid polluting your local gog setup
export XDG_CONFIG_HOME=/tmp/gog-bootstrap
mkdir -p $XDG_CONFIG_HOME/gogcli

# 2. Switch keyring backend to file (required for headless/container environments)
gog auth keyring file

# 3. Set a strong passphrase — this encrypts the token files on disk.
#    Save this value: you will need it in the SealedSecret in step 7.4.
export GOG_KEYRING_PASSWORD=$(openssl rand -hex 24)

# 4. Register the OAuth client credentials using the original file downloaded
#    from Google Cloud Console (must have "installed" root key — NOT the
#    credentials.json that gogcli generates internally)
gog auth credentials /path/to/google_client_secret.json

# 5. Complete the interactive browser-based OAuth flow
#    A browser window will open — log in and grant access
gog auth add josemaria.sola.duran@gmail.com --services gmail,calendar

# 6. Verify tokens were stored successfully
gog auth status
```

After a successful `gog auth status` you should see `josemaria.sola.duran@gmail.com` listed with `gmail, calendar` scopes active. The encrypted token keyring is now at:

```
/tmp/gog-bootstrap/gogcli/keyring/
├── token:default:josemaria.sola.duran@gmail.com
└── token:josemaria.sola.duran@gmail.com
```

> ⚠️ **Important:** The keyring is a **directory** containing two token files whose names include colons. Do not try to mount individual files — bundle the whole directory as a tar instead (see next step).

#### 7.3. Bundle the Keyring

Because the token filenames contain colons (`:`) — which are incompatible with Kubernetes `subPath` mounts — the entire keyring directory must be bundled as a tar archive:

```bash
tar -czf /tmp/gog-keyring.tar.gz -C /tmp/gog-bootstrap/gogcli keyring

# Verify contents
tar -tzf /tmp/gog-keyring.tar.gz
# Expected output:
# keyring/
# keyring/token:default:josemaria.sola.duran@gmail.com
# keyring/token:josemaria.sola.duran@gmail.com
```

#### 7.4. Create the SealedSecret

All credentials — including the OAuth token keyring tar and the keyring password — are stored in the single `openclaw` secret.

- `GOOGLE_CLIENT_SECRET_PATH` must point to the **original file downloaded from Google Cloud Console** (the one with the `"installed"` root key), not the `credentials.json` that gogcli generates internally.
- `GOG_KEYRING_PASSWORD` is the same passphrase you set in step 7.2 — the pod needs it to decrypt the token files at runtime.

```bash
export GEMINI_API_KEY=<value>
export GATEWAY_TOKEN=<value>
export HOOKS_TOKEN=<value>
export TELEGRAM_BOT_TOKEN=<value>
export WHATSAPP_PHONE_NUMBER=<value>
export GOOGLE_CLIENT_SECRET_PATH=<path/to/original/google_client_secret.json>
export GOG_KEYRING_TAR_PATH=/tmp/gog-keyring.tar.gz
export GOG_KEYRING_PASSWORD="a-strong-passphrase"   # same value used in step 7.2

kubectl create secret generic openclaw \
  --from-literal=google.gemini.api.key=${GEMINI_API_KEY} \
  --from-literal=gateway.token=${GATEWAY_TOKEN} \
  --from-literal=hooks.token=${HOOKS_TOKEN} \
  --from-literal=telegram.bot.token=${TELEGRAM_BOT_TOKEN} \
  --from-literal=whatsapp.phone.number=${WHATSAPP_PHONE_NUMBER} \
  --from-literal=gog.keyring.password=${GOG_KEYRING_PASSWORD} \
  --from-file=google_client_secret.json=${GOOGLE_CLIENT_SECRET_PATH} \
  --from-file=gog.keyring.tar.gz=${GOG_KEYRING_TAR_PATH} \
  --namespace=openclaw \
  --dry-run=client -o yaml | \
kubeseal \
  --format=yaml \
  --cert=$HOME/.kube/vps-jmsola-dev-sealed-secrets.cert > ./argocd/manifests/openclaw/base/secret.yaml
```

> ⚠️ The keyring tar contains your live OAuth refresh token. Treat it with the same sensitivity as a password. Never commit the raw (unsealed) secret YAML to the repository.

#### 7.5. Deploy and Verify

The Deployment init container (`setup-gog-credentials`) will automatically:
1. Extract the keyring tar into `/home/node/.openclaw/.config/gogcli/keyring/`
2. Register the OAuth client credentials via `gog auth credentials`

The following environment variables are set on both the init container and the main container to ensure gogcli operates correctly in the headless pod environment:

- `GOG_KEYRING_BACKEND=file` — use the on-disk file keyring instead of a system keychain
- `GOG_KEYRING_PASSWORD` — injected from the secret to decrypt the token files

Access tokens expire every ~1 hour but are **refreshed automatically and silently** using the stored refresh token and the OAuth client credentials — no human interaction or pod restart is ever needed.

Once ArgoCD syncs the application, verify that the pod sees the tokens correctly:

```bash
# Watch the rollout
kubectl rollout status deployment/openclaw -n openclaw

# Verify gog can see the authorized account inside the running pod
kubectl exec -n openclaw -it deploy/openclaw -- gog auth status
```

You should see `josemaria.sola.duran@gmail.com` listed as authenticated with `gmail, calendar` scopes.

#### 7.6. Approve Device Pairing

Once the application is deployed, connect via https://openclaw.vps.jmsola.dev/. You will need the `GATEWAY_TOKEN` defined above to pair your device. To approve the pairing, execute:

```bash
kubectl exec -n openclaw -it deploy/openclaw -- openclaw devices approve --latest
```

#### 7.7. Re-authorizing (Token Rotation)

The refresh token stored in the keyring is permanent and will not expire under normal usage. The only scenarios requiring re-authorization are:

- You manually revoke OpenClaw's access at [myaccount.google.com/permissions](https://myaccount.google.com/permissions)
- Google invalidates the token after 6 months of complete inactivity
- You delete or recreate the OAuth client in Google Cloud Console

In any of these cases, repeat steps **7.2 → 7.4** and push the updated `secret.yaml` to the GitOps repo. ArgoCD will automatically re-sync and roll out the updated secret.

---


## 🛡️ License


This setup is intended for personal use. Feel free to fork and adapt it to suit your own deployment needs.
