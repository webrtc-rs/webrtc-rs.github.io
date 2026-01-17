# Announcing `rtc` 0.8.0: WebRTC Stats Collection for Sans-I/O 📊

We're thrilled to announce **`rtc` 0.8.0**, a major milestone that brings **comprehensive WebRTC statistics collection** to our sans-I/O WebRTC implementation. This release implements the [W3C WebRTC Stats API](https://www.w3.org/TR/webrtc-stats/), enabling applications to monitor and diagnose peer connection health, media quality, and network performance—all without sacrificing the zero-overhead design principles of sansio.

## What's New in 0.8.0

### WebRTC Stats Collection API 📈

The headline feature is a **production-ready stats collection system** that continuously accumulates statistics during normal packet processing and provides instant snapshots on demand via the `get_stats()` API.

**Key capabilities:**

- **W3C WebRTC Stats compliant** - Implements standardized stats types and fields
- **Zero-cost accumulation** - Stats collected during normal packet processing, no extra queries
- **Instant snapshots** - No async waiting, no mutex locks, no I/O
- **Comprehensive coverage** - ICE, Transport, RTP streams, Data Channels, Codecs, and more
- **Sans-I/O design** - Explicit timestamp parameter, no hidden system calls
- **Type-safe access** - Strongly-typed stats structures with iterator helpers

**Statistics categories covered:**

- ✅ **ICE & Transport** - Candidate pairs, bytes/packets sent/received, RTT, STUN transactions
- ✅ **RTP Streams** - Inbound/outbound packet counts, loss, jitter, bitrate, RTCP feedback
- ✅ **Data Channels** - Messages sent/received, bytes transferred, buffer amounts
- ✅ **Codecs** - Payload types, MIME types, clock rates, channels
- ✅ **Certificates** - Fingerprints, algorithms, certificate chains
- ✅ **Peer Connection** - Data channels opened/closed, connection state

---

## Architecture: Continuous Accumulation, Instant Snapshots

The stats implementation follows the sansio philosophy: **accumulate incrementally during event processing, snapshot synchronously on demand**.

### How It Works

```
┌────────────────────────────────────────────────────────────────┐
│                    RTCPeerConnection                           │
│                                                                │
│   handle_read(packet)   ──────┐                                │
│                               ▼                                │
│                      ┌─────────────────┐                       │
│                      │  Handler        │                       │
│                      │  Pipeline       │                       │
│                      │  (ICE, DTLS,    │                       │
│                      │   SRTP, SCTP,   │                       │
│                      │   Interceptors) │                       │
│                      └────────┬────────┘                       │
│                               │                                │
│                               ▼                                │
│                      ┌─────────────────┐                       │
│                      │ RTCStatsAccumu- │                       │
│                      │ lator           │                       │
│                      │ (incremental    │                       │
│                      │  updates)       │                       │
│                      └────────┬────────┘                       │
│                               │                                │
│   get_stats(now,  ────────────┼────────┐                       │
│     StatsSelector::None)      ▼        ▼                       │
│                      snapshot()  RTCStatsReport                │
│                      (instant)   (returned immediately)        │
└────────────────────────────────────────────────────────────────┘
```

**Benefits over traditional approaches:**

- **No async coordination** - Stats ready instantly, no `await` needed
- **No locks or mutexes** - Single-threaded accumulation in event loop
- **No extra network I/O** - Stats collected as packets flow through
- **Deterministic timestamps** - Explicit `now` parameter enables testing

### Comparison with Other Implementations

| Aspect          | Pion (Go)              | Async WebRTC          | Sansio RTC (v0.8.0)              |
|-----------------|------------------------|-----------------------|----------------------------------|
| Collection      | WaitGroup + goroutines | tokio::join! async    | Synchronous accumulation         |
| Timing          | On-demand fetch        | On-demand async fetch | Continuous accumulation + snapshot |
| I/O             | Direct network access  | Async network         | No I/O, application-driven       |
| Threading       | Multi-threaded         | Async tasks           | Single-threaded, event-loop friendly |
| Synchronization | Mutex + WaitGroup      | Mutex + async         | None needed (zero locks!)        |

---

## Using the Stats API

### Basic Usage

```rust
use rtc::peer_connection::RTCPeerConnection;
use rtc::statistics::StatsSelector;
use std::time::Instant;

// Create and configure peer connection
let mut peer_connection = RTCPeerConnection::new(config)?;

// ... normal WebRTC operations (handle_read, poll_write, etc.) ...

// Get stats snapshot at any time (all stats)
let now = Instant::now();
let report = peer_connection.get_stats(now, StatsSelector::None);

// Access peer connection stats
if let Some(pc_stats) = report.peer_connection() {
    println!("Data channels opened: {}", pc_stats.data_channels_opened);
    println!("Data channels closed: {}", pc_stats.data_channels_closed);
}

// Iterate over inbound RTP streams
for inbound in report.inbound_rtp_streams() {
    println!("SSRC: {}", inbound.received_rtp_stream_stats.rtp_stream_stats.ssrc);
    println!("Packets received: {}", inbound.received_rtp_stream_stats.packets_received);
    println!("Bytes received: {}", inbound.bytes_received);
    println!("Packets lost: {}", inbound.received_rtp_stream_stats.packets_lost);
    println!("Jitter: {}", inbound.received_rtp_stream_stats.jitter);
}
```

### Filtering Stats with StatsSelector

The `get_stats()` API supports the W3C stats selection algorithm, allowing you to filter statistics for specific senders or receivers:

```rust
use rtc::statistics::StatsSelector;

// Get all stats for the entire connection
let all_stats = peer_connection.get_stats(now, StatsSelector::None);

// Get stats only for a specific sender (outbound streams)
let sender_stats = peer_connection.get_stats(now, StatsSelector::Sender(sender_id));

// Get stats only for a specific receiver (inbound streams)
let receiver_stats = peer_connection.get_stats(now, StatsSelector::Receiver(receiver_id));
```

This is particularly useful when you only need stats for a specific track:

```rust
// Get the sender ID from an RTP sender
let sender_id = peer_connection.add_track(track)?;

// Later, get stats only for that sender
let report = peer_connection.get_stats(Instant::now(), StatsSelector::Sender(sender_id));

// Only contains outbound RTP streams and related stats for this sender
for outbound in report.outbound_rtp_streams() {
    println!("Packets sent: {}", outbound.sent_rtp_stream_stats.packets_sent);
}
```

**Benefits of filtered stats:**
- Reduced memory allocation (smaller report)
- Faster processing (fewer stats to iterate)
- Cleaner API when you only care about specific streams

---

### Filtering by Stats Type

You can also filter the report by stats type after retrieval:

```rust
use rtc::statistics::stats::RTCStatsType;

let report = peer_connection.get_stats(now, StatsSelector::None);

// Filter by stats type
for entry in report.iter_by_type(RTCStatsType::RemoteCandidate) {
    if let RTCStatsReportEntry::RemoteCandidate(candidate) = entry {
        println!("Remote IP: {}", candidate.address.as_deref().unwrap_or("unknown"));
        println!("Port: {}", candidate.port);
    }
}

// Or iterate all entries with pattern matching
for entry in report.iter() {
    match entry {
        RTCStatsReportEntry::InboundRtp(stats) => { /* ... */ }
        RTCStatsReportEntry::OutboundRtp(stats) => { /* ... */ }
        RTCStatsReportEntry::IceCandidatePair(stats) => { /* ... */ }
        _ => {}
    }
}
```

---

## Periodic Stats Reporting


### Periodic Stats Reporting

Integrate stats into your event loop for periodic monitoring:

```rust
use std::time::{Duration, Instant};

const STATS_INTERVAL: Duration = Duration::from_secs(5);
let mut last_stats_time = Instant::now();

loop {
    // Normal event loop operations
    while let Some(msg) = peer_connection.poll_write() {
        socket.send_to(&msg.message, msg.transport.peer_addr).await?;
    }
    
    while let Some(event) = peer_connection.poll_event() {
        // Handle events
    }
    
    while let Some(message) = peer_connection.poll_read() {
        // Process messages
    }
    
    // Periodic stats collection
    let now = Instant::now();
    if now.duration_since(last_stats_time) >= STATS_INTERVAL {
        let report = peer_connection.get_stats(now, StatsSelector::None);
        
        // Log or export stats
        println!("\n=== WebRTC Stats ===");
        for inbound in report.inbound_rtp_streams() {
            println!("Track: {}", inbound.track_identifier);
            println!("  Packets: {}", inbound.received_rtp_stream_stats.packets_received);
            println!("  Loss: {}", inbound.received_rtp_stream_stats.packets_lost);
        }
        println!("====================\n");
        
        last_stats_time = now;
    }
    
    // Continue with timeout and I/O handling...
}
```

### Type-Safe Stats Access

The `RTCStatsReport` provides convenient accessors for common stats:

```rust
let report = peer_connection.get_stats(Instant::now(), StatsSelector::None);

// Direct access to peer connection stats
let pc_stats = report.peer_connection();

// Direct access to transport stats
let transport_stats = report.transport();

// Iterators for collections
let inbound_streams = report.inbound_rtp_streams();
let outbound_streams = report.outbound_rtp_streams();
let data_channels = report.data_channels();
let candidate_pairs = report.candidate_pairs();

// Generic access by ID
if let Some(entry) = report.get("specific-stats-id") {
    // Process specific stats entry
}

// Iterate all entries
for entry in report.iter() {
    match entry {
        RTCStatsReportEntry::InboundRtp(stats) => { /* ... */ }
        RTCStatsReportEntry::OutboundRtp(stats) => { /* ... */ }
        RTCStatsReportEntry::IceCandidatePair(stats) => { /* ... */ }
        _ => {}
    }
}
```

---

## New Example: Stats Monitoring

The [stats example](https://github.com/webrtc-rs/rtc/tree/master/examples/examples/stats) demonstrates comprehensive stats collection for incoming audio and video streams.

**Output example:**

```
=== WebRTC Stats ===

Inbound RTP Stats for: video/vp8
  SSRC: 1234567890
  Packets Received: 2450
  Bytes Received: 1361125
  Packets Lost: 5
  Jitter: 12.5

Inbound RTP Stats for: audio/opus
  SSRC: 987654321
  Packets Received: 4820
  Bytes Received: 245000
  Packets Lost: 0
  Jitter: 2.3

Remote Candidate: IP(192.168.1.100) Port(54321)
====================
```

---

## Design Principles: Sansio Stats Architecture

The stats implementation showcases core sansio design principles:

### 1. Incremental Accumulation

Stats are accumulated **incrementally** during normal `handle_read/handle_write/handle_event/handle_timeout` processing:

```rust
// In ICE handler
impl IceHandler {
    fn handle_read(&mut self, packet: TaggedBytesMut) -> Result<()> {
        // Process packet normally
        // ...
        
        // Update stats atomically as side effect
        if let Some(pair_id) = self.active_candidate_pair {
            self.stats.ice_candidate_pairs
                .get_mut(&pair_id)
                .on_packet_received(packet.message.len(), packet.now);
        }
        
        Ok(())
    }
}
```

No separate stats collection phase—stats update is a zero-cost side effect of normal processing.

### 2. Synchronous Snapshots

The `get_stats()` call returns **instantly** with current accumulated values:

```rust
pub fn get_stats(&mut self, now: std::time::Instant, selector: StatsSelector) -> RTCStatsReport {
    // Update ICE agent stats before taking snapshot
    self.update_ice_agent_stats();
    
    // Update codec stats from transceivers before taking snapshot
    self.update_codec_stats();
    
    // Return instant snapshot (no async, no waiting)
    self.pipeline_context.stats.snapshot_with_selector(now, selector)
}
```

No async coordination, no mutex locks, no waiting. Just a pure function call.

### 3. Explicit Timestamp Parameter

Following sansio principles, `get_stats()` takes an **explicit timestamp** rather than calling `Instant::now()` internally:

```rust
// Good: Caller controls time source
let report = peer_connection.get_stats(Instant::now(), StatsSelector::None);

// Enables deterministic testing
let report = peer_connection.get_stats(test_instant, StatsSelector::None);
```

This maintains sansio's promise: **no hidden I/O or system calls**.

### 4. Centralized Stats Storage

A single `RTCStatsAccumulator` in `PipelineContext` holds all stats, avoiding scattered state across components:

```rust
pub struct PipelineContext {
    // Other pipeline state...
    
    pub stats: RTCStatsAccumulator,  // Centralized stats storage
}

impl RTCStatsAccumulator {
    pub fn snapshot(&self, now: Instant) -> RTCStatsReport {
        let mut entries = Vec::new();
        
        // Collect snapshots from all accumulators
        entries.push(self.peer_connection.snapshot(now));
        entries.push(self.transport.snapshot(now));
        
        for (_, accumulator) in &self.ice_candidate_pairs {
            entries.push(accumulator.snapshot(now));
        }
        
        for (_, accumulator) in &self.inbound_rtp_streams {
            entries.push(accumulator.snapshot(now));
        }
        
        // ... more stat types ...
        
        RTCStatsReport::new(entries)
    }
}
```

### 5. Type-Safe, Iterator-Friendly API

The `RTCStatsReport` provides both generic and type-specific access patterns:

```rust
// Generic access
for entry in report.iter() { /* all entries */ }
for id in report.ids() { /* all IDs */ }
if let Some(entry) = report.get("specific-id") { /* by ID */ }

// Type-filtered access
for entry in report.iter_by_type(RTCStatsType::InboundRTP) { /* filtered */ }

// Direct type-specific accessors (zero-cost)
let pc_stats = report.peer_connection();  // Option<&RTCPeerConnectionStats>
let inbound = report.inbound_rtp_streams();  // Iterator<&RTCInboundRtpStreamStats>
let outbound = report.outbound_rtp_streams();  // Iterator<&RTCOutboundRtpStreamStats>
```

---

## Coverage: What Stats Are Available?

The implementation provides comprehensive coverage of W3C WebRTC Stats:

### ✅ Fully Implemented (95%+ coverage)

**Network & Transport:**
- `RTCIceCandidateStats` - Local and remote ICE candidates
- `RTCIceCandidatePairStats` - Bytes/packets sent/received, RTT, STUN transactions
- `RTCTransportStats` - DTLS state, SRTP cipher, selected candidate pair
- `RTCCertificateStats` - Certificate fingerprints and chains

**RTP Streams:**
- `RTCInboundRtpStreamStats` - Packets/bytes received, loss, jitter, FEC
- `RTCOutboundRtpStreamStats` - Packets/bytes sent, retransmits, quality limitation
- `RTCRemoteInboundRtpStreamStats` - Round-trip time from RTCP reports
- `RTCRemoteOutboundRtpStreamStats` - Remote sender info from RTCP

**Codecs & Channels:**
- `RTCCodecStats` - Payload types, MIME types, clock rates
- `RTCDataChannelStats` - Messages/bytes sent/received, buffer state
- `RTCPeerConnectionStats` - Data channels opened/closed

### 🔄 Application-Provided (By Design)

Some stats require media encoding/decoding information that lives outside the sansio protocol layer:

- `RTCAudioSourceStats` / `RTCVideoSourceStats` - Capture device stats
- `RTCAudioPlayoutStats` - Audio jitter buffer, concealment
- Encoder/Decoder stats - Frame encoding/decoding metrics

The sansio design provides **integration APIs** for applications to supply these stats, maintaining the separation between protocol (sansio) and media processing (application).

---

## Additional Improvements in v0.8.0

### New Examples & Integration Tests

- ✨ **stats example** - Comprehensive stats monitoring demonstration
- ✨ **stats integration tests** - Browser interop verification
- ✨ **W3C stats selection algorithm** - Filter stats by sender/receiver with `StatsSelector`
- ✨ **Multiple interop tests** - mdns, rtcp-processing, trickle-ice, ice-tcp, media-rejection
- ✨ play-from-disk-playlist-control, save-to-disk-av1, data-channels-simple examples
- ✨ rtcp-processing, ICE TCP active mode, trickle-ice, ice-tcp examples

### Bug Fixes

- 🐛 Fixed stats transport missing fields
- 🐛 Fixed noop interceptor bug for `handle_poll_read` (#28)
- 🐛 Fixed simulcast bidirectional issue (#20)
- 🐛 Fixed integration test for single media session verification (#11)
- 🐛 Fixed SDP generation bug for rejected media sections with mid

### Documentation & Code Quality

- 📚 Comprehensive stats API documentation with examples
- 🏗️ Refactored stats accumulator architecture for clarity
- 🏗️ Implemented W3C stats selection algorithm

### Parser Improvements

- 🔧 Accept unknown bandwidth types in SDP parser per RFC 8866 (#29)

---

## Why Stats Matter for WebRTC

### Monitoring & Diagnostics

WebRTC applications need real-time visibility into connection health:

- **Network quality** - Packet loss, jitter, RTT for adaptive bitrate
- **Media quality** - Frame rates, resolution, encoding quality
- **Troubleshooting** - Diagnose connectivity issues, performance bottlenecks
- **User experience** - Detect poor quality before users complain

### Production Requirements

For production WebRTC deployments:

- **SLA monitoring** - Track uptime, quality metrics
- **Capacity planning** - Understand bandwidth usage patterns
- **Cost optimization** - Identify inefficient codec/network configurations
- **Compliance** - Record call quality metrics for regulations

### Integration with Monitoring Systems

The sansio stats API makes integration straightforward:

```rust
// Export to Prometheus
let report = peer_connection.get_stats(Instant::now(), StatsSelector::None);
for inbound in report.inbound_rtp_streams() {
    prometheus_gauge!("webrtc_packets_received", 
        inbound.received_rtp_stream_stats.packets_received as f64);
    prometheus_gauge!("webrtc_packets_lost",
        inbound.received_rtp_stream_stats.packets_lost as f64);
}

// Log to structured logging
for outbound in report.outbound_rtp_streams() {
    info!(
        ssrc = outbound.sent_rtp_stream_stats.rtp_stream_stats.ssrc,
        packets_sent = outbound.sent_rtp_stream_stats.packets_sent,
        bytes_sent = outbound.sent_rtp_stream_stats.bytes_sent,
        "Outbound RTP stream stats"
    );
}

// Send to custom analytics
let analytics_data = serde_json::to_string(&report)?;
analytics_client.track("webrtc_stats", analytics_data)?;
```

---

## Migration Guide

### Upgrading from v0.7.x

The stats API is entirely **new** in v0.8.0, so no breaking changes to existing code. Simply start calling `get_stats()` when you want statistics:

```rust
use std::time::Instant;

// Your existing v0.7.x code continues to work
let mut peer_connection = RTCPeerConnection::new(config)?;

// New in v0.8.0: Get stats at any time
let report = peer_connection.get_stats(Instant::now(), StatsSelector::None);
```

### Enabling Stats Collection

Stats accumulation is **always enabled** (zero overhead by design). Just call `get_stats()` whenever you need a snapshot:

```rust
// In your event loop
let now = Instant::now();

// Process peer connection as usual
while let Some(msg) = peer_connection.poll_write() { /* ... */ }
while let Some(event) = peer_connection.poll_event() { /* ... */ }

// Get stats periodically or on-demand
if should_collect_stats {
    let report = peer_connection.get_stats(now, StatsSelector::None);
    // Use report...
}
```

---

## Feature Parity Update

Progress toward full feature parity with the `webrtc` crate:

✅ **Complete:**
- ICE, DTLS, SRTP/SRTCP, SCTP
- Data Channels (reliable & unreliable)
- RTP/RTCP, Media Tracks, SDP
- Peer Connection API
- Simulcast
- RTCP Interceptors (NACK, RTCP Reports, TWCC)
- mDNS Support
- **WebRTC Stats API** ← New in 0.8.0! 🎉

🎯 **Future Work:**
- Advanced bandwidth estimation algorithms
- Performance optimizations and benchmarking
- Additional RTCP features

---

## Links

- **GitHub**: https://github.com/webrtc-rs/rtc
- **Crate**: https://crates.io/crates/rtc
- **Docs**: https://docs.rs/rtc
- **Discord**: https://discord.gg/4Ju8UHdXMs
- **Examples**: https://github.com/webrtc-rs/rtc/tree/master/examples
- **Stats Collector Design**: https://webrtc.rs/blog/2026/01/17/stats-collector-design-sansio.html

---

## Full Changelog

### Added
- ✨ **WebRTC Stats API** - Complete W3C-compliant stats collection system
- ✨ `RTCPeerConnection::get_stats()` - Synchronous stats snapshot API with `StatsSelector`
- ✨ `StatsSelector` enum - Filter stats by sender, receiver, or all (W3C stats selection algorithm)
- ✨ `RTCStatsReport` - Type-safe stats report with iterator helpers
- ✨ `RTCStatsAccumulator` - Centralized stats accumulation system
- ✨ 15+ stats types covering ICE, Transport, RTP, Codecs, Data Channels
- ✨ **stats example** - Comprehensive stats monitoring demonstration
- ✨ Integration tests for stats API with browser interop
- ✨ Multiple new integration tests: mdns, rtcp-processing, trickle-ice, ice-tcp, media-rejection
- ✨ W3C stats selection algorithm implementation
- ✨ play-from-disk-playlist-control example
- ✨ save-to-disk-av1 example
- ✨ data-channels-simple example
- ✨ rtcp-processing example
- ✨ ICE TCP active mode support
- ✨ trickle-ice example
- ✨ ice-tcp example

### Changed
- 🔄 Refactored stats accumulator architecture for clarity
- 🔄 Improved stats ID management and naming
- 🔄 Enhanced RTP stream stats structure
- 🔄 Better codec stats accumulator implementation
- 🔄 Improved ICE candidate pair accumulator

### Fixed
- 🐛 Fixed stats transport missing fields
- 🐛 Fixed noop interceptor bug for `handle_poll_read` (#28)
- 🐛 Fixed simulcast bidirectional issue where both peers send over single m= line (#20)
- 🐛 Fixed integration test for single media session verification (#11)
- 🐛 Fixed SDP generation bug for rejected media sections with mid

### Improved
- 📚 Comprehensive inline documentation for stats API
- 📚 Updated README with stats example
- 🏗️ Cleaner accumulator architecture
- 🧪 Enhanced unit tests for stats accumulators
- 🧪 Better integration test coverage with browser interop
- 🔧 Accept unknown bandwidth types in SDP parser per RFC 8866 (#29)

---

## Commits

This release includes **50 commits** focused on stats implementation, testing, and quality improvements:

- Complete W3C WebRTC Stats API implementation
- 15+ accumulator types for different stats categories
- Comprehensive unit and integration tests
- Multiple new examples demonstrating various features
- Bug fixes and stability improvements
- Enhanced documentation and design documentation

**Detailed breakdown:**
- 152 files changed
- 22,859 insertions
- 2,504 deletions

---

## Relationship with `webrtc` Crate

As stated in previous announcements, `rtc` (sans-I/O) and `webrtc` (async) are **complementary**:

- **Use `webrtc`** for quick start with Tokio and async/await
- **Use `rtc`** for runtime independence, custom I/O, or maximum control

Both crates are actively maintained and share protocol implementations where possible. The stats implementation demonstrates the advantages of sans-I/O for:
- Zero-cost incremental accumulation
- Instant synchronous snapshots
- Deterministic testing with explicit timestamps
- Complete application control over collection timing

---

*Thanks to everyone who contributed feedback, bug reports, and feature requests! The WebRTC Stats implementation represents a significant milestone in providing production-ready monitoring capabilities while maintaining sansio's zero-overhead design principles. Special thanks to the WebRTC-rs community for their continued support.* 🦀

Feedback and contributions welcome on [GitHub](https://github.com/webrtc-rs/rtc)!
