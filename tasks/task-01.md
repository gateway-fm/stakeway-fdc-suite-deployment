# Task 01: Stakeway FDC Suite v1.2.4 Parallel Deployment

## Status: DEPLOYED - INDEXING IN PROGRESS

## Problem Statement

The Stakeway FDC Suite deployment at `/data/stakeway-fdc-suite-deployment` on server `10.75.123.150` needs to be upgraded from v1.2.0 to v1.2.4 of the upstream `flare-foundation/fdc-suite-deployment`.

### Why v1.2.4 requires a parallel deployment

From upstream RELEASES.md (v1.2.4-rc.1, 2026-04-17):
- PostgreSQL upgraded from 16 to 18
- XRP indexer updated to v2.0.0 (database schema change, requires full reindex)
- BTC and DOGE also require database migration or reindex
- Upstream explicitly recommends: "To reduce downtime, it is recommended to prepare a parallel deployment first"

### Current state of fork

- **Fork**: `gateway-fm/stakeway-fdc-suite-deployment`
- **Upstream**: `flare-foundation/fdc-suite-deployment`
- Fork `main` is 0 commits ahead, 18 commits behind upstream
- Server is on `stakeway-fork` branch with a **stuck interactive rebase** (editing commit 125bbc2)
- 15 files modified (uncommitted) — these represent the running production config
- 1 stash entry from the rebase attempt

### Key customizations applied on the server (manual edits to upstream files)

1. **BTC node**: Ports bound to `10.75.123.150` instead of `0.0.0.0`
2. **DOGE node**: Connected to external `gateway-shared` Docker network
3. **XRP node**: Admin port bound to `127.0.0.1:5005`, admin IPs include `172.30.0.1` (Docker gateway)
4. **All verifiers**: `extra_hosts: host.docker.internal:host-gateway` added
5. **DOGE verifier**: Connected to `gateway-shared` network
6. **XRP verifier**: Connected to `xrp_default` external network
7. **Web2 verifier**: DNS servers added
8. **Config files**: RPC auth credentials in bitcoin.conf, dogecoin.conf; admin IP bindings in rippled.conf
9. **Image SHA digest pinning**: All images pinned with `@sha256:...`

### Why the current approach was problematic

All customizations were applied as direct edits to upstream-tracked files, making `git merge` or `git rebase` from upstream result in conflicts on almost every file.

---

## Solution Design

### Architecture

- **Branch**: `stakeway-fork-2` created from upstream tag `v1.2.4`
- **Port offset**: +10000 for all services in -2 deployment
- **Compose approach**: Direct edits to base compose files for ports/container_names (override files can't replace ports on Docker Compose v2.21), override files for additive changes (extra_hosts, networks, dns)
- **Config files**: `*.local.conf` files (gitignored) for server-specific credentials
- **Server path**: `/data/stakeway-fdc-suite-deployment-2`

### Docker Compose v2.21 Limitation (Verified)

The server runs Docker Compose v2.21.0. Testing confirmed:
- Override files APPEND to port lists — they do NOT replace
- `!reset` syntax strips ports entirely instead of replacing (fixed in v2.24+)
- `!override` syntax is not recognized
- **Solution**: Modify ports/container_names directly in compose files on `stakeway-fork-2` branch

### Port Collision Verification

All 20 planned -2 ports verified free via `ss -tlnp`:
```
Planned: 18332, 18333, 32555, 32556, 15005, 16006, 61233, 61234,
         61235, 60051, 35431, 19501, 18401, 35432, 19502, 18402,
         35433, 19503, 19800, 19801
Result: NO COLLISIONS
```

### Disk Space

- `/data`: 3.9T total, 775 GB available (80% used)
- BTC full node: ~700GB, DOGE: ~100GB, XRP: ~50GB
- Risk: BTC blockchain data may be tight. Must monitor during sync.

---

## Implementation Steps

### Phase 1: GitHub — Sync fork and create branch
- [ ] Fast-forward fork `main` to upstream
- [ ] Create `stakeway-fork-2` from `v1.2.4` tag
- [ ] Commit compose file modifications (ports, container names)
- [ ] Commit override files (extra_hosts, networks, dns)
- [ ] Commit gitignore additions
- [ ] Commit STAKEWAY-DEPLOYMENT.md
- [ ] Commit tasks/task-01.md
- [ ] Add GitHub Action for upstream sync

### Phase 2: Server — Deploy -2
- [ ] Clone `stakeway-fork-2` to `/data/stakeway-fdc-suite-deployment-2`
- [ ] Copy .env from -1, update node URLs
- [ ] Copy local config files (bitcoin.local.conf, dogecoin.local.conf, rippled.local.conf)
- [ ] Run `generate-config.sh`
- [ ] Update XRP config.toml for -2 node URL
- [ ] Start nodes (BTC, DOGE, XRP)
- [ ] Verify nodes are syncing
- [ ] Start verifiers (BTC, DOGE, XRP, EVM, Web2)
- [ ] Verify verifiers are indexing
- [ ] Verify -1 containers are unaffected

### Phase 3: Monitor & Cutover
- [ ] Monitor node sync progress
- [ ] Monitor verifier indexing progress
- [ ] When caught up: stop -1, swap -2 to production ports
- [ ] Verify all services on production ports
- [ ] Clean up -1

---

## Running Services on Server (Baseline — DO NOT TOUCH)

```
NAMES                                                       STATUS
evm-verifier                                                Up 6 days
flare-validator-bf6d4ff7-5476-f627-e502-cf7abc1cc020        Up
flare-value-provider-c36bb01d-f0bf-c53a-283b-794eddbcb44f   Up
node-exporter-d4b121af-59b0-9e92-6b8e-2fbf4e586c66         Up
node-f5fec067-3cc6-2830-3166-23c277f74a1f                   Up
node-mainnet-btc                                            Up 6 days
node-mainnet-doge                                           Up 6 days
node-mainnet-xrp                                            Up
quazar-idx-daemon-55077739-c91e-2806-5605-e86bb68e172a       Up
verifier-btc-database                                       Up 6 days
verifier-btc-prune-blocks                                   Up 6 days
verifier-btc-server                                         Up 6 days
verifier-btc-verifier                                       Up 6 days
verifier-doge-database                                      Up 6 days
verifier-doge-index-blocks                                  Up 6 days
verifier-doge-prune-blocks                                  Up 6 days
verifier-doge-server                                        Up 6 days
verifier-doge-verifier                                      Up 6 days
verifier-xrp-database                                       Up 6 days
verifier-xrp-indexer                                        Up 6 days
verifier-xrp-verifier                                       Up 6 days
web2-verifier                                               Up 6 days
```

**RULE: None of these containers may be stopped, restarted, or modified at any point during this task.**

---

## Docker Networks on Server

```
NAME                    DRIVER
bridge                  bridge
btc_default             bridge
doge_default            bridge
eth_default             bridge
evm-verifier_default    bridge
gateway-shared          bridge
host                    host
none                    null
verifier-btc_default    bridge
verifier-doge_default   bridge
verifier-xrp_default    bridge
web2-verifier_default   bridge
xrp_default             bridge
```

---

## Key Decisions Log

| Decision | Rationale |
|---|---|
| +10000 port offset | Easy to remember, all verified free, no collisions with any existing service |
| Direct compose edits for ports (not overrides) | Docker Compose v2.21 appends ports from overrides — cannot replace. Verified via test on server. |
| Separate nodes in -2 (not sharing -1 nodes) | Full independence. v1.2.4 might have different node version requirements. |
| Nodes sync from scratch | No downtime on -1. BTC will take days but that's acceptable. |
| `stakeway-fork-2` branch (not editing main) | Clean separation. `main` stays synced with upstream. |
| Keep `stakeway-fork` branch for now | Clean up after cutover is complete. |
| Commit override files to git | They contain infra config (networks, extra_hosts), not secrets. |
| Gitignore `*.local.conf` | These contain RPC credentials — server-only. |

---

## Upstream v1.2.4 Changes Summary

From v1.2.0 (current) to v1.2.4:
1. Docker image registry migrated from Docker Hub to GHCR
2. All container images tagged with SHA256 digests
3. Ripple node updated from 2.6.2 to 3.1.2
4. PostgreSQL upgraded from 16 to 18
5. XRP indexer updated from v1.0.4 to v2.0.0 (schema change)
6. XRP verifier updated from v1.2.2 to v1.5.0
7. Web2 verifier updated from v1.3.0 to v1.4.0
8. EVM verifier: unchanged (v1.0.3)
9. BTC/DOGE indexer: unchanged (v1.0.2)
10. BTC/DOGE verifier API: unchanged (v1.2.2)
11. GitHub Actions CI added for releases
12. Database volume mount changed: `/var/lib/postgresql` (was `/var/lib/postgresql/data`)
13. XRP verifier: removed indexer-server, indexer-indexer, indexer-prune services (simplified)
14. XRP verifier: no longer has DNS in base compose
15. BTC/DOGE verifiers: database volume now uses named volume (was `./xxx-indexer-database` bind mount)

---

## Final Verification Checklist

- [x] Fork `main` sync PR created (PR #7 — main is branch-protected, requires PR merge)
- [x] `stakeway-fork-2` branch exists with all customizations (4 commits)
- [x] All -2 containers running with `-2` suffix names (17 containers)
- [x] No port collisions (all 20 -2 ports verified free via `ss -tlnp`)
- [x] -1 containers ALL still running and healthy (all showing "Up 6 days")
- [x] `docker compose config` validated in each -2 directory — correct ports, names, networks
- [x] BTC node syncing: block 233,378 (1.25% progress as of deployment)
- [x] DOGE node syncing: block 42,241 (0.56% progress as of deployment)
- [x] XRP node syncing: connecting to peers, downloading ledger
- [x] BTC verifier indexing (server healthy, indexer running)
- [x] DOGE indexer restarting as expected (node hasn't reached start_block yet)
- [x] XRP verifier started (indexer connecting to xrp-2 node via xrp-2_default network)
- [x] EVM verifier-2 running on port 19800
- [x] Web2 verifier-2 running on port 19801
- [x] `STAKEWAY-DEPLOYMENT.md` committed
- [x] `tasks/task-01.md` committed
- [x] GitHub Action for upstream sync committed
- [x] `git status` clean on server -2 (only data dirs and .env untracked)

## Issues Encountered During Deployment

1. **BTC node permissions**: Distroless images run as uid 65532, but bind-mount data dirs owned by root. Fixed by adding `user: "1000"` to node overrides and chowning data dirs.
2. **DOGE DNS resolution**: DOGE node couldn't resolve DNS seeds. Fixed by adding explicit DNS servers to DOGE override.
3. **Docker Compose project name collision**: Node compose files use directory name as project name, which would match -1. Fixed by creating `.env` with `COMPOSE_PROJECT_NAME=xxx-2` in each node and verifier directory.
4. **XRP named volume permissions**: Volume created as uid 65532, but `user: "1000"` override. Fixed by chowning the volume data.
5. **DOGE verifier indexer restarting**: Expected behavior — DOGE node hasn't synced to start_block (5469000) yet. Will auto-recover.
6. **Docker gateway-shared memory error**: Transient "cannot allocate memory" on network attach. Resolved on retry.

## Disk Space at Deployment

- `/data`: 3.9T total, 763 GB available (80% used)
- **Warning**: BTC full chain is ~700GB. Must monitor `df -h /data` during sync.

## Commits on stakeway-fork-2

1. `738f499` — STW: Add Stakeway v1.2.4 parallel deployment customizations (20 files)
2. `0dc771a` — STW: Add xrp-2_default network to XRP verifier override
3. `d4429f8` — STW: Add user: 1000 to node overrides for bind mount permissions
4. `310c82e` — STW: Add DNS servers to DOGE node override
5. `8f1c6c7` — STW: Update task-01.md with deployment verification results
6. `f6597ab` — STW: Revert BTC/XRP/EVM/Web2 to production ports and names for cutover

---

## Cutover Log (2026-04-21)

Cutover of all services except DOGE to production ports/names from the -2 deployment directory.
DOGE remains on -2 offset ports (node still syncing from genesis).

### Pre-cutover steps
1. Committed compose files with production ports/names to `stakeway-fork-2` (DOGE excluded)
2. Pulled latest on server
3. Updated `COMPOSE_PROJECT_NAME` in node `.env` files: `btc`, `xrp`, `evm-verifier`, `web2-verifier`
4. Updated root `.env`: `BTC_NODE_URL=http://10.75.123.150:8332`, `XRP_NODE_URL=http://node-mainnet-xrp:51234`
5. Ran `./generate-config.sh` to regenerate verifier configs with production URLs

### Cutover execution (one-by-one, nodes first)

| Step | Action | Result |
|---|---|---|
| 1 | Stop old BTC node (-1) | `docker compose down` in -1/nodes-mainnet/btc |
| 2 | Stop -2 BTC node, remove container | Had to manually `docker stop/rm node-mainnet-btc-2` (different project name) |
| 3 | Start new BTC node on production ports | `node-mainnet-btc` on `10.75.123.150:8332-8333`. Lock file error from shared data dir — resolved with restart. |
| 4 | Stop old XRP node (-1) | `docker compose down` in -1/nodes-mainnet/xrp |
| 5 | Stop -2 XRP node, copy volume data | Copied `xrp-2_ripple-mainnet-data` → `xrp_ripple-mainnet-data` (new project name = new volume name) |
| 6 | Start new XRP node on production ports | `node-mainnet-xrp` on `5005,6006,50051,51233-51235` |
| 7 | Stop old BTC verifier (-1) | `docker compose down` removed all 5 containers |
| 8 | Stop -2 BTC verifier, copy DB volume | Copied `verifier-btc-2_btc-indexer-database` → `verifier-btc_btc-indexer-database` |
| 9 | Start new BTC verifier on production ports | All 5 containers up: `25431, 9501, 8401` |
| 10 | Stop old XRP verifier (-1) | `docker compose down` removed all 3 containers |
| 11 | Stop -2 XRP verifier, copy DB volume | Copied `verifier-xrp-2_xrp-indexer-database` → `verifier-xrp_xrp-indexer-database` |
| 12 | Start new XRP verifier on production ports | All 3 containers up: `25433, 9503`. Indexer: "up to date" |
| 13 | Stop old EVM verifier, remove -2 | Started new `evm-verifier` on `:9800` |
| 14 | Stop old Web2 verifier, remove -2 | Started new `web2-verifier` on `:9801` |

### Post-cutover issues
1. **BTC indexer connection refused**: `host.docker.internal:8332` didn't work because BTC node binds to `10.75.123.150`, not `0.0.0.0`. Fixed by changing `BTC_NODE_URL` to `http://10.75.123.150:8332` and regenerating configs.
2. **BTC indexer waiting**: Node at block ~636K, start_block is 871,200. Will begin indexing once node catches up. Expected.
3. **Volume migration required**: Changing `COMPOSE_PROJECT_NAME` from `btc-2` to `btc` changes the volume name. Required copying data between named volumes for XRP and BTC verifier databases.

### Services still on -2 (DOGE — pending node sync)

| Container | Status | Port |
|---|---|---|
| node-mainnet-doge (-1) | Up 6 days | 22555 (production) |
| node-mainnet-doge-2 | Up 3 hours, syncing | 32555 (-2 offset) |
| verifier-doge-* (-1) | Up 6 days | 25432, 9502, 8402 (production) |
| verifier-doge-*-2 | Up 3 hours | 35432, 19502, 18402 (-2 offset) |

DOGE cutover will happen when the -2 DOGE node reaches block 5,469,000 and the indexer can start.

