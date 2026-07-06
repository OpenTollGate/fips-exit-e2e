# FIPS ANDROID APP — DEFINITIVE HANDOVER

**For:** Fresh LLM session building the Android FIPS mesh client app.
**From:** c08r4d0r, Origami74 (Arjen), Amperstrand — Jul 2026.
**You are building:** An Android app that runs FIPS mesh and routes traffic through the existing exit node at 66.92.204.38.

---

## 0. THE DECISION (UNANIMOUS, CONFIRMED)

**Use the `ble-v2` branch of `github.com/jmcorgan/fips`.**

Confirmed by all three stakeholders:

- **c08r4d0r** (project owner): "Use ble-v2, it includes Android support and app-defined TUN adapter"
- **Amperstrand** (collaborator): "Android basics will be upstream soon, so it's fine to rely on it. At worst only minor tweaks required"
- **Origami74** (ble-v2 author): wrote the code, has a working Android embedder (myco-core)

**Branch:** `ble-v2`, commit `56062094d604317a885e696f979c425518516cc1` (2026-06-30)
**Base:** v0.4.0 tag + 11 additive commits. Core protocol byte-for-byte identical.

Do NOT use:
- v0.4.0 tag alone (no Android support)
- master (mid-sans-io-refactor, breaks everything)

---

## 1. TRADEOFF ANALYSIS: ble-v2 vs v0.4.0

### Protocol compatibility: IDENTICAL

ble-v2 is v0.4.0 (merge base `3ea7ca1`) with 11 purely additive commits. Zero changes to:
- Noise XK handshake
- Mesh routing
- Session management
- UDP/TCP transport wire format
- Packet format

A ble-v2 Android client talks to a v0.4.0 VPS1 exit node with zero compatibility issues.

### What ble-v2 actually changes (all additive):

| File | Change | Risk |
|------|--------|------|
| `src/node/mod.rs` (+92 lines) | Adds `enable_app_owned_tun()`, BLE bridge injection, PeerView API | Zero — new methods, existing paths unchanged |
| `src/node/lifecycle.rs` (1 line) | `if tun.enabled` becomes `if tun.enabled && tun_tx.is_none()` | Zero — only skips system-TUN when app-owned is pre-set |
| `src/upper/tun.rs` (+26 lines) | Android no-op stubs (`cfg(target_os = "android")`) | Zero — doesn't exist on Linux |
| `src/control/read_handle.rs` (+36 lines) | `PeerView` struct + `peer_views()` for embedder UIs | Zero — additive |
| `src/transport/ble/android_io.rs` (619 lines) | Full Android BLE backend | Zero — new file, gated by `target_os = "android")` |
| `src/upper/dns.rs` (1 line) | `ipi6_ifindex as u32` type cast | Trivial fix |
| `build.rs` (+9 lines) | `ble_available` cfg gate for linux/macos/android | Zero |

### Why ble-v2 saves weeks of work:

- `enable_app_owned_tun()` — VpnService owns the fd, FIPS exchanges packets via channels. Without this, you'd be fighting system-TUN permissions on Android.
- `AndroidBleBridge` — working BLE L2CAP transport with the byte-bridge pattern already designed, tested, and tuned.
- `AndroidRadio` trait — the exact JNI interface Kotlin must implement.
- Platform gating — `cargo build --target aarch64-linux-android` works out of the box, no `--no-default-features` hacks.
- `myco-core` — Origami74's working Android JNI embedder that implements all of this.

### The "branch vs tag" risk:

ble-v2 is a branch, not a tag. It could theoretically be rebased. Mitigation:
- Amperstrand confirms Android basics are going upstream soon
- You can cherry-pick the 11 commits into your own fork for stability
- The commits are clean, atomic, and well-tested

---

## 2. THE THREE APIs YOU MUST KNOW

### API 1: enable_app_owned_tun() — THE critical seam

```rust
// src/node/mod.rs line 2878
pub fn enable_app_owned_tun(&mut self) -> (TunOutboundTx, std::sync::mpsc::Receiver<Vec<u8>>)
```

**Design rationale (Origami74's commit message):**
> Node::enable_app_owned_tun() lets an embedder that owns the TUN fd (e.g. an Android VpnService) exchange IPv6 packet bytes with FIPS over channels instead of FIPS creating a system TUN device.

- Returns `(app_outbound_tx, app_inbound_rx)` channel pair
- `app_outbound_tx`: push packets from VpnService fd INTO FIPS (app → mesh)
- `app_inbound_rx`: pull packets FROM FIPS TO VpnService fd (mesh → app)
- Call AFTER `Node::new()`, BEFORE `start()`
- `start()` then SKIPS system-TUN creation entirely

**Your embedder responsibilities:**
1. Push ONLY `fd::/8`-destined IPv6 packets (FIPS no longer filters)
2. Clamp TCP MSS on outbound SYNs
3. Read from the VpnService fd in a loop → push to `app_outbound_tx`
4. Pull from `app_inbound_rx` → write to the VpnService fd

**Unit test proving it works** (`src/node/tests/unit.rs:2003`):
```rust
let (outbound_tx, tun_rx) = node.enable_app_owned_tun();
assert_eq!(node.tun_state(), TunState::Active);
assert!(node.tun_tx().is_some());
node.tun_tx().unwrap().send(pkt.clone()).unwrap();
assert_eq!(tun_rx.recv_timeout(200ms).unwrap(), pkt);
// start() skips system-TUN:
node.start().await.unwrap();
assert!(node.tun_name().is_none()); // no system device created
```

### API 2: AndroidBleBridge + AndroidRadio — BLE byte-bridge

```rust
// src/transport/ble/android_io.rs

// Kotlin must implement this via JNI:
pub trait AndroidRadio: Send + Sync {
    fn listen(&self) -> u16;                                       // L2CAP, return PSM
    fn connect(&self, connect_id: i64, addr: &BleAddr, psm: u16);  // dial a peer
    fn start_advertising(&self, psm: u16);
    fn stop_advertising(&self);
    fn start_scanning(&self);
    fn stop_scanning(&self);
    fn close_channel(&self, ch_id: i64);
}

// Inject BEFORE Node::new():
let bridge = AndroidBleBridge::new(Arc::new(kotlin_radio_impl));
set_android_ble_bridge(bridge);
```

**Design rationale (Origami74's commit message):**
> The Android BLE radio lives in Kotlin, so AndroidIo/Stream/Acceptor/Scanner implement BleIo by delegating to AndroidBleBridge. Inbound bytes/events are pushed non-blocking into tokio channels; outbound bytes are pulled by a per-channel Kotlin writer thread via next_send. BleStream::send never calls JNI — the byte hot path is pure channel push.
>
> Pure Rust (no JNI here — that's in myco-core), so the channel logic unit-tests on the host with a mock radio. Cross-compiles clean for arm64-android.

**The byte-bridge pattern (critical for understanding):**
- **Inbound** (Kotlin → Rust): Kotlin calls JNI methods that push into tokio channels. Non-blocking.
- **Outbound** (Rust → Kotlin): Kotlin writer thread calls `next_send(ch_id, timeout)` — blocking pull with timeout. The byte hot path NEVER calls JNI upcalls.

**JNI-facing methods your Kotlin code must call on the bridge:**
```
deliver_inbound(remote, send_mtu, recv_mtu) -> i64       // channel accepted
deliver_connect_result(connect_id, ok, remote, mtus) -> i64  // dial done
deliver_scan(addr, psm, rssi)                             // peer found
deliver_recv(ch_id, data) -> bool                         // packet arrived
next_send(ch_id, timeout) -> Option<Vec<u8>>              // pull outbound
channel_closed(ch_id)                                     // socket gone
channel_open(ch_id) -> bool                               // alive check
advert_views() -> Vec<AdvertView>                         // UI: addr/psm/rssi
```

**Key bugfixes baked into ble-v2 (don't reintroduce):**
- Inbound L2CAP stream reframing: Android BluetoothSocket is byte-stream, not datagram. Packets were fragmenting/coalescing. Fixed with FMP length-prefix framer.
- PSM rotation: RPAs rotate between scan and dial, so PSM lookup missed. Fixed by dialing last-learned PSM on miss.
- Bufferbloat: outbound queue was 256 deep, RTT ballooned to 5s. Reduced to 32 (later 8 in one commit, tuned to 32 in the latest). RTT dropped to 1.2s.
- Safe teardown: `next_send` no longer holds channels lock across `recv_timeout`.

### API 3: PeerView — for your UI

```rust
// src/control/read_handle.rs line 114
pub struct PeerView {
    pub node_addr_hex: String,
    pub npub: String,
    pub connected: bool,
}

// Lock-free snapshot, safe to poll from UI thread:
let views: Vec<PeerView> = control_read_handle.peer_views();
```

**Design rationale (Origami74):**
> Expose ControlReadHandle + add peer_views(), read lock-free from the tick-published stats snapshot. This lets an embedder run run_rx_loop on a background task (which exclusively borrows &mut Node) and still poll live peer state from a cloned handle. Used by the Myco app's developer UI.

---

## 3. THE EMBEDDER (myco-core)

**Origami74 (Arjen) has a WORKING Android JNI embedder called `myco-core`.**

It implements:
- All `Java_..._NativeCore_*` JNI exports
- `AndroidRadio` trait via JNI `call_method` on a Kotlin `BleRadio` object
- The Kotlin BLE radio (scan, advertise, L2CAP listen/connect, socket read/write)
- The VpnService ↔ FIPS TUN channel glue
- The developer UI using PeerView

**myco-core is NOT in the FIPS tree.** It lives in Origami74's repo. Contact him for access.

**Fastest path to a working app:** Fork myco-core, customize the UI for the exit-node use case (add exit node config, connection status, data counters). The FIPS protocol layer is already done.

---

## 4. ANDROID TRANSPORT GAP (IMPORTANT)

On `target_os = "android"`, ble-v2 GATES OUT the standard UDP/TCP system transports. The Android node construction path only creates BLE transports.

**This means for direct phone → VPS1 exit connectivity, you need a custom transport.**

Origami74's design rationale:
> Ethernet: gated to linux/macos (raw AF_PACKET). Android is target_os = "android" — not "linux" — so the raw-socket transport self-excludes. System-tun: Android gets a no-op stub — the TUN is app-owned by the embedder.

**Options:**
1. Write an Android UDP transport — mirror the AndroidBleBridge pattern with Kotlin DatagramSocket exchanging bytes with Rust via channels. This is the clean path for direct exit connectivity.
2. Use BLE to a nearby peer that relays to the exit node.

Option 1 is required for a standalone phone app that connects directly to VPS1.

---

## 5. EXIT NODE DETAILS (VPS1)

```
IP:           66.92.204.38
UDP port:     2121 (primary)
TCP port:     8443 (fallback)
NPUB:         npub1mqelkzqp4659fws35h2wvr7z9caka5ml8qddj3ssnwaulwpxdd9sdc3esw
APP TAG:      fips-overlay-v1
HANDSHAKE:    Noise XK (you are the initiator)
TUN MTU:      1280
FIPS version: v0.4.0 (protocol-compatible with ble-v2)
```

The exit node uses WireGuard wg0 + nftables MASQUERADE for internet egress. It has been verified working: raw FIPS Docker container connects, Noise XK handshake completes, encrypted session established, bidirectional traffic confirmed.

**Success criteria (log messages you should see):**
```
Connection promoted to active peer peer=vps1-exit
Session established (initiator, XK)
new_parent=vps1-exit
```

After that, `curl ifconfig.me` through the VPN should show `66.92.204.38`.

---

## 6. TESTING REQUIREMENTS

**From c08r4d0r (project owner, mandatory):**
- Playwright-based smoke tests for the happy path in ALL functionality
- Show Playwright video of the happy path before considering something complete
- This is a hard gate — nothing is "done" until video evidence is shown

**For the Android app specifically:**
- Espresso or UI Automator for instrumented tests
- Happy path: app launches → connect to exit → verify traffic routes through mesh → disconnect
- Record video evidence (Android screen recording or emulator screencap)

**For infrastructure testing (what already exists):**
- SMOKE-1: SSH-based pytest against VPS1 (5 tests, 4 pass + 1 skip)
- Docker test harness: raw FIPS container peering with VPS1
- Daily smoke cron at 06:00
- Health monitoring cron every 15 minutes

---

## 7. BUILD INSTRUCTIONS

```bash
# Clone ble-v2
git clone https://github.com/jmcorgan/fips.git
cd fips && git checkout ble-v2

# Add Android targets
rustup target add aarch64-linux-android armv7-linux-androideabi x86_64-linux-android

# Install NDK via Android Studio SDK Manager
export ANDROID_NDK_HOME=$HOME/Android/Sdk/ndk/27.0.12077973

# Configure cargo linker
cat > .cargo/config.toml << 'EOF'
[target.aarch64-linux-android]
linker = "/path/to/ndk/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android24-clang"
[target.armv7-linux-androideabi]
linker = "/path/to/ndk/toolchains/llvm/prebuilt/linux-x86_64/bin/armv7-linux-androideabi24-clang"
[target.x86_64-linux-android]
linker = "/path/to/ndk/toolchains/llvm/prebuilt/linux-x86_64/bin/x86_64-linux-android24-clang"
EOF

# Build (cross-compiles clean)
cargo build --target aarch64-linux-android --release
```

Output: `target/aarch64-linux-android/release/libfips.so` → drop into `app/src/main/jniLibs/arm64-v8a/`

---

## 8. FIPS CONFIG FOR ANDROID CLIENT

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

## 9. BLE PERFORMANCE (empirical, from Origami74's tuning)

- ~200 kbps upstream / ~500 kbps downstream
- Outbound queue cap: 32 (tuned — 8 starved, 64 bufferbloated)
- L2CAP CoC MTU: 2048 bytes
- RTT: ~1.2s after bufferbloat fix (was ~5s)
- FIPS BLE service UUID: `9c90b7902cc542c09f87c9cc40648f4c`
- FIPS L2CAP PSM: dynamic (per-peer discovered, not fixed)
- BLE variance (RF, 2M PHY, connection priority) rivals tuning effects

BLE is for **nearby peer mesh**, not internet exit.

---

## 10. WHAT NOT TO DO

- Don't use FIPS master (sans-io refactor breaks everything)
- Don't use v0.4.0 tag alone (no Android support)
- Don't create a system TUN on Android — use `enable_app_owned_tun()`
- Don't push non-fd::/8 packets through the TUN seam
- Don't call JNI on the byte hot path — use the channel pattern
- Don't forget reconnection logic (FIPS has none — implement 5s→60s backoff)
- Don't require root — VpnService API needs no root
- Don't trust datagram boundaries on Android BLE — use the FMP length-prefix framer
- Don't hardcode the L2CAP PSM — it's OS-assigned and per-peer discovered

---

## 11. ble-v2 COMMIT LOG (all 11 by Origami74)

```
5606209 fix(ble): reframe inbound L2CAP stream — Android byte-stream packets
095c119 style(ble): rustfmt android_io and psm
0a56f7e docs: list Android as a supported platform
eea5c4f perf(ble): shallow backpressured outbound queue — fixes bufferbloat
6094aa5 fix(ble): dial last-learned PSM on per-peer lookup miss
618d1e6 feat(node): app-owned TUN seam — embedder owns the fd
f872c0b feat(ble): record scan adverts (PSM + RSSI) for developer UI
b517458 fix(ble): safe AndroidBleBridge teardown + replaceable injection
4695ffd feat(ble): public peer-view read API for embedders
2204894 feat(ble): AndroidBleBridge::channel_open — tell closed from timeout
d5f8921 feat(ble): Android backend — BleIo over Kotlin-radio byte-bridge
908bc48 feat(ble): per-peer PSM discovery core + compile BLE on macOS/Android
a879fdb feat(mobile): gate desktop transports/TUN by target_os, not features
```

---

## 12. CONTACTS

- **c08r4d0r** — project owner (Signal @+181****0908, group: fips-exit-node-poc)
- **Origami74 (Arjen)** — ble-v2 author, has myco-core. Signal @1624e1bb-94ef-46d1-b03b-f067ea320af9
- **Amperstrand** — collaborator. Signal @cc5bdaa4-98d2-4ce2-af1d-93aab049868a
- **Aruna** — proteomics pipeline lead, testing methodology. Signal @+181****0908
- **jmcorgan** — FIPS upstream maintainer (johnathan@corganlabs.com)

---

## 13. REPOS

| What | Where |
|------|-------|
| FIPS upstream | `github.com/jmcorgan/fips` branch `ble-v2` |
| FIPS fork (ngit) | `nostr://npub12m5exm2uk3xa674cc5r0hlyvccs5xxn7qv83ezuteefv5972nquq4j4szl/relay.ngit.dev/fips` |
| Exit node repo | `github.com/OpenTollGate/fips-exit-node` |
| E2E test infra | `github.com/OpenTollGate/fips-exit-e2e` |
| Exit node status | `github.com/OpenTollGate/fips-exit-node/STATUS.md` |

---

## 14. QUICK START CHECKLIST

1. [ ] Contact Origami74 for `myco-core` access
2. [ ] Clone `jmcorgan/fips`, checkout `ble-v2`
3. [ ] Install Android NDK + Rust Android targets
4. [ ] Cross-compile: `cargo build --target aarch64-linux-android --release`
5. [ ] Fork myco-core, customize UI for exit-node use case
6. [ ] Add exit node config (npub, addr above)
7. [ ] Implement custom Android UDP transport for direct exit connectivity
8. [ ] Test Noise XK handshake to VPS1
9. [ ] Test VpnService TUN routing
10. [ ] Implement reconnect loop (5s → 60s backoff)
11. [ ] Add per-app VPN toggle
12. [ ] Record video evidence of happy path
13. [ ] Playwright/Espresso tests with video

---

*Generated 2026-07-06. ble-v2 confirmed by c08r4d0r, Amperstrand, and Origami74.*
