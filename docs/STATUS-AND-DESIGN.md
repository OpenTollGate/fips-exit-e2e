# FIPS Exit Node PoC — Status & Design

**Date:** 2026-07-06  
**Author:** Hermes Agent (on behalf of c08r4d0r)  
**Repo:** `github.com/OpenTollGate/fips-exit-node`  
**Ngit:** `nostr://npub12m5exm2uk3xa674cc5r0hlyvccs5xxn7qv83ezuteefv5972nquq4j4szl/relay.ngit.dev/fips-exit-node`  
**Domain:** `fips-exit.orangesync.tech` (live, nsite gateway)  
**Dashboard:** `https://npub1laqt4pmrqsel4ak6z6nazptm99jj28m386zkmsgd9zadt7jq55jq9qfhhe.nsite.lol/`  
**VPS:** 66.92.204.38 (VPS1, Debian 13, via tollgate-infrastructure-kit)  
**FIPS Version:** v0.4.0 (pinned — NOT tracking master)  

---

## 1. What This Is

A proof-of-concept FIPS mesh exit node — a VPS that acts as a WireGuard-based
internet gateway for FIPS mesh network peers. FIPS mesh peers connect to the
VPS over UDP/TCP, get routed through a WireGuard tunnel, and reach the public
internet via nftables MASQUERADE.

### Architecture

```
[FIPS Mesh Peer] ──UDP :2121──→ [VPS1: FIPS Daemon] ──fips0──→ [Kernel IP Forward]
                                       │
                                  [WireGuard: wg0] ──10.99.99.0/24──
                                       │
                                  [nftables fips-exit: MASQUERADE on eth0]
                                       │
                                  [PUBLIC INTERNET]
```

### Data Flow

1. Test peer sends internet-bound traffic through nvpn mesh
2. nvpn encapsulates via FIPS (UDP :2121) to VPS1
3. VPS1 FIPS daemon decapsulates → delivers to local fips0 TUN interface
4. Kernel routes from fips0 → wg0 (WireGuard tunnel)
5. wg0 → nftables MASQUERADE → eth0 → internet
6. Return traffic reverses: eth0 → nftables → wg0 → kernel → fips0 → FIPS mesh → peer

### Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Mesh protocol | FIPS v0.4.0 (pinned) | Nostr-native p2p mesh; pinned to stable release, NOT tracking master |
| Routing layer | nvpn 4.0.87 | FIPS alone handles mesh handshake but not egress routing; nvpn provides tunnel network (10.44.0.0/16) + WireGuard exit management |
| Internet exit | WireGuard wg0 + nftables MASQUERADE | Simple, auditable, no dependencies on VPN providers |
| Identity | Nostr npub (FIPS node identity) | Each FIPS node has a Nostr keypair; exit advertises via kind 30078 events |
| Config mgmt | Ansible (in tollgate-infrastructure-kit) | Reproducible, version-controlled, single playbook deploys all VPS services |
| Domain | fips-exit.orangesync.tech | Caddy reverse proxy → nsite-gateway (port 3002) |
| Publishing | GitHub + ngit | All work published to both platforms |

### FIPS Version Pin

| Field | Value |
|-------|-------|
| Pinned version | v0.4.0 (upstream tagged release) |
| Tag author | Johnathan Corgan (jmcorgan/fips maintainer) |
| Tag date | 2026-06-27 |
| Pinned commit | `da2d0b7408fc98ffc17671b5a49a4d76ce504292` |
| Local fork commit | `7a3f8c7ff3fb5d230a376cf1519b27f162fa442e` (v0.4.0 + 4 commits) |
| VPS1 binary | `/usr/bin/fips` (17.6MB, built 2026-07-05 from local fork) |

**Policy:** DO NOT follow master. Upstream is actively refactoring toward sans-io
architecture. The v0.4.0 tag is the last stable release before the refactor.

---

## 2. Components Deployed on VPS1

| Component | Status | Details |
|-----------|--------|---------|
| FIPS daemon | ✅ ACTIVE | v0.4.0-derivative, UDP :2121, TCP :8443, persistent identity |
| WireGuard wg0 | ✅ UP | :51821, 10.99.99.1/24, peer 10.99.99.2 |
| nftables fips-exit | ✅ LOADED | MASQUERADE on wg0 → eth0 |
| IP forwarding | ✅ ENABLED | net.ipv4.ip_forward=1, persistent via sysctl.d |
| external_addr | ✅ SET | 66.92.204.38:8443 |
| Kind 30078 route advert | ✅ PUBLISHED | On relay.damus.io, nos.lol |
| Nostr identity | `npub1mqelkzqp4659fws35h2wvr7z9caka5ml8qddj3ssnwaulwpxdd9sdc3esw` |

---

## 3. Phase 1 — All EXIT Tasks Complete ✅

| Task | ID | Status | What was done |
|------|----|--------|---------------|
| EXIT-1 | t_1d9cacc0 | ✅ DONE | FIPS config fix — external_addr: 66.92.204.38:8443, TCP advertised |
| EXIT-2 | t_69e6a4ca | ✅ DONE | WireGuard wg0 + nftables MASQUERADE deployed on VPS1 |
| EXIT-3 | t_9f3e5f68 | ✅ DONE | Docker test peer (nvpn node-a) — nostr-vpn-e2e-fips-exit image built |
| EXIT-4 | t_025ad644 | ✅ DONE | E2E verified — raw FIPS Docker handshake + nostr-vpn WG exit test passed |
| EXIT-5 | t_bffcfbf7 | ✅ DONE | SMOKE-1: 4/4 pass, 1 skip (needs FIPS_EXIT_NPUB env) |
| EXIT-6 | t_d0fce0ec | ✅ DONE | Cashu payment gate integrated |
| EXIT-7 | t_06c2e379 | ✅ DONE | Kind 30078 route advertisement published |
| EXIT-8 | t_8adff567 | ✅ DONE | Soveng demo preparation |
| EXIT-9 | t_8c56d9b9 | ⏸️ BLOCKED | Production hardening — monitoring, reliability, docs (Phase 2) |

### SMOKE-1 Test Results (2026-07-06)

```
test_wireguard_peer_has_recent_handshake      ✅ PASS (handshake <5min ago)
test_nftables_fips_exit_masquerade_loaded     ✅ PASS (fips-exit table present)
test_ip_forwarding_enabled_and_egress_works    ✅ PASS (forward path confirmed)
test_nostr_kind_30078_route_advert_published   ⏭️ SKIP (needs FIPS_EXIT_NPUB env)
test_wireguard_tunnel_has_bidirectional_traffic ✅ PASS (rx>0 AND tx>0)
```

All 5 tests use feature detection (skip gracefully if absent). Run against VPS1
at 66.92.204.38 via SSH with passwordless sudo.

---

## 4. Raw FIPS Docker Node

A standalone raw FIPS Docker container was built from the upstream jmcorgan/fips
sidecar Dockerfile to replace the buggy nvpn test peer (which didn't reconnect
after VPS1 FIPS restarts).

**Image:** `fips-raw-node:latest` (231MB, debian:trixie-slim runtime)
**Build:** From `~/fips/examples/sidecar-nostr-relay/Dockerfile` (env-var-driven config)

### Verified Working

- ✅ Config generated from environment variables
- ✅ FIPS daemon starts (v0.4.0-derivative)
- ✅ Transports initialized: UDP :2121 + TCP :8443
- ✅ Connection to VPS1 established (1 peer)
- ✅ FIPS handshake: "Connection promoted to active peer peer=vps1-exit"
- ✅ Mesh parent switched: "new_parent=vps1-exit"
- ✅ Encrypted session established (Noise XK handshake)
- ✅ Session active: "Session established (initiator, XK)"

### Local E2E Test Passed

The nostr-vpn WireGuard exit e2e test (`docker-compose.wireguard-exit-e2e.yml`)
proved the full stack locally:

```
wireguard-exit docker e2e passed: node-a egressed via WG (5 icmp pkts),
and WG upstream ingress could not reach node-b
```

This confirms: nvpn mesh → WireGuard exit → internet egress works correctly,
with proper traffic source verification and security isolation.

---

## 5. Repo Structure

The FIPS exit node project lives in two repos:

### Primary: `github.com/OpenTollGate/fips-exit-node`

```
fips-exit-node/
├── STATUS.md                    # This document (status + design)
├── README.md                    # Quick-start guide
├── fips-pin.txt                 # FIPS version pin details
├── ansible/
│   ├── playbooks/13-fips.yml    # FIPS deployment playbook
│   └── roles/fips/              # Ansible role (tasks, handlers, templates)
├── tests/
│   └── test_fips_exit_node.py   # SMOKE-1 pytest suite
└── dashboard/
    └── index.html               # nsite status dashboard
```

**GitHub:** `https://github.com/OpenTollGate/fips-exit-node`  
**Ngit:** `nostr://npub12m5exm2uk3xa674cc5r0hlyvccs5xxn7qv83ezuteefv5972nquq4j4szl/relay.ngit.dev/fips-exit-node`  
**Gitworkshop:** `https://gitworkshop.dev/npub12m5exm2uk3xa674cc5r0hlyvccs5xxn7qv83ezuteefv5972nquq4j4szl/relay.ngit.dev/fips-exit-node`  

### E2E Test Infrastructure: `~/repos/fips-exit-e2e/`

```
fips-exit-e2e/
├── docker/
│   ├── Dockerfile.fips-node     # FIPS Docker image definition
│   ├── entrypoint.sh            # Env-var → fips.yaml generator
│   └── docker-compose.yml       # Test topology
├── scripts/
│   ├── run-node.sh              # Run FIPS Docker node
│   ├── get-vps1-identity.sh     # Get VPS1 FIPS npub
│   └── run-and-verify.sh        # Full run + verify
├── docs/
│   ├── STATUS-AND-DESIGN.md     # This file
│   └── PLAN.md                  # Implementation plan (phases 1-3)
└── fips / fipsctl / fipstop     # FIPS binaries
```

---

## 6. Testing & Monitoring

### Daily Smoke Test

- **Cron:** `fips-exit-smoke-daily` (runs at 06:00 daily)
- **Type:** `no_agent=True` (script-based, no LLM cost)
- **Script:** `~/.hermes/profiles/manager/scripts/fips-exit-smoke-daily.sh`
- **Delivery:** Origin (this Signal group)
- **Env:** FIPS_EXIT_HOST=66.92.204.38, FIPS_EXIT_SUDO=1
- **Next run:** 2026-07-07T06:00:00

### Health Monitoring

A separate `FIPS VPS1 Health` cron runs every 15 minutes and checks:
- FIPS daemon active
- Ports listening (:2121 UDP, :8443 TCP)
- Error storm detection
- WireGuard tunnel state

---

## 7. Domain & Hosting

**Domain:** `fips-exit.orangesync.tech`
**DNS:** `A fips-exit.orangesync.tech → 66.92.204.38` (Cloudflare, DNS-only)
**Proxy:** Caddy reverse proxy → nsite-gateway (localhost:3002)
**Dashboard:** Static HTML deployed via nsyte to Blossom + nsite gateway

The dashboard is live at two URLs:
- `https://fips-exit.orangesync.tech` (Caddy proxied)
- `https://npub1laqt4pmrqsel4ak6z6nazptm99jj28m386zkmsgd9zadt7jq55jq9qfhhe.nsite.lol/` (nsite gateway)

---

## 8. Pattern Research (for Phase 2)

Two projects were studied for reusable patterns:

### tests.tollgate.me (physical-router-test-automation)

| Pattern | Relevance to FIPS |
|---------|-------------------|
| Live dashboard via Nostr+Blossom (kind 30078) | 🟢 Adapt for FIPS smoke test results |
| PR-driven testing workflow | 🟢 Adapt for FIPS CI |
| Feature detection gating in tests | 🟢 Already adopted in SMOKE-1 |
| Fire-and-forget cloud VMs | 🟡 Heavy infra, skip for now |
| TollGate-specific tests | ❌ Not applicable |

### conwrt (Amperstrand)

| Pattern | Relevance to FIPS |
|---------|-------------------|
| UseCase plugin system (declarative config presets) | 🟢 Adapt for VPS deployment |
| vpn_node.py use case (WireGuard server + Nostr listing) | 🟢 **Directly analogous** |
| fips_bluetooth_rfcomm.py (FIPS YAML from typed params) | 🟢 **Direct FIPS reference** |
| E2E smoke tests via SSH | 🟢 Already adopted in SMOKE-1 |
| Nostr+Blossom artifact distribution | 🟢 Adapt for release publishing |
| OpenWrt flashing workflow | ❌ Not applicable |

---

## 9. Phase 2 — Planned Work

| Priority | Task | Description |
|----------|------|-------------|
| P0 | Multi-peer support | Verify multiple concurrent FIPS mesh peers can egress |
| P0 | Dashboard auto-refresh | data.json pattern with cron rebuild |
| P1 | Rate limiting | Per-npub traffic caps on WireGuard exit |
| P1 | Graceful reconnect | Ensure nvpn reconnects after VPS1 FIPS restart |
| P1 | CI pipeline | GitHub Actions builds + SMOKE-1 on push |
| P2 | Production hardening | Monitoring alerts, disk/CPU thresholds |
| P2 | conwrt UseCase adaptation | Formalize VPS deployment as declarative preset |

---

## 10. Key Files & Paths

| Resource | Path |
|----------|------|
| Ansible FIPS role | `tollgate-infrastructure-kit/ansible/roles/fips/` |
| FIPS playbook | `tollgate-infrastructure-kit/ansible/playbooks/13-fips.yml` |
| SMOKE-1 test suite | `test-fips-exit-smoke/tests/api/test_fips_exit_node.py` |
| VPS1 credentials | `tollgate-infrastructure-kit/.env` (gitignored) |
| FIPS Docker image | `fips-raw-node:latest` |
| FIPS upstream | `~/fips/` (ngit: nostr://.../fips) |
| nsite playbook | `tollgate-infrastructure-kit/ansible/playbooks/08-nsite-gateway.yml` |
| Daily smoke cron | `fips-exit-smoke-daily` (cron job ID: a2e789e3e49f) |
| FIPS health cron | `FIPS VPS1 Health` (cron job ID: b86d9af271a7) |
| Kanban board | `hermes kanban --board fips` |
| Status dashboard | `fips-exit.orangesync.tech` |

---

## 11. Contact / Ownership

- **Maintainer:** c08r4d0r / @9cab90c7-125c-488a-b568-4a4bc0e9f627
- **Contributors:** Amperstrand / @1624e1bb-94ef-46d1-b03b-f067ea320af9, Origami74
- **Upstream FIPS:** jmcorgan (johnathan@corganlabs.com) — v0.4.0 pinned
- **VPS:** TollGate infrastructure (tollgate-infrastructure-kit)
- **Relays:** relay1.orangesync.tech, ngit1.orangesync.tech, relay.damus.io, nos.lol
