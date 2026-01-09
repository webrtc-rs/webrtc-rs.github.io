# Announcing `rtc` 0.6.0: Interceptor Framework Complete 🎉

We're thrilled to announce **`rtc` 0.6.0**, a major milestone release that completes the **Interceptor framework** for our sans-I/O WebRTC implementation. This release achieves interceptor feature parity with the async-based `webrtc` crate, bringing RTCP feedback mechanisms for adaptive streaming, packet loss recovery, and congestion control.

## What's New in 0.6.0

### Complete Interceptor Framework 🚀

The headline feature is the **completion of the rtc-interceptor crate**, providing a comprehensive suite of RTP/RTCP processing interceptors. Built on the `sansio::Protocol` pattern, interceptors form a composable pipeline for packet processing, statistics collection, and quality control.

**Available Interceptors:**

#### RTCP Reports
- **`SenderReportInterceptor`** - Generates RTCP Sender Reports (SR) for local streams
- **`ReceiverReportInterceptor`** - Generates RTCP Receiver Reports (RR) with reception statistics

#### NACK (Negative Acknowledgement)
- **`NackGeneratorInterceptor`** - Detects packet loss and requests retransmissions (RFC 4585)
- **`NackResponderInterceptor`** - Buffers sent packets and handles retransmission requests

#### TWCC (Transport Wide Congestion Control)
- **`TwccSenderInterceptor`** - Adds transport-wide sequence numbers to outgoing RTP packets
- **`TwccReceiverInterceptor`** - Tracks packet arrival times and generates TransportLayerCC feedback

**Key capabilities:**
- Automatic packet loss detection and recovery via NACK
- Real-time quality statistics (packet loss, jitter, RTT)
- Bandwidth estimation feedback via TWCC
- Composable interceptor chains via `Registry`
- Stream binding for both local (sender) and remote (receiver) streams

**Example use cases:**
- Adaptive bitrate streaming with TWCC feedback
- Reliable media delivery with NACK retransmissions
- Network quality monitoring with RTCP reports
- Simulcast layer selection based on bandwidth estimates

### Enhanced Media Examples 📹

All media examples now use `register_default_interceptors()` to enable RTCP feedback:

- `broadcast` - Multi-party media distribution with quality reports
- `play-from-disk-*` - File playback with adaptive retransmission
- `reflect` - Media relay with transport-wide feedback
- `rtp-forwarder` - RTP bridging with loss recovery
- `simulcast` - **Multi-resolution streaming with full RTCP feedback**
- `save-to-disk-*` - Recording with quality monitoring
- `swap-tracks` - Dynamic track switching with seamless feedback

**Simulcast Breakthrough:** The simulcast example now achieves **highest quality streaming** from browsers thanks to proper RTCP feedback. Browsers receive Receiver Reports and TWCC feedback, enabling them to optimize sending bitrate and quality layer selection.

### Generic Interceptor Support 🔧

`RTCPeerConnection` now supports generic interceptor types:

```rust
// Before (v0.5.x)
let config = RTCConfigurationBuilder::new().build();
let pc = RTCPeerConnection::new(config)?;

// After (v0.6.0) - with interceptors
let mut media_engine = MediaEngine::default();
let registry = Registry::new();
let registry = register_default_interceptors(registry, &mut media_engine)?;

let config = RTCConfigurationBuilder::new()
    .with_media_engine(media_engine)
    .with_interceptor_registry(registry)
    .build();

let pc = RTCPeerConnection::new(config)?;
```

The generic `RTCPeerConnection<I>` allows full type safety while maintaining zero-cost abstractions. The default `NoopInterceptor` ensures existing code continues working without changes.

---

## API Changes

This release includes breaking changes to support the interceptor framework:

### RTCConfiguration Generic Parameter

`RTCConfiguration` is now generic over the interceptor type:

```rust
// Default (no interceptors)
RTCConfiguration<NoopInterceptor>

// With custom interceptor chain
RTCConfiguration<impl Interceptor>
```

### RTCConfigurationBuilder Methods

New builder methods for configuring interceptors:

```rust
pub fn with_interceptor_registry<P>(
    self, 
    registry: Registry<P>
) -> RTCConfigurationBuilder<P>
where
    P: Interceptor
```

These are called automatically during transceiver creation but can be used for custom stream management.

### New Types

- **`interceptor::Registry`** - Builder for composing interceptor chains
- **`interceptor::StreamInfo`** - Stream metadata (SSRC, codec, RTCP feedback, extensions)
- **`interceptor::Packet`** - RTP or RTCP packet enum for interceptor processing
- **`interceptor::TaggedPacket`** - Packet with transport context

---

## Interceptor Framework Design

### Sans-I/O Architecture

Interceptors follow the same `sansio::Protocol` pattern as the rest of the stack:

```rust
pub trait Interceptor:
    sansio::Protocol<
        TaggedPacket,
        TaggedPacket,
        (),
        Rout = TaggedPacket,
        Wout = TaggedPacket,
        Eout = (),
        Time = Instant,
        Error = shared::error::Error,
    > + Sized
{
    // Stream lifecycle
    fn bind_local_stream(&mut self, info: &StreamInfo);
    fn unbind_local_stream(&mut self, info: &StreamInfo);
    fn bind_remote_stream(&mut self, info: &StreamInfo);
    fn unbind_remote_stream(&mut self, info: &StreamInfo);
}
```

### Composable Chains

Interceptors wrap each other, forming a processing chain:

```rust
let chain = Registry::new()
    .with(SenderReportBuilder::new()
        .with_interval(Duration::from_secs(1))
        .build())
    .with(ReceiverReportBuilder::new()
        .with_interval(Duration::from_secs(1))
        .build())
    .with(NackGeneratorBuilder::new()
        .with_size(512)
        .with_interval(Duration::from_millis(100))
        .build())
    .with(NackResponderBuilder::new()
        .with_size(1024)
        .build())
    .with(TwccSenderBuilder::new().build())
    .with(TwccReceiverBuilder::new()
        .with_interval(Duration::from_millis(100))
        .build())
    .build();
```

### No Direction Concept

Unlike PeerConnection handlers which reverse order between read/write, interceptors process all operations in the **same structural order** (outer → inner):

```text
handle_read:    A → B → C → NoopInterceptor
handle_write:   A → B → C → NoopInterceptor
poll_read:      A ← B ← C ← NoopInterceptor
poll_write:     A ← B ← C ← NoopInterceptor
```

This symmetric design simplifies reasoning about packet flow.

---

## Quick Start: Default Interceptors

For most applications, use `register_default_interceptors()`:

```rust
use rtc::peer_connection::RTCPeerConnection;
use rtc::peer_connection::configuration::RTCConfigurationBuilder;
use rtc::peer_connection::configuration::interceptor_registry::register_default_interceptors;
use rtc::peer_connection::configuration::media_engine::MediaEngine;
use interceptor::Registry;

// Setup interceptors
let mut media_engine = MediaEngine::default();
let registry = Registry::new();
let registry = register_default_interceptors(registry, &mut media_engine)?;

// Create peer connection
let config = RTCConfigurationBuilder::new()
    .with_media_engine(media_engine)
    .with_interceptor_registry(registry)
    .build();

let pc = RTCPeerConnection::new(config)?;
```

This enables:
- ✅ NACK for packet loss recovery (video only)
- ✅ RTCP Sender and Receiver Reports
- ✅ Simulcast header extensions
- ✅ TWCC receiver for congestion feedback

---

## Migration Guide

### Updating Peer Connection Creation

```rust
// Old code (v0.5.x)
use rtc::peer_connection::RTCPeerConnection;
use rtc::peer_connection::configuration::RTCConfigurationBuilder;

let config = RTCConfigurationBuilder::new().build();
let pc = RTCPeerConnection::new(config)?;

// New code (v0.6.0) - with default interceptors
use rtc::peer_connection::RTCPeerConnection;
use rtc::peer_connection::configuration::RTCConfigurationBuilder;
use rtc::peer_connection::configuration::interceptor_registry::register_default_interceptors;
use rtc::peer_connection::configuration::media_engine::MediaEngine;
use interceptor::Registry;

let mut media_engine = MediaEngine::default();
let registry = Registry::new();
let registry = register_default_interceptors(registry, &mut media_engine)?;

let config = RTCConfigurationBuilder::new()
    .with_media_engine(media_engine)
    .with_interceptor_registry(registry)
    .build();

let pc = RTCPeerConnection::new(config)?;
```

### Keeping Existing Behavior (No Interceptors)

If you don't need interceptors, your code continues working without changes:

```rust
// This still works - uses NoopInterceptor by default
let config = RTCConfigurationBuilder::new().build();
let pc = RTCPeerConnection::new(config)?;
```

### Custom Interceptor Configuration

For fine-grained control, configure individual interceptors:

```rust
use rtc::peer_connection::configuration::interceptor_registry::*;

// Only NACK (no TWCC or reports)
let registry = configure_nack(registry, &mut media_engine);

// Or only TWCC
let registry = configure_twcc(registry, &mut media_engine)?;

// Or only RTCP reports
let registry = configure_rtcp_reports(registry);
```

---

## Performance Improvements

### Eliminated SystemTime::now() Calls

All `SystemTime::now()` calls have been removed from the `sansio::Protocol` hot path (issue #16). Time is now provided explicitly via `handle_timeout(now: Instant)`:

**Benefits:**
- ✅ Fully deterministic protocol behavior
- ✅ Simplified unit testing with controlled time
- ✅ Better performance (no syscalls in packet processing)
- ✅ More idiomatic sans-I/O design

### New Time Abstractions

Added `shared::time::SystemInstant` for time conversions between unix and ntp time.

---

## Testing & Validation

### Comprehensive Test Coverage

- **Unit tests** - All interceptors have extensive unit test coverage
- **Integration tests** - Full interceptor chain tests in `rtc-interceptor/tests/`
- **Interop tests** - Browser interoperability tests in `rtc/tests/`
  - `interceptor_rtcp_reports_interop.rs` - SR/RR validation
  - `simulcast_*_interop.rs` - Simulcast with RTCP feedback
  - All media examples validated with real browsers

### Real-World Validation

The simulcast example demonstrates the real-world impact:

**Before v0.6.0:** Browsers send lowest quality due to lack of RTCP feedback

**After v0.6.0:** Browsers send highest quality, optimizing based on RR and TWCC feedback

---

## What's Next: Interceptor Deep Dive 📝

A detailed technical blog post is coming soon to explore:

- Interceptor framework architecture and design principles
- How interceptors integrate with the sans-I/O pipeline
- Implementation details of NACK, RTCP reports, and TWCC
- Custom interceptor development guide
- Performance characteristics and optimization techniques

Stay tuned for the deep dive into the interceptor framework!

---

## Getting Started

### Installation

```toml
[dependencies]
rtc = "0.6.0"
```

### Quick Example

```rust
use rtc::peer_connection::RTCPeerConnection;
use rtc::peer_connection::configuration::RTCConfigurationBuilder;
use rtc::peer_connection::configuration::interceptor_registry::register_default_interceptors;
use rtc::peer_connection::configuration::media_engine::MediaEngine;
use rtc::sansio::Protocol;
use rtc::interceptor::Registry;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Setup with interceptors
    let mut media_engine = MediaEngine::default();
    let registry = Registry::new();
    let registry = register_default_interceptors(registry, &mut media_engine)?;
    
    let config = RTCConfigurationBuilder::new()
        .with_media_engine(media_engine)
        .with_interceptor_registry(registry)
        .build();
    
    let mut pc = RTCPeerConnection::new(config)?;

    // Create and set local description
    let offer = pc.create_offer(None)?;
    pc.set_local_description(offer)?;

    // Sans-I/O event loop
    loop {
        while let Some(msg) = pc.poll_write() {
            // Send to network (includes RTCP reports, NACK, TWCC feedback)
        }
        
        while let Some(event) = pc.poll_event() {
            // Handle state changes
        }
        
        while let Some(message) = pc.poll_read() {
            // Process application messages (RTP/RTCP/DataChannel)
        }
        
        // Handle I/O and timeouts
        // ...
    }
}
```

Check out the [examples directory](https://github.com/webrtc-rs/rtc/tree/master/examples/examples) for complete working code, especially the [simulcast example](https://github.com/webrtc-rs/rtc/tree/master/examples/examples/simulcast)!

---

## Feature Parity Update

Progress toward full feature parity with the `webrtc` crate:

✅ **Complete:**
- ICE, DTLS, SRTP/SRTCP, SCTP
- Data Channels (reliable & unreliable)
- RTP/RTCP, Media Tracks, SDP
- Peer Connection API
- Simulcast
- **RTCP Interceptors** ← New in 0.6.0!
  - NACK (packet loss recovery)
  - Sender/Receiver Reports
  - TWCC (congestion control feedback)

🎯 **Future Work:**
- Advanced bandwidth estimation algorithms
- Custom QoS policies
- Performance analytics and observability

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
- ✨ Complete `rtc-interceptor` crate with NACK, RTCP Reports, and TWCC
- ✨ `SenderReportInterceptor` and `ReceiverReportInterceptor` for RTCP SR/RR
- ✨ `NackGeneratorInterceptor` and `NackResponderInterceptor` for packet loss recovery
- ✨ `TwccSenderInterceptor` and `TwccReceiverInterceptor` for congestion control
- ✨ `Registry` builder for composing interceptor chains
- ✨ `register_default_interceptors()` convenience function
- ✨ `configure_nack()`, `configure_rtcp_reports()`, `configure_twcc()` helpers
- ✨ `StreamInfo` type for stream metadata and capabilities
- ✨ `Packet` and `TaggedPacket` types for interceptor processing
- ✨ `shared::time::SystemInstant` for time conversions
- ✨ Stream binding APIs: `bind_local_stream()`, `unbind_local_stream()`, `bind_remote_stream()`, `unbind_remote_stream()`
- ✨ Comprehensive integration tests for interceptors
- ✨ Browser interop tests for RTCP feedback

### Changed
- 💥 `RTCConfiguration<I>` - Now generic over interceptor type
- 💥 `RTCConfigurationBuilder::with_interceptor_registry()` - Configure interceptor chain
- 💥 All `sansio::Protocol` implementations - Removed `SystemTime::now()` calls (issue #16)
- 🔄 All media examples - Now use `register_default_interceptors()`
- 🔄 Simulcast example - Now achieves highest quality with RTCP feedback

### Improved
- 📚 Extensive inline documentation for interceptor framework
- 📚 README updates in `rtc-interceptor` subdirectories (NACK, TWCC, Reports)
- 📚 API documentation with practical examples
- 🏗️ More idiomatic sans-I/O design (time is passed explicitly)
- 🏗️ Better performance (eliminated syscalls in hot path)
- 🏗️ Fully deterministic protocol behavior for testing

### Fixed
- 🐛 Sender/Receiver Report calculation bugs
- 🐛 TWCC sequence number handling
- 🐛 NACK retransmission timing
- 🐛 Stream lifecycle management

---

## Commits

This release includes **39 commits** with major contributions to interceptor functionality, testing, and documentation. Key highlights:

- Complete NACK interceptor implementation (generator + responder)
- Complete TWCC interceptor implementation (sender + receiver + recorder)
- RTCP Reports interceptors with accurate statistics
- Integration with all media examples
- Comprehensive test coverage (unit + integration + interop)
- Performance improvements (eliminated `SystemTime::now()`)
- Stream binding lifecycle management
- Generic interceptor support in `RTCPeerConnection`

---

## Relationship with `webrtc` Crate

As stated in previous announcements, `rtc` (sans-I/O) and `webrtc` (async) are **complementary**:

- **Use `webrtc`** for quick start with Tokio and async/await
- **Use `rtc`** for runtime independence, custom I/O, or maximum control

Both crates are actively maintained and share protocol implementations where possible. The interceptor framework has been greatly improved in `rtc` due to the architectural advantages of the sans-I/O design for testing and composability.

---

*Thanks to everyone who contributed feedback, bug reports, and feature requests! The interceptor framework represents months of careful design and implementation work. Special thanks to the Rust WebRTC community for their continued support.* 🦀

Feedback and contributions welcome on [GitHub](https://github.com/webrtc-rs/rtc)!
