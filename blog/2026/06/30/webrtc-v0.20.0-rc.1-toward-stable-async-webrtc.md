# WebRTC v0.20.0-rc.1: Toward Stable Async-Friendly WebRTC Built on Sans-I/O

We're excited to announce **webrtc v0.20.0-rc.1**, released on **June 30, 2026**. This release candidate is the milestone we were aiming for when we shipped **v0.20.0-alpha.1** in March: the async `webrtc` crate now reaches **feature parity with the Sans-I/O `rtc` crate**, and the async example suite now matches the `rtc` examples wherever an async/networked wrapper makes sense.

Since `v0.20.0-alpha.1`, we've added mDNS support, TURN relay support, ICE TCP support, stronger socket error handling, more integration tests, and a much broader example set. The result is a far more complete, resilient, and practical async WebRTC stack for Rust.

---

## What's New

**v0.20.0-rc.1** is the feature-complete release candidate for the new async `webrtc` architecture. Rather than introducing another large architectural shift, this release focuses on closing the remaining feature gaps between the async crate and the Sans-I/O `rtc` core.

### ✅ Async `webrtc` Reaches Feature Parity with Sans-I/O `rtc`

With **v0.20.0-rc.1**, the async layer is no longer just a thin rewrite of the old API surface. It now exposes the same practical capabilities as the underlying Sans-I/O core:

- mDNS query-and-gather
- TURN relay candidate gathering
- ICE TCP, including active/passive flows
- full Trickle ICE variants
- stats collection
- richer media and codec examples

In other words: if you can build it with the Sans-I/O `rtc` crate, you can now build it with the async `webrtc` crate too.

### ✅ Example Parity: 20 to 33 Runnable Cargo Examples

At **v0.20.0-alpha.1**, the async crate shipped with 20 examples, mostly covering the old v0.17.x surface area. For **rc.1**, the examples in `Cargo.toml` have grown to **33 runnable examples**, bringing the async crate in line with the Sans-I/O example set.

New async examples added since alpha.1 include:

| Area | Examples |
|------|----------|
| **ICE / Connectivity** | `mdns-query-and-gather`, `trickle-ice`, `trickle-ice-host`, `trickle-ice-srflx`, `trickle-ice-relay`, `ice-tcp`, `ice-tcp-active-offer`, `ice-tcp-passive-answer` |
| **Media / Control** | `rtcp-processing`, `save-to-disk-av1`, `play-from-disk-playlist-control`, `simulcast_add_transceiver_from_kind` |
| **Observability** | `stats` |

This is an important milestone for users migrating from Sans-I/O `rtc` prototypes to production async applications: the async crate is no longer missing the advanced scenarios.

### ✅ mDNS Support for Privacy-Friendly Host Candidates

The async driver now supports **mDNS-based candidate gathering and resolution**, allowing local host candidates to be advertised through `.local` names instead of leaking raw LAN IP addresses.

This brings the async crate up to date with modern browser behavior and makes local-network interop much more realistic. The new support includes both:

- **gather**: publishing local host candidates as mDNS names
- **query**: resolving remote mDNS candidates during connection setup

The new `mdns-query-and-gather` example and dedicated integration tests exercise this flow end-to-end.

### ✅ ICE Relay Support via TURN

The async stack now supports **relay candidate gathering** through TURN servers.

Under the hood, the new TURN relayer manages allocation, permission creation, channel data handling, and relay candidate publication, then feeds the decapsulated packets back into the peer connection driver. This is a major step for users operating behind restrictive NATs or firewalls where direct host or server-reflexive connectivity is not enough.

The new `trickle-ice-relay` example demonstrates relay-only gathering with `RTCIceTransportPolicy::Relay`.

### ✅ ICE TCP Support, Including Active/Passive Mode

Async `webrtc` now supports **ICE over TCP**.

That includes:

- passive TCP candidates gathered from local listeners
- active TCP candidates for dialing remote passive endpoints
- TCP packet framing/decoding inside the async transport layer
- driver support for mixed UDP/TCP connectivity paths

This matters for deployments where UDP is blocked or unreliable. The new `ice-tcp`, `ice-tcp-active-offer`, and `ice-tcp-passive-answer` examples show how to bring up a WebRTC data channel over TCP and how to handle active/passive negotiation correctly.

### ✅ Better Handling for Socket Write and Receive Errors

One of the rough edges called out after alpha.1 was error handling in the background driver. A transient socket problem could stop gathering progress or destabilize the event loop.

That has been tightened up significantly:

- retryable receive errors such as `Interrupted`, `WouldBlock`, `ConnectionRefused`, `ConnectionReset`, and `TimedOut` are now handled explicitly
- STUN and TURN gathering paths react to socket write failures without tearing down unrelated driver state
- TCP read failures are isolated to the affected stream instead of poisoning the whole connection path

This makes long-lived peer connections much more resilient in real networks, especially when probing STUN/TURN infrastructure or dealing with lossy TCP/UDP environments.

---

## Architecture: How It Works

The async architecture introduced in alpha.1 is still the same: a clean user-facing `PeerConnection` API backed by a background `PeerConnectionDriver` that bridges the Sans-I/O `rtc` core and the selected async runtime.

A lot of the work since alpha.1 was not just "adding features", but making that driver cleaner and more robust as those features landed.

The `PeerConnectionDriver` is now structured around clearer phases:

1. `poll_writes()`
2. `poll_events()`
3. `poll_reads()`
4. `poll_timeout()` / `handle_timeout()`

The transport-specific responsibilities were also separated out more cleanly:

- `RTCStunGatherer` for STUN-based gathering
- `RTCTurnRelayer` for TURN allocations and relay traffic
- `RTCTcpTransport` for ICE TCP listener/stream management

That split made it possible to add mDNS, TURN relay, and TCP support without turning the driver into a monolith.

It also reduced the amount of lock churn in the hot path by draining core writes, events, and reads into local vectors before processing them asynchronously.

---

## Configuring mDNS, TURN, and TCP in Async `webrtc`

The builder surface remains the same clean async API introduced in alpha.1, but it now drives a much richer transport stack:

```rust
use std::sync::Arc;
use std::time::Duration;

use rtc::ice::mdns::MulticastDnsMode;
use webrtc::peer_connection::{
    PeerConnectionBuilder, PeerConnectionEventHandler, RTCConfigurationBuilder,
    RTCIceServer, SettingEngine,
};

#[derive(Clone)]
struct MyHandler;

#[async_trait::async_trait]
impl PeerConnectionEventHandler for MyHandler {}

let mut setting_engine = SettingEngine::default();
setting_engine.set_multicast_dns_mode(MulticastDnsMode::QueryAndGather);
setting_engine.set_multicast_dns_timeout(Some(Duration::from_secs(5)));

let pc = PeerConnectionBuilder::new()
    .with_configuration(
        RTCConfigurationBuilder::default()
            .with_ice_servers(vec![RTCIceServer {
                urls: vec!["turn:turn.example.com:3478?transport=udp".to_owned()],
                username: "user".to_owned(),
                credential: "pass".to_owned(),
            }])
            .build(),
    )
    .with_setting_engine(setting_engine)
    .with_handler(Arc::new(MyHandler))
    .with_udp_addrs(vec!["0.0.0.0:0"])
    .with_tcp_addrs(vec!["0.0.0.0:8443"])
    .build()
    .await?;
```

The same async builder now covers the advanced connectivity options that previously only existed in the Sans-I/O layer.

---

## Browser Interop and AppRTC

Another useful piece of the ecosystem is now available again: **[https://appr.tc](https://appr.tc)** is online as a public peer-to-peer signaling service.

Its implementation lives in the `apprtc` codebase in the webrtc-rs ecosystem, and it is especially helpful for **browser interoperability testing**. With AppRTC infrastructure available, it is much easier to test async `webrtc` implementations against real browsers such as:

- Chrome
- Safari
- other standards-compliant WebRTC browsers

For anyone validating P2P signaling and connection setup outside of local toy examples, this makes rc.1 much easier to evaluate in realistic browser-to-Rust scenarios.

---

## On the Road to Stable

With **v0.20.0-rc.1**, the big missing async features are in place. The focus from here to stable is much narrower:

1. bug fixing and interoperability polish
2. broader browser validation
3. performance tuning and cleanup

That is exactly what an RC should represent: not a moving target, but a feature-complete release candidate that is ready for hardening.

---

## Try It Out

Update your `Cargo.toml`:

```toml
[dependencies]
webrtc = "0.20.0-rc.1"
```

Or with `smol`:

```toml
[dependencies]
webrtc = { version = "0.20.0-rc.1", default-features = false, features = ["runtime-smol"] }
```

Then explore the [examples](https://github.com/webrtc-rs/webrtc/tree/master/examples), especially:

- `mdns-query-and-gather`
- `trickle-ice-relay`
- `ice-tcp`
- `ice-tcp-active-offer` / `ice-tcp-passive-answer`
- `stats`

Check out the [examples](https://github.com/webrtc-rs/webrtc/tree/master/examples), run the new networking scenarios, and try browser interop testing with AppRTC. We'd love feedback while the release is still in RC.

---

## Get Involved

This is still the best time to shake out edge cases before the stable release:

- **Try the RC**: Run the new mDNS, TURN, and ICE TCP examples in your environment
- **Test interoperability**: Validate behavior against Chrome, Safari, and other WebRTC implementations
- **File issues**: Help us catch bugs in real network conditions
- **Contribute**: Tests, docs, examples, and runtime improvements are all welcome

Join our community:

- **GitHub**: https://github.com/webrtc-rs/webrtc
- **Sans-I/O core (`rtc`)**: https://github.com/webrtc-rs/rtc
- **Discord**: https://discord.gg/4Ju8UHdXMs
- **Discussions**: https://github.com/webrtc-rs/webrtc/discussions

---

## Conclusion

**v0.20.0-rc.1** brings the async `webrtc` crate to the point we set out to reach in March:

- **Feature parity with Sans-I/O `rtc`**
- **Example parity for async-supported scenarios**
- **mDNS, TURN relay, and ICE TCP support**
- **Much stronger socket and transport error handling**
- **A cleaner, more modular driver architecture**

The async-friendly, runtime-agnostic WebRTC design is no longer just a promising rewrite — it is now approaching stable as a complete implementation.

Thank you to everyone who tested the alpha and beta releases and helped push async `webrtc` to this point. 🦀

---

## Links

- **GitHub**: https://github.com/webrtc-rs/webrtc
- **rtc (Sans-I/O)**: https://github.com/webrtc-rs/rtc
- **Examples**: https://github.com/webrtc-rs/webrtc/tree/master/examples
- **Docs**: https://docs.rs/webrtc
- **AppRTC**: https://appr.tc
- **Discord**: https://discord.gg/4Ju8UHdXMs

---

## Further Reading

- [Building Async-Friendly webrtc on Sans-I/O rtc: Architecture Design and Roadmap](/blog/2026/01/31/async-friendly-webrtc-architecture)
- [WebRTC v0.20.0-alpha.1: Async-Friendly WebRTC Built on Sans-I/O](/blog/2026/03/01/webrtc-v0.20.0-alpha.1-async-webrtc-on-sansio)
