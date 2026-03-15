# GEMINI Project Overview: Contabo VPS GitOps with ArgoCD

This document provides a comprehensive overview of the `contabo-vps-gitops-argocd` project, intended to guide future development and maintenance efforts.

## Project Overview

This repository implements a GitOps workflow to manage a personal Kubernetes (K3s) cluster hosted on a Contabo VPS. It uses ArgoCD to automatically sync the cluster's state with the configuration defined in this repository.

The core of the project follows the **"App of Apps" pattern**. A root ArgoCD application (`vps-jmsola-dev`) is responsible for deploying other ArgoCD `Application` resources, which in turn manage the individual components and applications running on the cluster.

### Key Technologies & Concepts

*   **Kubernetes:** K3s (a lightweight Kubernetes distribution).
*   **GitOps:** ArgoCD for continuous delivery and state synchronization.
*   **Configuration Management:** Kustomize is used to manage Kubernetes manifests, with a `base` and `overlay` structure for environment-specific configurations.
*   **Secret Management:** [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) is used to safely store encrypted secrets in a public Git repository.
*   **Applications:** The cluster hosts several applications, including:
    *   `cert-manager`: For automated TLS certificate management.
    *   `reflector`: To mirror secrets and configmaps across namespaces.
    *   `argocd-image-updater`: To automatically update container images.
    *   Custom applications like `mexc-crypto-bot`, `mexc-crypto-futures-bot`, and `openclaw`.

## Deployment Workflow

This project is fully automated through GitOps. There are no manual build or run commands for deployment.

1.  **Make Changes:** Modify or add Kubernetes manifests within the `argocd/manifests` directory or update application definitions in `argocd/apps`.
2.  **Commit and Push:** Push the changes to the `main` branch of the GitHub repository.
3.  **Automatic Sync:** ArgoCD detects the changes in the repository and automatically applies them to the Kubernetes cluster to match the desired state defined in Git.

### Local Interaction

*   **Cluster CLI:** Use `kubectl` configured with the appropriate `KUBECONFIG` file to interact with the cluster directly.
*   **ArgoCD CLI:** Use `argocd` to manage ArgoCD itself (e.g., check sync status, trigger manual syncs).
*   **Secret Management:** Use the `kubeseal` CLI to encrypt new secrets into `SealedSecret` resources. The process is documented in `README.md`.

## Development Conventions

### Directory Structure

The repository is organized to support the GitOps workflow and Kustomize's base/overlay pattern.

*   `argocd/project.yaml`: Defines the ArgoCD `AppProject` and the root `Application`.
*   `argocd/apps/`: Contains the ArgoCD `Application` manifest for each deployed component (the "apps" in the "App of Apps" pattern). Each file in this directory tells ArgoCD to manage a new application.
*   `argocd/manifests/`: Contains the raw Kubernetes manifests for the applications defined in `argocd/apps/`.
    *   `argocd/manifests/<app-name>/base/`: Holds the default, environment-agnostic Kubernetes resources for an application, along with a `kustomization.yaml`.
    *   `argocd/manifests/<app-name>/overlay/<env-name>/`: Contains environment-specific patches and configurations (e.g., for the `contabo` environment). The `kustomization.yaml` here points to the `base` and applies the patches.

### Adding a New Application

1.  **Create Manifests:** Create a new directory under `argocd/manifests/` for your application. Inside, create a `base` directory with your Kubernetes manifests (`deployment.yaml`, `service.yaml`, etc.) and a `kustomization.yaml` file listing these resources.
2.  **Create Overlays (if needed):** If you need environment-specific settings, create an `overlay/contabo` directory and add patches (e.g., for an `IngressRoute` or to add environment variables).
3.  **Define the ArgoCD Application:** Create a new YAML file in the `argocd/apps/` directory. This file will contain an `Application` custom resource that points to the manifest path you created in step 1 (e.g., `argocd/manifests/my-new-app/overlay/contabo`).
4.  **Commit and Push:** Commit all the new files and push them to the `main` branch. ArgoCD will automatically deploy your new application.
