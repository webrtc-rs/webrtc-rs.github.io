# Announcing `rtc` 0.7.0: mDNS Support for Privacy-Preserving WebRTC 🎉

We're excited to announce **`rtc` 0.7.0**, a significant release that brings **multicast DNS (mDNS) support** to our sans-I/O WebRTC implementation. This release enables privacy-preserving peer connections by hiding local IP addresses with `.local` hostnames, following [RFC 6762](https://www.rfc-editor.org/rfc/rfc6762.html) and WebRTC best practices.

## What's New in 0.7.0

### mDNS Support for IP Privacy 🔒

The headline feature is **comprehensive mDNS support** across the stack. mDNS allows WebRTC peers to exchange ICE candidates without exposing local IP addresses, addressing privacy concerns in modern browsers and applications.

**Key capabilities:**

- **Query-only mode** - Resolve `.local` hostnames from remote peers
- **Query-and-gather mode** - Both resolve remote hostnames and hide your own IP
- **Sans-I/O design** - mDNS implementation follows `sansio::Protocol` pattern
- **Seamless integration** - mDNS is deeply integrated into the ICE agent
- **Configurable modes** - Disabled, QueryOnly, or QueryAndGather via `SettingEngine`

**Privacy benefits:**

- Prevent IP address leakage to remote peers
- Comply with privacy-focused browser policies (Firefox, Safari)
- Support WebRTC in privacy-sensitive applications
- Follow W3C WebRTC security guidelines

---

## Architecture: Three-Level mDNS Integration

The mDNS implementation spans three layers of the stack, demonstrating the composability of the sans-I/O architecture:

### 1. `rtc-mdns` Crate: Sans-I/O Protocol Implementation

A new standalone crate implementing mDNS as a `sansio::Protocol`:

```rust
use rtc_mdns::{MdnsConfig, Mdns, MdnsEvent};
use sansio::Protocol;
use std::time::{Duration, Instant};

// Create mDNS connection
let config = MdnsConfig::default()
    .with_query_interval(Duration::from_secs(1))
    .with_local_names(vec!["myhost.local".to_string()])
    .with_local_ip(local_ip);

let mut mdns = Mdns::new(config);

// Query for a hostname
let query_id = mdns.query("remote-peer.local");

// Sans-I/O event loop
loop {
    // 1. Send outgoing mDNS packets
    while let Some(packet) = mdns.poll_write() {
        socket.send_to(&packet.message, packet.transport.peer_addr).await?;
    }
    
    // 2. Handle mDNS events
    while let Some(event) = mdns.poll_event() {
        match event {
            MdnsEvent::QueryAnswered { query_id, addr } => {
                println!("Resolved to: {}", addr);
            }
            _ => {}
        }
    }
    
    // 3. Process incoming packets and timeouts
    // mdns.handle_read(packet)?;
    // mdns.handle_timeout(now)?;
}
```

**Features of `rtc-mdns`:**

- RFC 6762 compliant mDNS implementation
- Query and server modes
- Automatic query retries with configurable intervals
- Multiple concurrent query tracking
- Zero I/O dependencies (pure protocol logic)

### 2. ICE Agent Integration: Seamless `.local` Resolution

The mDNS protocol is integrated directly into `rtc-ice::Agent`:

```rust
// ICE agent transparently uses mDNS when needed
impl Protocol for Agent {
    fn handle_read(&mut self, message: TaggedBytesMut) -> Result<()> {
        // mDNS packets automatically routed to internal mDNS handler
        if message.transport.peer_addr.port() == MDNS_PORT {
            self.mdns.handle_read(message)?;
        } else {
            // Regular STUN/ICE packets
        }
    }
    
    fn poll_write(&mut self) -> Option<TaggedBytes> {
        // Outgoing mDNS queries sent seamlessly
        if let Some(packet) = self.mdns.poll_write() {
            return Some(packet);
        }
        // Regular ICE packets...
    }
}
```

**ICE improvements:**

- Automatic `.local` hostname resolution during connectivity checks
- mDNS queries triggered on-demand when encountering `.local` candidates
- Proper handling of mDNS responses in candidate pair evaluation
- Fixed `find_remote_candidate()` bug for `.local` remote addresses

### 3. PeerConnection Configuration: User-Friendly API

Configure mDNS behavior through `SettingEngine`:

```rust
use rtc::peer_connection::configuration::RTCConfigurationBuilder;
use rtc::peer_connection::configuration::setting_engine::SettingEngine;
use rtc::ice::mdns::MulticastDnsMode;
use std::time::Duration;

let mut setting_engine = SettingEngine::default();

// Option 1: Query-only (resolve remote .local addresses)
setting_engine.set_multicast_dns_mode(MulticastDnsMode::QueryOnly);

// Option 2: Query-and-gather (hide your IP + resolve remote)
setting_engine.set_multicast_dns_mode(MulticastDnsMode::QueryAndGather);
setting_engine.set_multicast_dns_local_name("my-peer.local".to_string());
setting_engine.set_multicast_dns_local_ip(Some(local_ip));

// Optional: Set query timeout
setting_engine.set_multicast_dns_timeout(Some(Duration::from_secs(10)));

let config = RTCConfigurationBuilder::new()
    .with_setting_engine(setting_engine)
    .build();

let peer_connection = RTCPeerConnection::new(config)?;
```

**Three mDNS modes:**

- **`Disabled`** - No mDNS support (default for compatibility)
- **`QueryOnly`** - Resolve `.local` hostnames from remote peers only
- **`QueryAndGather`** - Resolve remote + advertise local hostname (full privacy)

---

## New Example: mDNS Query and Gather

The [mdns-query-and-gather example](https://github.com/webrtc-rs/rtc/tree/master/examples/examples/mdns-query-and-gather) demonstrates how WebRTC.rs hides local IP addresses using mDNS:

```rust
// Configure full mDNS support
let mut setting_engine = SettingEngine::default();
setting_engine.set_multicast_dns_mode(MulticastDnsMode::QueryAndGather);
setting_engine.set_multicast_dns_local_name(
    "webrtc-rs-hides-local-ip-by-mdns.local".to_string()
);
setting_engine.set_multicast_dns_local_ip(Some(local_addr.ip()));

let config = RTCConfigurationBuilder::new()
    .with_setting_engine(setting_engine)
    .build();

let mut pc = RTCPeerConnection::new(config)?;

// Add candidate with local IP - mDNS will hide it in SDP
let candidate = CandidateHostConfig {
    base_config: CandidateConfig {
        address: local_addr.ip().to_string(),  // Real IP
        port: local_addr.port(),
        // ... mDNS transparently converts to .local hostname
    },
    ..Default::default()
}.new_candidate_host()?;

pc.add_local_candidate(candidate)?;

// SDP will show: "webrtc-rs-hides-local-ip-by-mdns.local" instead of IP
```

**Key demonstration:**

This example showcases a unique aspect of the sans-I/O design: **handling multiple I/O sockets in a single event loop**. The application multiplexes:

1. **mDNS multicast socket** - For sending/receiving mDNS queries (port 5353)
2. **WebRTC peer connection socket** - For ICE/DTLS/RTP/RTCP traffic

```rust
let mdns_socket = UdpSocket::from_std(MulticastSocket::new().into_std()?)?;
let pc_socket = UdpSocket::bind(format!("{host}:{port}")).await?;

loop {
    // Route outgoing packets to correct socket
    while let Some(msg) = pc.poll_write() {
        if msg.transport.peer_addr.port() == MDNS_PORT {
            mdns_socket.send_to(&msg.message, msg.transport.peer_addr).await?;
        } else {
            pc_socket.send_to(&msg.message, msg.transport.peer_addr).await?;
        }
    }
    
    // Multiplex incoming packets
    tokio::select! {
        Ok((n, peer_addr)) = mdns_socket.recv_from(&mut mdns_buf) => {
            pc.handle_read(TaggedBytesMut { /* mDNS packet */ })?;
        }
        Ok((n, peer_addr)) = pc_socket.recv_from(&mut pc_buf) => {
            pc.handle_read(TaggedBytesMut { /* WebRTC packet */ })?;
        }
    }
}
```

This demonstrates the flexibility of sans-I/O architecture - the protocol layer doesn't care how many sockets you use or how you multiplex them. The caller has complete control over I/O strategy.

---

## Additional Improvements

### Bug Fixes and Stability

- **Fixed interceptor initialization** - Interceptors now start after connection is established (#19)
- **Fixed simulcast test flakiness** - Resolved SRTP duplicate index issue (#18)
- **Fixed ICE candidate resolution** - Proper `.local` address handling in `find_remote_candidate()`
- **Fixed mDNS query lifecycle** - Correct timeout handling and query state management
- **Optimized DTLS handshake** - Removed unused `remote_addr` parameter

### Testing Improvements

- Increased timeout for simulcast interop tests (15s → 25s)
- Better test stability for browser interoperability tests
- Enhanced mDNS unit and integration tests

### Documentation Updates

- Comprehensive `SettingEngine` mDNS configuration docs
- Updated README with mDNS example reference
- Inline documentation for `MulticastDnsMode` and related APIs
- Fixed `MediaStreamTrack::new()` API documentation after v0.5.0 changes

---

## Why mDNS Matters for WebRTC

### Privacy Concerns

Traditional WebRTC ICE gathering exposes local IP addresses in SDP offers/answers. This creates privacy risks:

- **Location tracking** - IP addresses reveal geographic location
- **Network topology** - Exposes internal network structure
- **Identity correlation** - IP addresses can link user identities across sessions

### Modern Browser Policies

Privacy-focused browsers have responded:

- **Firefox** - Uses mDNS by default for local candidates
- **Safari** - Requires mDNS for privacy-sensitive contexts
- **Chrome** - Working towards mDNS support for privacy

### WebRTC.rs Solution

With v0.7.0, Rust applications can now match browser privacy standards:

```rust
// Old behavior (v0.6.0): SDP exposes IP
// candidate:1 1 udp 192.168.1.100 50000 typ host

// New behavior (v0.7.0): SDP uses mDNS hostname
// candidate:1 1 udp webrtc-rs-hides-local-ip-by-mdns.local 50000 typ host
```

Remote peers resolve the hostname via mDNS queries on the local network, maintaining privacy while enabling connectivity.

---

## Migration Guide

### Enabling mDNS in Existing Applications

If you're upgrading from v0.6.0, mDNS is **disabled by default** for backward compatibility. To enable:

```rust
// Step 1: Configure mDNS mode
let mut setting_engine = SettingEngine::default();
setting_engine.set_multicast_dns_mode(MulticastDnsMode::QueryOnly);

// Step 2: Apply to configuration
let config = RTCConfigurationBuilder::new()
    .with_setting_engine(setting_engine)
    .build();

let mut pc = RTCPeerConnection::new(config)?;
```

### Full Privacy Mode (Query and Gather)

For maximum privacy with IP hiding:

```rust
use std::net::IpAddr;

let local_ip: IpAddr = get_local_ip(); // Your method to get local IP

let mut setting_engine = SettingEngine::default();
setting_engine.set_multicast_dns_mode(MulticastDnsMode::QueryAndGather);
setting_engine.set_multicast_dns_local_name("myapp.local".to_string());
setting_engine.set_multicast_dns_local_ip(Some(local_ip));
setting_engine.set_multicast_dns_timeout(Some(Duration::from_secs(10)));

let config = RTCConfigurationBuilder::new()
    .with_setting_engine(setting_engine)
    .build();
```

### Socket Management for mDNS

When using mDNS, route packets based on destination port:

```rust
const MDNS_PORT: u16 = 5353;

loop {
    while let Some(msg) = pc.poll_write() {
        if msg.transport.peer_addr.port() == MDNS_PORT {
            // Send via mDNS multicast socket
            mdns_socket.send_to(&msg.message, msg.transport.peer_addr).await?;
        } else {
            // Send via regular WebRTC socket
            pc_socket.send_to(&msg.message, msg.transport.peer_addr).await?;
        }
    }
}
```

---

## Sans-I/O Benefits Demonstrated

This release showcases why sans-I/O architecture is powerful for WebRTC:

### 1. Protocol Composability

mDNS implementation (`rtc-mdns::Mdns`) is itself a `sansio::Protocol`, making it trivial to embed inside another protocol (`rtc-ice::Agent`):

```rust
struct Agent {
    mdns: Mdns,  // Embedded sans-I/O protocol
    // other fields...
}

impl Protocol for Agent {
    fn poll_write(&mut self) -> Option<TaggedBytes> {
        // Delegate to embedded protocol
        if let Some(packet) = self.mdns.poll_write() {
            return Some(packet);
        }
        // Agent's own logic...
    }
}
```

No callbacks, no async complexity - just simple method delegation.

### 2. Multi-Socket I/O Control

The caller controls how many sockets to use and how to multiplex them. The mdns-query-and-gather example uses two sockets, but you could use:

- One socket with port filtering
- Separate threads per socket
- Different async runtimes per socket
- Mix blocking and async I/O

The protocol layer doesn't impose any I/O strategy.

### 3. Testability

mDNS protocol logic is fully testable without real network I/O:

```rust
#[test]
fn test_mdns_query_response() {
    let mut mdns = Mdns::new(config);
    let query_id = mdns.query("test.local");
    
    // Simulate sending query
    let packet = mdns.poll_write().unwrap();
    
    // Simulate receiving response
    let response = create_mdns_response("test.local", "192.168.1.100");
    mdns.handle_read(response).unwrap();
    
    // Verify event
    let event = mdns.poll_event().unwrap();
    assert!(matches!(event, MdnsEvent::QueryAnswered { .. }));
}
```

No mocking frameworks, no network setup - just pure protocol testing.

---

## Getting Started

### Installation

```toml
[dependencies]
rtc = "0.7.0"
```

### Quick Example with mDNS

```rust
use rtc::peer_connection::RTCPeerConnection;
use rtc::peer_connection::configuration::RTCConfigurationBuilder;
use rtc::peer_connection::configuration::setting_engine::SettingEngine;
use rtc::ice::mdns::MulticastDnsMode;
use rtc::mdns::{MDNS_PORT, MulticastSocket};
use rtc::sansio::Protocol;
use rtc::shared::{TaggedBytesMut, TransportContext, TransportProtocol};
use bytes::BytesMut;
use tokio::net::UdpSocket;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Enable mDNS
    let mut setting_engine = SettingEngine::default();
    setting_engine.set_multicast_dns_mode(MulticastDnsMode::QueryOnly);
    
    let config = RTCConfigurationBuilder::new()
        .with_setting_engine(setting_engine)
        .build();
    
    let mut pc = RTCPeerConnection::new(config)?;

    // Create and set local description
    let offer = pc.create_offer(None)?;
    pc.set_local_description(offer)?;

    // Create two sockets: one for mDNS multicast, one for peer connection
    let mdns_socket = UdpSocket::from_std(MulticastSocket::new().into_std()?)?;
    let pc_socket = UdpSocket::bind("0.0.0.0:0").await?;
    let pc_local_addr = pc_socket.local_addr()?;
    
    let mut mdns_buf = vec![0u8; 2000];
    let mut pc_buf = vec![0u8; 2000];

    // Sans-I/O event loop with multi-socket handling
    loop {
        // 1. Send outgoing packets to appropriate socket
        while let Some(msg) = pc.poll_write() {
            if msg.transport.peer_addr.port() == MDNS_PORT {
                mdns_socket.send_to(&msg.message, msg.transport.peer_addr).await?;
            } else {
                pc_socket.send_to(&msg.message, msg.transport.peer_addr).await?;
            }
        }
        
        // 2. Handle state changes
        while let Some(event) = pc.poll_event() {
            // Handle events
        }
        
        // 3. Process application messages
        while let Some(message) = pc.poll_read() {
            // Process RTP/RTCP/DataChannel messages
        }
        
        // 4. Multiplex I/O from both sockets
        tokio::select! {
            Ok((n, peer_addr)) = mdns_socket.recv_from(&mut mdns_buf) => {
                pc.handle_read(TaggedBytesMut {
                    now: std::time::Instant::now(),
                    transport: TransportContext {
                        local_addr: "0.0.0.0:5353".parse().unwrap(),
                        peer_addr,
                        ecn: None,
                        transport_protocol: TransportProtocol::UDP,
                    },
                    message: BytesMut::from(&mdns_buf[..n]),
                })?;
            }
            Ok((n, peer_addr)) = pc_socket.recv_from(&mut pc_buf) => {
                pc.handle_read(TaggedBytesMut {
                    now: std::time::Instant::now(),
                    transport: TransportContext {
                        local_addr: pc_local_addr,
                        peer_addr,
                        ecn: None,
                        transport_protocol: TransportProtocol::UDP,
                    },
                    message: BytesMut::from(&pc_buf[..n]),
                })?;
            }
        }
    }
}
```

Check out the [mdns-query-and-gather example](https://github.com/webrtc-rs/rtc/tree/master/examples/examples/mdns-query-and-gather) for the complete working code!

---

## Feature Parity Update

Progress toward full feature parity with the `webrtc` crate:

✅ **Complete:**
- ICE, DTLS, SRTP/SRTCP, SCTP
- Data Channels (reliable & unreliable)
- RTP/RTCP, Media Tracks, SDP
- Peer Connection API
- Simulcast
- RTCP Interceptors (NACK, Reports, TWCC)
- **mDNS Support** ← New in 0.7.0!

🎯 **Future Work:**
- Advanced bandwidth estimation algorithms
- Performance optimizations and benchmarking
- Additional privacy features (TURN over TLS, etc.)

---

## Links

- **GitHub**: https://github.com/webrtc-rs/rtc
- **Crate**: https://crates.io/crates/rtc
- **Docs**: https://docs.rs/rtc
- **Discord**: https://discord.gg/4Ju8UHdXMs
- **Examples**: https://github.com/webrtc-rs/rtc/tree/master/examples

---

## Full Changelog

### Added
- ✨ **`rtc-mdns` crate** - Complete sans-I/O mDNS implementation (RFC 6762)
- ✨ mDNS integration in `rtc-ice::Agent` for `.local` hostname resolution
- ✨ `MulticastDnsMode` enum (Disabled, QueryOnly, QueryAndGather)
- ✨ `SettingEngine::set_multicast_dns_mode()` configuration API
- ✨ `SettingEngine::set_multicast_dns_local_name()` for IP hiding
- ✨ `SettingEngine::set_multicast_dns_local_ip()` for local IP binding
- ✨ `SettingEngine::set_multicast_dns_timeout()` for query timeout control
- ✨ `mdns-query-and-gather` example demonstrating IP privacy
- ✨ `MulticastSocket` utility for mDNS multicast I/O
- ✨ Comprehensive mDNS documentation and examples

### Changed
- 🔄 `rtc-ice::Agent` - Integrated mDNS protocol for candidate resolution
- 🔄 ICE candidate evaluation - Handles `.local` hostnames transparently

### Fixed
- 🐛 Interceptor initialization timing - Start after connection established (#19)
- 🐛 Simulcast test flakiness - SRTP duplicate index handling (#18)
- 🐛 `find_remote_candidate()` - Correct `.local` address matching
- 🐛 mDNS query lifecycle - Proper timeout and state management
- 🐛 mDNS answer processing - Fixed local IP extraction from A records
- 🐛 DTLS handshake - Removed unused `remote_addr` parameter

### Improved
- 📚 Complete `SettingEngine` mDNS documentation with examples
- 📚 Updated main README with mDNS example reference
- 📚 Enhanced inline documentation for all mDNS APIs
- 🏗️ Refactored ICE agent for cleaner mDNS integration
- 🏗️ Better multicast socket abstraction
- 🧪 Increased test timeouts for better stability (15s → 25s)

---

## Commits

This release includes **19 commits** focused on mDNS implementation, integration, and stability:

- Complete sans-I/O mDNS protocol implementation
- Deep ICE agent integration for `.local` resolution
- User-friendly configuration via `SettingEngine`
- Comprehensive example with multi-socket I/O
- Bug fixes for interceptor timing and simulcast stability
- Documentation improvements across the board

---

## Relationship with `webrtc` Crate

As stated in previous announcements, `rtc` (sans-I/O) and `webrtc` (async) are **complementary**:

- **Use `webrtc`** for quick start with Tokio and async/await
- **Use `rtc`** for runtime independence, custom I/O, or maximum control

Both crates are actively maintained and share protocol implementations where possible. The mDNS feature demonstrates the advantages of sans-I/O for protocol composability and flexible I/O management.

---

*Thanks to everyone who contributed feedback, bug reports, and feature requests! The mDNS implementation represents significant work in privacy-preserving WebRTC. Special thanks to the WebRTC-rs community for their continued support.* 🦀

Feedback and contributions welcome on [GitHub](https://github.com/webrtc-rs/rtc)!
