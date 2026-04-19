# Livtorgex Beta

A beta trading platform deployment on a dedicated server using Kubernetes.

## Server

| | |
|---|---|
| **Host** | `65.109.35.177` |
| **User** | `root` |
| **Hostname** | `livtorgexbeta` |
| **OS** | Ubuntu 24.04, x86_64 |
| **Domain** | `trading.sinvoid.me` (Cloudflare) |
| **Backtesting Disk** | `/dev/sdb1` mounted at `/home/backtesting` |

## SSH Connection

| User | Key | Command |
|---|---|---|
| `root` | `~/.ssh/trading/id_ed25519` | `ssh -i ~/.ssh/trading/id_ed25519 root@65.109.35.177` |
| `trading` | `~/.ssh/trading/trading_user_id_ed25519` | `ssh -i ~/.ssh/trading/trading_user_id_ed25519 trading@65.109.35.177` |

The `trading` user has passwordless sudo. Prefer it over root for all setup work.

## Repositories

Organization: https://github.com/LivTorgEx

- Backend: `trading-platform-rust` — Rust-based trading platform with multiple services
- Infrastructure: k8s manifests/helm charts in the organization

## Stack

- **Orchestration:** Kubernetes (k8s) + k9s
- **Registry:** Nexus (`docker.nexus.livtorgex.com`)
- **Monitoring:** SigNoz
- **Exchange:** OKX, BingX
- **Domain:** `trading.sinvoid.me` via Cloudflare DNS

## Exchange Server Info

| Exchange | Server |
|---|---|
| `BingX` | `trading.sinvoid.me` |

## Storage Layout

| Mount | Device | Size | Purpose |
|---|---|---:|---|
| `/` | `/dev/sda3` | `503G` | OS, k3s, container runtime |
| `/home` | `/dev/sda4` | `3.1T` | user home and general data |
| `/home/backtesting` | `/dev/sdb1` | `3.6T` | dedicated backtesting storage |

## Services

`api`, `api_aptos`, `api_company`, `telegram`, `worker_bot`, `worker_loader`, `worker_projection`, `strategy_nft`, `backtest_core`, `webhook_company`

## Kubernetes Usage

All `kubectl` commands must be run as the `trading` user (has kubeconfig at `~/.kube/config`).

```sh
ssh -i ~/.ssh/trading/trading_user_id_ed25519 trading@65.109.35.177
```

### List pods

```sh
kubectl get pod -n livtorgex
```

### PostgreSQL

Connect via exec into the pod:

```sh
kubectl exec -n livtorgex $(kubectl get pod -n livtorgex -l app.kubernetes.io/name=postgresql -o jsonpath='{.items[0].metadata.name}') -- psql -U trading -d trading
```

Or port-forward and connect locally:

```sh
kubectl port-forward svc/postgresql -n livtorgex 5432:5432 --address 0.0.0.0 &
# Password: kubectl exec ... -- env | grep POSTGRES_PASSWORD
psql -h 65.109.35.177 -U trading -d trading
```

### RabbitMQ

Port-forward the management UI:

```sh
kubectl port-forward svc/rabbitmq -n livtorgex 15672:15672 --address 0.0.0.0 &
# Credentials: kubectl exec -n livtorgex rabbitmq-rabbitmq-0 -- env | grep RABBITMQ
# Open: http://65.109.35.177:15672
```

### View logs

```sh
kubectl logs -n livtorgex <pod-name> --tail=100 -f
```

## Running worker_bot locally against k8s

> **Use WSL Arch Linux** for local development. Two reasons:
> 1. Building on Windows fails (`aws-lc-sys` requires Linux tooling)
> 2. Building from the NTFS mount (`/mnt/c/`) causes rustc ICE — source must be on the WSL native filesystem

The worker bot reads all config from env vars / `.env`. Run it inside WSL Arch and point it at the remote k8s services via SSH tunnel.

### 1. Copy source to WSL native filesystem

Do this once (re-run after pulling changes):

```sh
wsl -d archlinux -u root
rsync -a --exclude='target/' /mnt/c/Users/inthe/projects/livtorgex/beta/trading-platform-rust/ /root/projects/trading-platform-rust/
cd /root/projects/trading-platform-rust
```

### 2. SSH tunnel (run in background)

```sh
ssh -i /mnt/c/Users/inthe/.ssh/trading/trading_user_id_ed25519 trading@65.109.35.177 \
  -L 5432:10.43.216.82:5432 \
  -L 5672:10.43.95.159:5672 \
  -L 50153:10.43.135.145:50153 \
  -N &
```

### 3. Get credentials

```sh
# PostgreSQL password
kubectl exec -n livtorgex $(kubectl get pod -n livtorgex -l app.kubernetes.io/name=postgresql -o jsonpath='{.items[0].metadata.name}') -- env | grep POSTGRES_PASSWORD

# RabbitMQ user/password
kubectl exec -n livtorgex rabbitmq-rabbitmq-0 -- env | grep RABBITMQ_DEFAULT

# Exchange secret key (EXCHANGE_SECRET_KEY)
kubectl get secret -n livtorgex worker-bot-binance-livtorgex-main-exchange -o jsonpath='{.data.EXCHANGE_SECRET_KEY}' | base64 -d
```

### 4. Create `.env` in `trading-platform-rust/`

```env
DATABASE_URL=postgres://trading:<POSTGRES_PASSWORD>@localhost:5432/trading
RABBIT_MQ_ADDR=amqp://admin:<RABBITMQ_DEFAULT_PASS>@localhost:5672/%2f
PROJECTION_RPC=localhost:50153

EXCHANGE_NAME=Binance # or BingX / OKX
COMPANY_CODE=LIVTORGEX
SERVER_NAME=main

EXCHANGE_API=fapi.binance.com
EXCHANGE_WS=fstream.binance.com
EXCHANGE_SECRET_KEY=<from k8s secret>

RUST_LOG=info
```

### 5. Stop the server-side worker bot

Before running locally, scale down the server bot to avoid two instances competing on the same RabbitMQ queue:

```sh
kubectl scale deployment worker-bot-binance-livtorgex-main -n livtorgex --replicas=0
```

Restore it when done:

```sh
kubectl scale deployment worker-bot-binance-livtorgex-main -n livtorgex --replicas=1
```

### 6. Run

```sh
cargo run -p trading --bin worker_bot
```
