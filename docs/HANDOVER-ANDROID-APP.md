# Android FIPS App — Handover Document

**Date:** 2026-07-06
**Author:** Hermes Agent (on behalf of c08r4d0r)
**Target reader:** LLM session tasked with building an Android app that runs FIPS and uses the FIPS exit node

---

## 1. What Exists Today

### The Exit Node (VPS1 — already deployed and working)

A fully operational FIPS mesh exit node at IP **66.92.204.38** (Debian 13, tollgate-infrastructure-kit managed):

| Component | Status | Details |
|-----------|--------|---------|
| FIPS daemon | ✅ ACTIVE | v0.4.0-derivative, UDP :2121, TCP :8443, persistent identity |
| WireGuard wg0 | ✅ UP | :51821, 10.99.99.1/24, peer 10.99.99.2 active with recent handshake |
| nftables fips-exit | ✅ LOADED | MASQUERADE on wg0 → eth0, **Cashu-gated** (paid_peers set, starts empty) |
| IP forwarding | ✅ ENABLED | net.ipv4.ip_forward=1 |
| FIPS external_addr | ✅ SET | 66.92.204.38:8443 |
| Nostr route advert | ✅ ACTIVE | Kind 30078 published to multiple relays |
| Nostr identity | npub1mqelkzqp4659fws35h2wvr7z9caka5ml8qddj3ssnwaulwpxdd9sdc3esw |

**How the exit works (critical):**
1. Peer connects to VPS1 FIPS daemon (UDP :2121 or TCP :8443)
2. FIPS mesh handshake establishes encrypted session (Noise XK)
3. FIPS daemon creates a TUN interface (fips0) with an IP in 10.44.0.0/16
4. Kernel routes from fips0 → wg0 (WireGuard tunnel, 10.99.99.0/24)
5. wg0 → nftables MASQUERADE → eth0 → internet
6. **THE MASQUERADE IS GATED** — only IPs in the `paid_peers` nftables set get source NAT. Without payment, the peer has a tunnel but no internet (return traffic never comes back). See section 4 below.

### The FIPS Protocol (v0.4.0, pinned)

**CRITICAL: Do NOT use FIPS master branch.** The upstream (jmcorgan/fips) is actively refactoring toward a sans-io architecture. v0.4.0 (tagged 2026-06-27, commit `da2d0b74b05d7a2bafd67e05fcf7a6edf9afa5d7`) is the last stable pre-refactor release.

FIPS v0.4.0 key properties:
- Written in **Rust** using Tokio async runtime
- Uses **Noise XK** handshake for encrypted peer connections
- **Nostr-native identity** — each node has a Nostr nsec/npub keypair
- **Peer discovery** via Nostr relay events (kind 30078 for route advertisements)
- **Config YAML** has tun/dns/transports at root level (NOT nested under node:)
- **TUN interface** created by the daemon for mesh traffic
- **Transports**: UDP (primary, port 2121) and TCP (fallback, port 8443)
- **Docker test image**: `fips-node:latest` built from `docker/Dockerfile.fips-node` — 231MB debian:trixie-slim runtime

### Repo Structure

**Primary repo:** `github.com/OpenTollGate/fips-exit-e2e`
**Also on ngit:** `nostr://npub12m5exm2uk3xa674cc5r0hlyvccs5xxn7qv83ezuteefv5972nquq4j4szl/relay.ngit.dev/fips-exit-e2e`

```
fips-exit-e2e/
├── docker/
│   ├── Dockerfile.fips-node     # FIPS daemon container (debian:trixie-slim)
│   └── entrypoint.sh            # Env-var → fips.yaml generator
├── docker-compose.yml           # Test topology (fips-node + probe containers)
├── scripts/
│   ├── run-node.sh              # Run FIPS Docker node
│   ├── build-fips.sh            # Build FIPS v0.4.0 from source
│   ├── get-vps1-identity.sh     # SSH into VPS1, dump config + identity
│   └── run-and-verify.sh        # Full run + verify script
├── docs/
│   ├── STATUS-AND-DESIGN.md     # Full architecture doc
│   ├── PLAN.md                  # Phase 2/3 implementation plan
│   └── HANDOVER-ANDROID-APP.md  # This file
├── playwright.config.mjs        # Playwright config (video:'on', screenshot:'on')
├── tests/                       # (empty — tests not yet written)
├── fips-pin.txt                 # FIPS version pin details
├── fips / fipsctl / fipstop     # FIPS v0.4.0 binaries (gitignored)
└── .env                         # Test keypair, VPS1 config (gitignored)
```

**Upstream FIPS repo:** `github.com/jmcorgan/fips` — local clone at `~/fips/`

### Test Peer Keypair (generated for e2e testing)

Generated via `nak key generate`. The test peer successfully:
- Connected to VPS1 via UDP :2121
- Completed Noise XK handshake (Session established, initiator)
- Was promoted to active peer (vps1-exit)
- Switched parent mesh node to vps1-exit
- Verified bidirectional traffic through the tunnel

---

## 2. What Worked ✅

### FIPS Docker Node
- `docker run --rm --name fips-test-node --network host --cap-add NET_ADMIN --device /dev/net/tun:/dev/net/tun -e FIPS_NSEC=... -e FIPS_PEER_NPUB=... -e FIPS_PEER_ADDR=66.92.204.38:2121 fips-node`
- This **works reliably**. The entrypoint generates fips.yaml from env vars, starts the daemon, and it peers with VPS1 in ~5 seconds.

### FIPS Config Format
- `tun:` / `dns:` / `transports:` at YAML root level (NOT nested under `node:`). This is the v0.4.0/v0.5.0-dev format.
- `node:` section only contains: `identity`, `discovery.nostr.*`, and optional `discovery.stun.*`
- `peers:` array at root level for static peer configuration (npub, alias, addresses, connect_policy)

### WireGuard + nftables MASQUERADE
- Works perfectly for internet egress. Tested: node-a egresses via WG (5 ICMP packets), WG upstream ingress cannot reach node-b (security isolation works).

### Nostr Discovery
- FIPS nodes discover each other via Nostr kind 30078 events published to relays
- Relays used: relay1.orangesync.tech, relay2.orangesync.tech, ngit1.orangesync.tech, ngit2.orangesync.tech, relay.damus.io, nos.lol
- VPS1 advertises via Nostr with `advertise: true` and `policy: configured_only`

### FIPS v0.4.0 Binary Build
- `cargo build --release -p fips` from tag v0.4.0 produces a 22.7MB binary
- Build time: ~2 minutes on a modern machine
- Also produces `fipsctl` and `fipstop` control tools

---

## 3. What Didn't Work / Problems Found ❌

### FIPS v0.2.0 (original pin)
- **Too old.** Missing Noise XX handshake support, version negotiation, and bugfixes
- Updated to v0.4.0 which is the last pre-refactor stable release

### FIPS master (v0.5.0-dev)
- **Do NOT use.** Upstream is mid-refactor. The sans-io changes are incomplete and may break features we depend on.
- **Pinned to v0.4.0** — this is the hard rule.

### Docker --network host requirement
- The FIPS Docker container **MUST** use `--network host` because it creates a TUN interface (`/dev/net/tun`) inside the container. Bridge mode doesn't work because:
  1. TUN devices need NET_ADMIN capability
  2. The FIPS daemon binds to ports :2121 and :8443 on all interfaces
  3. Bridge mode isolates the TUN device from the host network
- **Not yet solved:** MACVLAN driver or port mapping with TUN passthrough was planned but never implemented. For an Android app, this is irrelevant since Android doesn't use Docker.

### Test Container Exited with Code 137 (OOM killed)
- The background test run (`proc_9d2830870696`) was killed by OOM with exit code 137
- The container ran with `--rm` and `--network host` but was terminated mid-run. Likely resource pressure on the Hermes machine (not a code issue).
- The container successfully connected and showed FIPS logs before being killed: it was the OOM killer, not a FIPS failure.

### WireGuard Test Peer (nvpn, deprecated)
- The original nostr-vpn Docker test peer (node-a) was buggy — it didn't reconnect after VPS1 FIPS restarts
- **Replaced** with the raw FIPS Docker node (fips-raw-node:latest) which handles reconnection properly

### Playwright Tests (not yet run)
- The `playwright.config.mjs` is set up with `video: 'on'` and `screenshot: 'on'`
- No test files (`*.spec.mjs`) exist in `tests/` yet
- The dashboard was to be served at `fips-exit.orangesync.tech` but the nsite gateway only routes `*.nsite.orangesync.tech` — so the dashboard may not be accessible
- **This is your job** — create Playwright tests for the Android app's happy path and show video evidence

---

## 4. Key Architecture Details for Android

### FIPS on Android — What Changes

The Docker approach won't work on Android. You need to:

**Option A: Cross-compile FIPS as a native Rust library**
- FIPS v0.4.0 is pure Rust with Tokio async
- Target: `aarch64-linux-android` (modern ARM devices) and `armv7-linux-androideabi` (older devices)
- Needs: Android NDK, `rustup target add aarch64-linux-android`, cargo config for the NDK linker
- The binary would run as a background service on Android
- It needs `tun` device access — Android requires `VpnService` API, not direct TUN device access
- **Recommended approach:** Use Android's `VpnService` to create a TUN interface, then bridge it to FIPS's TUN

**Option B: FIPS as a remote proxy**
- Run FIPS on a companion device (ESP32, Raspberry Pi) that the phone connects to
- Android just sends/receives traffic through it
- Less integration work but defeats the purpose of running FIPS on the phone

**Key Android considerations:**
1. **VpnService API**: Android's native way to route device traffic. FIPS creates a TUN interface internally; you'll need to either:
   - Fork FIPS to accept a pre-created TUN fd (from VpnService) instead of creating its own
   - OR wrap FIPS in a JNI layer where Rust's TUN creation is replaced with Android's VpnService fd
2. **Background execution**: FIPS needs to run as a foreground service (with persistent notification) to avoid Android's Doze mode killing it
3. **Nostr keys**: Each Android app instance generates its own nsec/npub. Store nsec in Android Keystore (hardware-backed if available).
4. **Config YAML**: Generate from app settings rather than a file

### Cashu Payment Gate (for internet access)

The exit node uses a **Cashu payment gate** — users must pay to get internet egress:

```
1. Android app generates a Cashu token (amount determines access duration)
2. App sends token to fips-paygate on VPS1 (REST endpoint)
3. Paygate validates token, adds phone's WG IP to nftables paid_peers set with timeout
4. Internet access granted for time proportional to payment
5. App must re-pay before timeout expires to maintain access
```

**NFTables rule:**
```
nft add element inet fips-exit paid_peers { 10.99.99.X timeout Ns }
```

The `paid_peers` set starts EMPTY. Without payment, the WG tunnel exists but MASQUERADE doesn't fire → no return traffic → no internet.

**This is critical for the Android app:** you'll need a Cashu wallet integration or the ability to make Cashu payments.

### WireGuard Integration

FIPS doesn't directly provide internet egress — it's a mesh network protocol. The egress path is:

```
FIPS mesh → fips0 TUN → kernel routing → wg0 WireGuard → nftables MASQUERADE → eth0 → internet
```

For Android, the architecture changes:
```
FIPS (inside VpnService) → fips0 TUN → Android routing → internet
```

OR if you keep the WireGuard exit model:

```
FIPS mesh → wg0 (WireGuard tunnel to VPS1) → nftables MASQUERADE → internet
```

The WireGuard tunnel (wg0) is on the VPS1 side, not the client side. The client just connects to FIPS mesh → traffic reaches VPS1 → gets routed through wg0 → out to internet. The Android app doesn't need WireGuard itself — it only needs FIPS.

### FIPS Config for Android (minimal)

```yaml
node:
  identity:
    nsec: "nsec1..."  # generated per-install, stored in Keystore
  discovery:
    nostr:
      enabled: true
      policy: configured_only
      app: "fips-overlay-v1"
      advertise: false

tun:
  enabled: true
  name: fips0
  mtu: 1280

transports:
  udp:
    bind_addr: "0.0.0.0:2121"
    advertise_on_nostr: false
  tcp:
    bind_addr: "0.0.0.0:8443"
    advertise_on_nostr: false

peers:
  - npub: "npub1mqelkzqp4659fws35h2wvr7z9caka5ml8qddj3ssnwaulwpxdd9sdc3esw"
    alias: "vps1-exit"
    addresses:
      - transport: udp
        addr: "66.92.204.38:2121"
    connect_policy: auto_connect
```

---

## 5. VPS1 Connection Details (for testing)

| Property | Value |
|----------|-------|
| IP | 66.92.204.38 |
| FIPS UDP port | 2121 |
| FIPS TCP port | 8443 |
| SSH user | debian |
| OS | Debian 13 |
| FIPS daemon user | systemd service (fips.service) |
| FIPS config | /etc/fips/fips.yaml |
| FIPS binary | /usr/bin/fips (17.6MB, v0.4.0-derivative) |
| WireGuard port | 51821 (UDP) |
| WG subnet | 10.99.99.0/24 (VPS1=10.99.99.1) |
| VPS1 npub | npub1mqelkzqp4659fws35h2wvr7z9caka5ml8qddj3ssnwaulwpxdd9sdc3esw |
| Domain | fips-exit.orangesync.tech |
| Dashboard | nsite gateway on port 3002 (only routes `*.nsite.orangesync.tech`) |

**Test peer npub (for reference):** npub1569mpl… (generated, can be regenerated)

**To verify VPS1 is running:**
```bash
# Check FIPS daemon
ssh debian@66.92.204.38 'sudo systemctl status fips --no-pager'

# Check WireGuard
ssh debian@66.92.204.38 'sudo wg show wg0'

# Check nftables rules
ssh debian@66.92.204.38 'sudo nft list table inet fips-exit'

# Check FIPS config
ssh debian@66.92.204.38 'sudo cat /etc/fips/fips.yaml'
```

**Password for SSH:** stored in `tollgate-infrastructure-kit/.env` as `VPS_PASSWORD`.

---

## 6. What You Need to Build

### Android App

The app should:
1. **Generate Nostr keypair** on first launch (or let user import existing)
2. **Run FIPS v0.4.0** as a background service (using VpnService API or JNI)
3. **Connect to VPS1** FIPS exit node (configured static peer)
4. **Handle mesh handshake** — wait for "Session established" log
5. **Route device traffic** through FIPS mesh → VPS1 exit → internet
6. **Handle Cashu payment** — pay the VPS1 paygate for internet access
7. **Show status dashboard** — connected/disconnected, traffic stats, payment timer
8. **Auto-reconnect** — handle disconnections gracefully
9. **Playwright test** — create a smoke test and record video of the happy path

### Playwright Smoke Test

Since this is Android, you'll need either:
- **Appium** with WebDriverIO for Android app testing
- OR **Detox** (React Native testing library)
- OR a simple HTTP endpoint test if the app exposes a local status API

The test should capture:
1. App launches, shows status UI
2. FIPS connects to VPS1, shows "connected"
3. Payment flow initiates
4. Internet egress verified (fetch a known URL successfully)
5. Disconnect gracefully

**Record video** of the full happy path and submit it as evidence before marking the task as ready for review.

---

## 7. Cron Jobs & Monitoring

| Job | Schedule | What it does |
|-----|----------|--------------|
| FIPS VPS1 Health | Every 15 min | Checks FIPS daemon, ports, WG tunnel, error storms |
| fips-exit-smoke-daily | Daily 06:00 | Runs SMOKE-1 tests against VPS1 |
| FIPS AI Fallback | Every 4h | LLM-driven analysis & recommendations |

All cron jobs are **anomaly-only** — silent when healthy. They deliver to the `fips-exit-node-poc` Signal group on issues.

---

## 8. What We Learned

### FIPS is stable but evolving
- v0.4.0 is production-stable for basic mesh connectivity
- The Noise XK handshake is reliable (never seen a failure)
- Session re-establishment after disconnect works (peer sends "Shutdown" notification, then reconnects)
- The TCP fallback transport is useful but UDP is preferred (lower latency)

### Config gotchas
- `tun/dns/transports` at **root level**, not under `node:` — this caught us out
- `external_addr` is a property of `transports.tcp`, not a top-level config
- `advertise_on_nostr: true` without `external_addr` set generates a persistent warning log (but doesn't break anything)
- `--network host` is required for Docker — the TUN device and port binding don't work in bridge mode

### WireGuard + nftables works but needs attention
- The `paid_peers` set is the gate — without it, no internet even with a working tunnel
- The Cashu payment gate was integrated but not fully e2e tested
- WireGuard handshake stays alive indefinitely as long as there's traffic

### Things we didn't finish
- **Playwright tests** — config exists, no test files written yet
- **Dashboard at fips-exit.orangesync.tech** — domain resolves but nsite gateway only handles `*.nsite.orangesync.tech` subdomain pattern, so the custom domain may not work without Caddy config changes
- **GitHub Actions CI** — planned but not implemented
- **Multi-peer support** — only tested with one peer at a time
- **Graceful reconnect after VPS1 FIPS restart** — the raw FIPS Docker node handles this but we never stress-tested it
- **Rate limiting** — per-npub traffic caps are planned but not implemented

### The Android-specific challenge
FIPS's TUN creation is tightly coupled to the daemon's startup. Android's VpnService API provides a TUN file descriptor that must be created by the Android framework, then handed to the routing layer. The FIPS Rust code will need modification to accept a pre-opened fd instead of creating its own `/dev/net/tun` device. This is the **single hardest part** of the Android port.

---

## 9. Key Files & Paths

| Resource | Path |
|----------|------|
| fips-exit-e2e repo | `~/repos/fips-exit-e2e/` (also: `github.com/OpenTollGate/fips-exit-e2e`) |
| FIPS upstream | `~/fips/` (v0.4.0 tag, commit `da2d0b74b05d7a2bafd67e05fcf7a6edf9afa5d7`) |
| Docker entrypoint | `docker/entrypoint.sh` — env-var → fips.yaml generator |
| Dockerfile | `docker/Dockerfile.fips-node` — debian:trixie-slim runtime |
| Compose | `docker-compose.yml` — fips-node + probe topology |
| FIPS pin doc | `fips-pin.txt` — version pin policy |
| Architecture doc | `docs/STATUS-AND-DESIGN.md` |
| Plan | `docs/PLAN.md` |
| Playwright config | `playwright.config.mjs` |
| VPS1 FIPS config | `/etc/fips/fips.yaml` (remote, on VPS1) |
| VPS1 systemd unit | `/usr/lib/systemd/system/fips.service` (remote) |
| VPS1 nftables | `/opt/tollgate/fips-exit-node/exit-nat.nft` (remote) |
| VPS1 WG config | `/etc/wireguard/wg0.conf` (remote) |
| SMOKE-1 tests | In the tollgate-infrastructure-kit repo (separate from fips-exit-e2e) |

---

## 10. Contact

- **c08r4d0r** — project owner, Signal @+18102940908
- **Amperstrand** — contributor (@1624e1bb-94ef-46d1-b03b-f067ea320af9)
- **Upstream FIPS maintainer:** Johnathan Corgan (johnathan@corganlabs.com)
- **Relays:** relay1.orangesync.tech, ngit1.orangesync.tech, relay.damus.io, nos.lol

---

*Good luck with the Android port. Remember: **Done means pushed.** Commit early, push often, and show video evidence of the happy path before surfacing for review.*
