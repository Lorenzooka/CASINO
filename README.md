# MetaWin-like NFT Raffle Platform — Starter Monorepo

This is a production-minded **starter skeleton** for a web-based, NFT-reward raffle/casino platform.
It includes:

- **contracts/** — Solidity contracts (Hardhat + TypeScript scripts)
- **web/** — Next.js 14 (App Router) + wagmi + Web3Modal
- **services/indexer/** — chain event listener -> PostgreSQL
- **services/scheduler/** — cron to invoke on-chain `requestDraw()`
- **db/** — Prisma schema (PostgreSQL)
- **docs/** — Provably Fair explainer + API sketch
- **data/** — OpenSea CSV template examples

## Quick Start

### 1) Contracts
```bash
cd contracts
npm i
cp .env.example .env
# Fill in RPC URLs, PRIVATE_KEY, ETHERSCAN keys, CHAINLINK VRF config.
npx hardhat compile
```

### 2) Web
```bash
cd web
npm i
cp .env.example .env
npm run dev
```

### 3) Database & Prisma
```bash
cd db
npm i
cp .env.example .env
npx prisma generate
npx prisma migrate dev --name init
```

### 4) Services
```bash
# Indexer
cd services/indexer && npm i && cp .env.example .env
# Scheduler
cd ../scheduler && npm i && cp .env.example .env
```

## Notes
- Contracts are **stubs** with clear TODOs and places to bind Chainlink VRF V2.5.
- Always run audits and add geoblocking + KYC/AML before going live.
- See `docs/provably-fair.md` for the randomness proof model.


---

## TRANSCEND Update (VRF v2.5 + wagmi write)

- **RaffleManager** upgraded to **Chainlink VRF v2.5** (subscription method) with native token payment toggle.
- Added real **coordinator/keyHash** placeholders for **Polygon Amoy** and **Base Sepolia** in `contracts/.env.example`.
- **Deploy script** wires roles and passes VRF params from env.
- **Web client** now performs a real `buyTickets()` transaction using **wagmi v2**.

> Next: expose a `getRaffle()` view to read `ticketPriceWei` on-chain and remove the demo price from `.env`.


## Payout / Claim Flow
1) Admin létrehozza a raffle‑t → `createRaffle()`
2) Admin beállítja a nyereményt:
   - `setPrizeETH(id, totalAmountWei)` – egyenlő rész az összes nyertesnek
   - `setPrizeERC20(id, token, totalAmount)` – egyenlő rész
   - `setPrizeNFTs(id, nft, tokenIds[])` – `winnersCount` darab NFT (min. ennyi)
3) Admin feltölti a **PrizeVault**‑ot a megfelelő eszközökkel.
4) Sorsolás záráskor: `requestDraw(id)` → VRF teljesít → nyertesek beíródnak.
5) Nyertes bejelentkezik és hívja: `claimPrize(id)` → Vault kifizet.

> Megjegyzés: a mostani implementáció ismétlődő nyertes(eke)t is kijelölhet, deduplikálást igény szerint adj hozzá (set/hash). AUDIT KÖTELEZŐ.


## ERC20 jegyvásárlás (USDC mód)
- Raffle létrehozásakor add meg a `payToken` címet (pl. USDC – 6 decimális).
- A web kliens automatikusan `approve` → `buyTicketsERC20` flow-t hív.
- Az ár `ticketPrice` **token legkisebb egységében** értendő (pl. USDC = 6 decimális).

## Több nyeremény
- **Per‑winner** (ETH/ ERC20): `setPerWinnerETH` vagy `setPerWinnerERC20` egy összeg **fejenként**.
- **Item prizes** (ERC721 / MINT_NFT): `addItemPrizeERC721` vagy `addItemPrizeMintNFT` többször is hívható; a draw után az első *K* nyerteshez kerülnek hozzárendelésre.

## Új admin oldalak
- **/admin** – raffle létrehozás, per‑winner és item prize konfiguráció, kézi draw.
- **/admin/list** – on‑chain lista az összes raffle‑ről.

> Megjegyzés: árfolyam alapú (USD → native) árazás opcionálisan Chainlink Price Feeds‑szel bővíthető — itt szándékosan nem kötöttük be, mert Amoyon a feedek korlátozottak lehetnek. Prod buildhez be tudom illeszteni ETH/USD feeddel a választott hálózatra.

# OMEGA Build Additions

- USD-pegged ticket pricing via Chainlink price feeds (`createRaffleUSD` + `quoteTicketCost`).
- Admin can set feeds: `setNativeUsdFeed`, `setTokenUsdFeed`.
- Admin list (`/admin/raffles`) has **DRAW NOW (VRF)** and **CSV export**.
- KYC/AML stub service: `services/kyc-aml` with `/api/kyc/check` endpoint.
- Monitor service: `services/monitor` with webhook alerts for anomalies.
- Security: `pauseAll()` / `unpauseAll()` and PrizeVault emergency withdraw (when paused).

## Feeds
Set in deploy or via admin page:
- `NATIVE_USD_FEED` (e.g. MATIC/USD on Polygon Amoy)
- `USDC_USD_FEED` (map with `setTokenUsdFeed(USDC, FEED)`)

## CSV Export
- `/api/admin/export` (all raffles) or `/api/admin/export?id=123` (single).



## CELESTIAL DEPLOY Additions
- **Gnosis Safe / Admin Guard**: `deployTimelockAndWire.ts` deploys an OZ **TimelockController** (SAFE as proposer/executor) and grants **ADMIN_ROLE** to the timelock for **RaffleManager** and **PrizeVault**.
- **EIP‑2612 Permit path**: client tries `permit()` (gasless approval) before fallback to `approve` for ERC20/USDC tickets.
- **Admin Stats**: `/admin/stats` with a Chart.js line chart (API stub at `/api/stats/entries`).

### Timelock wiring
```bash
cd contracts
npx hardhat run scripts/deployTimelockAndWire.ts --network polygon_amoy \
  --show-stack-traces
# env:
# SAFE_ADDRESS=0xYourSafe RAFFLE_MANAGER=0xRaffle PRIZE_VAULT=0xVault MIN_DELAY=3600
```
Verify that timelock has **ADMIN_ROLE**, then renounce deployer's admin role.


# SOLARIS FINAL Additions (2025-10-05)
- **Permit2 one‑click buy** button on raffle page (ERC20/USD módok): EIP‑712 aláírás + `buyTicketsWithPermit2(...)` hívás.
- **Safe Tx Builder helper** a `/admin/raffles` oldalon: calldata generálás + Safe link.
- **Idősor analitika**: `/admin/stats/timeseries` (Next API → RPC log scan → Chart.js vonaldiagram).

> Figyelem: a timeseries API demó módon durván becsüli a blokktartományt és egyszerű nyers log bucketelést használ. Prod‑ban célszerű az indexer DB‑t lekérdezni vagy dedikált graf szolgáltatást használni.


# AURORA RUNBOOK Additions
- **Runbook**: `docs/runbook.md` (SLIs/SLOs, RTO/RPO, SOP, roles)
- **Incident playbooks**: `docs/incident-playbooks/*` (web outage, RPC, VRF, DB, security)
- **On-call & Postmortem templates**: `docs/oncall-schedule-template.md`, `docs/postmortem-template.md`
- **Canary rollout**: `ops/canary.md` + `k8s/canary/web-canary.yaml` + **GitHub `canary.yml`**
- **Health endpoint**: `/api/health`
- **Smoke test**: `node scripts/smoke-check.mjs`
- **Load tests (k6)**: `tests/k6/k6_smoke.js`, `tests/k6/k6_stress.js`


# NEBULA METRICS
- Prometheus Operator resources in `k8s/monitoring/*`: Blackbox exporter, Probes, ServiceMonitors, Alerts.
- Grafana dashboards JSON in `grafana/dashboards/*` (import these).
- App metrics:
  - Web: `/api/metrics` (prom-client)
  - Monitor: port 9464 `/metrics` (tickets, claims, VRF latency, indexer lag)
  - KYC-AML: `/metrics` (requests counter)
- Argo Rollouts automated analysis: `k8s/argo/analysis-template.yaml` (availability & latency from Prometheus).


# AEON SECURE
- Key mgmt terv: `docs/security/aeon-secure-key-mgmt.md`
- Safe/Zodiac policy: `docs/security/safe-policy.md`, `ops/safe/zodiac-roles-config.json`
- Signer rotáció log: `docs/security/rotation-log.md`
- Tx Builder payloadok: `ops/safe/txs/payloads.json`
- Security CI: `.github/workflows/security.yml` (Semgrep, Slither, Trivy), Dependabot
- Vészleállítás: `ops/scripts/emergency_freeze.md`
- Hozzáférés-mátrix: `docs/security/access-matrix.csv`
