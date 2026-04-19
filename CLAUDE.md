# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Layout

```
beta/
├── trading-platform-rust/   # Main Rust monorepo (all service code lives here)
└── livtorgex-server/        # Kubernetes/Helm infrastructure manifests
```

Work almost always happens inside `trading-platform-rust/`.

## Build & Run

All commands run from `trading-platform-rust/`.

```sh
cargo build -p <package>
cargo run -p trading --bin worker_bot
cargo run -p trading --bin worker_projection
cargo run -p api
```

**Important:** Build only in WSL Arch Linux on the native filesystem (`/root/projects/trading-platform-rust/`). Building on Windows fails (`aws-lc-sys` requires Linux tooling). Building from the NTFS mount (`/mnt/c/`) causes a rustc ICE. After pulling changes on Windows, sync first:

```sh
rsync -a --exclude='target/' /mnt/c/Users/inthe/projects/livtorgex/beta/trading-platform-rust/ /root/projects/trading-platform-rust/
```

## Database Migrations (SEA-ORM)

```sh
sea-orm-cli migrate generate <name>           # create a new migration file
cargo run -p migration -- up                  # apply migrations
sea-orm-cli generate entity -o models/src/entities --with-serde both  # regenerate ORM entities
```

## Deployment

```sh
./scripts/build_and_k8s_deploy.sh [binary_name]   # git pull → build → push to Nexus → k8s rollout
```

GitHub Actions (`.github/workflows/build-and-deploy.yaml`) handles multi-arch (amd64/arm64) Docker builds and pushes to `docker.nexus.livtorgex.com`.

## Architecture

The workspace has ~21 crates. Key groupings:

| Crate(s) | Role |
|---|---|
| `api`, `api_aptos`, `api_company` | REST APIs (Rocket 0.5) |
| `trading/` | All worker binaries: `worker_bot`, `worker_projection`, `worker_loader`, `indicator_saver`, etc. |
| `telegram` | Telegram bot service |
| `backtest/` | Backtesting engine (`backtest_core`) |
| `strategy_nft` | NFT strategy worker |
| `webhook_company` | Webhook handler |
| `models/` | SEA-ORM entities (auto-generated — do not hand-edit) |
| `migration/` | Database schema migrations |
| `proto/`, `stream_schema/` | Protobuf definitions and stream schemas |
| `lte_rabbitmq/` | RabbitMQ client wrapper |
| `libs/lte_exchange/` | Exchange API abstraction (OKX, Binance, BingX) |
| `libs/ta-rs/` | Technical analysis library (git submodule) |

**Async runtime:** tokio. **ORM:** SEA-ORM + PostgreSQL. **Messaging:** RabbitMQ (lapin) + gRPC (tonic). **Serialization:** serde/serde_json. **Numerics:** `bigdecimal`, `rust_decimal`, `ordered-float`.

## Local Development Against k8s

The worker services read all config from `.env`. To run locally against the live k8s cluster:

1. Open SSH tunnel (map PostgreSQL 5432, RabbitMQ 5672, Projection RPC 50153):
   ```sh
   ssh -i /mnt/c/Users/inthe/.ssh/trading/trading_user_id_ed25519 trading@65.109.35.177 \
     -L 5432:10.43.216.82:5432 \
     -L 5672:10.43.95.159:5672 \
     -L 50153:10.43.135.145:50153 \
     -N &
   ```

2. Create `.env` in `trading-platform-rust/` (see `.env.example`). Key variables:
   - `DATABASE_URL`, `RABBIT_MQ_ADDR`, `PROJECTION_RPC`
   - `EXCHANGE_NAME` (Binance, OKX, or BingX), `COMPANY_CODE`, `SERVER_NAME`
   - `EXCHANGE_API`, `EXCHANGE_WS`, `EXCHANGE_SECRET_KEY`
   - `TELOXIDE_TOKEN` (Telegram bot)

3. Scale down the server-side worker before running locally to avoid queue conflicts:
   ```sh
   kubectl scale deployment worker-bot-binance-livtorgex-main -n livtorgex --replicas=0
   # restore when done:
   kubectl scale deployment worker-bot-binance-livtorgex-main -n livtorgex --replicas=1
   ```

## Kubernetes Operations

All `kubectl` commands run as the `trading` user (has kubeconfig):

```sh
ssh -i ~/.ssh/trading/trading_user_id_ed25519 trading@65.109.35.177
kubectl get pod -n livtorgex
kubectl logs -n livtorgex <pod-name> --tail=100 -f
```

Connect to PostgreSQL:
```sh
kubectl exec -n livtorgex $(kubectl get pod -n livtorgex -l app.kubernetes.io/name=postgresql -o jsonpath='{.items[0].metadata.name}') -- psql -U trading -d trading
```

## Multi-Instance Support

The platform supports multiple deployment instances. Key env vars that differentiate them:
- `COMPANY_CODE`: `LIVTORGEX`, `AIWE`, `ZEMAN`
- `SERVER_NAME`: `main`, `simulation`, `alpha`
- `EXCHANGE_NAME`: `Binance`, `OKX`, `BingX`
