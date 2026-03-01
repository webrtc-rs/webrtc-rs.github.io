# WebRTC v0.20.0-alpha.1: Async-Friendly WebRTC Built on Sans-I/O

We're excited to announce the first pre-release of **webrtc v0.20.0-alpha.1** — a ground-up rewrite of the async `webrtc` crate built on top of the Sans-I/O `rtc` protocol core. This milestone delivers on the promise we made in January 31: a runtime-agnostic, async-friendly WebRTC implementation in Rust with no callback hell, no memory leaks, and no Tokio lock-in.

---

## What's New

**v0.20.0-alpha.1** is a complete rewrite of the `webrtc` crate. Rather than patching the Tokio-coupled v0.17.x architecture, we started fresh with a thin async layer on top of the battle-tested Sans-I/O `rtc` crate.

### ✅ Runtime Agnostic — Tokio and smol Today, More Tomorrow

The new `webrtc` crate supports **multiple async runtimes** through feature flags:

```toml
# Tokio (default)
[dependencies]
webrtc = "0.20.0-alpha.1"

# smol
[dependencies]
webrtc = { version = "0.20.0-alpha.1", default-features = false, features = ["runtime-smol"] }
```

Switching runtimes is a one-line change — your application logic stays identical. The `Runtime` trait abstracts spawning, UDP sockets, timers, channels, mutexes, and DNS resolution, making it straightforward to add support for additional runtimes (async-std, embassy, etc.) in the future.

### ✅ Full Async API Parity with Sans-I/O `rtc`

Every operation exposed by the Sans-I/O `rtc` crate now has an `async fn` counterpart in the `webrtc` crate:

- `create_offer` / `create_answer`
- `set_local_description` / `set_remote_description`
- `add_ice_candidate` / `restart_ice`
- `create_data_channel` / `send` / `send_text`
- `add_track` / `remove_track` / `add_transceiver_from_track` / `add_transceiver_from_kind`
- `get_stats`
- And more…

### ✅ All v0.17.x Examples Ported

The [`examples/`](https://github.com/webrtc-rs/webrtc/tree/master/examples) directory ships with **20 working examples** that cover the same scenarios as the old v0.17.x crate:

| Category | Examples |
|----------|----------|
| **Data Channels** | `data-channels`, `data-channels-close`, `data-channels-create`, `data-channels-flow-control`, `data-channels-offer-answer`, `data-channels-simple` |
| **Media Playback** | `play-from-disk-vpx`, `play-from-disk-h26x`, `play-from-disk-renegotiation` |
| **Media Recording** | `save-to-disk-vpx`, `save-to-disk-h26x` |
| **Advanced Media** | `simulcast`, `swap-tracks`, `insertable-streams`, `reflect` |
| **Networking** | `rtp-forwarder`, `rtp-to-webrtc`, `broadcast` |
| **ICE** | `ice-restart` |

Each example demonstrates the new trait-based event handler API, making them excellent starting points for your own projects.

---

## Architecture: How It Works

The v0.20.0 architecture follows the [Quinn](https://github.com/quinn-rs/quinn)-inspired pattern we described in our [January design post](/blog/2026/01/31/async-friendly-webrtc-architecture):

```
┌─────────────────────────────────────────────────────────┐
│  Application (your code)                                │
│  ┌───────────────────────────────────────────────────┐  │
│  │  PeerConnectionEventHandler trait                 │  │
│  │  (on_track, on_ice_candidate, on_data_channel…)   │  │
│  └───────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────┤
│  webrtc crate (async layer)                             │
│  ┌──────────────────┐  ┌──────────────────────────┐     │
│  │ PeerConnection   │  │ PeerConnectionDriver     │     │
│  │ (API surface)    │◄─┤ (event loop + I/O)       │     │
│  └──────────────────┘  └──────────────────────────┘     │
│  ┌──────────────────────────────────────────────────┐   │
│  │ Runtime trait (Tokio / smol / …)                 │   │
│  └──────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────┤
│  rtc crate (Sans-I/O protocol core)                     │
│  ICE · DTLS · SRTP · SCTP · RTP/RTCP · Stats            │
└─────────────────────────────────────────────────────────┘
```

### The Builder Pattern

Creating a peer connection is clean and explicit:

```rust
use webrtc::peer_connection::{
    PeerConnectionBuilder, PeerConnectionEventHandler,
    RTCConfigurationBuilder, RTCIceServer,
    RTCPeerConnectionIceEvent, RTCPeerConnectionState,
};
use std::sync::Arc;

#[derive(Clone)]
struct MyHandler;

#[async_trait::async_trait]
impl PeerConnectionEventHandler for MyHandler {
    async fn on_ice_candidate(&self, event: RTCPeerConnectionIceEvent) {
        println!("ICE candidate: {:?}", event.candidate);
    }

    async fn on_connection_state_change(&self, state: RTCPeerConnectionState) {
        println!("Connection state: {:?}", state);
    }
}

let pc = PeerConnectionBuilder::new()
    .with_configuration(
        RTCConfigurationBuilder::default()
            .with_ice_servers(vec![RTCIceServer {
                urls: vec!["stun:stun.l.google.com:19302".to_owned()],
                ..Default::default()
            }])
            .build(),
    )
    .with_handler(Arc::new(MyHandler))
    .with_udp_addrs(vec!["0.0.0.0:0"])
    .build()
    .await?;

let offer = pc.create_offer(None).await?;
pc.set_local_description(offer).await?;
```

**No Arc cloning. No Box::new. No triple-nesting closures.** Just implement a trait and build.

### The Driver: Event Loop Under the Hood

The `PeerConnectionDriver` runs a background event loop that bridges the Sans-I/O `rtc` core and the async runtime. It follows the classic Sans-I/O loop:

1. **`poll_write()`** — flush outgoing packets to the network
2. **`poll_event()`** — dispatch protocol events to your `PeerConnectionEventHandler`
3. **`poll_read()`** — route incoming RTP/RTCP and data channel messages to the appropriate `TrackRemote` or `DataChannel`
4. **`poll_timeout()`** → `sleep()` → **`handle_timeout()`** — manage ICE, DTLS, and SCTP timers
5. **`recv_from()`** → **`handle_read()`** — feed incoming network packets into the protocol core

This is coordinated using `futures::select!`, which is runtime-agnostic — no `tokio::select!` coupling.

### ICE Gathering: Sans-I/O All the Way Down

ICE candidate gathering is also implemented as a Sans-I/O state machine (`RTCIceGatherer`) that uses STUN clients from the `rtc` crate. Host candidates are generated purely from local socket addresses. Server-reflexive candidates are gathered by driving STUN binding requests through the same `poll_write()` / `handle_read()` / `poll_event()` cycle. The gatherer implements the `sansio::Protocol` trait, keeping the entire ICE layer testable and runtime-agnostic.

---

## How This Differs from the Original Design

In our [January 31 architecture post](/blog/2026/01/31/async-friendly-webrtc-architecture), we outlined a proposed design. The implementation in v0.20.0-alpha.1 closely follows that vision, but with some pragmatic differences evolved during development:

### Runtime Abstraction: Broader Than Planned

The original design proposed a `Runtime` trait with three methods: `spawn`, `wrap_udp_socket`, and `new_timer`. The actual implementation goes further — the runtime module also abstracts:

- **Async mutexes** (`AsyncMutex` trait) — tokio::sync::Mutex vs smol::lock::Mutex
- **Async channels** (`AsyncSender` / `AsyncReceiver` traits) — mpsc channels for driver communication
- **Broadcast channels** — for pub/sub patterns
- **Async notify primitives** (`AsyncNotify` trait)
- **DNS resolution** (`resolve_host`)
- **Interval timers**, **timeout helpers**, and **block_on**

These are currently exposed as compile-time type aliases selected by feature flags (e.g., `pub type Mutex<T> = TokioMutex<T>`) rather than dynamic dispatch through the `Runtime` trait. This avoids runtime overhead in the hot path but means only one runtime can be active per compilation — a practical trade-off.

### Event Handling: `&self` Not `&mut self`

The original design proposed `PeerConnectionEventHandler` with `&mut self` methods for direct state mutation. The implementation uses `&self` instead:

```rust
// Original proposal
async fn on_track(&mut self, track: Track) { ... }

// Actual implementation
async fn on_track(&self, track: Arc<dyn TrackRemote>) { ... }
```

This makes the handler `Arc`-shareable and avoids holding a mutable borrow across `await` points in the driver's event loop. Users who need mutable state can use interior mutability (e.g., `Mutex`, `RwLock`, or atomics) inside their handler struct.

### Object-Safe Traits for Type Erasure

The implementation uses object-safe `async_trait` traits (`PeerConnection`, `DataChannel`, `TrackRemote`, `RtpTransceiver`, etc.) so the generic interceptor type parameter `I` doesn't leak into user code. `PeerConnectionBuilder::build()` returns `impl PeerConnection` rather than exposing the concrete `PeerConnectionImpl<I>`. This gives users a clean, non-generic API while keeping the interceptor framework fully pluggable internally.

### Channel-Based Driver Communication

Rather than user code directly calling into the Sans-I/O core (which would require locking the core mutex on every operation), the implementation uses an internal channel (`PeerConnectionDriverEvent`) to communicate between the API surface and the driver event loop. Operations like `send()` on a data channel write to the Sans-I/O core and then notify the driver via `try_send(WriteNotify)` to flush outgoing packets. This decouples the API thread from the I/O thread cleanly.

### No Stream-Based API (Yet)

The original design mentioned an optional pull-based stream API as an alternative to the trait handler. This hasn't been implemented in alpha.1 — the trait-based `PeerConnectionEventHandler` is the sole event delivery mechanism. Stream wrappers could be layered on top in a future release for users who prefer that pattern.

---

## Examples: Before and After

### Data Channel (v0.17.x → v0.20.0)

**Before (v0.17.x) — Callback Hell:**

```rust
let pc = Arc::new(api.new_peer_connection(config).await?);

let pc_clone = Arc::clone(&pc);
pc.on_peer_connection_state_change(Box::new(move |s| {
    let pc = Arc::clone(&pc_clone);
    Box::pin(async move { println!("State: {s}"); })
}));

let pc_clone2 = Arc::clone(&pc);
pc.on_data_channel(Box::new(move |dc| {
    let dc_clone = Arc::clone(&dc);
    Box::pin(async move {
        dc_clone.on_open(Box::new(move || {
            Box::pin(async move { println!("Opened!"); })
        }));
    })
}));
```

**After (v0.20.0) — Trait Handler:**

```rust
#[derive(Clone)]
struct MyHandler;

#[async_trait::async_trait]
impl PeerConnectionEventHandler for MyHandler {
    async fn on_connection_state_change(&self, state: RTCPeerConnectionState) {
        println!("State: {:?}", state);
    }

    async fn on_data_channel(&self, dc: Arc<dyn DataChannel>) {
        // Poll for events on the data channel
        while let Some(evt) = dc.poll().await {
            match evt {
                DataChannelEvent::OnOpen => println!("Opened!"),
                DataChannelEvent::OnMessage(msg) => println!("Message: {:?}", msg),
                _ => {}
            }
        }
    }
}

let pc = PeerConnectionBuilder::new()
    .with_handler(Arc::new(MyHandler))
    .with_udp_addrs(vec!["0.0.0.0:0"])
    .build()
    .await?;
```

Cleaner, shorter, and no risk of memory leaks from dangling callbacks.

---

## Future Work

This is an alpha release — there's more to do before we reach a stable v0.20.0. Here's what's on the roadmap:

### More Examples

The async `webrtc` crate currently ports the core v0.17.x examples. We plan to add parity with the full set of Sans-I/O `rtc` examples, including:

- `ice-tcp` / `ice-tcp-active-passive` — TCP transport for ICE
- `mdns-query-and-gather` — mDNS-based candidate gathering
- `perfect-negotiation` — W3C perfect negotiation pattern
- `trickle-ice` / `trickle-ice-host` / `trickle-ice-srflx` / `trickle-ice-relay` — Trickle ICE variants
- `rtcp-processing` — RTCP packet inspection
- `save-to-disk-av1` — AV1 codec support
- `stats` — WebRTC statistics API
- `simulcast_bidirection` — Bidirectional simulcast
- And more…

### ICE Gathering Improvements

Several known issues with ICE gathering are being tracked and worked on:

- **IPv6 ICE gather failure** ([#774](https://github.com/webrtc-rs/webrtc/issues/774)) — Currently, IPv6 address gathering can fail when the STUN server doesn't resolve to an IPv6 address. We need more robust address-family handling.
- **Better socket recv error handling** ([#777](https://github.com/webrtc-rs/webrtc/issues/777)) — Today, a socket receive error terminates the entire driver event loop. We need graceful recovery for transient errors (e.g., ICMP unreachable) rather than a hard stop.
- **Localhost STUN timeout** ([#778](https://github.com/webrtc-rs/webrtc/issues/778)) — When `stun:stun.l.google.com:19302` is configured and the only local address is `127.0.0.1`, gathering takes excessively long to reach `RTCIceGatheringState::Complete`. STUN requests to external servers from localhost will never succeed; we should detect this case and skip them.

### H.265 Codec Fixes

The H.265 packetizer/depacketizer has known issues in the simulcast and H.26x play-from-disk/save-to-disk examples ([#779](https://github.com/webrtc-rs/webrtc/issues/779)). This needs investigation and verification across different stream configurations.

### Runtime Abstraction Improvements

The current runtime abstraction uses compile-time feature flags with type aliases:

```rust
#[cfg(feature = "runtime-tokio")]
pub type Mutex<T> = TokioMutex<T>;

#[cfg(feature = "runtime-smol")]
pub type Mutex<T> = SmolMutex<T>;
```

While this works and avoids dynamic dispatch overhead, it has limitations:

- Only one runtime per compilation
- Adding a new runtime requires modifying the `webrtc` crate itself
- Primitives like `Mutex`, `Sender`, `Receiver`, `Notify`, etc. are not part of the `Runtime` trait

We plan to introduce a **`RuntimeFactory` abstraction** that moves these primitives behind the `Runtime` trait (or companion traits), enabling external runtime implementations without forking the crate. This would allow third-party crates to implement runtime support (e.g., `webrtc-embassy`) by simply implementing a trait.

### Performance & Testing

- Performance benchmarks and optimization
- Comprehensive browser interoperability testing (Chrome, Firefox, Safari, Edge)
- Deterministic testing leveraging the Sans-I/O architecture
- Memory leak detection and verification

---

## Try It Out

Install via Cargo:

```toml
[dependencies]
webrtc = "0.20.0-alpha.1"
```

Or with smol:

```toml
[dependencies]
webrtc = { version = "0.20.0-alpha.1", default-features = false, features = ["runtime-smol"] }
```

Check out the [examples](https://github.com/webrtc-rs/webrtc/tree/master/examples) to get started. We'd love your feedback — this is an alpha, and now is the best time to shape the API.

---

## Get Involved

This is a community-driven project and we welcome contributions:

- **Try the alpha**: Run the examples, build something, and report what works and what doesn't
- **File issues**: Help us find bugs and rough edges
- **Contribute**: Help with examples, runtime adapters, documentation, or tests
- **Discuss**: Share your use cases and API feedback

Join our community:

- **GitHub**: https://github.com/webrtc-rs/webrtc
- **Sans-I/O core (rtc)**: https://github.com/webrtc-rs/rtc
- **Discord**: https://discord.gg/4Ju8UHdXMs
- **Discussions**: https://github.com/webrtc-rs/webrtc/discussions

---

## Conclusion

**v0.20.0-alpha.1** brings the async-friendly, runtime-agnostic WebRTC vision to life. Built on the proven Sans-I/O `rtc` crate, it delivers:

- **No callback hell** — clean trait-based event handling
- **No memory leaks** — explicit ownership, no dangling closures
- **No Tokio lock-in** — Tokio and smol today, more runtimes coming
- **Full API parity** — every Sans-I/O operation has an async counterpart
- **20 working examples** — ported from v0.17.x and ready to use

This is the beginning of the next chapter for webrtc-rs. We're shipping early to get feedback, iterate, and build the best WebRTC library in the Rust ecosystem together.

Thank you to all the contributors who made this possible. 🦀

---

## Links

- **GitHub**: https://github.com/webrtc-rs/webrtc
- **rtc (Sans-I/O)**: https://github.com/webrtc-rs/rtc
- **Examples**: https://github.com/webrtc-rs/webrtc/tree/master/examples
- **Docs**: https://docs.rs/webrtc
- **Discord**: https://discord.gg/4Ju8UHdXMs

---

## Further Reading

- [Building Async-Friendly webrtc on Sans-I/O rtc: Architecture Design and Roadmap](/blog/2026/01/31/async-friendly-webrtc-architecture)
- [WebRTC v0.17.0: Feature Freeze and Shifting to Sans-I/O](/blog/2026/01/31/webrtc-v0.17.0-feature-freeze-sansio-shift)
