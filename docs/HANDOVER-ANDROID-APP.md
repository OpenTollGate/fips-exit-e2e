# Handover: Android FIPS Mesh Client App

**Author:** c08r4d0r via Hermes Agent (Jul 2026)
**Purpose:** Give a future LLM session everything it needs to build an Android app that runs the FIPS mesh protocol and uses the existing FIPS exit node as its internet gateway.

---

## What You're Building

An Android app that:

1. Runs the FIPS mesh protocol natively (or wrapped) on Android
2. Connects as a peer to the existing VPS1 exit node at `66.92.204.38:2121` (UDP) or `66.92.204.38:8443` (TCP)
3. Routes device traffic through the FIPS mesh → exit node → WireGuard → internet
4. Shows connection status, data counters, and peer list

---

## 1. What Exists Today

### Exit Node (VPS1 — 66.92.204.38)

A fully operational FIPS mesh exit node running on a Debian 13 VPS:

| Component | Detail |
|-----------|--------|
| **FIPS daemon** | v0.4.0-derivative (Rust binary, 17.6MB), running as systemd unit |
| **Transports** | UDP :2121 (primary), TCP :8443 (fallback) |
| **TUN interface** | `fips0`, MTU 1280 |
| **WireGuard** | `wg0`, 10.99.99.1/24, peer at 10.99.99.2 |
| **NAT** | nftables MASQUERADE from wg0 → eth0 |
| **Identity** | `npub1mqelkzqp4659fws35h2wvr7z9caka5ml8qddj3ssnwaulwpxdd9sdc3esw` |
| **Nostr relays** | relay1.orangesync.tech, relay2.orangesync.tech, ngit1.orangesync.tech, ngit2.orangesync.tech, relay.damus.io, nos.lol |
| **Route advert** | Kind 30078 published to damus.io, nos.lol |
| **External addr** | 66.92.204.38:8443 (TCP advertised) |

### FIPS Protocol (v0.4.0 — pinned, DO NOT track master)

- **Language:** Rust
- **Upstream:** `github.com/jmcorgan/fips`
- **Pin commit:** `da2d0b7408fc98ffc17671b5a49a4d76ce504292`
- **Local fork:** ngit at `nostr://npub12m5exm2uk3xa674cc5r0hlyvccs5xxn7qv83ezuteefv5972nquq4j4szl/relay.ngit.dev/fips`
- **Handshake:** Noise XK (initiator role for clients)
- **Mesh discovery:** Nostr-based (kind 30078 events, app tag `fips-overlay-v1`)

### Repos

| Repo | URL | Contents |
|------|-----|----------|
| `fips-exit-node` | `github.com/OpenTollGate/fips-exit-node` | STATUS.md, README, ansible roles, SMOKE-1 tests, dashboard HTML |
| `fips-exit-e2e` | `github.com/OpenTollGate/fips-exit-e2e` | Docker test harness, scripts, FIPS binaries, docs |
| `fips` (fork) | ngit relay.ngit.dev/fips | Upstream source + 4 cherry-picked commits |

---

## 2. Architecture (Data Flow)

```
[Android App: FIPS Client]
       │
       │  FIPS Mesh Protocol (UDP :2121 or TCP :8443)
       │  Noise XK handshake → Encrypted session
       ▼
[VPS1: FIPS Daemon v0.4.0]
       │
       │  Decapsulates → delivers to local fips0 TUN
       ▼
[Kernel IP Forward: fips0 → wg0]
       │
       ▼
[WireGuard wg0 — 10.99.99.0/24 tunnel]
       │
       ▼
[nftables fips-exit: MASQUERADE on eth0]
       │
       ▼
[PUBLIC INTERNET]
```

**Critical path for Android:** The app must:
1. Generate or import a Nostr keypair (nsec)
2. Start FIPS protocol with the exit node's npub as peer
3. Create a TUN device locally (requires `android.permission.TUN` or VPN API)
4. Route selected traffic through that TUN → FIPS mesh → exit node → internet

---

## 3. FIPS Config — What the Android App Must Generate

The VPS1 exit node uses this YAML config. The Android client's config is similar but with client-side settings:

```yaml
# Client-side FIPS config
node:
  identity:
    nsec: "<generated-or-imported-client-nsec>"
  discovery:
    nostr:
      enabled: true
      policy: configured_only
      app: "fips-overlay-v1"
      advertise: false           # Phone should NOT advertise as exit

tun:
  enabled: true
  name: fips0
  mtu: 1280

transports:
  udp:
    bind_addr: "0.0.0.0:2121"    # Or ephemeral port
  tcp:
    bind_addr: "0.0.0.0:8443"

# Static peer config to reach the exit node
peers:
  - npub: "npub1mqelkzqp4659fws35h2wvr7z9caka5ml8qddj3ssnwaulwpxdd9sdc3esw"
    alias: "vps1-exit"
    addresses:
      - transport: udp
        addr: "66.92.204.38:2121"
    connect_policy: auto_connect
```

---

## 4. What Works (Verified)

| Feature | Status | Evidence |
|---------|--------|----------|
| FIPS daemon start | ✅ | Raw Docker container starts cleanly |
| UDP transport | ✅ | Connects to VPS1 :2121 |
| TCP transport | ✅ | Config has TCP :8443 as fallback |
| Noise XK handshake | ✅ | "Session established (initiator, XK)" logged |
| Peer promotion | ✅ | "Connection promoted to active peer" confirmed |
| Mesh parent switch | ✅ | "new_parent=vps1-exit" logged |
| Encrypted session | ✅ | Full duplex after handshake |
| E2E traffic | ✅ | nostr-vpn test proved egress (5 ICMP pkts) |

**The Docker test peer** (`fips-raw-node:latest` at 231MB) is the closest thing to what the Android client needs to do. It was built from `~/fips/examples/sidecar-nostr-relay/Dockerfile` — an env-var-driven entrypoint that generates the FIPS YAML at runtime.

---

## 5. What Did NOT Work / Lessons Learned

### ❌ nvpn Test Peer Doesn't Reconnect
The nostr-vpn 4.0.87 Docker image used as a test peer (EXIT-3) did **not** reconnect after VPS1 FIPS was restarted. The container kept running but the FIPS connection stayed dead. We abandoned it in favor of a **raw FIPS Docker node** built directly from the upstream sidecar Dockerfile — which connects fresh every time.

**Lesson for Android:** The FIPS client must handle reconnection. If the exit node restarts or the transport drops, the client must re-initiate the Noise XK handshake. Don't rely on any wrapper (nvpn) to handle this — do it at the FIPS protocol level.

### ❌ FIPS v0.4.0 Has No Reconnect Logic
The upstream daemon doesn't retry failed peer connections. If the first handshake attempt fails, it moves on. The Android app needs **its own reconnect loop**: retry every 5s with exponential backoff, capped at 60s.

### ❌ FIPS v0.5.0-dev (master) is Breaking
Upstream is in the middle of a **sans-io refactor**. The v0.4.0 tag is the LAST stable pre-refactor release. Master may compile but the config format and protocol behavior have changed. **DO NOT use master. Pin to v0.4.0 commit `da2d0b7408fc98ffc17671b5a49a4d76ce504292`.**

### ❌ TUN Device is Mandatory
FIPS requires `/dev/net/tun` — it creates a `fips0` virtual interface. On Android you have two options:
- **Option A (recommended):** Use Android's `VpnService` API which creates a TUN-like interface automatically. You get all traffic routing for free.
- **Option B (harder):** Compile FIPS with TUN support and get root/JNI access. Requires root or a system app.

### ❌ nftables Not Available on Android
The exit node uses nftables for MASQUERADE. The Android client does NOT need nftables — it's a **client**, not an exit. It only needs to connect to the mesh and route its own traffic through the mesh.

### ❌ Rust Cross-Compilation is Tricky
FIPS is Rust. Cross-compiling for Android (arm64-v8a, armeabi-v7a, x86_64) requires the Android NDK and careful cargo config. See section 8 below.

### ❌ VPS1 FIPS Has No Rate Limiting (Yet)
Phase 2 planned work includes per-npub rate limiting. Until then: one Android client can consume all available egress bandwidth. Don't stress-test the exit without coordination.

---

## 6. What We Learned About FIPS's Internal Architecture

Based on the Rust source structure at v0.4.0:

```
fips/
├── Cargo.toml              # Workspace root
├── fips-core/              # Protocol core (Noise handshake, mesh logic)
│   ├── src/
│   │   ├── handshake.rs    # Noise XK protocol implementation
│   │   ├── mesh.rs         # Mesh topology management
│   │   ├── session.rs      # Encrypted session state
│   │   └── transport.rs    # Transport abstraction
│   └── Cargo.toml
├── fips-daemon/            # The actual daemon binary
│   ├── src/
│   │   ├── main.rs         # Entrypoint, config parser, signal handling
│   │   ├── tun.rs          # TUN interface management
│   │   └── nostr.rs        # Nostr discovery/advertisement
│   └── Cargo.toml
├── fips-net/               # Network layer (UDP, TCP transports)
│   ├── src/
│   │   ├── noise.rs        # Noise protocol wire encoding
│   │   └── socket.rs       # Socket management
│   └── Cargo.toml
├── examples/               # Example deployments
│   └── sidecar-nostr-relay/ # The Dockerfile we used
└── docs/                   # Protocol specification
```

**For Android, the most relevant crates are:**
- `fips-core` — the pure logic (Noise handshake, mesh, session). This could be extracted as a library.
- `fips-net` — the network I/O (UDP/TCP sockets). Android-compatible with `std::net`.
- `fips-daemon/tun.rs` — TUN interface. This needs Android-specific reimplementation.

The Noise XK handshake sequence is:
1. Client generates ephemeral keypair
2. Client sends Noise handshake message (static + ephemeral public keys)
3. Server responds with its static key + encrypted session key material
4. Both derive shared secret via Noise XK pattern
5. Symmetric encrypted session established

---

## 7. What the Android App Needs To Do

### Core Requirements

| # | Requirement | Approach |
|---|------------|----------|
| 1 | Nostr keypair | Generate `nsec`/`npub` on first launch (e.g. `nostr-tool` or `nak` lib). Store in Android KeyStore. |
| 2 | FIPS protocol | Port fips-core + fips-net to Android (Rust → JNI, or reimplement in Kotlin/Java). See section 8. |
| 3 | TUN/VPN | Use Android `VpnService.Builder` to create a TUN interface and route traffic. |
| 4 | Connect to exit node | Initiate Noise XK handshake to `66.92.204.38:2121` (UDP). Fallback to TCP `:8443`. |
| 5 | Encrypt/decrypt | All mesh traffic is encrypted via Noise session. The app just sends/receives encrypted packets. |
| 6 | Route traffic | Once FIPS session is established, route device traffic through `fips0` TUN → exit node. |
| 7 | Status display | Show: connected/disconnected, peer list, bytes transferred, handshake time, exit node info. |
| 8 | Reconnection | Auto-retry: 5s → 10s → 20s → 40s → 60s max, capped. Reset on successful handshake. |

### Nice-to-Have

| # | Feature |
|---|---------|
| 9 | Select which apps route through FIPS (per-app VPN) |
| 10 | Kill switch: block non-FIPS traffic when disconnected |
| 11 | Show exit node's route advertisement (kind 30078 from Nostr) |
| 12 | Dark mode / Material Design 3 |

---

## 8. Rust → Android Cross-Compilation (The Hard Part)

The FIPS daemon is pure Rust with these Android-compatible dependencies:

- `tokio` — async runtime (Android compatible with the right features)
- `noise-rust-crypto` — Noise protocol (pure Rust, no C deps)
- `ed25519-dalek` — key exchange
- `serde` / `serde_yaml` — config parsing
- `tun` crate — TUN creation (needs platform-specific support)
- `reqwest` / `nostr-sdk` — Nostr API calls (optional for static config)

### Strategy Options

**Option A: Rust cross-compilation via JNI (Recommended)**

```bash
# Install Android NDK targets
rustup target add aarch64-linux-android armv7-linux-androideabi x86_64-linux-android

# Create .cargo/config.toml with NDK linker paths
# Build .so for each ABI
cargo build --target aarch64-linux-android --release
cargo build --target armv7-linux-androideabi --release
cargo build --target x86_64-linux-android --release
```

The resulting `.so` files are loaded via JNI in Android. The TUN/VPN part uses Android's `VpnService` API in Kotlin, then passes packets to the Rust FIPS core via JNI for encryption/decryption.

**Option B: Pure Kotlin reimplementation** (Harder, more work)

Reimplement the Noise XK handshake + mesh protocol from scratch in Kotlin. The `fips-core` crate is ~3K LOC. A Kotlin port would be similar. Not recommended unless cross-compilation fails.

**Option C: Termux + FIPS binary** (Quick-and-dirty proof of concept)

Install FIPS binary in Termux on Android. Works for testing but not for a production app. No TUN/VPN integration without root.

---

## 9. Key Test Vectors

To verify the Android client works, test against these known-good values:

| Parameter | Value |
|-----------|-------|
| Exit node IP | `66.92.204.38` |
| FIPS port (UDP) | `2121` |
| FIPS port (TCP) | `8443` |
| Exit node npub | `npub1mqelkzqp4659fws35h2wvr7z9caka5ml8qddj3ssnwaulwpxdd9sdc3esw` |
| App tag | `fips-overlay-v1` |
| Handshake type | `Noise XK` (initiator role) |
| TUN MTU | `1280` |
| Expected log | `"Connection promoted to active peer"` |
| Expected log | `"Session established (initiator, XK)"` |
| Expected log | `"new_parent=vps1-exit"` |

**Known good FIPS client config** (from Docker test peer):
```yaml
# /etc/fips/fips.yaml on the peer
node:
  identity:
    nsec: "<test-peer-nsec>"
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
dns:
  enabled: true
  bind_addr: "127.0.0.1"
transports:
  udp:
    bind_addr: "0.0.0.0:2121"
    advertise_on_nostr: true
    mtu: 1472
  tcp:
    bind_addr: "0.0.0.0:8443"
    advertise_on_nostr: true
peers:
  - npub: "npub1mqelkzqp4659fws35h2wvr7z9caka5ml8qddj3ssnwaulwpxdd9sdc3esw"
    alias: "vps1-exit"
    addresses:
      - transport: udp
        addr: "66.92.204.38:2121"
    connect_policy: auto_connect
```

---

## 10. What NOT To Do

- ❌ **Don't track FIPS master.** Pin to v0.4.0 commit `da2d0b74`. The sans-io refactor will change everything.
- ❌ **Don't use nvpn.** It breaks on reconnect. Go raw FIPS protocol.
- ❌ **Don't hardcode secrets.** Use Android KeyStore for nssec.
- ❌ **Don't assume the exit node has infinite bandwidth.** This is a PoC on a shared VPS.
- ❌ **Don't require root.** Use `VpnService` API (no root needed).
- ❌ **Don't ignore reconnection.** The exit node may restart for updates.

---

## 11. Quick Reference: Files to Read First

| File | What It Teaches |
|------|-----------------|
| `github.com/OpenTollGate/fips-exit-node/STATUS.md` | Full architecture, design decisions, deployment details |
| `github.com/OpenTollGate/fips-exit-node/fips-pin.txt` | FIPS version pin policy |
| `github.com/jmcorgan/fips/examples/sidecar-nostr-relay/` | The canonical Docker pattern we used |
| `github.com/c03rad0r/fips-exit-e2e/docker/entrypoint.sh` | Env-var → FIPS YAML generator |
| `github.com/c03rad0r/fips-exit-e2e/docker/Dockerfile.fips-node` | Raw FIPS Docker container |
| `github.com/c03rad0r/fips-exit-e2e/scripts/run-and-verify.sh` | End-to-end test script |
| `github.com/jmcorgan/fips/fips-core/src/handshake.rs` | Noise XK handshake (the heart of the protocol) |
| `github.com/jmcorgan/fips/fips-daemon/src/main.rs` | FIPS daemon entrypoint |

---

## 12. Contacts

- **c08r4d0r** — project owner (Signal @+18102940908)
- **jmcorgan** — FIPS upstream maintainer (johnathan@corganlabs.com)
- **Amperstrand** — collaborator, conwrt UseCase patterns
- **Relay operators** — relay1.orangesync.tech, ngit1.orangesync.tech

---

*Generated 2026-07-06. Last known good state: Phase 1 complete, all 9 EXIT tasks done, daily smoke test running, FIPS v0.4.0 pinned.*
