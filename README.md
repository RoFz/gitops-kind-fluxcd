# gitops-kind-fluxcd

Reference implementation for running a local Kubernetes cluster with [kind](https://kind.sigs.k8s.io/), managed via [FluxCD](https://fluxcd.io/) using a structured Kustomize overlay layout with cert-manager TLS and Traefik ingress.

## Overview

This repository demonstrates how to manage a local kind cluster entirely through GitOps using FluxCD. The structure follows the [flux2-kustomize-helm-example](https://github.com/fluxcd/flux2-kustomize-helm-example) reference architecture with infrastructure controllers, infrastructure configs, and applications as separate Kustomization layers with explicit dependency ordering.

### Deployment layers

```text
infra-controllers     (Helm releases: Traefik, cert-manager, Vault, Prometheus, CSI driver)
       |
       v (dependsOn)
infra-configs         (ClusterIssuer, TLS options, Traefik middlewares and certificates)
       |
       v (dependsOn)
apps                  (podinfo, kubernetes-dashboard, simple-vault-client)
```

### Infrastructure controllers

| Component | Namespace | Chart |
| --- | --- | --- |
| Traefik | traefik | traefik/traefik |
| cert-manager | cert-manager | jetstack/cert-manager |
| kube-prometheus-stack | monitoring | prometheus-community/kube-prometheus-stack |
| Vault (HA, Raft) | vault | hashicorp/vault |
| Secrets Store CSI Driver | kube-system | secrets-store-csi-driver |
| metrics-server (optional) | kube-system | metrics-server/metrics-server |

### Applications

| App | Namespace | URL |
| --- | --- | --- |
| podinfo | podinfo | <https://podinfo.k8s.example.internal> |
| kubernetes-dashboard | kubernetes-dashboard | <https://dashboard.k8s.example.internal> |
| simple-vault-client | vault-demo | http://\<cluster-ip\>/simple-vault-client |
| kafka (commented out) | kafka | - |

### Key patterns

- **Kustomize overlays**: `apps/base/` contains base manifests; `apps/production/` applies environment-specific patches
- **HelmRelease with remediation**: all Helm releases configure install/upgrade retry policies
- **TLS via cert-manager**: wildcard certificate for `*.k8s.example.internal` set as Traefik default certificate
- **TLS options**: global TLS 1.2+ policy and cipher suite restrictions applied via Traefik `TLSOption`
- **Secure headers middleware**: `secure-headers-all` middleware strips server-identifying response headers
- **Ingress**: standard Kubernetes `Ingress` resources with Traefik annotations (as opposed to IngressRoute CRDs)

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/)
- [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [flux CLI](https://fluxcd.io/flux/installation/)
- A GitHub personal access token with `repo` scope (for bootstrapping)
- An internal CA (root or intermediate) with the certificate and private key available locally

## Getting started

### 1. Create the kind cluster

```bash
kind create cluster --name my-cluster --config configs/kind-config-my-cluster.yaml
```

### 2. Create the internal CA secret

```bash
kubectl create namespace cert-manager
kubectl create secret tls internal-ca \
  --cert=path/to/ca.crt \
  --key=path/to/ca.key \
  --namespace cert-manager
```

### 3. Bootstrap Flux

```bash
export GITHUB_TOKEN=<your-token>

flux bootstrap github \
  --owner=<your-github-username> \
  --repository=gitops-kind-fluxcd \
  --branch=main \
  --path=./fluxcd/clusters/kind-my-cluster \
  --personal
```

Flux will install its own components, commit `gotk-components.yaml` to the repository, and begin reconciling the cluster state.

### 4. Add /etc/hosts entries

```text
127.0.0.1   traefik.k8s.example.internal
127.0.0.1   dashboard.k8s.example.internal
127.0.0.1   podinfo.k8s.example.internal
127.0.0.1   grafana.k8s.example.internal
127.0.0.1   prometheus.k8s.example.internal
127.0.0.1   vault.k8s.example.internal
```

### 5. Vault post-install steps

Vault requires manual initialisation after deployment:

```bash
export VAULT_K8S_NAMESPACE=vault

# Initialise
kubectl exec -n $VAULT_K8S_NAMESPACE vault-0 -- vault operator init \
    -key-shares=1 -key-threshold=1 -format=json > /tmp/vault-init/cluster-keys.json

# Unseal both pods
export VAULT_UNSEAL_KEY=$(jq -r ".unseal_keys_b64[]" /tmp/vault-init/cluster-keys.json)
kubectl exec -n $VAULT_K8S_NAMESPACE vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY
kubectl exec -n $VAULT_K8S_NAMESPACE vault-1 -- vault operator unseal $VAULT_UNSEAL_KEY

# Enable kv-v2
export CLUSTER_ROOT_TOKEN=$(jq -r ".root_token" /tmp/vault-init/cluster-keys.json)
kubectl exec -n $VAULT_K8S_NAMESPACE vault-0 -- vault login $CLUSTER_ROOT_TOKEN
kubectl exec -n $VAULT_K8S_NAMESPACE vault-0 -- vault secrets enable -path=secret kv-v2
```

> Store `/tmp/vault-init/cluster-keys.json` securely. Never commit it to this repository.

## Repository structure

```text
gitops-kind-fluxcd/
├── configs/
│   └── kind-config-my-cluster.yaml
└── fluxcd/
    ├── apps/
    │   ├── base/
    │   │   ├── kafka/
    │   │   │   ├── kustomization.yaml
    │   │   │   ├── namespace.yaml
    │   │   │   ├── release.yaml
    │   │   │   └── repository.yaml
    │   │   ├── kubernetes-dashboard/
    │   │   │   ├── ingress.yaml
    │   │   │   ├── kustomization.yaml
    │   │   │   ├── namespace.yaml
    │   │   │   ├── release.yaml
    │   │   │   ├── repository.yaml
    │   │   │   └── serviceaccounts.yaml
    │   │   ├── podinfo/
    │   │   │   ├── ingress.yaml
    │   │   │   ├── kustomization.yaml
    │   │   │   ├── namespace.yaml
    │   │   │   ├── release.yaml
    │   │   │   └── repository.yaml
    │   │   └── simple-vault-client/
    │   │       ├── deployment.yaml
    │   │       ├── ingress.yaml
    │   │       ├── kustomization.yaml
    │   │       ├── namespace.yaml
    │   │       ├── service.yaml
    │   │       └── serviceaccount.yaml
    │   └── production/
    │       ├── kustomization.yaml
    │       └── podinfo-values.yaml
    ├── clusters/
    │   └── kind-my-cluster/
    │       ├── flux-system/
    │       │   ├── gotk-components.yaml
    │       │   ├── gotk-sync.yaml
    │       │   └── kustomization.yaml
    │       ├── apps.yaml
    │       └── infrastructure.yaml
    └── infrastructure/
        ├── configs/
        │   ├── cert-manager.yaml
        │   ├── kustomization.yaml
        │   └── traefik.yaml
        └── controllers/
            ├── cert-manager.yaml
            ├── grafana.yaml
            ├── kustomization.yaml
            ├── metrics-server.yaml
            ├── prometheus-stack.yaml
            ├── secrets-store-csi-driver.yaml
            ├── traefik.yaml
            └── vault.yaml
```

## Adding a new application

1. Create a folder under `fluxcd/apps/base/<app-name>/` with `namespace.yaml`, `kustomization.yaml`, and Helm or raw manifests
2. Add the folder to `fluxcd/apps/production/kustomization.yaml`
3. Add environment-specific patches in `fluxcd/apps/production/` if needed
4. Add the hostname to `/etc/hosts`

## Adding a new infrastructure controller

1. Create `fluxcd/infrastructure/controllers/<name>.yaml` with `HelmRepository` and `HelmRelease`
2. Add it to `fluxcd/infrastructure/controllers/kustomization.yaml`
3. If it needs post-install configuration, add a corresponding file to `fluxcd/infrastructure/configs/`

## Adapting for your environment

| Placeholder | Replace with |
| --- | --- |
| `<your-github-username>` | Your GitHub username |
| `k8s.example.internal` | Your internal domain |
| `my-cluster` | Your cluster name |
| `internal-ca` | Your ClusterIssuer name (if different) |
| `<your-admin-password>` | A strong Grafana admin password |

## Security notes

- The `internal-ca` secret is created manually and never committed to this repository
- No secrets, private keys, or tokens are stored in this repository
- `gotk-components.yaml` is generated and managed by Flux; do not edit it manually
- The Vault init output (`cluster-keys.json`) must be stored securely outside this repository
- The `admin-user` service account in `kubernetes-dashboard` has `cluster-admin` privileges; restrict this for non-local environments
