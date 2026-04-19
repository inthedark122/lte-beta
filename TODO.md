# Beta Server Setup TODO

## 1. Server Prerequisites

- [x] Update & upgrade system packages (`apt update && apt upgrade`)
- [x] Install Git (2.43.0)
- [x] Install Docker + Docker Compose plugin (29.3.0)
- [x] Install kubectl (v1.32.13)
- [x] Install k9s (v0.50.18)
- [x] Install Helm (v3.20.1)
- [x] Install Rust toolchain (rustc 1.94.0)
- [x] Install `jq`, `rsync`, `build-essential` and other utilities

## 2. Build & Push Services

- [x] Add GitHub SSH deploy key to server
- [x] Clone `trading-platform-rust` repo + submodules (dev branch, `/opt/trading/trading-platform-rust`)
- [x] Login to Nexus Docker registry (`docker.nexus.livtorgex.com`)
- [x] Build all service Docker images
- [x] Push images to Nexus registry

## 3. Kubernetes Setup

- [x] Install / configure k8s cluster on server (k3s single-node, v1.34.5)
- [x] Clone k8s infrastructure repo from LivTorgEx org (`/opt/trading/livtorgex-server`)
- [x] Configure `kubectl` context (`~/.kube/config`)
- [x] Install nginx-ingress controller
- [x] Install cert-manager
- [ ] Apply namespace and RBAC configs
- [ ] Configure secrets / `values-secrets.yaml`
- [ ] Deploy all services via Helm charts
- [ ] Verify all pods are running with k9s

## 4. Monitoring — SigNoz

- [ ] Deploy SigNoz to cluster
- [ ] Configure services to send traces/metrics to SigNoz
- [ ] Verify SigNoz dashboard is accessible

## 5. Domain & TLS

- [ ] Point `trading.sinvoid.me` DNS (Cloudflare) to `65.109.35.177`
- [ ] Install ingress controller (e.g. nginx-ingress)
- [ ] Configure ingress rules for `trading.sinvoid.me`
- [ ] Setup TLS (cert-manager + Let's Encrypt or Cloudflare proxy)
- [ ] Verify HTTPS access on `trading.sinvoid.me`
