# FajroX Raffle — MetaWin‑style (Paste‑this README)

A production‑ready monorepo for an **NFT raffle platform** with Chainlink VRF, USDC/ETH ticketing (Permit2 one‑click), **Gnosis Safe** admin, full **observability** (Prometheus/Grafana or Operator), **Argo canary** rollouts, VRF latency metrics, **Security CI**, and a Safe **Roles generator** (ARCANE SHIELD).

> Want an interactive, step‑by‑step setup with me? Open an issue or ping me and say **AGENT MODE**.

---

## What’s inside

- **contracts/** — Solidity: `RaffleManager`, `PrizeVault`, `PrizeNFT`, Hardhat scripts
- **web/** — Next.js App Router (user UI + admin, stats, Safe helpers, Roles generator)
- **services/** — Node services: `monitor` (on-chain watcher + VRF latency), `indexer`, `scheduler`, `kyc-aml` (stub)
- **k8s/** — Kubernetes manifests (base deploys, observability, Argo rollouts, cron backups)
- **ops/** — docker‑compose, helm values, load tests (k6), chaos drills, Safe utilities
- **docs/** — deploy playbook, runbook, observability, VRF, security, roles
- **.github/** — CI/CD workflows (CI, staging/prod CD, canary), security scans

**Core features**
- Multi‑prize raffles (ETH / ERC20 / ERC721 / on‑chain MINT_NFT)
- USD‑indexed ticket pricing (Chainlink price feeds)
- Chainlink VRF v2.x draws + `adminDrawNow` override
- **Permit2** one‑click ERC20 purchase
- **Gnosis Safe** admin guard (multisig) + **Zodiac Roles & Delay** modules
- **Roles generator** (ABI → Zodiac JSON) — UI & CLI
- **Observability**: Prometheus + Grafana + Alertmanager (vanilla or Operator/CRDs)
- **Argo canary** with Prometheus AnalysisTemplate (go/no‑go on SLI)
- **VRF latency** metric `vrf_latency_seconds` + alerting
- **Security CI** (Slither, Semgrep, Trivy, Dependabot)

---

## Requirements
- Node.js **20+**
- Docker (for local stack) and Git
- (Prod/Staging) `kubectl`, `helm`, a Kubernetes cluster
- EVM RPC (e.g., Polygon Amoy testnet via Alchemy/Infura)

---

## Quickstart (local)

1. **Init repo**
```bash
git init
git add .
git commit -m "init"
git branch -M main
git remote add origin <YOUR_GITHUB_REPO_URL>
git push -u origin main
```

2. **Env files (copy examples)**
- `contracts/.env.example` → **`.env`** (fill VRF keys, ADMIN_SAFE, PERMIT2, network)
- `web/.env.example` → **`.env.local`**
  - `NEXT_PUBLIC_RPC_URL=<your_amoy_rpc>`
  - `NEXT_PUBLIC_CHAIN_ID=80002`
  - `NEXT_PUBLIC_RAFFLE_MANAGER=<after_deploy>`
  - optional: `NEXT_PUBLIC_PERMIT2_ADDRESS`, `NEXT_PUBLIC_SAFE_ADDRESS`
- `services/*/.env` as needed (DATABASE_URL, RAFFLE_MANAGER, RPCs)

3. **Local stack (Docker)**
```bash
cd ops
docker compose -f docker-compose.yml up --build
# web: http://localhost:3000
# kyc-aml: http://localhost:8081
# postgres: localhost:5432  (user/pass/raffle)
```

4. **Compile & deploy contracts (Polygon Amoy)**
```bash
cd contracts
npm ci
cp .env.example .env
npx hardhat compile
npx hardhat run scripts/deploy.ts --network polygon_amoy
# Note addresses: RAFFLE_MANAGER, PRIZE_VAULT, PRIZE_NFT
```

5. **Configure VRF & Safe**
- Create/fund Chainlink **VRF** subscription and add **RaffleManager** as **consumer**.
- Optional: set native↔USD price feed (`setNativeUsdFeed`).
- Safe guard + Permit2:
```bash
RAFFLE_MANAGER=0x... ADMIN_SAFE=0xYourSafe PERMIT2=0x000000000022D473030F116dDEE9F6B43aC78BA3 npx hardhat run scripts/setupSafe.ts --network polygon_amoy
```

6. **Run web (dev)**
```bash
cd web
npm ci
cp .env.example .env.local   # fill RPC + RAFFLE_MANAGER
npm run dev
# Admin: /admin, Safe tools: /admin/safe, Roles: /admin/safe/roles, Stats: /admin/stats
```

---

## Admin & Ops (what you can do)

- **Create raffles & prizes** (ETH / ERC20 / NFT / MINT_NFT)
- **Buy tickets** (user) or **Buy with Permit2** (one‑click ERC20)
- **Draw now** (admin) or scheduled draws via scheduler
- **Safe tools**: calldata builder, link to Safe queue
- **Roles generator**: paste ABI → download Zodiac **JSON**
- **Stats**: entrants, winners, drawn flags, **tickets/hour**
- **CSV export** on admin lists

---

## Safe policy (recommended)

- **Owner Safe (3/5)** — full admin incl. withdraw (**Delay** module 30–120 min)
- **Ops Safe (2/3)** — day‑to‑day ops: `adminDrawNow`, `setVRFParams`, `setPermit2`
- **Guardian Safe (2/2)** — break‑glass only: `pauseAll()`

**Generate roles from ABI**
- UI: open **`/admin/safe/roles`**
- CLI:
```bash
node ops/safe/generate-roles-from-abi.mjs   contracts/artifacts/contracts/RaffleManager.sol/RaffleManager.json   contracts/artifacts/contracts/PrizeVault.sol/PrizeVault.json   ops/safe/generated-roles.json   --targets RAFFLE_MANAGER=0x...,PRIZE_VAULT=0x...
```

---

## Observability & Canary

**Vanilla stack** (`k8s/observability/*`)
- Web metrics: `/api/metrics`, Monitor: `/metrics`, KYC: `/metrics`
- Grafana dashboards via ConfigMap
- Alerts: **HighErrorRate5m**, **HighLatencyP95**, **NoTickets30m**, **VRFHighLatencyP95**

**Operator stack** (`k8s/observability-operator/*`)
- `ServiceMonitor` / `PrometheusRule` / `AlertmanagerConfig` CRDs
- Grafana sidecar auto‑loads dashboard ConfigMaps

**Canary (Argo Rollouts)**  
`k8s/argo/rollout-web-with-analysis.yaml` — steps 10% → 50% → 100%, with **Prometheus** AnalysisTemplate (error rate & p95 latency).

**VRF metric**
- `vrf_latency_seconds` histogram from **monitor** service; alert on **p95 > 300s**.

---

## CI/CD & Secrets

**Workflows**
- CI: `.github/workflows/ci.yml` (contracts compile, web build)
- CD Staging: `.github/workflows/cd-staging.yml` (build, GHCR push, k8s apply)
- CD Prod: `.github/workflows/cd-prod.yml` (tag `v*.*.*` → deploy prod)
- Canary trigger: `.github/workflows/cd-canary.yml`
- Security CI: `.github/workflows/security.yml` (Slither, Semgrep, Trivy)
- Dependabot: `.github/dependabot.yml`

**Set these GitHub Secrets (examples)**
```
REGISTRY_TOKEN
KUBE_CONFIG_BASE64
K8S_NAMESPACE_STAGING
K8S_NAMESPACE_PROD
RPC_URL
DATABASE_URL
SENTRY_DSN_WEB
SENTRY_DSN_NODE
MONITOR_WEBHOOK_URL
RAFFLE_MANAGER            # after deploy
PERMIT2_ADDRESS           # optional override
USE_OPERATOR_OBS=true     # to enable Operator-based observability in CD
```

---

## Useful commands

Rollback web to previous image:
```bash
kubectl -n <ns> set image deployment/web web=ghcr.io/<owner>/<repo>-web:<prevTag>
```

Emergency freeze (break‑glass):  
Use **/admin/safe** to generate `pauseAll()` calldata, execute via **Guardian Safe (2/2)**.

k6 load test:
```bash
k6 run ops/k6/traffic.js -e BASE_URL=https://<your-env-url>
```

Chaos drill (pod kill):
```bash
kubectl apply -f ops/chaos/pod-kill.yaml -n <ns>
```

---

## Docs index (read these next)

- Deploy playbook — `docs/deploy-playbook.md`
- On‑call runbook — `docs/runbook/aurora-runbook.md`
- Observability (vanilla) — `docs/nebula-metrics.md`
- Observability (Operator) — `docs/quantum-obs.md`
- VRF metrics — `docs/singularity-vrf.md`
- Security & key mgmt — `docs/security/aeon-secure-key-mgmt.md`
- ARCANE SHIELD (roles generator) — `docs/security/arcane-shield.md`

---

## Repo tree (short)

```
contracts/   web/   services/   k8s/   ops/   docs/   .github/
```

## License
MIT (add a `LICENSE` file).

---

### Need help?
Say **AGENT MODE** and I’ll guide you through creating secrets, deploying contracts, wiring Safe, launching web, enabling canary & observability, and running a test raffle.
