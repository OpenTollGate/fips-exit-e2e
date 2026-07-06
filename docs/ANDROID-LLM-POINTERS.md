# ANDROID FIPS APP — POINTERS FOR LLM SESSION

**You are building an Android app that runs the FIPS mesh protocol and uses the existing FIPS exit node as its internet gateway. This document is your entire starting context.**

---

## DECISION: USE THE `ble-v2` BRANCH

**Branch:** `ble-v2` on `github.com/jmcorgan/fips`
**Commit:** `56062094d604317a885e696f979c425518516cc1` (2026-06-30)
**Base:** v0.4.0 tag + 11 additive commits by Origami74 (Arjen)

This is CONFIRMED by the project owner. Android basics will be upstreamed soon. At worst you'll need minor tweaks. Do NOT use v0.4.0 tag alone — it has zero Android support. Do NOT use master — it's mid-sans-io-refactor and breaks everything.

ble-v2 is protocol-identical to v0.4.0. It adds transport backends (BLE) and embedder APIs (app-owned TUN) without touching the wire format, handshake, or routing. A ble-v2 client talks to a v0.4.0 exit node with zero compatibility issues.

---

## THE THREE APIs YOU NEED

### 1. App-Owned TUN (THE critical one)

```rust
// src/node/mod.rs line 2878
impl Node {
    /// Call after Node::new(), BEFORE start().
    /// Returns two channels. The app owns the TUN fd (Android VpnService).
    pub fn enable_app_owned_tun(&mut self) -> (TunOutboundTx, std::sync::mpsc::Receiver<Vec<u8>>)
}
```

- `TunOutboundTx` = `tokio::sync::mpsc::Sender<Vec<u8>>` — push app→mesh packets here
- `Receiver<Vec<u8>>` = std mpsc receiver — pull mesh→app packets from here
- After calling this, `start()` SKIPS system-TUN creation entirely
- Your embedder MUST: push only fd::/8-destined IPv6 packets, clamp TCP MSS on outbound SYNs

**Unit test proving it works** (from `src/node/tests/unit.rs` line 2003):
```rust
let (outbound_tx, tun_rx) = node.enable_app_owned_tun();
assert_eq!(node.tun_state(), TunState::Active);
assert!(node.tun_tx().is_some());
// mesh→app round trip works:
node.tun_tx().unwrap().send(pkt.clone()).unwrap();
assert_eq!(tun_rx.recv_timeout(200ms).unwrap(), pkt);
```

### 2. Android BLE Bridge

```rust
// src/transport/ble/android_io.rs

// The Kotlin radio must implement this trait (via JNI):
pub trait AndroidRadio: Send + Sync {
    fn listen(&self) -> u16;                    // Open L2CAP, return PSM
    fn connect(&self, connect_id: i64, addr: &BleAddr, psm: u16);
    fn start_advertising(&self, psm: u16);
    fn stop_advertising(&self);
    fn start_scanning(&self);
    fn stop_scanning(&self);
    fn close_channel(&self, ch_id: i64);
}

// Create and inject BEFORE Node::new():
let bridge = AndroidBleBridge::new(Arc::new(your_radio_impl));
set_android_ble_bridge(bridge);
// Node::new() will pick it up via android_ble_bridge()
```

The bridge byte-bridge pattern (from source comments):
- **Inbound** (Kotlin→Rust): Kotlin calls `bridge.deliver_recv(ch_id, data)` — non-blocking push into tokio channels
- **Outbound** (Rust→Kotlin): Kotlin writer thread calls `bridge.next_send(ch_id, timeout)` — blocking pull with timeout
- The byte hot path NEVER calls JNI. `BleStream::send` only pushes into a std channel.

Key bridge methods your JNI layer must call:
```
deliver_inbound(remote, send_mtu, recv_mtu) -> i64   // Kotlin accepted a channel
deliver_connect_result(connect_id, ok, remote, mtus) -> i64  // dial completed
deliver_scan(addr, psm, rssi)                         // peer discovered
deliver_recv(ch_id, data) -> bool                     // packet arrived
next_send(ch_id, timeout) -> Option<Vec<u8>>          // pull outbound packet
channel_closed(ch_id)                                 // socket gone
channel_open(ch_id) -> bool                           // check if alive
advert_views() -> Vec<AdvertView>                     // for UI: addr/psm/rssi
```

### 3. PeerView (for your UI)

```rust
// src/control/read_handle.rs line 114
pub struct PeerView {
    pub node_addr_hex: String,  // peer's node address
    pub npub: String,           // resolved Nostr pubkey
    pub connected: bool,        // in active peer table?
}

// Get a snapshot from a ControlReadHandle clone:
let views: Vec<PeerView> = control_read_handle.peer_views();
```

This is lock-free, safe to poll from the UI thread.

---

## THE EMBEDDER PATTERN (myco-core)

Origami74 (Arjen, Signal @1624e1bb-94ef-46d1-b03b-f067ea320af9) has a WORKING Android JNI embedder called **`myco-core`**. It implements:

- All `Java_..._NativeCore_*` JNI exports
- `AndroidRadio` trait via JNI `call_method` on a Kotlin `BleRadio` object
- The Kotlin BLE radio (scan, advertise, L2CAP listen/connect, socket read/write)
- The VpnService ↔ FIPS TUN channel glue

**`myco-core` is NOT in the FIPS tree.** Contact Origami74 for access. Forking myco-core is the fastest path to a working app — it already works with ble-v2.

---

## EXIT NODE (VPS1) — WHAT YOU'RE CONNECTING TO

```
IP:         66.92.204.38
UDP port:   2121 (primary)
TCP port:   8443 (fallback)
NPUB:       npub1mqelkzqp4659fws35h2wvr7z9caka5ml8qddj3ssnwaulwpxdd9sdc3esw
APP TAG:    fips-overlay-v1
HANDSHAKE:  Noise XK (you are the initiator)
TUN MTU:    1280
```

Verified working: raw FIPS Docker container connects, Noise XK handshake completes, encrypted session established, bidirectional traffic confirmed, internet egress through WireGuard+nftables.

---

## YOUR APP'S ARCHITECTURE

```
┌─────────────── Android App ───────────────┐
│                                            │
│  Kotlin UI (Compose)                       │
│    ├── Connect/Disconnect button           │
│    ├── Peer list (from PeerView)           │
│    ├── Data counters (rx/tx bytes)         │
│    ├── BLE scan results (from AdvertView)  │
│    └── Settings (which apps route)         │
│                                            │
│  VpnService (owns TUN fd)                  │
│    ├── ParcelFileDescriptor from Builder   │
│    ├── Read thread: fd → JNI → app_outbound│
│    └── Write thread: app_inbound → JNI → fd│
│                                            │
│  ──────── JNI boundary ────────            │
│                                            │
│  Rust FIPS Core (libfips.so)               │
│    ├── Node (from ble-v2 branch)           │
│    ├── enable_app_owned_tun() → channels   │
│    ├── set_android_ble_bridge() → BLE      │
│    ├── Noise XK handshake                  │
│    ├── Mesh routing (fd00::/8 IPv6 ULA)    │
│    ├── UDP/TCP transport (if you add one)  │
│    └── BLE transport (AndroidBleBridge)    │
│                                            │
└────────────────────────────────────────────┘
        │                        │
   UDP/TCP :2121/:8443    BLE L2CAP CoC
        │                        │
        ▼                        ▼
  [VPS1 Exit Node]      [Nearby FIPS Peer]
```

---

## ANDROID TRANSPORT GAP (IMPORTANT)

On `target_os = "android"`, ble-v2 GATES OUT the standard UDP/TCP system transports (they're `cfg(unix)` but the Android construction path only creates BLE transports). This means:

**For BLE mesh:** Works out of the box via AndroidBleBridge.

**For direct internet exit (phone → VPS1):** You need to add a custom transport. Two options:
1. Write an Android UDP transport (similar byte-bridge pattern as BLE — Kotlin owns the DatagramSocket, exchanges bytes with Rust via channels)
2. Use BLE to a nearby peer that relays to the exit node

Option 1 is what you need for a standalone phone app. The pattern is well-established in the codebase — mirror AndroidBleBridge for UDP.

---

## BUILD INSTRUCTIONS

```bash
# 1. Clone and checkout
git clone https://github.com/jmcorgan/fips.git
cd fips && git checkout ble-v2

# 2. Add Android targets
rustup target add aarch64-linux-android armv7-linux-androideabi x86_64-linux-android

# 3. Install NDK (via Android Studio SDK Manager)
export ANDROID_NDK_HOME=$HOME/Android/Sdk/ndk/27.0.12077973

# 4. Configure cargo linker
cat > .cargo/config.toml <<EOF
[target.aarch64-linux-android]
linker = "$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android24-clang"
[target.armv7-linux-androideabi]
linker = "$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/bin/armv7-linux-androideabi24-clang"
[target.x86_64-linux-android]
linker = "$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/bin/x86_64-linux-android24-clang"
EOF

# 5. Build (ble-v2 cross-compiles clean)
cargo build --target aarch64-linux-android --release
```

The `.so` file lands in `target/aarch64-linux-android/release/libfips.so`. Drop it into `app/src/main/jniLibs/arm64-v8a/`.

---

## FIPS CONFIG FOR ANDROID CLIENT

```yaml
node:
  identity:
    nsec: "<from Android KeyStore>"
  discovery:
    nostr:
      enabled: true
      policy: configured_only
      app: "fips-overlay-v1"
      advertise: false

tun:
  enabled: true              # required, but start() skips system-TUN with app-owned
  name: fips0
  mtu: 1280

transports:
  ble:
    enabled: true             # optional, for nearby peer mesh
    mtu: 2048

peers:
  - npub: "npub1mqelkzqp4659fws35h2wvr7z9caka5ml8qddj3ssnwaulwpxdd9sdc3esw"
    alias: "vps1-exit"
    addresses:
      - transport: udp
        addr: "66.92.204.38:2121"
    connect_policy: auto_connect
```

---

## SUCCESS CRITERIA (verified log messages)

When your app connects to VPS1, you should see these in the FIPS logs:

```
Connection promoted to active peer peer=vps1-exit
Session established (initiator, XK)
new_parent=vps1-exit
```

After that, bidirectional traffic should flow. Test with `curl ifconfig.me` through the VPN — it should show VPS1's IP (66.92.204.38).

---

## BLE PERFORMANCE (empirical, from source comments)

- ~200 kbps upstream / ~500 kbps downstream
- Outbound queue cap: 32 packets (tuned)
- L2CAP CoC MTU: 2048 bytes
- FIPS BLE service UUID: `9c90b7902cc542c09f87c9cc40648f4c`
- FIPS L2CAP PSM: `0x0085` (dynamic range)
- BLE is for nearby peer mesh, NOT for internet exit

---

## WHAT NOT TO DO

- Don't use FIPS master branch (sans-io refactor in progress)
- Don't use v0.4.0 tag alone (no Android support)
- Don't create a system TUN on Android — use `enable_app_owned_tun()`
- Don't push non-fd::/8 packets through the TUN seam
- Don't call JNI on the byte hot path — use the channel pattern
- Don't forget reconnection logic (FIPS has none — implement 5s→60s backoff)
- Don't require root — VpnService API needs no root

---

## CONTACTS

- **c08r4d0r** — project owner (Signal group: fips-exit-node-poc)
- **Origami74 (Arjen)** — ble-v2 author, has myco-core (working Android embedder). Signal @1624e1bb-94ef-46d1-b03b-f067ea320af9
- **jmcorgan** — FIPS upstream maintainer

---

## REPOS

| What | Where |
|------|-------|
| FIPS upstream | `github.com/jmcorgan/fips` branch `ble-v2` |
| FIPS fork (ngit) | `nostr://npub12m5exm2uk3xa674cc5r0hlyvccs5xxn7qv83ezuteefv5972nquq4j4szl/relay.ngit.dev/fips` |
| Exit node repo | `github.com/OpenTollGate/fips-exit-node` |
| E2E test infra | `github.com/OpenTollGate/fips-exit-e2e` |
| Full handover doc | `github.com/OpenTollGate/fips-exit-e2e/docs/HANDOVER-ANDROID-APP.md` |
| Status & design | `github.com/OpenTollGate/fips-exit-node/STATUS.md` |

---

*Generated 2026-07-06. ble-v2 confirmed as the path forward by c08r4d0r.*
