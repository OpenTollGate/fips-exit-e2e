# Android FIPS App — Handover for Android LLM Session

**Date:** 2026-07-06
**For:** LLM session building the Android FIPS mesh client app
**From:** Hermes Agent session (fips-exit-node-poc), c08r4d0r, Origami74

---

## TL;DR

Build an Android app that runs FIPS mesh networking and routes traffic through the exit node at 66.92.204.38. Use the **`ble-v2` branch** of `github.com/jmcorgan/fips` — it already has Android support built in. The exit node is live and tested. Your job is the Android client.

---

## 1. Use ble-v2. Here's Why.

**Advice from three project members:**

> **@cc5bdaa4:** "Android basics will be upstream soon, so it's fine to rely on it. At worst only minor tweaks required."

> **@1624e1bb (Arjen/Origami74):** Wrote all 12 ble-v2 commits. Has a working Android embedder (`myco-core`). Says the TUN seam + BLE backend are production-ready.

> **@9cab90c7 (c08r4d0r):** Verified via diff that ble-v2 is v0.4.0 + purely additive changes. Zero protocol changes.

### ble-v2 is NOT a fork. It's v0.4.0 + additive Android commits.

**Protocol-critical files: ZERO CHANGES**
- Noise XK handshake: unchanged
- Mesh routing: unchanged
- Session management: unchanged
- UDP/TCP transport: unchanged
- Packet format: unchanged

**What ble-v2 actually adds (verified from diff):**
1. `src/node/mod.rs` (+92 lines) — `enable_app_owned_tun()` method, purely additive
2. `src/transport/ble/` — new BLE transport module with Android backend
3. `src/upper/tun.rs` — Android gets no-op TUN stub (system TUN creation skipped)
4. `src/transport/mod.rs` — Ethernet gated by `target_os` (excluded on Android)
5. `build.rs` — auto-detects `target_os=android`, sets cfg flags automatically

**The 12 ble-v2 commits (all by Origami74, June 2026):**

```
5606209 fix(ble): reframe inbound L2CAP stream so packets survive non-SeqPacket backends
095c119 style(ble): rustfmt android_io and psm
0a56f7e docs: list Android as a supported platform
eea5c4f perf(ble): shallow, backpressured outbound queue to fix bufferbloat
6094aa5 fix(ble): dial the last-learned PSM on a per-peer lookup miss
618d1e6 feat(node): app-owned TUN seam — embedder owns the fd, FIPS uses channels
f872c0b feat(ble): record scan adverts (PSM + RSSI) for the developer UI
b517458 fix(ble): safe AndroidBleBridge teardown + replaceable injection
4695ffd feat(ble): public peer-view read API for embedders running run_rx_loop
2204894 feat(ble): AndroidBleBridge::channel_open — let next_send tell closed from timeout
d5f8921 feat(ble): Android backend — BleIo over a Kotlin-radio byte-bridge
908bc48 feat(ble): per-peer PSM discovery core + compile BLE on macOS/Android
a879fdb feat(mobile): gate desktop transports/TUN by target_os, not features
```

**Bottom line:** A phone running ble-v2 connects to the VPS1 exit (running v0.4.0) with byte-for-byte identical protocol compatibility. No interop risk.

---

## 2. The Exit Node (Already Live)

VPS1 at **66.92.204.38** is fully operational:

| Component | Status |
|-----------|--------|
| FIPS daemon v0.4.0 | ✅ Active (UDP :2121, TCP :8443) |
| WireGuard wg0 | ✅ Up (10.99.99.0/24) |
| nftables MASQUERADE | ✅ Loaded (Cashu-gated) |
| IP forwarding | ✅ Enabled |
| Nostr route advert | ✅ Published (kind 30078) |
| VPS1 npub | `npub1mqelkzqp4659fws35h2wvr7z9caka5ml8qddj3ssnwaulwpxdd9sdc3esw` |

**How egress works:**
1. Phone connects to VPS1 FIPS daemon (UDP/TCP)
2. Noise XK handshake → encrypted session
3. FIPS creates TUN interface, routes through WireGuard
4. nftables MASQUERADE → internet
5. **The MASQUERADE is Cashu-gated** — `paid_peers` nftables set starts empty. Without payment, tunnel exists but no internet (return traffic dropped).

---

## 3. The Integration API (from ble-v2 source)

### Node lifecycle for Android embedder

```rust
// 1. Create node from config
let mut node = Node::new(config)?;

// 2. Set up app-owned TUN (BEFORE start!)
//    This returns two channel ends and tells FIPS to skip system-TUN creation
let (app_outbound_tx, app_inbound_rx) = node.enable_app_owned_tun();

// 3. Get a control handle for lock-free UI polling
let control_handle = node.control_read_handle();

// 4. Start transports + initiate peer connections
node.start().await?;

// 5. Move node to background task for the RX event loop
tokio::spawn(async move {
    node.run_rx_loop().await
});

// Meanwhile on the VpnService fd thread:
//   loop {
//       let pkt = read_from_fd(tun_fd);     // VpnService TUN
//       app_outbound_tx.try_send(pkt);       // app → mesh
//       if let Ok(mesh_pkt) = app_inbound_rx.recv_timeout(Duration::from_millis(50)) {
//           write_to_fd(tun_fd, &mesh_pkt);  // mesh → app
//       }
//   }

// 6. To stop:
node.stop().await?;  // drops packet channel, rx_loop exits
```

### Channel types (IMPORTANT — they differ)

| Direction | Type | Why |
|-----------|------|-----|
| app → mesh | `tokio::sync::mpsc::Sender<Vec<u8>>` | Drained by `run_rx_loop` (async) |
| mesh → app | `std::sync::mpsc::Receiver<Vec<u8>>` | App reads from blocking JNI thread |

Use `recv_timeout` on `app_inbound_rx`, NOT blocking `recv`. The JNI thread needs to check a shutdown flag between polls.

### Transport availability on Android

| Transport | Android | Notes |
|-----------|:-------:|-------|
| UDP | ✅ | Primary — connects to exit node |
| TCP | ✅ | Fallback — connects to exit node |
| Ethernet | ❌ | Needs raw sockets (AF_PACKET) |
| BLE | ✅ | Peer-to-peer mesh (optional) |

The app connects to VPS1 directly via UDP/TCP. BLE is a bonus for nearby peer mesh.

### ControlReadHandle — UI polling without touching Node

```rust
pub struct PeerView {
    pub node_addr_hex: String,
    pub npub: String,
    pub connected: bool,
}

// Lock-free read via ArcSwap — call from any thread
let views: Vec<PeerView> = control_handle.peer_views();
```

### BLE Bridge (if implementing peer-to-peer)

**Rust side (already in FIPS):**
```rust
pub trait AndroidRadio: Send + Sync {
    fn listen(&self) -> u16;
    fn connect(&self, connect_id: i64, addr: &BleAddr, psm: u16);
    fn start_advertising(&self, psm: u16);
    fn stop_advertising(&self);
    fn start_scanning(&self);
    fn stop_scanning(&self);
    fn close_channel(&self, ch_id: i64);
}

pub fn set_android_ble_bridge(bridge: Arc<AndroidBleBridge>)

// JNI-facing methods:
impl AndroidBleBridge {
    pub fn new(radio: Arc<dyn AndroidRadio>) -> Arc<Self>
    pub fn deliver_inbound(&self, remote: BleAddr, send_mtu: u16, recv_mtu: u16) -> i64
    pub fn deliver_connect_result(&self, connect_id: i64, ok: bool, remote: BleAddr, send_mtu: u16, recv_mtu: u16) -> i64
    pub fn deliver_scan(&self, addr: BleAddr, psm: u16, rssi: i32)
    pub fn deliver_recv(&self, ch_id: i64, data: &[u8]) -> bool
    pub fn next_send(&self, ch_id: i64, timeout: Duration) -> Option<Vec<u8>>
    pub fn channel_open(&self, ch_id: i64) -> bool
}
```

**Kotlin side (you write):**
- `BleRadio` class implementing radio operations via JNI
- Per-channel writer threads pulling via `next_send` (blocking with timeout)
- `BluetoothLeScanner` / `BluetoothLeAdvertiser` / L2CAP socket management
- BLE throughput: ~200/500 kbps up/down (empirical)
- Outbound queue depth: 32 (empirically tuned, not proven optimum)

---

## 4. Embedder Contract (MUST FOLLOW)

Three responsibilities the system-TUN reader normally handles, bypassed in app-owned mode:

1. **Filter destinations:** Push only `fd00::/8`-destined IPv6 packets to `app_outbound_tx`. FIPS does NOT filter in app-owned mode. Non-fd00::/8 packets will be misrouted.

2. **Clamp TCP MSS:** On outbound SYNs, clamp MSS to fit within TUN MTU. Without this, cold TCP connections wedge — silent drops, no PTB feedback through userspace TUN. This is the #1 cause of "connected but no data flows."

3. **Use recv_timeout:** When pulling from `app_inbound_rx`, use `recv_timeout(Duration)` so the JNI thread can check a shutdown flag between polls for clean service teardown.

---

## 5. Cross-Compilation

```bash
rustup target add aarch64-linux-android armv7-linux-androideabi

# ~/.cargo/config.toml:
# [target.aarch64-linux-android]
# linker = "$NDK/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android24-clang"

# Build — NO special flags needed, build.rs auto-detects target_os=android
cargo build --target aarch64-linux-android --release -p fips
```

**Build.rs auto-sets these cfg gates:**
```
target_os = "android" → ble_available (BLE module compiles)
target_os = "android" → DefaultBleTransport = BleTransport<AndroidIo>
target_os = "android" → system-tun = no-op stub
target_os = "android" → Ethernet excluded
```

**Watch out for:**
- `ring = "0.17"` — crypto with C/assembly. Needs NDK C compiler. May need `RING_*` env vars.
- `tun = "0.8.7"` — compiles on Android but TUN creation path is skipped (app-owned TUN). Effectively unused at runtime.
- `nostr-sdk = "0.44"` — may need `rustls-tls` instead of native-tls for Android TLS.

---

## 6. Config for Android Client

```yaml
node:
  identity:
    nsec: "nsec1..."  # generated per-install, stored in Android Keystore
  discovery:
    nostr:
      enabled: true
      policy: configured_only
      app: "fips-overlay-v1"
      advertise: false

tun:
  enabled: true
  # On Android with enable_app_owned_tun(), these are informational only.
  # The VpnService owns the fd.
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

## 7. Cashu Payment Gate

Internet access requires Cashu payment. The `paid_peers` nftables set starts empty:

```
nft add element inet fips-exit paid_peers { 10.99.99.X timeout Ns }
```

**Android payment flow:**
1. App generates Cashu token
2. App sends token to fips-paygate on VPS1 (REST endpoint)
3. Paygate validates, adds phone's WG IP to `paid_peers` with timeout
4. Internet granted for time proportional to payment
5. App must re-pay before timeout

---

## 8. Testing

### What's been tested (exit node side)
- FIPS v0.4.0 Docker test peer connects to VPS1: ✅
- Noise XK handshake: ✅ (never seen a failure)
- WireGuard + nftables MASQUERADE egress: ✅
- Session re-establishment after disconnect: ✅
- SMOKE-1 tests (5 tests, 4 pass + 1 conditional skip): ✅

### What has NOT been tested
- Android client (no Android device or emulator available to this session)
- ble-v2 cross-compilation to `aarch64-linux-android` (not attempted)
- The `enable_app_owned_tun()` channel round-trip on a real device
- BLE transport between two physical devices
- Cashu payment flow end-to-end on mobile

### How to verify your Android build works

**Success indicators (from VPS1 logs when a peer connects):**
```
Connection promoted to active peer peer=vps1-exit
Session established (initiator, XK)
new_parent=vps1-exit
```

**Test vectors:**
| Parameter | Value |
|-----------|-------|
| Exit node IP | `66.92.204.38` |
| FIPS UDP | `2121` |
| FIPS TCP | `8443` |
| Exit npub | `npub1mqelkzqp4659fws35h2wvr7z9caka5ml8qddj3ssnwaulwpxdd9sdc3esw` |
| App tag | `fips-overlay-v1` |
| Handshake | Noise XK (phone is initiator) |
| TUN MTU | `1280` |

**SSH to VPS1 for debugging:**
```bash
ssh debian@66.92.204.38 'sudo systemctl status fips --no-pager'
ssh debian@66.92.204.38 'sudo wg show wg0'
ssh debian@66.92.204.38 'sudo nft list table inet fips-exit'
ssh debian@66.92.204.38 'sudo journalctl -u fips --no-pager -n 50'
```

---

## 9. The Fastest Path: myco-core

FIPS source comments reference **`myco-core`** — Origami74's working Android embedder. It implements:
- `Java_..._NativeCore_*` JNI exports
- `AndroidRadio` trait via JNI
- Kotlin BLE radio (scan, advertise, L2CAP, sockets)
- VpnService ↔ FIPS TUN channel glue

**myco-core is NOT in the FIPS repo.** Contact Origami74 (Arjen) for access. It already works with ble-v2. Fork it and customize the UI for the exit-node use case.

---

## 10. What NOT To Do

- ❌ **Don't use FIPS master.** Master is mid-sans-io-refactor. Use ble-v2.
- ❌ **Don't create a system TUN on Android.** Use `enable_app_owned_tun()`.
- ❌ **Don't push non-fd00::/8 packets through the TUN seam.** FIPS doesn't filter them in app-owned mode.
- ❌ **Don't skip TCP MSS clamping.** Cold connections will wedge silently.
- ❌ **Don't use blocking recv on app_inbound_rx.** Use recv_timeout for clean shutdown.
- ❌ **Don't hardcode nsec.** Use Android Keystore.
- ❌ **Don't require root.** VpnService API needs no root.
- ❌ **Don't ignore reconnection.** Exit node may restart. Implement backoff (5s → 60s).

---

## 11. Key Source Files in ble-v2

| File | Purpose |
|------|---------|
| `src/node/mod.rs` (~line 2878) | `enable_app_owned_tun()` — the TUN seam |
| `src/node/lifecycle.rs` (line 1057) | `start()` — skips TUN creation when app-owned |
| `src/node/lifecycle.rs` (line 1463) | `stop()` — clean shutdown |
| `src/node/handlers/rx_loop.rs` | `run_rx_loop()` — the main event loop |
| `src/control/read_handle.rs` | `ControlReadHandle` + `PeerView` — lock-free UI polling |
| `src/transport/ble/android_io.rs` | Full Android BLE backend (734 lines) |
| `src/transport/ble/mod.rs` | BLE module, wires `AndroidIo` as `DefaultBleTransport` |
| `src/transport/ble/psm.rs` | Per-peer PSM (L2CAP port) discovery |
| `src/upper/tun.rs` | TUN device — Android gets no-op stub |
| `docs/design/fips-ipv6-adapter.md` | Design doc — "App-Owned TUN" section documents the contract |
| `build.rs` | Auto-detects target_os, sets cfg gates |
| `Cargo.toml` | Platform-gated dependencies |

---

## 12. Repos

| Repo | URL |
|------|-----|
| FIPS upstream | `github.com/jmcorgan/fips` branch `ble-v2` |
| Exit node e2e | `github.com/OpenTollGate/fips-exit-e2e` |
| Exit node e2e (ngit) | `nostr://npub12m5exm2uk3xa674cc5r0hlyvccs5xxn7qv83ezuteefv5972nquq4j4szl/relay.ngit.dev/fips-exit-e2e` |

---

## 13. Contacts

- **c08r4d0r** (@9cab90c7) — project owner, Signal +181****0908
- **Origami74 / Arjen** (@1624e1bb) — ble-v2 author, has myco-core reference embedder, Signal @1624e1bb-94ef-46d1-b03b-f067ea320af9
- **@cc5bdaa4** — confirmed Android basics heading upstream
- **Johnathan Corgan** — FIPS upstream maintainer, johnathan@corganlabs.com
- **Relays:** relay1.orangesync.tech, ngit1.orangesync.tech, relay.damus.io, nos.lol

---

*Start from ble-v2. Contact Arjen for myco-core. The exit node is live at 66.92.204.38. Show video evidence of the happy path before surfacing for review.*
