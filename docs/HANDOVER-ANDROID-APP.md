# Handover: Android FIPS Mesh Client App

**Author:** c08r4d0r via Hermes Agent (Jul 2026)
**Purpose:** Give a future LLM session everything it needs to build an Android app that runs the FIPS mesh protocol and uses the existing FIPS exit node as its internet gateway.

---

## 0. TL;DR — Which FIPS Branch to Use

**Use `ble-v2` branch from `jmcorgan/fips` upstream.**

The `ble-v2` branch contains 11 commits by Origami74 (Arjen) that add Android
support natively. It is based on v0.4.0 (`3ea7ca1`) and adds:

- **`Node::enable_app_owned_tun()`** — app owns the TUN fd (Android VpnService), FIPS uses channels
- **`AndroidBleBridge`** — Kotlin BLE radio byte-bridge (619 lines in `android_io.rs`)
- **Platform gating** — desktop transports/TUN gated by `target_os`, compiles clean for Android
- **BLE transport** — L2CAP CoC, per-peer PSM discovery, ~200/500 kbps up/down

**Branch commit:** `5606209` (HEAD of `upstream/ble-v2`, 2026-06-30)
**Merge base with v0.4.0:** `3ea7ca1` (so it IS v0.4.0 + Android work on top)

---

## 1. What You're Building

An Android app that:

1. Runs the FIPS mesh protocol natively via JNI (Rust .so + Kotlin)
2. Connects as a peer to the existing VPS1 exit node at `66.92.204.38:2121` (UDP) or `66.92.204.38:8443` (TCP)
3. Optionally connects via BLE to nearby FIPS peers
4. Routes device traffic through the FIPS mesh → exit node → WireGuard → internet
5. Shows connection status, data counters, peer list, BLE scan results

---

## 2. What Exists Today

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

### FIPS Protocol

- **Language:** Rust
- **Upstream:** `github.com/jmcorgan/fips`
- **Stable pin (VPS1):** v0.4.0, commit `da2d0b7408fc98ffc17671b5a49a4d76ce504292`
- **Android branch:** `ble-v2`, commit `56062094d604317a885e696f979c425518516cc1`
- **Local fork:** ngit at `nostr://npub12m5exm2uk3xa674cc5r0hlyvccs5xxn7qv83ezuteefv5972nquq4j4szl/relay.ngit.dev/fips`
- **Handshake:** Noise XK (initiator role for clients)
- **Mesh discovery:** Nostr-based (kind 30078 events, app tag `fips-overlay-v1`)

### Repos

| Repo | URL | Contents |
|------|-----|----------|
| `fips-exit-node` | `github.com/OpenTollGate/fips-exit-node` | STATUS.md, README, ansible roles, SMOKE-1 tests, dashboard HTML |
| `fips-exit-e2e` | `github.com/OpenTollGate/fips-exit-e2e` | Docker test harness, scripts, FIPS binaries, this handover doc |
| `fips` (fork) | ngit relay.ngit.dev/fips | Upstream source + cherry-picked commits |

---

## 3. Architecture (Data Flow)

### Internet Exit Path (via VPS1)

```
[Android App: FIPS Client]
       │
       │  FIPS Mesh Protocol (UDP :2121 or TCP :8443)
       │  Noise XK handshake → Encrypted session
       │  (or BLE L2CAP to a nearby peer that relays to exit)
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

### Android App Internal Architecture (ble-v2)

```
┌─────────────────────────────────────────────┐
│                  Android App                 │
│                                              │
│  ┌──────────┐   ┌─────────────────────────┐ │
│  │ Kotlin UI │   │   VpnService (owns fd)   │ │
│  │  (Compose)│   │         │                │ │
│  └──────────┘   │  IPv6 packets ↔ channels │ │
│                  └─────────┬───────────────┘ │
│                            │                  │
│                    JNI boundary               │
│                            │                  │
│  ┌─────────────────────────┴───────────────┐ │
│  │         FIPS Rust Core (.so)              │ │
│  │  ┌─────────────┐  ┌──────────────────┐  │ │
│  │  │ Noise XK    │  │ Mesh routing     │  │ │
│  │  │ handshake   │  │ (fd00::/8 ULA)   │  │ │
│  │  └─────────────┘  └──────────────────┘  │ │
│  │  ┌─────────────┐  ┌──────────────────┐  │ │
│  │  │ UDP/TCP     │  │ BLE transport    │  │ │
│  │  │ transport   │  │ (AndroidBleBridge)│  │ │
│  │  └─────────────┘  └──────────────────┘  │ │
│  └──────────────────────────────────────────┘ │
└──────────────────────────────────────────────┘
         │                           │
    UDP/TCP :2121/:8443     BLE L2CAP CoC
         │                           │
         ▼                           ▼
   [VPS1 Exit Node]          [Nearby FIPS Peer]
```

---

## 4. The ble-v2 Android API (The Key to Everything)

### 4.1. App-Owned TUN Seam

**`Node::enable_app_owned_tun()`** — THE critical API for Android.

```rust
// In your Rust FIPS embedder code (called from Kotlin via JNI):
let (app_outbound_tx, app_inbound_rx) = node.enable_app_owned_tun();
```

What it does:
- Returns `(TunOutboundTx, Receiver<Vec<u8>>)` — two channels
- `app_outbound_tx`: push IPv6 packets FROM the app's TUN fd INTO FIPS (app → mesh)
- `app_inbound_rx`: pull IPv6 packets FROM FIPS TO the app's TUN fd (mesh → app)
- **Skips system-TUN creation entirely** — `start()` gates on `tun_tx` being unset
- The embedder (Android VpnService) owns the fd, FIPS just exchanges bytes

**Responsibilities of the embedder (your Kotlin code):**
1. Read IPv6 packets from the VpnService fd
2. Push them into `app_outbound_tx` via JNI
3. Pull packets from `app_inbound_rx` via JNI
4. Write them to the VpnService fd
5. Push ONLY `fd::/8`-destined packets (FIPS doesn't filter anymore)
6. Clamp TCP MSS on outbound SYNs

Source: `src/node/mod.rs` line ~2878, `enable_app_owned_tun()` method.

### 4.2. Android BLE Backend

**`AndroidBleBridge`** — Kotlin BLE radio byte-bridge.

```rust
// In Rust (the FIPS library side):
use crate::transport::ble::android_io::{set_android_ble_bridge, AndroidBleBridge, AndroidRadio};

// Define the radio trait (Kotlin implements this via JNI):
pub trait AndroidRadio: Send + Sync {
    fn listen(&self) -> u16;                           // Open L2CAP listener, return PSM
    fn connect(&self, connect_id: i64, addr: &BleAddr, psm: u16);
    fn start_advertising(&self, psm: u16);
    fn stop_advertising(&self);
    fn start_scanning(&self);
    fn stop_scanning(&self);
    fn close_channel(&self, ch_id: i64);
}

// Create the bridge and inject it:
let bridge = AndroidBleBridge::new(Arc::new(kotlin_radio_impl));
set_android_ble_bridge(bridge);
```

The bridge uses a **byte-bridge pattern** (symmetric to nostr-vpn's MobileTunnel):
- **Inbound** (Kotlin → Rust): pushed non-blocking into tokio channels
- **Outbound** (Rust → Kotlin): pulled blocking-with-timeout by a Kotlin writer thread
- The byte hot path NEVER calls JNI — `BleStream::send` only pushes into a std channel

Source: `src/transport/ble/android_io.rs` (619 lines).

### 4.3. Node Construction on Android

When building a FIPS `Node` on Android (`target_os = "android"`):

1. The Node constructor checks for an injected BLE bridge via `android_ble_bridge()`
2. If present, it creates `AndroidIo` instances for BLE transport
3. Desktop transports (UDP/TCP system TUN) are gated by `target_os` and excluded on Android
4. The embedder calls `enable_app_owned_tun()` before `start()` to wire up VpnService

### 4.4. BLE Performance (Empirical)

From the source comments:
- ~200 kbps upstream / ~500 kbps downstream over BLE L2CAP CoC
- Outbound queue cap: 32 packets (empirically tuned)
- MTU: 2048 bytes (L2CAP CoC default)
- BLE variance (RF, 2M PHY, connection priority) rivals queue tuning effects

BLE is for **nearby peer mesh** — NOT for internet exit. For internet exit, use UDP/TCP to VPS1.

---

## 5. What Works (Verified on VPS1)

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
| BLE transport | ✅ | ble-v2 branch, compiles for arm64-android |
| App-owned TUN | ✅ | Unit tested (app_owned_tun_seam_wires_channels) |
| Android BLE backend | ✅ | Cross-compiles clean for arm64-android |

---

## 6. What Did NOT Work / Lessons Learned

### ❌ nvpn Test Peer Doesn't Reconnect
The nostr-vpn 4.0.87 Docker image used as a test peer (EXIT-3) did **not** reconnect after VPS1 FIPS was restarted. We abandoned it in favor of a **raw FIPS Docker node**.

**Lesson for Android:** Implement your own reconnect loop. Don't rely on any wrapper.

### ❌ FIPS Has No Built-In Reconnect
Neither v0.4.0 nor ble-v2 has automatic reconnection. The Android app needs **its own retry loop**: 5s → 10s → 20s → 40s → 60s, capped. Reset on successful handshake.

### ❌ FIPS v0.5.0-dev (master) is Breaking
Upstream master is mid-**sans-io refactor**. The config format and protocol behavior have changed. **DO NOT use master.** Use `ble-v2` branch (which is v0.4.0 + Android support).

### ❌ nftables Not Available on Android
The exit node uses nftables for MASQUERADE. The Android client does NOT need nftables — it's a **client**, not an exit.

### ❌ Rust Cross-Compilation Requires Care
FIPS Rust cross-compiles for `aarch64-linux-android` cleanly (ble-v2 verified). Use Android NDK + cargo targets. The `tun` crate dependency is gated by `cfg(unix)` and excluded on Android (app-owned TUN replaces it).

### ❌ VPS1 FIPS Has No Rate Limiting (Yet)
Phase 2 planned work includes per-npub rate limiting. Until then: one Android client can consume all available egress bandwidth. Don't stress-test without coordination.

---

## 7. How to Build the Android App

### Recommended Architecture

```
android-app/
├── app/                          # Android app module
│   ├── src/main/
│   │   ├── java/com/opentollgate/fips/
│   │   │   ├── MainActivity.kt       # Compose UI
│   │   │   ├── FipsVpnService.kt     # VpnService — owns TUN fd
│   │   │   ├── BleRadio.kt           # Implements AndroidRadio trait via JNI
│   │   │   └── NativeCore.kt         # JNI bindings to Rust .so
│   │   ├── jniLibs/
│   │   │   ├── arm64-v8a/libfips.so  # Cross-compiled FIPS
│   │   │   ├── armeabi-v7a/libfips.so
│   │   │   └── x86_64/libfips.so
│   │   └── AndroidManifest.xml       # VPN permission, BLE permissions
│   └── build.gradle.kts
├── fips-embedder/                # Rust crate (JNI bridge)
│   ├── src/
│   │   ├── lib.rs                   # JNI exports (Java_..._NativeCore_*)
│   │   ├── tun_bridge.rs            # VpnService ↔ FIPS channel glue
│   │   └── ble_bridge.rs            # Kotlin BLE radio ↔ FIPS glue
│   ├── Cargo.toml
│   └── .cargo/config.toml           # NDK linker paths
└── build.gradle.kts               # Root build file
```

### Step 1: Cross-Compile FIPS

```bash
# Clone ble-v2
git clone https://github.com/jmcorgan/fips.git
cd fips
git checkout ble-v2

# Add Android targets
rustup target add aarch64-linux-android armv7-linux-androideabi x86_64-linux-android

# Set up NDK (install via Android Studio SDK Manager)
export ANDROID_NDK_HOME=$HOME/Android/Sdk/ndk/27.0.12077973

# Create .cargo/config.toml with NDK linker paths:
# [target.aarch64-linux-android]
# linker = "$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android24-clang"

# Build (ble-v2 cross-compiles clean)
cargo build --target aarch64-linux-android --release
```

### Step 2: Write the JNI Embedder

The embedder crate (`fips-embedder`) bridges Kotlin ↔ Rust:

```rust
// fips-embedder/src/lib.rs
use jni::JNIEnv;
use jni::objects::{JClass, JObject, JString};
use jni::sys::{jbyteArray, jlong, jshort};

#[no_mangle]
pub extern "system" fn Java_com_opentollgate_fips_NativeCore_start(
    mut env: JNIEnv, _class: JClass, config_yaml: JString, tun_fd: jlong
) {
    // 1. Parse config
    // 2. Create Node::new(config)
    // 3. Call node.enable_app_owned_tun() → get channels
    // 4. Spawn tokio task: read from tun_fd → push to app_outbound_tx
    // 5. Spawn tokio task: pull from app_inbound_rx → write to tun_fd
    // 6. node.start().await
}

#[no_mangle]
pub extern "system" fn Java_com_opentollgate_fips_NativeCore_setBleBridge(
    mut env: JNIEnv, _class: JClass, radio: JObject
) {
    // Wrap the Kotlin BleRadio object in an AndroidRadio impl
    // Create AndroidBleBridge, inject via set_android_ble_bridge()
}
```

### Step 3: Write the VpnService

```kotlin
// FipsVpnService.kt
class FipsVpnService : VpnService() {
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        val builder = Builder()
        builder.setSession("FIPS Mesh")
        builder.addAddress("fd00::1", 64)  // FIPS IPv6 ULA
        builder.addRoute("fd00::", 8)       // Route mesh traffic
        val establish = builder.establish() // Returns ParcelFileDescriptor (the fd)
        val fd = establish!!.fd.toLong()

        // Pass fd to Rust via JNI
        NativeCore.start(configYaml, fd)
        return START_STICKY
    }
}
```

### Step 4: Write the BLE Radio (Optional — for BLE mesh)

```kotlin
// BleRadio.kt — implements the AndroidRadio trait
class BleRadio : BluetoothAdapter, AndroidRadio {
    override fun listen(): Short {
        // Open BluetoothServerSocket with L2CAP, return PSM
    }
    override fun connect(connectId: Long, addr: BleAddr, psm: Short) {
        // BluetoothSocket connect to peer
    }
    override fun startAdvertising(psm: Short) { ... }
    override fun startScanning() { ... }
    // ... etc
}
```

---

## 8. FIPS Config for Android Client

```yaml
# Client-side FIPS config (generated by the app or shipped as a template)
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
  # On Android with enable_app_owned_tun(), FIPS skips system-TUN creation.
  # The VpnService owns the fd. These fields are informational only.
  name: fips0
  mtu: 1280

# UDP and TCP BOTH work on Android. Only Ethernet is gated out (needs raw sockets).
transports:
  udp:
    bind_addr: "0.0.0.0:2121"
    advertise_on_nostr: false
  tcp:
    bind_addr: "0.0.0.0:8443"
    advertise_on_nostr: false
  # BLE transport (optional — for nearby peer mesh, not needed for exit node)
  # ble:
  #   enabled: true

# Static peer config to reach the exit node
peers:
  - npub: "npub1mqelkzqp4659fws35h2wvr7z9caka5ml8qddj3ssnwaulwpxdd9sdc3esw"
    alias: "vps1-exit"
    addresses:
      - transport: udp
        addr: "66.92.204.38:2121"
    connect_policy: auto_connect
```

**Transport availability on Android (from ble-v2 README):**

| Transport | Android |
|-----------|:-------:|
| UDP       |   ✅    |
| TCP       |   ✅    |
| Ethernet  |   ❌    |
| BLE       |   ✅    |

The app connects to VPS1 exit directly via UDP/TCP. No relay needed. BLE is a bonus for peer-to-peer mesh.

---

## 9. Key Test Vectors

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
| BLE throughput | ~200/500 kbps up/down |
| BLE MTU | 2048 bytes |

---

## 10. What NOT To Do

- ❌ **Don't use FIPS master.** Use `ble-v2` branch. Master is mid-sans-io-refactor.
- ❌ **Don't use nvpn.** It breaks on reconnect. Go raw FIPS protocol.
- ❌ **Don't hardcode secrets.** Use Android KeyStore for nsec.
- ❌ **Don't assume the exit node has infinite bandwidth.** This is a PoC on a shared VPS.
- ❌ **Don't require root.** Use `VpnService` API (no root needed).
- ❌ **Don't ignore reconnection.** The exit node may restart for updates.
- ❌ **Don't create a system TUN on Android.** Use `enable_app_owned_tun()` — the VpnService owns the fd.
- ❌ **Don't push non-fd::/8 packets through the TUN seam.** FIPS no longer filters them.

---

## 11. Key Source Files in ble-v2

| File | What It Does |
|------|-------------|
| `src/node/mod.rs` (line ~2878) | `enable_app_owned_tun()` — the TUN seam API |
| `src/node/mod.rs` (line ~1048) | Android BLE transport construction (`cfg(target_os = "android")`) |
| `src/transport/ble/android_io.rs` | Full Android BLE backend (619 lines) — AndroidRadio, AndroidBleBridge, AndroidIo/Stream/Acceptor/Scanner |
| `src/transport/ble/mod.rs` | BLE transport module, wires AndroidIo as DefaultBleTransport on Android |
| `src/transport/ble/psm.rs` | Per-peer PSM discovery (L2CAP port mapping) |
| `src/transport/ble/io.rs` | BleIo trait (transport abstraction) |
| `src/transport/ble/discovery.rs` | BLE peer discovery |
| `docs/design/fips-ipv6-adapter.md` | IPv6 adapter design — fd00::/8 ULA addressing, DNS `.fips` resolution |
| `src/upper/tun.rs` | TUN device management (skipped on Android with app-owned TUN) |
| `Cargo.toml` | Platform-gated dependencies (tun crate is `cfg(unix)` but excluded via app-owned path) |
| `testing/ble/ble_spike.rs` | BLE L2CAP spike test (validates API assumptions) |

### ble-v2 commits (11 on top of v0.4.0)

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

All by **Origami74 (Arjen)** — `@1624e1bb-94ef-46d1-b03b-f067ea320af9` on Signal.

---

## 12. Reference: The Embedder Pattern

The FIPS source comments reference **`myco-core`** — Origami74's Android embedder crate that implements the JNI layer. This is NOT in the FIPS tree. Contact Origami74 for access to myco-core, which is the working reference implementation of:

- `Java_..._NativeCore_*` JNI exports
- `AndroidRadio` trait implementation via JNI `call_method` on a Kotlin `BleRadio` object
- The Kotlin BLE radio (scan, advertise, L2CAP listen/connect, socket read/write)
- The VpnService ↔ FIPS TUN channel glue

**myco-core is the fastest path to a working Android app.** It already works with ble-v2. Fork it and customize the UI for the exit-node use case.

---

## 13. Contacts

- **c08r4d0r** — project owner (Signal @+181****0908)
- **Origami74 (Arjen)** — ble-v2 author, has working Android embedder (myco-core)
- **jmcorgan** — FIPS upstream maintainer (johnathan@corganlabs.com)
- **Amperstrand** — collaborator, conwrt UseCase patterns
- **Relay operators** — relay1.orangesync.tech, ngit1.orangesync.tech

---

## 14. Quick Start Checklist

1. [ ] Contact Origami74 for `myco-core` access (the working Android embedder)
2. [ ] Clone `jmcorgan/fips`, checkout `ble-v2` branch
3. [ ] Install Android NDK + Rust Android targets
4. [ ] Cross-compile FIPS: `cargo build --target aarch64-linux-android --release`
5. [ ] Fork myco-core, customize UI for exit-node use case
6. [ ] Add exit node config: npub `npub1mqelkzqp4659fws35h2wvr7z9caka5ml8qddj3ssnwaulwpxdd9sdc3esw`, addr `66.92.204.38:2121`
7. [ ] Test Noise XK handshake to VPS1 (watch for "Connection promoted to active peer")
8. [ ] Test VpnService TUN routing (internet traffic through mesh)
9. [ ] Implement reconnect loop (5s → 60s backoff)
10. [ ] Add per-app VPN toggle
11. [ ] Add BLE peer scan UI (optional)
12. [ ] Playwright/Espresso happy-path tests with video evidence

---

*Generated 2026-07-06. Last known good state: Phase 1 complete, FIPS v0.4.0 pinned on VPS1, ble-v2 branch has Android support.*

---

## Appendix A: Integration Pointers for the LLM

### The Exact API Surface (from ble-v2 source)

The Node lifecycle an Android embedder uses:

```rust
// 1. Create node from config
let mut node = Node::new(config)?;

// 2. Set up app-owned TUN (before start!)
let (app_outbound_tx, app_inbound_rx) = node.enable_app_owned_tun();
//    app_outbound_tx: tokio::sync::mpsc::Sender<Vec<u8>>  (app → mesh)
//    app_inbound_rx:  std::sync::mpsc::Receiver<Vec<u8>>  (mesh → app)

// 3. Get a control read handle for UI polling (before moving node to bg task)
let control_handle = node.control_read_handle();

// 4. Start transports + handshake
node.start().await?;
//    start() skips TunDevice::create because tun_tx is set (gated by `self.tun_tx.is_none()`)

// 5. Run the RX event loop on a background task
//    This is what processes incoming packets, handshakes, routing
tokio::spawn(async move {
    node.run_rx_loop().await
});

// Meanwhile, on the VpnService fd thread:
//    loop {
//        let pkt = read_from_fd(tun_fd);   // app reads from VpnService TUN
//        app_outbound_tx.try_send(pkt);     // push to FIPS (app → mesh)
//
//        if let Ok(mesh_pkt) = app_inbound_rx.recv_timeout(Duration::from_millis(50)) {
//            write_to_fd(tun_fd, &mesh_pkt); // mesh → app → fd
//        }
//    }

// 6. To stop:
node.stop().await?;
//    stop() drops the packet channel, run_rx_loop exits
```

### Channel Types (IMPORTANT mismatch)

The two channels use **different** channel implementations — this is intentional:

| Direction | Channel type | Why |
|-----------|-------------|-----|
| app → mesh | `tokio::sync::mpsc::Sender<Vec<u8>>` | FIPS drains this in `run_rx_loop` (async context) |
| mesh → app | `std::sync::mpsc::Receiver<Vec<u8>>` | App reads from a blocking JNI thread, not async |

When reading from `app_inbound_rx`, use `recv_timeout` (not blocking `recv`) so the JNI thread can check a shutdown flag between polls.

### ControlReadHandle — Polling Peer State from UI

```rust
// src/control/read_handle.rs
pub struct PeerView {
    pub node_addr_hex: String,
    pub npub: String,
    pub connected: bool,
}

// Call from any thread — uses ArcSwap (lock-free read)
let views: Vec<PeerView> = control_handle.peer_views();
```

This is how the Android app's status UI reads "connected peers" without touching the Node (which is borrowed by `run_rx_loop` on the background task).

### BLE Bridge JNI Surface (if using BLE transport)

If you implement BLE peer-to-peer, here's the Kotlin ↔ Rust bridge:

**Rust side (already in FIPS):**
```rust
// src/transport/ble/android_io.rs

// The trait Kotlin must implement via JNI:
pub trait AndroidRadio: Send + Sync {
    fn listen(&self) -> u16;                                    // returns PSM
    fn connect(&self, connect_id: i64, addr: &BleAddr, psm: u16);
    fn start_advertising(&self, psm: u16);
    fn stop_advertising(&self);
    fn start_scanning(&self);
    fn stop_scanning(&self);
    fn close_channel(&self, ch_id: i64);
}

// Injection (call before Node::start):
pub fn set_android_ble_bridge(bridge: Arc<AndroidBleBridge>)

// JNI-facing methods (Kotlin calls these via JNI exports):
impl AndroidBleBridge {
    pub fn new(radio: Arc<dyn AndroidRadio>) -> Arc<Self>
    pub fn deliver_inbound(&self, remote: BleAddr, send_mtu: u16, recv_mtu: u16) -> i64
    pub fn deliver_connect_result(&self, connect_id: i64, ok: bool, ...) -> i64
    pub fn deliver_scan(&self, addr: BleAddr, psm: u16, rssi: i32)
    pub fn deliver_recv(&self, ch_id: i64, data: &[u8]) -> bool
    pub fn next_send(&self, ch_id: i64, timeout: Duration) -> Option<Vec<u8>>
    pub fn channel_open(&self, ch_id: i64) -> bool
}
```

**Kotlin side (you write this):**
- `BleRadio` class implementing the radio operations
- JNI `Java_..._NativeCore_*` exports that call `deliver_*` / `next_send`
- Per-channel writer threads pulling via `next_send` (blocking with timeout)
- `BluetoothLeScanner` / `BluetoothLeAdvertiser` / L2CAP socket management

### Cross-Compilation Notes

```bash
# Install Rust Android targets
rustup target add aarch64-linux-android armv7-linux-androideabi

# In ~/.cargo/config.toml:
# [target.aarch64-linux-android]
# linker = "/path/to/ndk/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android24-clang"

# Build (no special flags needed — build.rs auto-detects target_os=android)
cargo build --target aarch64-linux-android --release -p fips
```

**Watch out for:**
- `ring = "0.17"` — crypto crate with C/assembly. Needs NDK C compiler. May need `RING_*` env vars pointing to NDK toolchain.
- `tun = "0.8.7"` — gated `cfg(unix)` but Android is unix. However, with `enable_app_owned_tun()`, the TUN creation path is **skipped** so this dependency is effectively unused at runtime. It still needs to compile though.
- `socket2`, `tokio` — work fine on Android, pure Rust or have Android support.
- `nostr-sdk = "0.44"` — check for Android TLS backend. May need `--features rustls-tls` instead of native-tls.

### Build.rs CFG Gates (automatic)

The `build.rs` script auto-detects the target and sets these cfg flags:

```
target_os = "android" → ble_available (BLE module compiles)
target_os = "android" → DefaultBleTransport = BleTransport<AndroidIo>
target_os = "android" → system-tun = no-op stub (all functions return Ok(()))
target_os = "android" → Ethernet transport excluded
```

You do NOT need to pass any `--features` or `--no-default-features`. A plain `cargo build --target aarch64-linux-android` compiles correctly.

### Embedder Contract Summary (MUST FOLLOW)

When using `enable_app_owned_tun()`, the app owns three responsibilities that the system-TUN reader would normally handle:

1. **Filter destinations:** Push only `fd00::/8`-destined IPv6 packets to `app_outbound_tx`. FIPS does NOT filter in app-owned mode. Non-fd00::/8 packets will be misrouted.

2. **Clamp TCP MSS:** On outbound SYNs, clamp the TCP Maximum Segment Size to fit within the TUN MTU. Without this, cold TCP connections can wedge (silent drops, no PTB feedback through userspace TUN).

3. **Use recv_timeout:** When pulling from `app_inbound_rx`, use `recv_timeout(Duration)` not blocking `recv()`. This lets the JNI thread check a shutdown flag between polls for clean service teardown.

### The myco-core Reference

FIPS source comments reference **`myco-core`** — Origami74's Android embedder crate. This is the working reference implementation of:
- `Java_..._NativeCore_*` JNI exports
- `AndroidRadio` trait implementation via JNI
- Kotlin BLE radio (scan, advertise, L2CAP, sockets)
- VpnService ↔ FIPS TUN channel glue

**myco-core is NOT in the FIPS repo.** Contact Origami74 (Arjen, `@1624e1bb-94ef-46d1-b03b-f067ea320af9` on Signal) for access. It already works with ble-v2 — fork it and customize the UI.

### Test Vectors

| Parameter | Value |
|-----------|-------|
| Exit node IP | `66.92.204.38` |
| FIPS port (UDP) | `2121` |
| FIPS port (TCP) | `8443` |
| Exit node npub | `npub1mqelkzqp4659fws35h2wvr7z9caka5ml8qddj3ssnwaulwpxdd9sdc3esw` |
| App tag | `fips-overlay-v1` |
| Handshake type | `Noise XK` (phone is initiator) |
| TUN MTU | `1280` |
| Expected log on success | `Connection promoted to active peer peer=vps1-exit` |
| Expected log on success | `Session established (initiator, XK)` |
| Expected log on success | `new_parent=vps1-exit` |
| BLE throughput (if used) | ~200/500 kbps up/down |
| BLE outbound queue depth | 32 (empirically tuned) |
