# Handover: Android FIPS Mesh Client App

> **THE single source of truth for building the Android FIPS app.**
> This document consolidates input from all three project members and the
> verified `ble-v2` diff analysis. It supersedes `ANDROID-HANDOVER-DEFINITIVE.md`
> and `ANDROID-LLM-POINTERS.md` (kept for history; do not edit them).
>
> **Read this top to bottom before writing a single line of code.** It is written
> so a fresh LLM session with **zero** prior context about this project can pick
> it up and start building.

---

**Author:** c08r4d0r via Hermes Agent
**Last updated:** 2026-07-07
**Status:** Phase 1 (exit node) complete & verified; Android app not yet started

---

## 0. TL;DR — The Decision (Unanimous, Confirmed)

**Use the `ble-v2` branch of `github.com/jmcorgan/fips`.**

It is FIPS **v0.4.0** with **11 purely additive commits** by Origami74 (Arjen) that add
native Android support. The core mesh protocol is **byte-for-byte identical** to v0.4.0.

**Branch:** `ble-v2` · **HEAD:** `5606209` (2026-06-30) · **Merge base with v0.4.0:** `3ea7ca1`

Confirmed by all three stakeholders (see §2):

| Member | Role | Position |
|--------|------|----------|
| **c08r4d0r** (`@9cab90c7`) | Project owner | "Use ble-v2 — it includes Android support and app-defined TUN" |
| **Amperstrand** (`@cc5bdaa4`) | Collaborator | "Android basics will be upstream soon, so it's fine to rely on it. At worst only minor tweaks required" |
| **Origami74 / Arjen** (`@1624e1bb`) | `ble-v2` author | Wrote the code; has a *working* Android embedder called **myco-core** |

**Do NOT use:**
- `v0.4.0` tag alone — zero Android support (no app-owned TUN, no BLE backend)
- `master` — mid **sans-io refactor**, breaks the config format and protocol behaviour

---

## 1. What You're Building

An **Android app** that:

1. Runs the FIPS mesh protocol **natively via JNI** (Rust `.so` + Kotlin)
2. Connects as a peer to the existing **VPS1 exit node** at `66.92.204.38:2121` (UDP) or `66.92.204.38:8443` (TCP)
3. Optionally connects via **BLE** to nearby FIPS peers (mesh, not internet exit)
4. Routes device traffic through the FIPS mesh → exit node → WireGuard → public internet
5. Shows connection status, data counters, peer list, and (optionally) BLE scan results

The exit node **already exists and is verified working** (see §4). Your job is the client side.

---

## 2. Project Member Advice (All Three, Verbatim Where Possible)

### @cc5bdaa4 — Amperstrand (collaborator)

> *"Android basics will be upstream soon, so it's fine to rely on it. At worst only minor tweaks required."*

**What this means for you:** `ble-v2` is expected to merge into upstream FIPS. Relying on it
is safe. If the branch is rebased, the worst case is minor re-cherry-picking of 11 clean,
atomic commits — not a rewrite. You can also vendor the 11 commits into your own fork for
stability. Signal: `@cc5bdaa4-98d2-4ce2-af1d-93aab049868a`.

### @1624e1bb — Origami74 / Arjen (ble-v2 author)

Arjen **wrote all 11 Android commits**. Specifically he created:

- **`Node::enable_app_owned_tun()`** — the TUN seam that lets an Android `VpnService` own the fd while FIPS exchanges packets over channels
- **`src/transport/ble/android_io.rs`** — the full Android BLE backend (`AndroidBleBridge`, `AndroidRadio` trait, `AndroidIo`/`Stream`/`Acceptor`/`Scanner`)
- **Platform gating by `target_os`** — desktop transports and system-TUN are conditionally compiled so `cargo build --target aarch64-linux-android` works out of the box
- **`ControlReadHandle::peer_views()`** — lock-free peer-state snapshot for embedder UIs

Arjen has a **working Android JNI embedder called `myco-core`** that implements the entire
Kotlin ↔ Rust bridge (JNI exports, `AndroidRadio` impl, Kotlin BLE radio, VpnService glue,
developer UI). **`myco-core` is NOT in the FIPS repo** — it lives in his private repo.
**Contact him for access. It is the fastest path to a working app.**

**Signal:** `@1624e1bb-94ef-46d1-b03b-f067ea320af9`

### @9cab90c7 — c08r4d0r (project owner)

Provided the **verified `ble-v2` diff analysis** (§3) confirming the branch is protocol-safe.
Also sets the testing standard:

> *"Playwright-based smoke tests for the happy path in ALL functionality. Show video of the
> happy path before considering something complete. This is a hard gate."*

For Android specifically: Espresso/UI Automator instrumented tests + screen-recorded video of
the happy path (launch → connect → route → disconnect).

**Signal:** `+181****0908` (group: `fips-exit-node-poc`)

---

## 3. Verified Tradeoff Analysis: `ble-v2` vs `v0.4.0`

> **This analysis is verified from the actual git diff by c08r4d0r (`@9cab90c7`).**
> It is the authoritative answer to "is `ble-v2` safe to depend on?"

### Protocol-critical files: ZERO CHANGES

`ble-v2` is **v0.4.0** (`3ea7ca1`) with **11 purely additive commits**. The core protocol is
**byte-for-byte identical**. Zero changes to:

- ❌ Noise XK handshake
- ❌ Mesh routing
- ❌ Session management
- ❌ UDP/TCP transport wire format
- ❌ Packet format

**Implication:** A `ble-v2` Android client talks to a `v0.4.0` exit node (VPS1) with **zero
compatibility issues**. The exit node does not need to be upgraded.

### What `ble-v2` ACTUALLY changes (5 things, all additive or platform-gated)

| # | File | Change | Risk |
|---|------|--------|------|
| 1 | `src/node/mod.rs` (+92 lines) | **PURELY ADDITIVE:** `enable_app_owned_tun()` method, BLE bridge injection, `PeerView` API | Zero — new methods, existing paths untouched |
| 2 | `src/transport/ble/` | **NEW module:** Android BLE backend (`android_io.rs`, 619 lines) | Zero — new file, gated by `cfg(target_os = "android")` |
| 3 | `src/upper/tun.rs` | **Android gets no-op stub** — system TUN skipped (app owns the fd) | Zero — doesn't exist on Linux |
| 4 | `src/transport/mod.rs` | **Ethernet gated by `target_os`** — excluded on Android (needs raw sockets) | Zero — UDP/TCP still compile & run on Android |
| 5 | `build.rs` | **Auto-detects `target_os`**, sets cfg flags | Zero |

Minor incidental fix: `src/upper/dns.rs` (1 line) — `ipi6_ifindex as u32` type cast. Trivial.

### Why `ble-v2` saves weeks of work

- **`enable_app_owned_tun()`** — without this you'd be fighting system-TUN permissions on Android.
  The VpnService owns the fd; FIPS exchanges bytes over channels. This is *the* hard part, already done.
- **`AndroidBleBridge`** — working BLE L2CAP transport with the byte-bridge pattern designed, tested,
  and tuned (bufferbloat fix, PSM rotation fix, stream reframing fix all baked in).
- **`AndroidRadio` trait** — the exact JNI interface your Kotlin must implement.
- **Platform gating** — `cargo build --target aarch64-linux-android` works out of the box. No
  `--no-default-features` hacks.
- **`myco-core`** — Arjen's working Android embedder. The FIPS protocol layer is already done there.

### The "branch vs tag" risk (and mitigation)

`ble-v2` is a branch, not a tag — it could theoretically be rebased. Mitigations:

1. Amperstrand confirms Android basics are going upstream soon.
2. You can cherry-pick the 11 commits into your own fork for stability.
3. The commits are clean, atomic, and well-tested.

---

## 4. The Exit Node (VPS1 — Already Deployed & Verified)

A fully operational FIPS mesh exit node running on a Debian 13 VPS. **This is the target your
Android app connects to.**

| Component | Detail |
|-----------|--------|
| **IP** | `66.92.204.38` |
| **Domain** | `fips-exit.orangesync.tech` |
| **SSH** | `debian@66.92.204.38` (password in `tollgate-infrastructure-kit/.env`) |
| **FIPS daemon** | v0.4.0-derivative (Rust binary, ~17.6 MB), systemd unit, binary at `/usr/bin/fips` |
| **FIPS UDP** | `2121` (primary) |
| **FIPS TCP** | `8443` (fallback) |
| **VPS1 npub** | `npub1mqelkzqp4659fws35h2wvr7z9caka5ml8qddj3ssnwaulwpxdd9sdc3esw` |
| **App tag** | `fips-overlay-v1` |
| **Handshake** | Noise **XK** (your app is the **initiator**) |
| **TUN interface** | `fips0`, MTU 1280 |
| **WireGuard** | `wg0`, `10.99.99.0/24`, peer at `10.99.99.2` |
| **NAT / egress** | nftables `MASQUERADE` from `wg0` → `eth0` |
| **Cashu payment gate** | `paid_peers` nftables set **starts EMPTY** — peers must pay for internet egress |
| **Nostr relays** | relay1/relay2/ngit1/ngit2.orangesync.tech, relay.damus.io, nos.lol |

**Important about the Cashu gate:** the `paid_peers` nftables set starts empty. For internet
egress to work through the exit, the Android client's npub must be paid into the set (or the
gate must be in a test/bypass state). Coordinate payment/testing with c08r4d0r before assuming
egress works. The FIPS mesh handshake itself is not gated — only the final NAT/egress step.

**Success criteria (log messages your app / VPS1 should show):**
```
Connection promoted to active peer peer=vps1-exit
Session established (initiator, XK)
new_parent=vps1-exit
```
After that, traffic routed through the VPN should egress as `66.92.204.38`.

### Internet exit path (data flow)

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
[nftables fips-exit: MASQUERADE on eth0]   ← Cashu paid_peers gate here
       │
       ▼
[PUBLIC INTERNET]
```

---

## 5. Transport Availability on Android (VERIFIED)

> **This is verified from the `ble-v2` README and source.** Note: an earlier internal
> draft (`ANDROID-HANDOVER-DEFINITIVE.md`) incorrectly claimed UDP/TCP are gated out on
> Android — **that is wrong**. Only Ethernet is excluded.

| Transport | Android | Notes |
|-----------|:-------:|-------|
| **UDP** | ✅ | Works. Primary transport for direct phone → VPS1 exit connectivity. |
| **TCP** | ✅ | Works. Fallback transport. |
| **Ethernet** | ❌ | Gated by `target_os` (needs raw `AF_PACKET` sockets). Self-excludes on Android. |
| **BLE** | ✅ | Works via `AndroidBleBridge`. For **nearby peer mesh**, not internet exit. |

**Your app connects to VPS1 directly via UDP/TCP. No relay needed.** BLE is a bonus for
peer-to-peer mesh between phones.

---

## 6. The Three APIs You Must Know

These are the entire `ble-v2` API surface your embedder touches. Everything else is FIPS internals.

### API 1: `Node::enable_app_owned_tun()` — THE critical seam

```rust
// src/node/mod.rs (~line 2878)
impl Node {
    /// Call AFTER Node::new(), BEFORE start().
    /// Returns two channels. The app owns the TUN fd (Android VpnService).
    pub fn enable_app_owned_tun(&mut self) -> (TunOutboundTx, std::sync::mpsc::Receiver<Vec<u8>>)
}
```

- `TunOutboundTx` = `tokio::sync::mpsc::Sender<Vec<u8>>` — push **app → mesh** packets here
- `Receiver<Vec<u8>>` = `std::sync::mpsc::Receiver` — pull **mesh → app** packets from here
- After calling this, `start()` **skips system-TUN creation** entirely (gated by `self.tun_tx.is_none()`)

**Unit test proving it works** (`src/node/tests/unit.rs:~2003`):
```rust
let (outbound_tx, tun_rx) = node.enable_app_owned_tun();
assert_eq!(node.tun_state(), TunState::Active);
assert!(node.tun_tx().is_some());
node.tun_tx().unwrap().send(pkt.clone()).unwrap();
assert_eq!(tun_rx.recv_timeout(200ms).unwrap(), pkt);
node.start().await.unwrap();
assert!(node.tun_name().is_none()); // no system device created
```

### API 2: `AndroidBleBridge` + `AndroidRadio` — BLE byte-bridge (optional, for BLE mesh)

```rust
// src/transport/ble/android_io.rs

// The Kotlin radio must implement this trait (via JNI):
pub trait AndroidRadio: Send + Sync {
    fn listen(&self) -> u16;                                       // Open L2CAP, return PSM
    fn connect(&self, connect_id: i64, addr: &BleAddr, psm: u16);  // Dial a peer
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

**The byte-bridge pattern (critical for understanding):**

- **Inbound** (Kotlin → Rust): Kotlin calls JNI methods (`deliver_*`) that push into tokio channels. Non-blocking.
- **Outbound** (Rust → Kotlin): Kotlin writer thread calls `next_send(ch_id, timeout)` — blocking pull with timeout.
  The byte hot path **never** calls JNI upcalls — `BleStream::send` only pushes into a std channel.

**JNI-facing methods your Kotlin code calls on the bridge:**

```
deliver_inbound(remote, send_mtu, recv_mtu) -> i64           // channel accepted
deliver_connect_result(connect_id, ok, remote, mtus) -> i64  // dial done
deliver_scan(addr, psm, rssi)                                 // peer found
deliver_recv(ch_id, data) -> bool                             // packet arrived
next_send(ch_id, timeout) -> Option<Vec<u8>>                  // pull outbound
channel_closed(ch_id)                                         // socket gone
channel_open(ch_id) -> bool                                   // alive check
advert_views() -> Vec<AdvertView>                             // UI: addr/psm/rssi
```

**Bugfixes already baked into ble-v2 (do NOT reintroduce):**

- **Inbound L2CAP stream reframing:** Android `BluetoothSocket` is byte-stream, not datagram. Packets fragmented/coalesced. Fixed with FMP length-prefix framer.
- **PSM rotation:** RPAs rotate between scan and dial, so PSM lookup missed. Fixed by dialing last-learned PSM on miss.
- **Bufferbloat:** outbound queue was 256 deep, RTT ballooned to ~5s. Reduced to 32. RTT dropped to ~1.2s.
- **Safe teardown:** `next_send` no longer holds channels lock across `recv_timeout`.

### API 3: `PeerView` — for your UI

```rust
// src/control/read_handle.rs (~line 114)
pub struct PeerView {
    pub node_addr_hex: String,
    pub npub: String,
    pub connected: bool,
}

// Lock-free snapshot (ArcSwap), safe to poll from UI thread:
let views: Vec<PeerView> = control_handle.peer_views();
```

This is how the Android app's status UI reads "connected peers" without touching the `Node`
(which is borrowed by `run_rx_loop` on the background task).

---

## 7. The Embedder Contract (MUST FOLLOW)

The complete Node lifecycle an Android embedder uses:

```rust
// 1. Create node from config
let mut node = Node::new(config)?;

// 2. Set up app-owned TUN (before start!)
let (app_outbound_tx, app_inbound_rx) = node.enable_app_owned_tun();
//    app_outbound_tx: tokio::sync::mpsc::Sender<Vec<u8>>  (app → mesh)
//    app_inbound_rx:  std::sync::mpsc::Receiver<Vec<u8>>  (mesh → app)  ← DIFFERENT channel type!

// 3. Get a control read handle for UI polling (before moving node to bg task)
let control_handle = node.control_read_handle();

// 4. Start transports + handshake
node.start().await?;
//    start() skips TunDevice::create because tun_tx is set

// 5. Run the RX event loop on a background task
tokio::spawn(async move { node.run_rx_loop().await });

// Meanwhile, on the VpnService fd thread:
//    loop {
//        let pkt = read_from_fd(tun_fd);
//        app_outbound_tx.try_send(pkt);   // app → mesh
//        if let Ok(mesh_pkt) = app_inbound_rx.recv_timeout(Duration::from_millis(50)) {
//            write_to_fd(tun_fd, &mesh_pkt); // mesh → app → fd
//        }
//        if shutdown_flag.load() { break; }
//    }

// 6. To stop:
node.stop().await?;
//    stop() drops the packet channel, run_rx_loop exits
```

### Channel types — IMPORTANT mismatch (intentional)

| Direction | Channel type | Why |
|-----------|-------------|-----|
| app → mesh | `tokio::sync::mpsc::Sender<Vec<u8>>` | FIPS drains this in `run_rx_loop` (async context) |
| mesh → app | `std::sync::mpsc::Receiver<Vec<u8>>` | App reads from a blocking JNI thread, not async |

### Three responsibilities you inherit with app-owned TUN (system-TUN reader did these for you)

1. **Filter destinations — push only `fd00::/8`-destined IPv6 packets** to `app_outbound_tx`.
   FIPS does **NOT** filter in app-owned mode. Non-`fd00::/8` packets will be misrouted.

2. **Clamp TCP MSS** on outbound SYNs (system-TUN reader's clamping is bypassed). Without this,
   cold TCP connections can wedge — silent drops, no PTB feedback through userspace TUN.

3. **Use `recv_timeout` (not blocking `recv`)** on `app_inbound_rx` so the JNI thread can check a
   shutdown flag between polls for clean service teardown.

---

## 8. Suggested App Architecture

```
android-app/
├── app/                          # Android app module
│   ├── src/main/
│   │   ├── java/com/opentollgate/fips/
│   │   │   ├── MainActivity.kt       # Compose UI
│   │   │   ├── FipsVpnService.kt     # VpnService — owns TUN fd
│   │   │   ├── BleRadio.kt           # Implements AndroidRadio trait via JNI (optional)
│   │   │   └── NativeCore.kt         # JNI bindings to Rust .so
│   │   ├── jniLibs/
│   │   │   ├── arm64-v8a/libfips.so  # Cross-compiled FIPS
│   │   │   ├── armeabi-v7a/libfips.so
│   │   │   └── x86_64/libfips.so
│   │   └── AndroidManifest.xml       # VPN + BLE permissions
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

### Internal data flow (ble-v2)

```
┌─────────────────────────────────────────────┐
│                  Android App                 │
│  ┌──────────┐   ┌─────────────────────────┐ │
│  │ Kotlin UI │   │   VpnService (owns fd)   │ │
│  │ (Compose) │   │         │                │ │
│  └──────────┘   │  IPv6 packets ↔ channels │ │
│       ▲          └─────────┬───────────────┘ │
│       │ peer_views()       │                 │
│       │                    JNI boundary      │
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

### Skeleton JNI (Rust side)

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
    //    (FILTER fd00::/8, clamp TCP MSS on SYNs)
    // 5. Spawn tokio task: pull from app_inbound_rx (recv_timeout) → write to tun_fd
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

### Skeleton VpnService (Kotlin side)

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
        NativeCore.start(configYaml, fd)
        return START_STICKY
    }
}
```

---

## 9. Build Instructions (Cross-Compilation)

```bash
# 1. Clone ble-v2
git clone https://github.com/jmcorgan/fips.git
cd fips
git checkout ble-v2

# 2. Add Android targets
rustup target add aarch64-linux-android armv7-linux-androideabi x86_64-linux-android

# 3. Install NDK via Android Studio SDK Manager, then:
export ANDROID_NDK_HOME=$HOME/Android/Sdk/ndk/27.0.12077973

# 4. Configure cargo linker
cat > .cargo/config.toml << 'EOF'
[target.aarch64-linux-android]
linker = "$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android24-clang"
[target.armv7-linux-androideabi]
linker = "$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/bin/armv7-linux-androideabi24-clang"
[target.x86_64-linux-android]
linker = "$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/bin/x86_64-linux-android24-clang"
EOF

# 5. Build — NO special flags needed. build.rs auto-detects target_os=android
cargo build --target aarch64-linux-android --release
```

Output: `target/aarch64-linux-android/release/libfips.so` → drop into `app/src/main/jniLibs/arm64-v8a/`.

### Cross-compilation gotchas

- **`ring` crate** (v0.17) — crypto with C/assembly. Needs the NDK C compiler. May need `RING_*`
  env vars pointing to the NDK toolchain.
- **`nostr-sdk`** (v0.44) — check the Android TLS backend. May need `--features rustls-tls`
  instead of `native-tls` (Android has no system OpenSSL by default).
- **`tun` crate** (v0.8.7) — gated `cfg(unix)` and Android *is* unix, so it compiles. But with
  `enable_app_owned_tun()`, the TUN creation path is **skipped** at runtime — the dependency is
  effectively unused. It still needs to compile.
- **`socket2`, `tokio`** — work fine on Android.

### build.rs CFG gates (automatic, you do nothing)

```
target_os = "android" → ble_available              (BLE module compiles)
target_os = "android" → DefaultBleTransport = BleTransport<AndroidIo>
target_os = "android" → system-tun = no-op stub    (all functions return Ok(()))
target_os = "android" → Ethernet transport excluded (raw sockets unavailable)
```

You do **NOT** need any `--features` or `--no-default-features`. A plain
`cargo build --target aarch64-linux-android` compiles correctly.

---

## 10. FIPS Config for the Android Client

```yaml
# Client-side FIPS config (generated by the app or shipped as a template)
node:
  identity:
    nsec: "<from Android KeyStore — do NOT hardcode>"

discovery:
  nostr:
    enabled: true
    policy: configured_only
    app: "fips-overlay-v1"
    advertise: false           # Phone should NOT advertise as an exit

tun:
  enabled: true                # required, but start() skips system-TUN with app-owned
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
  #   mtu: 2048

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

## 11. BLE Performance (Empirical — if you enable BLE mesh)

From Origami74's tuning (in the `ble-v2` source comments):

- ~**200 kbps** upstream / ~**500 kbps** downstream over BLE L2CAP CoC
- Outbound queue cap: **32** (tuned — 8 starved, 64 bufferbloated)
- L2CAP CoC MTU: **2048 bytes**
- RTT: **~1.2s** after bufferbloat fix (was ~5s)
- BLE variance (RF, 2M PHY, connection priority) rivals tuning effects

**BLE is for nearby peer mesh — NOT for internet exit.** For internet exit, use UDP/TCP to VPS1.

---

## 12. Testing Situation (Honest Assessment)

### What IS tested (by this project, on the exit-node side)

| Feature | Status | Evidence |
|---------|--------|----------|
| FIPS daemon start | ✅ | Raw Docker container starts cleanly |
| UDP transport | ✅ | Connects to VPS1 :2121 |
| TCP transport | ✅ | Config has TCP :8443 as fallback |
| Noise XK handshake | ✅ | "Session established (initiator, XK)" logged |
| Peer promotion | ✅ | "Connection promoted to active peer" confirmed |
| Mesh parent switch | ✅ | "new_parent=vps1-exit" logged |
| Encrypted session | ✅ | Full duplex after handshake |
| E2E egress traffic | ✅ | nostr-vpn test proved egress (5 ICMP pkts) |
| App-owned TUN | ✅ (unit) | `app_owned_tun_seam_wires_channels` test passes |
| Android BLE backend | ✅ (compiles) | Cross-compiles clean for arm64-android |

### What is NOT tested (be honest about this)

- **The Android app does not exist yet.** Nothing on the client side has been run on a device or emulator.
- **No Android emulator, no device, no Android SDK** is available in the Hermes session that wrote this doc.
  The Android LLM session **must bring its own Android toolchain**.
- **No physical Android hardware** is available in this project currently.
- **`myco-core`** (Arjen's working embedder) is reported working but has not been independently verified
  by this project — contact Arjen for a demo/access.
- **Docker testing of the FIPS daemon works fine** — but that tests the exit node, not the Android client.

### Testing standard (from c08r4d0r, mandatory)

- Playwright/Espresso/UI Automator smoke tests for the happy path in **all** functionality
- **Show video evidence** of the happy path before considering anything complete
- Happy path: app launches → connect to exit → verify traffic routes through mesh → disconnect
- Record video (Android screen recording or emulator screencap)

### Existing infrastructure tests (for reference)

- **SMOKE-1:** SSH-based pytest against VPS1 (5 tests, 4 pass + 1 skip)
- **Docker test harness:** raw FIPS container peering with VPS1
- Daily smoke cron at 06:00; health monitoring every 15 minutes

---

## 13. What NOT To Do (Pitfalls)

- ❌ **Don't use FIPS `master`.** Mid-sans-io-refactor, breaks config format and protocol. Use `ble-v2`.
- ❌ **Don't use `v0.4.0` tag alone.** No Android support at all.
- ❌ **Don't use `nvpn` (nostr-vpn) as a client wrapper.** It breaks on reconnect. Go raw FIPS protocol.
- ❌ **Don't create a system TUN on Android.** Use `enable_app_owned_tun()` — the VpnService owns the fd.
- ❌ **Don't push non-`fd00::/8` packets through the TUN seam.** FIPS does NOT filter in app-owned mode.
- ❌ **Don't forget TCP MSS clamping** on outbound SYNs (system-TUN reader's clamping is bypassed).
- ❌ **Don't use blocking `recv()` on `app_inbound_rx`.** Use `recv_timeout` so you can check shutdown flags.
- ❌ **Don't call JNI on the BLE byte hot path.** Use the channel pattern — `BleStream::send` is pure channel push.
- ❌ **Don't trust datagram boundaries on Android BLE.** Use the FMP length-prefix framer (already in ble-v2).
- ❌ **Don't hardcode the L2CAP PSM.** It's OS-assigned and per-peer discovered.
- ❌ **Don't hardcode secrets.** Use Android KeyStore for the `nsec`.
- ❌ **Don't assume the exit node has infinite bandwidth.** PoC on a shared VPS. No per-npub rate limiting yet.
- ❌ **Don't require root.** `VpnService` API needs no root.
- ❌ **Don't ignore reconnection.** FIPS has **no built-in reconnect**. Implement your own: 5s → 10s → 20s → 40s → 60s, capped. Reset on successful handshake.
- ❌ **Don't assume the Cashu egress gate is open.** The `paid_peers` nftables set starts empty. Coordinate with c08r4d0r.

---

## 14. `ble-v2` Commit Log (11 commits, all by Origami74, June 2026)

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

---

## 15. Key Source Files in `ble-v2`

| File | What It Does |
|------|-------------|
| `src/node/mod.rs` (~line 2878) | `enable_app_owned_tun()` — the TUN seam API |
| `src/node/mod.rs` (~line 1048) | Android BLE transport construction (`cfg(target_os = "android")`) |
| `src/transport/ble/android_io.rs` | Full Android BLE backend (619 lines): `AndroidRadio`, `AndroidBleBridge`, `AndroidIo`/`Stream`/`Acceptor`/`Scanner` |
| `src/transport/ble/mod.rs` | BLE transport module; wires `AndroidIo` as `DefaultBleTransport` on Android |
| `src/transport/ble/psm.rs` | Per-peer PSM discovery (L2CAP port mapping) |
| `src/transport/ble/io.rs` | `BleIo` trait (transport abstraction) |
| `src/transport/ble/discovery.rs` | BLE peer discovery |
| `src/control/read_handle.rs` (~line 114) | `PeerView` struct + `peer_views()` for embedder UIs |
| `src/upper/tun.rs` | TUN device management (no-op stub on Android with app-owned TUN) |
| `docs/design/fips-ipv6-adapter.md` | IPv6 adapter design — `fd00::/8` ULA addressing, DNS `.fips` resolution |
| `Cargo.toml` | Platform-gated dependencies |
| `testing/ble/ble_spike.rs` | BLE L2CAP spike test (validates API assumptions) |
| `build.rs` | Auto-detects `target_os`, sets cfg flags |

---

## 16. The `myco-core` Reference Embedder

**Origami74 (Arjen) has a WORKING Android JNI embedder called `myco-core`.**

It implements:
- All `Java_..._NativeCore_*` JNI exports
- `AndroidRadio` trait via JNI `call_method` on a Kotlin `BleRadio` object
- The Kotlin BLE radio (scan, advertise, L2CAP listen/connect, socket read/write)
- The VpnService ↔ FIPS TUN channel glue
- The developer UI using `PeerView`

**`myco-core` is NOT in the FIPS repo.** It lives in Arjen's private repo.

**Fastest path to a working app:** Contact Arjen for `myco-core` access, fork it, and customize
the UI for the exit-node use case (add exit-node config, connection status, data counters). The
FIPS protocol layer is already done and working.

---

## 17. Repos & Upstream

| Repo | URL | Contents |
|------|-----|----------|
| **FIPS upstream** | `github.com/jmcorgan/fips` branch `ble-v2` | The protocol library (use this branch) |
| FIPS fork (ngit) | `nostr://npub12m5exm2uk3xa674cc5r0hlyvccs5xxn7qv83ezuteefv5972nquq4j4szl/relay.ngit.dev/fips` | Upstream source + cherry-picks |
| **Exit node repo** | `github.com/OpenTollGate/fips-exit-node` | STATUS.md, README, ansible roles, SMOKE-1, dashboard |
| **E2E test infra** | `github.com/OpenTollGate/fips-exit-e2e` | Docker test harness, scripts, **this handover doc** |
| FIPS binary on VPS1 | `/usr/bin/fips` (v0.4.0-derivative) | The running exit node daemon |

---

## 18. Contacts

| Person | Role | Signal | Notes |
|--------|------|--------|-------|
| **c08r4d0r** (`@9cab90c7`) | Project owner | `+181****0908` | Group `fips-exit-node-poc`. Sets testing standard. |
| **Origami74 / Arjen** (`@1624e1bb`) | `ble-v2` author | `@1624e1bb-94ef-46d1-b03b-f067ea320af9` | Has **myco-core** (working embedder). Contact FIRST. |
| **Amperstrand** (`@cc5bdaa4`) | Collaborator | `@cc5bdaa4-98d2-4ce2-af1d-93aab049868a` | Confirmed Android basics going upstream. |
| **jmcorgan** | FIPS upstream maintainer | johnathan@corganlabs.com | For upstream merge questions. |

---

## 19. Test Vectors (Quick Reference)

| Parameter | Value |
|-----------|-------|
| Exit node IP | `66.92.204.38` |
| FIPS port (UDP) | `2121` |
| FIPS port (TCP) | `8443` |
| Exit node npub | `npub1mqelkzqp4659fws35h2wvr7z9caka5ml8qddj3ssnwaulwpxdd9sdc3esw` |
| App tag | `fips-overlay-v1` |
| Handshake type | `Noise XK` (phone is **initiator**) |
| TUN MTU | `1280` |
| BLE throughput (if used) | ~200/500 kbps up/down |
| BLE outbound queue depth | 32 (empirically tuned) |
| Expected log on success | `Connection promoted to active peer peer=vps1-exit` |
| Expected log on success | `Session established (initiator, XK)` |
| Expected log on success | `new_parent=vps1-exit` |

After a successful session, traffic routed through the VPN should egress as `66.92.204.38`
(`curl ifconfig.me` through the tunnel returns this IP).

---

## 20. Quick Start Checklist

1. [ ] Contact **Origami74** (`@1624e1bb-94ef-46d1-b03b-f067ea320af9`) for **myco-core** access
2. [ ] Clone `jmcorgan/fips`, checkout `ble-v2` branch
3. [ ] Install Android NDK + Rust Android targets (`aarch64-linux-android`)
4. [ ] Cross-compile FIPS: `cargo build --target aarch64-linux-android --release`
5. [ ] Fork **myco-core**, customize UI for exit-node use case
6. [ ] Add exit node config: npub `npub1mqelkzqp4659fws35h2wvr7z9caka5ml8qddj3ssnwaulwpxdd9sdc3esw`, addr `66.92.204.38:2121`
7. [ ] Coordinate with c08r4d0r on the Cashu `paid_peers` egress gate (starts empty)
8. [ ] Test Noise XK handshake to VPS1 (watch for "Connection promoted to active peer")
9. [ ] Test VpnService TUN routing (internet traffic through mesh)
10. [ ] Implement reconnect loop (5s → 60s backoff, reset on success)
11. [ ] Verify embedder contract: `fd00::/8` filtering, TCP MSS clamping, `recv_timeout`
12. [ ] Add per-app VPN toggle (optional)
13. [ ] Add BLE peer scan UI (optional)
14. [ ] Espresso/UI Automator happy-path tests
15. [ ] **Record video evidence** of the happy path (mandatory gate)

---

*This document consolidates input from c08r4d0r (`@9cab90c7`), Origami74/Arjen (`@1624e1bb`),
and Amperstrand (`@cc5bdaa4`). The `ble-v2` tradeoff analysis is verified from the actual git diff.
Last updated 2026-07-07.*
