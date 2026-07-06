# FIPS Exit Node PoC — Status & Design

**Date:** 2026-07-06
**Author:** Hermes Agent (on behalf of c03rad0r)
**Repo:** github.com/OpenTollGate/fips-exit-e2e (proposed)
**Domain:** fips-exit.orangesync.tech (pending nsite deploy)
**VPS:** 66.92.204.38 (VPS1 via tollgate-infrastructure-kit)
**FIPS Version:** v0.5.0-dev (pinned rev 30c5808e09-dirty)

---

## 1. What This Is

A proof-of-concept FIPS mesh exit node — a VPS that acts as a WireGuard-based
internet gateway for FIPS mesh network peers. FIPS mesh peers connect to the
VPS over UDP/TCP, get routed through a WireGuard tunnel, and reach the public
internet via nftables MASQUERADE.

### Architecture

```
[FIPS Mesh Peer] --UDP/TCP--> [VPS1: FIPS Daemon] --TUN(fips0)--> [Kernel IP Forward]
                                   |
                              [WireGuard: wg0] --10.99.99.0/24--
                                   |
                              [nftables fips-exit table: MASQUERADE on eth0]
                                   |
                              [PUBLIC INTERNET]
```

### Components Deployed on VPS1

| Component | Status | Details |
|-----------|--------|---------|
| FIPS daemon | ✅ Running | v0.5.0-dev, UDP:2121, TCP:8443, persistent identity |
| WireGuard wg0 | ✅ Running | :51821, 10.99.99.1/24, peer 10.99.99.2 |
| nftables fips-exit | ✅ Loaded | MASQUERADE on eth0 for 10.99.99.0/24 |
| IP forwarding | ✅ Enabled | net.ipv4.ip_forward=1, persistent |
| external_addr | ✅ Set | 66.92.204.38:8443 |
| Kind 30078 route advert | ✅ Published | On relay.damus.io, nos.lol |

---

## 2. Phase 1 Complete (All EXIT Tasks ✅)

| Task | ID | Status | What was done |
|------|----|--------|---------------|
| EXIT-1 | t_1d9cacc0 | ✅ done | FIPS config fix — added external_addr + Nostr discovery |
| EXIT-2 | t_69e6a4ca | ✅ done | WireGuard wg0 + nftables MASQUERADE deployed via SSH script |
| EXIT-3 | t_9f3e5f68 | ✅ done | Docker test peer (nvpn node-a) created on VPS1 |
| EXIT-4 | t_025ad644 | ✅ done | E2E mesh → peer → WG → internet path verified |
| EXIT-5 | t_bffcfbf7 | ✅ done | 5 SMOKE-1 tests PASS against live VPS1 (16s, no skips) |
| EXIT-6 | t_d0fce0ec | ✅ done | Cashu payment gate integrated |
| EXIT-7 | t_06c2e379 | ✅ done | Kind 30078 route advertisement published |
| EXIT-8 | t_8adff567 | ✅ done | Soveng demo preparation |

---

## 3. Raw FIPS Docker Node (Unblocks Phase 2)

The nvpn-based test peer had a reconnect bug — after VPS1 FIPS restart, it
would not re-establish the FIPS handshake (stuck on "0 peers, 0 reachable").
We built a standalone raw FIPS Docker container to replace it.

### Design

**Repo:** `~/repos/fips-exit-e2e/` → proposed `github.com/OpenTollGate/fips-exit-e2e`

```
fips-exit-e2e/
├── docker/
│   ├── Dockerfile.fips-node    # debian:trixie-slim + fips daemon + wireguard-tools
│   └── entrypoint.sh           # Generates fips.yaml from env vars
├── docker-compose.yml          # Topology: fips-node + probe container
├── scripts/
│   ├── build-fips.sh           # Build fips from source, copy binary
│   ├── run-node.sh             # Run raw FIPS Docker node
│   ├── verify-node.sh          # Verify connection to VPS1
│   └── get-vps1-identity.sh    # Get VPS1 FIPS identity
├── .env.template               # Config template
├── fips                        # FIPS daemon binary (22.7MB)
├── fipsctl                     # FIPS control tool
├── fipstop                     # FIPS stop tool
└── docs/
    ├── STATUS-AND-DESIGN.md    # This file
    └── PLAN.md                 # Full Phases 2-3 plan
```

### How it Works

1. Container starts with environment variables:
   - `FIPS_NSEC` — test peer's Nostr secret key
   - `FIPS_PEER_NPUB` — VPS1's Nostr public key
   - `FIPS_PEER_ADDR` — VPS1's address (66.92.204.38:2121)
2. Entrypoint generates `/etc/fips/fips.yaml` from env vars
3. FIPS daemon starts, connects to VPS1 via UDP :2121
4. FIPS mesh handshake completes, peer promoted to active

### Verified Working

✅ Config generated correctly
✅ FIPS daemon starts (v0.5.0-dev)
✅ Transports initialized: 2 (UDP + TCP)
✅ Connection to VPS1 established: 1 peer
✅ FIPS handshake complete: "Peer promoted to active peer=vps1-exit"
✅ Mesh parent switched: "new_parent=vps1-exit"
✅ Survives VPS1 restart (raw FIPS reconnects automatically — unlike nvpn)

### Key Decision: `--network host`

The container MUST run with `--network host` because FIPS needs to bind UDP :2121
and TCP :8443 — Docker bridge NAT doesn't expose these ports correctly for the
FIPS mesh protocol's UDP transport.

---

## 4. Test Infrastructure (SMOKE-1)

5 smoke tests in `~/worktrees/test-fips-exit-smoke/tests/api/test_fips_exit_node.py`
that verify the exit node against live VPS1:

| Test | What it checks |
|------|----------------|
| `test_wireguard_peer_has_recent_handshake` | WG handshake within 5min |
| `test_nftables_fips_exit_masquerade_loaded` | fips-exit nftables table |
| `test_ip_forwarding_enabled_and_egress_works` | ip_forward=1 + internet reachable |
| `test_nostr_kind_30078_route_advert_published` | Route advert on relays |
| `test_wireguard_tunnel_has_bidirectional_traffic` | rx>0 AND tx>0 on WG peer |

All 5 pass against VPS1 in ~16s. Feature-detection gating: tests skip gracefully
when VPS is unreachable or feature absent.

---

## 5. Pending Decisions

1. **Repo location:** `github.com/OpenTollGate/fips-exit-e2e` (confirmed by c08r4d0r)
2. **FIPS version:** Pin to rev `30c5808e09-dirty` (works now, don't follow master)
3. **Domain:** `fips-exit.orangesync.tech` — deploy via nsite when dashboard ready
4. **Test frequency:** Daily + per push (cron + GitHub Actions)
5. **Deployment:** Use existing VPS infrastructure (tollgate-infrastructure-kit)
6. **Publishing:** All work to ngit (nostr git) + GitHub

---

## 6. Next Steps (Phase 2)

1. **Move repo** to OpenTollGate org on GitHub
2. **Create ngit repo** and push all work
3. **Build FIPS from a pinned version** — save the working binary, create a CI build
4. **Fix `--network host` dependency** — try Docker MACVLAN or user-defined bridge
   with explicit port mapping as a cleaner alternative
5. **Port SMOKE-1 tests** to work against local Docker topology
6. **Add Nostr discovery relays** to config so test peer finds VPS1 via relay
7. **Set up daily cron** — run SMOKE-1 against VPS1 every 24h, alert on failure
8. **Set up per-push CI** — GitHub Actions builds Docker image, runs smoke tests
9. **Deploy nsite dashboard** at fips-exit.orangesync.tech
10. **Document** AGENTS.md + runbook for operators

---

## 7. Related Projects & Patterns Studied

### conwrt (Amperstrand)
OpenWrt flashing framework. Key patterns: use case plugin system, device model
JSON, E2E test infrastructure. **Not directly applicable** — it's OpenWrt UCI-
oriented, not Docker YAML.

### physical-router-test-automation (OpenTollGate)
Multi-tier test framework for TollGate routers. Key patterns: VM provider
abstraction (SHC/GCP/Local), feature detection gating, cloud lab fire-and-forget,
`gate_bug_fix()` regression tracking. **Partially applicable** — adopt the VM
abstraction and test gating, but build FIPS-specific test logic.

### nostr-vpn (c03rad0r)
nvpn + FIPS integration. Contains `Dockerfile.e2e` (nvpn with embedded-fips),
`docker-compose.exit-node-e2e.yml` (full e2e topology), and test scripts.
**Source of the raw FIPS approach** — our Dockerfile.fips-node was adapted
from the FIPS upstream sidecar Dockerfile.
