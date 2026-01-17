# Stats Collector Design: An Incremental Accumulation Approach

## Introduction

WebRTC statistics are essential for monitoring connection health, debugging issues, and implementing adaptive quality control. The W3C WebRTC specification defines a comprehensive [Statistics API](https://www.w3.org/TR/webrtc-stats/) that exposes metrics ranging from ICE connectivity to RTP stream quality. However, collecting these statistics in a sans-I/O implementation presents unique design challenges.

In previous articles, we explored how WebRTC can be modeled as a [pure protocol pipeline](/blog/2026/01/04/building-webrtc-pipeline-with-sansio.html) and how [interceptors](/blog/2026/01/09/interceptor-design-principle-sansio.html) process RTP/RTCP packets using the `sansio::Protocol` trait. This article examines how RTC collects W3C-compliant statistics without async runtimes, background tasks, or locks—using an **incremental accumulation** pattern that aligns naturally with the polling-based design.

### Prerequisites

This article builds on concepts introduced in:

- **[Building WebRTC's Pipeline with `sansio::Protocol`](/blog/2026/01/04/building-webrtc-pipeline-with-sansio.html)** — Covers the `sansio::Protocol` trait, the polling model (`handle_*`/`poll_*` methods), and how WebRTC's protocol layers are composed as a pipeline.
- **[Interceptor Design Principle](/blog/2026/01/09/interceptor-design-principle-sansio.html)** — Explains how RTP/RTCP processing interceptors implement `sansio::Protocol`.

Familiarity with the W3C WebRTC Statistics API is helpful but not required.

---

## The Challenge: Stats Collection in Sans-I/O

Traditional WebRTC implementations collect statistics using one of two approaches:

### Approach 1: On-Demand Fetching (Pion/Go)

```go
func (pc *PeerConnection) GetStats() StatsReport {
    var wg sync.WaitGroup
    var statsCollector StatsCollector

    wg.Add(len(pc.transceivers))
    for _, t := range pc.transceivers {
        go func(t *RTPTransceiver) {
            defer wg.Done()
            t.collectStats(&statsCollector)
        }(t)
    }
    wg.Wait()
    return statsCollector.report()
}
```

This approach spawns goroutines to fetch stats from various components, requiring synchronization with WaitGroups and mutexes.

### Approach 2: Async Fetching (async webrtc)

```rust
pub async fn get_stats(&self) -> RTCStatsReport {
    let mut stats = vec![];

    // Parallel async collection
    let (ice_stats, dtls_stats, sctp_stats) = tokio::join!(
        self.ice_transport.get_stats(),
        self.dtls_transport.get_stats(),
        self.sctp_transport.get_stats(),
    );

    stats.extend(ice_stats);
    stats.extend(dtls_stats);
    stats.extend(sctp_stats);

    RTCStatsReport::new(stats)
}
```

This approach uses async/await with `tokio::join!` for parallel collection, requiring an async runtime.

### The Problem

Both approaches share a fundamental issue: they perform I/O or synchronization during `getStats()`. This conflicts with the sans-I/O principle where protocol logic should be:

- **Deterministic** — Given the same inputs, produce the same outputs
- **Non-blocking** — Never wait for locks or I/O
- **Testable** — No runtime dependencies

How can we collect statistics without violating these principles?

---

## The Solution: Incremental Accumulation

The answer is to **invert the data flow**. Instead of fetching stats when requested, we accumulate them incrementally as events occur, then take a snapshot when `get_stats()` is called.

### Core Principle

```
Traditional:    get_stats() → fetch from components → synchronize → return
Sans-I/O:       events occur → accumulate incrementally → get_stats() returns snapshot
```

This pattern has several key properties:

1. **Zero-cost collection** — Stats are updated as a side effect of normal packet processing
2. **Always up-to-date** — Counters reflect the latest state without stale data
3. **Instant snapshots** — `get_stats()` is a cheap, synchronous operation
4. **No synchronization** — Single-threaded design eliminates locks

### Comparison Table

| Aspect          | Pion (Go)              | async webrtc          | sansio rtc                           |
|-----------------|------------------------|-----------------------|--------------------------------------|
| Collection      | WaitGroup + goroutines | tokio::join! async    | Synchronous accumulation             |
| Timing          | On-demand fetch        | On-demand async fetch | Continuous accumulation + snapshot   |
| I/O             | Direct network access  | Async network         | No I/O, application-driven           |
| Threading       | Multi-threaded         | Async tasks           | Single-threaded, event-loop friendly |
| Synchronization | Mutex + WaitGroup      | Mutex + async         | None needed                          |

---

## Architecture Overview

The statistics system is organized into three layers:

```
┌────────────────────────────────────────────────────────────────────────────┐
│                          RTCPeerConnection                                 │
│  ┌────────────────────────────────────────────────────────────────────────┐│
│  │                        RTCStatsAccumulator                             ││
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐   ││
│  │  │ ICE Stats    │ │ Transport    │ │ RTP Stream   │ │ DataChannel  │   ││
│  │  │ Accumulators │ │ Accumulator  │ │ Accumulators │ │ Accumulators │   ││
│  │  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘   ││
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐   ││
│  │  │ Codec        │ │ Certificate  │ │ PeerConn     │ │ MediaSource  │   ││
│  │  │ Accumulators │ │ Accumulators │ │ Accumulator  │ │ Accumulators │   ││
│  │  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘   ││
│  └────────────────────────────────────────────────────────────────────────┘│
│                                                                            │
│  pub fn get_stats(&mut self, now: Instant, selector: StatsSelector)        │
│      -> RTCStatsReport                                                     │
│      └─> Collects snapshots from all accumulators, builds report           │
└────────────────────────────────────────────────────────────────────────────┘
```

### Layer 1: Accumulators

Each stats category has a dedicated accumulator struct that tracks counters and state:

```rust
/// Accumulated statistics for an inbound RTP stream
#[derive(Debug, Default)]
pub struct InboundRtpStreamAccumulator {
    pub ssrc: SSRC,
    pub kind: RtpCodecKind,
    pub transport_id: String,

    // Packet counters - incremented per packet
    pub packets_received: u64,
    pub bytes_received: u64,
    pub header_bytes_received: u64,
    pub last_packet_received_timestamp: Option<Instant>,

    // Quality metrics - updated from RTCP feedback
    pub packets_lost: i64,
    pub jitter: f64,

    // RTCP feedback sent
    pub nack_count: u32,
    pub fir_count: u32,
    pub pli_count: u32,

    // ... additional fields
}
```

Accumulators provide event-driven update methods:

```rust
impl InboundRtpStreamAccumulator {
    pub fn on_rtp_received(&mut self, payload_bytes: usize, header_bytes: usize, now: Instant) {
        self.packets_received += 1;
        self.bytes_received += payload_bytes as u64;
        self.header_bytes_received += header_bytes as u64;
        self.last_packet_received_timestamp = Some(now);
    }

    pub fn on_nack_sent(&mut self) {
        self.nack_count += 1;
    }

    pub fn on_pli_sent(&mut self) {
        self.pli_count += 1;
    }
}
```

### Layer 2: Master Accumulator

The `RTCStatsAccumulator` aggregates all category-specific accumulators:

```rust
/// Master statistics accumulator for a peer connection.
#[derive(Debug, Default)]
pub(crate) struct RTCStatsAccumulator {
    /// Peer connection level stats
    pub(crate) peer_connection: PeerConnectionStatsAccumulator,

    /// Transport stats
    pub(crate) transport: TransportStatsAccumulator,

    /// ICE candidate pairs keyed by pair ID
    pub(crate) ice_candidate_pairs: HashMap<String, IceCandidatePairAccumulator>,

    /// Inbound RTP stream accumulators keyed by SSRC
    pub(crate) inbound_rtp_streams: HashMap<SSRC, InboundRtpStreamAccumulator>,

    /// Outbound RTP stream accumulators keyed by SSRC
    pub(crate) outbound_rtp_streams: HashMap<SSRC, OutboundRtpStreamAccumulator>,

    /// Data channel stats keyed by channel ID
    pub(crate) data_channels: HashMap<RTCDataChannelId, DataChannelStatsAccumulator>,

    // ... codecs, certificates, media sources
}
```

### Layer 3: Stats Report

The `snapshot()` method produces an immutable `RTCStatsReport`:

```rust
impl RTCStatsAccumulator {
    /// Creates a snapshot of all accumulated stats at the given timestamp.
    pub(crate) fn snapshot(&self, now: Instant) -> RTCStatsReport {
        let mut entries = Vec::new();

        // Peer connection stats
        entries.push(RTCStatsReportEntry::PeerConnection(
            self.peer_connection.snapshot(now),
        ));

        // Transport stats
        entries.push(RTCStatsReportEntry::Transport(self.transport.snapshot(now)));

        // Inbound RTP streams
        for (ssrc, stream) in &self.inbound_rtp_streams {
            let id = format!("RTCInboundRTPStream_{}_{}", stream.kind, ssrc);
            entries.push(RTCStatsReportEntry::InboundRtp(stream.snapshot(now, &id)));
        }

        // ... additional stat types

        RTCStatsReport::new(entries)
    }
}
```

---

## Handler Integration

The key insight is that statistics are collected **as packets flow through the handler pipeline**, not as a separate operation.

### Data Flow

```
handle_read(packet)
       │
       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            Handler Pipeline                                 │
│                                                                             │
│   ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐   │
│   │Demuxer  │───▶│  ICE    │───▶│  DTLS   │───▶│  SCTP   │───▶│DataChan │   │
│   │Handler  │    │Handler  │    │Handler  │    │Handler  │    │Handler  │   │
│   └────┬────┘    └────┬────┘    └────┬────┘    └────┬────┘    └────┬────┘   │
│        │              │              │              │              │        │
│        ▼              ▼              ▼              ▼              ▼        │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                      RTCStatsAccumulator                            │   │
│   │  Updates stats as packets flow through the pipeline                 │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│   ┌─────────┐    ┌─────────┐    ┌─────────┐                                 │
│   │  SRTP   │───▶│Intercep │───▶│Endpoint │                                 │
│   │Handler  │    │Handler  │    │Handler  │                                 │
│   └────┬────┘    └────┬────┘    └────┬────┘                                 │
│        │              │              │                                      │
│        ▼              ▼              ▼                                      │
│   Update SRTP    Update RTP      Update Track                               │
│   Stats          Stream Stats    Stats                                      │
└─────────────────────────────────────────────────────────────────────────────┘
       │
       ▼
poll_read() -> RTCMessage

get_stats(now, selector) -> RTCStatsReport  (instant snapshot, no fetching)
```

### Handler Stats Collection Points

Each handler updates relevant statistics during its normal processing:

| Handler         | Stats Updated                        | Trigger                               |
|-----------------|--------------------------------------|---------------------------------------|
| **Demuxer**     | Transport (packets, bytes)           | handle_read/handle_write              |
| **ICE**         | Candidate pair (packets, bytes, RTT) | handle_read/handle_write, STUN events |
| **DTLS**        | Transport (DTLS state, cipher)       | Handshake completion                  |
| **DTLS**        | Certificates (fingerprint, DER)      | Handshake completion                  |
| **Interceptor** | RTP stream (packets, bytes, RTCP)    | handle_read/handle_write              |
| **DataChannel** | Data channel (messages, bytes)       | handle_read/handle_write              |
| **Endpoint**    | Track references                     | Track events                          |

### Example: Interceptor Handler

The interceptor handler updates RTP stream statistics on every packet:

```rust
impl<'a, I: Interceptor> InterceptorHandler<'a, I> {
    fn handle_read(&mut self, msg: TaggedRTCMessageInternal) -> Result<()> {
        if let RTCMessageInternal::Rtp(RTPMessage::Packet(Packet::Rtp(rtp_packet))) = &msg.message {
            let ssrc = rtp_packet.header.ssrc;

            // Update inbound RTP stats
            if let Some(stream) = self.stats.inbound_rtp_streams.get_mut(&ssrc) {
                stream.on_rtp_received(
                    rtp_packet.payload.len(),
                    rtp_packet.header.marshal_size(),
                    msg.now,
                );
            }
        }

        // Continue processing...
        Ok(())
    }
}
```

RTCP feedback is also tracked:

```rust
fn process_read_rtcp_for_stats(&mut self, rtcp_packets: &[RtcpPacket]) {
    for packet in rtcp_packets {
        match packet {
            RtcpPacket::SenderReport(sr) => {
                // Update remote sender stats for inbound streams
                if let Some(stream) = self.stats.inbound_rtp_streams.get_mut(&sr.ssrc) {
                    stream.on_rtcp_sr_received(
                        sr.packet_count as u64,
                        sr.octet_count as u64,
                        msg.now,
                    );
                }
            }
            RtcpPacket::ReceiverReport(rr) => {
                // Update remote receiver stats for outbound streams
                for report in &rr.reports {
                    if let Some(stream) = self.stats.outbound_rtp_streams.get_mut(&report.ssrc) {
                        stream.on_rtcp_rr_received(
                            report.last_sequence_number as u64,
                            report.total_lost as u64,
                            report.jitter as f64,
                            report.fraction_lost as f64 / 256.0,
                            0.0, // RTT calculated separately
                        );
                    }
                }
            }
            // Handle NACK, PLI, FIR, etc.
            _ => {}
        }
    }
}
```

---

## Explicit Timestamp and Selector Parameters

A subtle but important design choice: `get_stats()` takes an explicit timestamp and selector rather than calling `Instant::now()` internally or always returning all stats.

```rust
impl<I: Interceptor> RTCPeerConnection<I> {
    /// Returns a snapshot of WebRTC statistics.
    ///
    /// # Arguments
    /// * `now` - The timestamp for the snapshot
    /// * `selector` - Controls which statistics are included
    pub fn get_stats(&mut self, now: Instant, selector: StatsSelector) -> RTCStatsReport {
        self.pipeline_context.stats.snapshot_with_selector(now, selector)
    }
}
```

This design choice:

- **Enables deterministic testing** — Tests can provide fixed timestamps for reproducible results
- **Follows sans-I/O principles** — No hidden I/O (getting current time is I/O)
- **Allows batch snapshots** — Multiple calls with the same timestamp produce consistent reports
- **Supports W3C selection algorithm** — Filter stats to a specific sender or receiver

---

## Application-Provided Stats (Roadmap)

Some statistics cannot be collected at the protocol layer because they depend on media encoding/decoding, which is handled by the application. These include:

- **Decoder stats**: frames decoded, key frames, decode time, decoder implementation
- **Encoder stats**: frames encoded, key frames, encode time, encoder implementation
- **Audio source stats**: audio level, echo cancellation metrics
- **Video source stats**: frame dimensions, frame rate
- **Audio playout stats**: playout delay, synthesized samples

### Design Considerations

The sans-I/O architecture creates a clear boundary: the library handles **protocol**, the application handles **I/O and media processing**. This means the library cannot directly observe encoder/decoder behavior.

A future API could allow applications to report these stats:

```rust
// Potential future API (not yet implemented)

/// Decoder statistics provided by the application
#[derive(Debug, Clone, Default)]
pub struct DecoderStatsUpdate {
    pub frames_decoded: u32,
    pub key_frames_decoded: u32,
    pub frames_rendered: u32,
    pub frame_width: u32,
    pub frame_height: u32,
    pub total_decode_time: f64,
    pub decoder_implementation: String,
}

/// Encoder statistics provided by the application
#[derive(Debug, Clone, Default)]
pub struct EncoderStatsUpdate {
    pub frames_encoded: u32,
    pub key_frames_encoded: u32,
    pub frame_width: u32,
    pub frame_height: u32,
    pub total_encode_time: f64,
    pub encoder_implementation: String,
}
```

The application would report these via a dedicated API:

```rust
// Potential future API (not yet implemented)

// Report decoder stats for an inbound video stream
pc.update_decoder_stats(ssrc, DecoderStatsUpdate {
    frames_decoded: 1000,
    key_frames_decoded: 50,
    frame_width: 1920,
    frame_height: 1080,
    ..Default::default()
});

// Stats are then included in the next get_stats() call
let stats = pc.get_stats(Instant::now(), StatsSelector::None);
```

This design is under consideration and may be refined based on real-world usage patterns. The key principle remains: protocol-level stats are collected automatically, while media-level stats require explicit application input.

---

## Stats Selector: W3C Selection Algorithm

The W3C WebRTC specification defines a [stats selection algorithm](https://www.w3.org/TR/webrtc/#the-stats-selection-algorithm) that allows filtering statistics to a specific sender or receiver. This is useful when you only need stats for a particular media track rather than the entire connection.

### The Problem with Unfiltered Stats

Consider a video conferencing application with multiple participants. Each peer connection might have:
- 3 outbound video streams (simulcast layers)
- 1 outbound audio stream
- N inbound video streams (one per participant)
- N inbound audio streams

Calling `get_stats()` with no filter returns stats for **all** streams, which can be overwhelming when debugging a specific stream. The W3C selection algorithm solves this by filtering to only the relevant stats.

### StatsSelector Enum

RTC implements the selection algorithm through the `StatsSelector` enum:

```rust
pub enum StatsSelector {
    /// Gather stats for the whole connection.
    ///
    /// Returns all available statistics objects including peer connection,
    /// transport, ICE candidates, codecs, data channels, and all RTP streams.
    None,

    /// Gather stats for a specific RTP sender.
    ///
    /// Returns:
    /// - All `RTCOutboundRtpStreamStats` for streams being sent by this sender
    /// - All stats objects referenced by those outbound streams (transport,
    ///   codec, remote inbound stats, etc.)
    Sender(RTCRtpSenderId),

    /// Gather stats for a specific RTP receiver.
    ///
    /// Returns:
    /// - All `RTCInboundRtpStreamStats` for streams being received by this receiver
    /// - All stats objects referenced by those inbound streams (transport,
    ///   codec, remote outbound stats, etc.)
    Receiver(RTCRtpReceiverId),
}
```

### Selection Algorithm Implementation

When a selector is provided, the algorithm returns:

1. **Primary stats** — The RTP stream stats for the selected sender/receiver
2. **Referenced stats** — All stats objects that the primary stats reference

For a **Sender** selection:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    StatsSelector::Sender(sender_id)                         │
├─────────────────────────────────────────────────────────────────────────────┤
│  Primary Stats:                                                             │
│    • RTCOutboundRtpStreamStats (for each SSRC of this sender)               │
│    • RTCRemoteInboundRtpStreamStats (from RTCP Receiver Reports)            │
│                                                                             │
│  Referenced Stats:                                                          │
│    • RTCTransportStats (transport used by the stream)                       │
│    • RTCCodecStats (codec used by the stream)                               │
│    • RTCIceCandidatePairStats (current candidate pair)                      │
│    • RTCIceCandidateStats (local and remote candidates)                     │
│    • RTCCertificateStats (DTLS certificates)                                │
└─────────────────────────────────────────────────────────────────────────────┘
```

For a **Receiver** selection:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   StatsSelector::Receiver(receiver_id)                      │
├─────────────────────────────────────────────────────────────────────────────┤
│  Primary Stats:                                                             │
│    • RTCInboundRtpStreamStats (for each SSRC of this receiver)              │
│    • RTCRemoteOutboundRtpStreamStats (from RTCP Sender Reports)             │
│                                                                             │
│  Referenced Stats:                                                          │
│    • RTCTransportStats (transport used by the stream)                       │
│    • RTCCodecStats (codec used by the stream)                               │
│    • RTCIceCandidatePairStats (current candidate pair)                      │
│    • RTCIceCandidateStats (local and remote candidates)                     │
│    • RTCCertificateStats (DTLS certificates)                                │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Usage Examples

```rust
use rtc::statistics::StatsSelector;

// Get all stats for the entire connection
let all_stats = pc.get_stats(Instant::now(), StatsSelector::None);
println!("Total stats entries: {}", all_stats.len());

// Get stats for a specific sender (e.g., video sender)
let video_sender_id = pc.get_senders()
    .find(|s| s.track().map(|t| t.kind() == RtpCodecKind::Video).unwrap_or(false))
    .map(|s| s.id());

if let Some(sender_id) = video_sender_id {
    let sender_stats = pc.get_stats(Instant::now(), StatsSelector::Sender(sender_id));

    // Only contains outbound video stats and referenced objects
    for outbound in sender_stats.outbound_rtp_streams() {
        println!("Outbound SSRC {}: {} packets sent, {} bytes",
            outbound.ssrc,
            outbound.packets_sent,
            outbound.bytes_sent);
    }
}

// Get stats for a specific receiver (e.g., to debug incoming stream quality)
let receiver_id = /* ... */;
let receiver_stats = pc.get_stats(Instant::now(), StatsSelector::Receiver(receiver_id));

for inbound in receiver_stats.inbound_rtp_streams() {
    println!("Inbound SSRC {}: {} packets received, {} lost, jitter: {:.2}ms",
        inbound.ssrc,
        inbound.packets_received,
        inbound.packets_lost,
        inbound.jitter * 1000.0);
}
```

### Performance Benefit

The selector also provides a performance benefit: when you only need stats for one stream, filtering avoids the overhead of collecting and returning stats for all streams:

```rust
// Efficient: only collects stats for one sender
let focused = pc.get_stats(now, StatsSelector::Sender(sender_id));

// Less efficient for single-stream monitoring: collects everything
let all = pc.get_stats(now, StatsSelector::None);
let filtered: Vec<_> = all.outbound_rtp_streams()
    .filter(|s| s.sender_id == sender_id)
    .collect();
```

---

## Coverage Summary

The implementation covers a significant portion of the W3C WebRTC Stats API:

| Stats Type                      | Coverage    | Notes                                   |
|---------------------------------|-------------|-----------------------------------------|
| RTCCodecStats                   | 100%        | Registered on-demand per W3C spec       |
| RTCDataChannelStats             | 100%        | Messages, bytes, state                  |
| RTCIceCandidateStats            | 100%        | All candidate properties                |
| RTCIceCandidatePairStats        | 89%         | Bitrate estimation requires BWE         |
| RTCPeerConnectionStats          | 100%        | Data channels opened/closed             |
| RTCTransportStats               | 100%        | ICE, DTLS, SRTP state                   |
| RTCCertificateStats             | 100%        | Fingerprint, DER certificate            |
| RTCInboundRtpStreamStats        | 60%         | Decoder stats require app API (roadmap) |
| RTCOutboundRtpStreamStats       | 67%         | Encoder stats require app API (roadmap) |
| RTCRemoteInboundRtpStreamStats  | 83%         | From RTCP RR                            |
| RTCRemoteOutboundRtpStreamStats | 83%         | From RTCP SR                            |
| RTCMediaSourceStats             | Roadmap     | Requires app API for capture stats      |
| RTCAudioPlayoutStats            | Roadmap     | Requires app API for playout stats      |

---

## Benefits

### 1. Zero Runtime Dependencies

Stats collection requires no async runtime, no background tasks, and no locks. The entire system runs synchronously in the handler pipeline's event loop.

### 2. Predictable Performance

`get_stats()` is a cheap iteration over HashMap entries followed by struct copies. There's no network I/O, no lock contention, and no waiting.

### 3. Deterministic Testing

With explicit timestamps and no hidden I/O, tests can verify exact statistics values:

```rust
#[test]
fn test_inbound_rtp_stats() {
    let mut pc = create_test_peer_connection();
    let fixed_time = Instant::now();

    // Simulate receiving packets
    pc.handle_read(create_rtp_packet(ssrc: 12345, payload: 100 bytes), fixed_time);
    pc.handle_read(create_rtp_packet(ssrc: 12345, payload: 150 bytes), fixed_time);

    let stats = pc.get_stats(fixed_time, StatsSelector::None);
    let inbound = stats.inbound_rtp_streams().next().unwrap();

    assert_eq!(inbound.packets_received, 2);
    assert_eq!(inbound.bytes_received, 250);
}
```

### 4. Natural Integration

Stats collection happens as a side effect of normal packet processing. There's no separate "stats collection phase" that could interfere with real-time performance.

---

## Conclusion

The incremental accumulation pattern provides a clean solution for WebRTC statistics collection in a sans-I/O architecture. By updating counters as events occur and taking snapshots on demand, we achieve:

- **Compliance** with the W3C WebRTC Statistics API
- **Performance** through zero-cost incremental updates
- **Testability** through deterministic, timestamp-parameterized snapshots
- **Simplicity** by eliminating async coordination and locking

This approach demonstrates how the constraints of sans-I/O design can lead to simpler, more efficient implementations. The key insight is that many "on-demand" operations in traditional architectures can be inverted to "continuous accumulation + instant snapshot" patterns.

---

## References

- [W3C WebRTC 1.0: Real-Time Communication Between Browsers](https://www.w3.org/TR/webrtc/)
- [W3C Identifiers for WebRTC's Statistics API](https://www.w3.org/TR/webrtc-stats/)
- [RTC Statistics Implementation](https://github.com/webrtc-rs/rtc/tree/master/rtc/src/statistics)
- [Building WebRTC's Pipeline with `sansio::Protocol`](/blog/2026/01/04/building-webrtc-pipeline-with-sansio.html)
- [Interceptor Design Principle](/blog/2026/01/09/interceptor-design-principle-sansio.html)
