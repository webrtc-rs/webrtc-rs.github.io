# RTC Feature Complete: What's Next for Sans-I/O WebRTC

## Introduction

With the release of **rtc 0.8.0**, the sans-I/O WebRTC implementation has reached a significant milestone: **full feature parity** with the async-based `webrtc` crate and **comprehensive W3C WebRTC API compliance**. This article reflects on what we've achieved and outlines the roadmap for what comes next.

---

## Achievement: Feature Parity with Async WebRTC

The `rtc` crate now provides all the functionality of the `webrtc` crate, reimagined with sans-I/O principles. Here's a summary of the complete feature set:

### Protocol Stack

| Layer | Feature | Status |
|-------|---------|--------|
| **ICE** | Host, SRFLX, Relay candidates | ✅ Complete |
| **ICE** | Trickle ICE | ✅ Complete |
| **ICE** | ICE-TCP (passive & active) | ✅ Complete |
| **ICE** | mDNS for privacy | ✅ Complete |
| **DTLS** | DTLS 1.2 with certificate fingerprints | ✅ Complete |
| **SRTP** | AES-CM, AES-GCM cipher suites | ✅ Complete |
| **SCTP** | Reliable & unreliable data channels | ✅ Complete |
| **RTP** | Header extensions, payload types | ✅ Complete |
| **RTCP** | SR, RR, NACK, PLI, FIR, TWCC | ✅ Complete |

### Peer Connection API

| Feature | Status |
|---------|--------|
| `createOffer()` / `createAnswer()` | ✅ Complete |
| `setLocalDescription()` / `setRemoteDescription()` | ✅ Complete |
| `addIceCandidate()` (local & remote) | ✅ Complete |
| `addTrack()` / `removeTrack()` | ✅ Complete |
| `createDataChannel()` | ✅ Complete |
| `getStats()` with `StatsSelector` | ✅ Complete |
| `getSenders()` / `getReceivers()` | ✅ Complete |
| `getTransceivers()` | ✅ Complete |
| Connection state events | ✅ Complete |
| ICE state events | ✅ Complete |
| Data channel events | ✅ Complete |
| Track events | ✅ Complete |

### Interceptor Framework

| Interceptor | Purpose | Status |
|-------------|---------|--------|
| **NACK Generator** | Request retransmission of lost packets | ✅ Complete |
| **NACK Responder** | Respond to NACK with cached packets | ✅ Complete |
| **Sender Report** | Generate RTCP SR for senders | ✅ Complete |
| **Receiver Report** | Generate RTCP RR for receivers | ✅ Complete |
| **TWCC Sender** | Add transport-wide sequence numbers | ✅ Complete |
| **TWCC Receiver** | Generate TWCC feedback | ✅ Complete |
| **Simulcast** | RID/MID header extensions | ✅ Complete |

### WebRTC Stats API

| Stats Type | Coverage |
|------------|----------|
| RTCPeerConnectionStats | 100% |
| RTCTransportStats | 100% |
| RTCIceCandidateStats | 100% |
| RTCIceCandidatePairStats | 89% |
| RTCCertificateStats | 100% |
| RTCCodecStats | 100% |
| RTCDataChannelStats | 100% |
| RTCInboundRtpStreamStats | 60%* |
| RTCOutboundRtpStreamStats | 67%* |
| RTCRemoteInboundRtpStreamStats | 83% |
| RTCRemoteOutboundRtpStreamStats | 83% |

*Media encoding/decoding stats require application-provided data (roadmap item)

---

## Achievement: W3C WebRTC Specification Compliance

The `rtc` crate follows the W3C WebRTC specification closely:

### Specification Conformance

- **[WebRTC 1.0](https://www.w3.org/TR/webrtc/)** — Peer connection lifecycle, SDP negotiation, ICE handling
- **[WebRTC Stats](https://www.w3.org/TR/webrtc-stats/)** — Statistics identifiers and types
- **[JSEP](https://datatracker.ietf.org/doc/html/rfc8829)** — JavaScript Session Establishment Protocol
- **[ICE](https://datatracker.ietf.org/doc/html/rfc8445)** — Interactive Connectivity Establishment
- **[Trickle ICE](https://datatracker.ietf.org/doc/html/rfc8838)** — Incremental ICE candidate exchange
- **[mDNS](https://datatracker.ietf.org/doc/html/rfc6762)** — Multicast DNS for ICE candidates
- **[DTLS 1.2](https://datatracker.ietf.org/doc/html/rfc6347)** — Datagram Transport Layer Security
- **[SRTP](https://datatracker.ietf.org/doc/html/rfc3711)** — Secure Real-time Transport Protocol
- **[SCTP over DTLS](https://datatracker.ietf.org/doc/html/rfc8261)** — Data channel transport

---

## Achievement: Sans-I/O Architecture Benefits

By building WebRTC with sans-I/O principles, we've achieved:

### Runtime Independence

```rust
// Works with any async runtime or no runtime at all
let mut pc = RTCPeerConnection::new(config)?;

// You control the I/O loop
loop {
    // Poll for outgoing data
    while let Some(msg) = pc.poll_write() {
        socket.send_to(&msg.message, msg.transport.peer_addr)?;
    }

    // Handle incoming data
    let (n, peer_addr) = socket.recv_from(&mut buf)?;
    pc.handle_read(TaggedBytesMut { ... })?;

    // Handle timeouts
    pc.handle_timeout(Instant::now())?;
}
```

### Deterministic Testing

```rust
#[test]
fn test_with_controlled_time() {
    let fixed_time = Instant::now();

    // All operations use explicit timestamps
    pc.handle_read(packet, fixed_time)?;
    pc.handle_timeout(fixed_time)?;

    let stats = pc.get_stats(fixed_time, StatsSelector::None);
    assert_eq!(stats.packets_received, expected);
}
```

### Zero Hidden I/O

- No background tasks or threads
- No hidden network operations
- No implicit timers
- Complete application control

---

## What's Next: The Roadmap

With feature parity achieved, the focus shifts to four key areas: **browser interoperability**, **performance**, **test coverage**, and **code quality**.

---

## Focus 1: Browser Interoperability

Ensuring seamless interoperability with all major browsers is critical for real-world deployments. While the core protocol implementation is complete, comprehensive browser testing and compatibility verification is ongoing.

### Target Browsers

| Browser | Platform | Status | Priority |
|---------|----------|--------|----------|
| **Chrome/Chromium** | Windows, macOS, Linux | 🔄 In Progress | High |
| **Firefox** | Windows, macOS, Linux | 🔄 In Progress | High |
| **Safari** | macOS, iOS | 📋 Planned | High |
| **Edge** | Windows | 📋 Planned | Medium |
| **Mobile Chrome** | Android | 📋 Planned | Medium |
| **Mobile Safari** | iOS | 📋 Planned | Medium |

### Interoperability Test Scenarios

**Data Channels:**
- [ ] Reliable ordered channels
- [ ] Unreliable unordered channels
- [ ] Multiple concurrent channels
- [ ] Large message fragmentation
- [ ] Binary and text messages

**Media:**
- [ ] Audio-only calls (Opus codec)
- [ ] Video-only streams (VP8, VP9, H.264)
- [ ] Audio + Video combined
- [ ] Simulcast with layer switching
- [ ] Screen sharing

**ICE & Connectivity:**
- [ ] Direct host-to-host connection
- [ ] STUN-assisted connectivity
- [ ] TURN relay fallback
- [ ] Trickle ICE candidate exchange
- [ ] ICE restart mid-session
- [ ] mDNS candidate handling

**SDP Negotiation:**
- [ ] Offer/Answer exchange
- [ ] Renegotiation (add/remove tracks)
- [ ] Codec negotiation
- [ ] Extension negotiation
- [ ] Rejected media sections

### Browser-Specific Quirks

Each browser has its own WebRTC implementation quirks that need to be handled:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     Browser Compatibility Matrix                            │
├─────────────────────────────────────────────────────────────────────────────┤
│  Issue                          │ Chrome │ Firefox │ Safari │ Edge          │
├─────────────────────────────────┼────────┼─────────┼────────┼───────────────┤
│  SDP format variations          │   ✓    │    ✓    │   ⚠    │    ✓          │
│  ICE candidate formatting       │   ✓    │    ✓    │   ⚠    │    ✓          │
│  DTLS role handling             │   ✓    │    ✓    │   ⚠    │    ✓          │
│  Data channel negotiation       │   ✓    │    ✓    │   ✓    │    ✓          │
│  Simulcast configuration        │   ✓    │    ⚠    │   ⚠    │    ✓          │
│  TWCC support                   │   ✓    │    ✓    │   ⚠    │    ✓          │
└─────────────────────────────────┴────────┴─────────┴────────┴───────────────┘
  ✓ = Works as expected   ⚠ = Requires special handling   ✗ = Known issues
```

### Automated Browser Testing

**Planned infrastructure:**

1. **Selenium/Playwright tests** — Automated browser control for E2E testing
2. **WebDriver BiDi** — Modern browser automation protocol
3. **CI integration** — Run browser tests on every PR
4. **Cross-platform matrix** — Test on Windows, macOS, Linux

```yaml
# Example CI configuration (planned)
browser-interop:
  strategy:
    matrix:
      browser: [chrome, firefox, safari, edge]
      os: [ubuntu-latest, macos-latest, windows-latest]
  steps:
    - run: cargo build --release
    - run: ./run-browser-tests.sh ${{ matrix.browser }}
```

### Known Issues to Address

- Safari SDP parsing edge cases
- Firefox simulcast layer negotiation
- Mobile browser power management
- Browser-specific codec preferences
- ICE candidate timing differences

---

## Focus 2: Performance Engineering

Performance is not an afterthought—it's a core requirement for real-time communication. This focus area encompasses systematic benchmarking, profiling, and optimization across the entire stack.

### Benchmarking Infrastructure

Before optimizing, we need to measure. A comprehensive benchmarking infrastructure is essential.

**Planned benchmark suite:**

```rust
// Using criterion for statistical rigor
use criterion::{criterion_group, criterion_main, Criterion, Throughput};

fn bench_datachannel_throughput(c: &mut Criterion) {
    let mut group = c.benchmark_group("datachannel");

    for size in [64, 1024, 16384, 65536].iter() {
        group.throughput(Throughput::Bytes(*size as u64));
        group.bench_with_input(
            BenchmarkId::new("send", size),
            size,
            |b, &size| {
                b.iter(|| {
                    dc.send(&message[..size])
                });
            },
        );
    }
    group.finish();
}

fn bench_rtp_pipeline(c: &mut Criterion) {
    c.bench_function("rtp_parse", |b| {
        b.iter(|| RtpPacket::unmarshal(&packet_bytes))
    });

    c.bench_function("rtp_marshal", |b| {
        b.iter(|| packet.marshal_to(&mut buffer))
    });

    c.bench_function("srtp_encrypt", |b| {
        b.iter(|| context.encrypt_rtp(&mut packet))
    });

    c.bench_function("srtp_decrypt", |b| {
        b.iter(|| context.decrypt_rtp(&mut packet))
    });
}

criterion_group!(benches, bench_datachannel_throughput, bench_rtp_pipeline);
criterion_main!(benches);
```

**Benchmark categories:**

| Category | Metrics | Tools |
|----------|---------|-------|
| Throughput | Messages/sec, Bytes/sec | criterion, custom |
| Latency | p50, p99, p999 | criterion, hdr_histogram |
| Memory | Allocations, peak usage | dhat, heaptrack |
| CPU | Cycles per operation | perf, flamegraph |

### Profiling and Analysis

**Profiling workflow:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Performance Analysis Workflow                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   1. Baseline         2. Profile           3. Analyze                       │
│   ┌─────────┐        ┌─────────┐          ┌─────────┐                       │
│   │ Run     │───────▶│ Collect │─────────▶│ Generate│                       │
│   │ Bench   │        │ Samples │          │ Reports │                       │
│   └─────────┘        └─────────┘          └─────────┘                       │
│        │                  │                    │                            │
│        ▼                  ▼                    ▼                            │
│   criterion          perf record          flamegraph                        │
│   results            + perf script        + hotspot analysis                │
│                                                                             │
│   4. Optimize         5. Validate          6. Document                      │
│   ┌─────────┐        ┌─────────┐          ┌─────────┐                       │
│   │ Apply   │───────▶│ Re-run  │─────────▶│ Record  │                       │
│   │ Changes │        │ Bench   │          │ Gains   │                       │
│   └─────────┘        └─────────┘          └─────────┘                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Profiling tools:**

- **perf** — Linux performance counters, CPU profiling
- **flamegraph** — Visualize hot code paths
- **heaptrack** — Memory allocation profiling
- **cargo-llvm-lines** — Generic code bloat analysis
- **valgrind/cachegrind** — Cache behavior analysis

### DataChannel Optimization

WebRTC DataChannels are increasingly used for high-throughput applications. Optimization targets:

**SCTP layer:**

| Optimization | Description | Expected Impact |
|--------------|-------------|-----------------|
| Chunk batching | Combine small messages into fewer SCTP chunks | Reduce overhead 20-40% |
| Zero-copy I/O | Avoid buffer copies in send/receive path | Reduce CPU usage |
| TSN tracking | Optimize sequence number management | Reduce memory allocations |
| Congestion control | Tune SCTP congestion parameters | Improve throughput stability |

**Application layer:**

- Message framing optimization
- Backpressure handling
- Buffer pool for allocations

**Performance targets:**

| Metric | Baseline | Target     | Notes |
|--------|----------|------------|-------|
| Throughput (reliable, ordered) | TBD | > 500 Mbps | Single channel |
| Throughput (unreliable) | TBD | > 1 Gbps   | Best-effort |
| Latency (1KB message) | TBD | < 1 ms     | p99 |
| Messages/second | TBD | > 100K     | Small messages |

### RTP/RTCP Pipeline Optimization

Media transport is latency-sensitive and high-volume.

**Packet processing:**

```
Incoming RTP Packet
        │
        ▼
┌───────────────┐
│ UDP Receive   │ ← Goal: zero-copy receive
└───────┬───────┘
        │
        ▼
┌───────────────┐
│ SRTP Decrypt  │ ← Goal: hardware AES-NI
└───────┬───────┘
        │
        ▼
┌───────────────┐
│ RTP Parse     │ ← Goal: minimal validation
└───────┬───────┘
        │
        ▼
┌───────────────┐
│ Interceptors  │ ← Goal: inline, no allocations
└───────┬───────┘
        │
        ▼
┌───────────────┐
│ Jitter Buffer │ ← Goal: lock-free, pre-allocated
└───────┬───────┘
        │
        ▼
    Application
```

**Specific optimizations:**

- **SIMD parsing** — Use SIMD instructions for header parsing where beneficial
- **AES-NI** — Ensure hardware acceleration for SRTP
- **Inline interceptors** — Compile-time interceptor composition (already implemented via generics)
- **Pre-allocated buffers** — Avoid per-packet allocations
- **Branch prediction** — Optimize common code paths

### ICE Performance

Connection establishment time directly impacts user experience.

**Optimization areas:**

| Phase | Current | Target | Approach |
|-------|---------|--------|----------|
| Candidate gathering | TBD | < 100ms | Parallel STUN queries |
| Connectivity checks | TBD | < 500ms | Prioritized pair testing |
| DTLS handshake | TBD | < 200ms | Session resumption |
| Total time-to-media | TBD | < 1s | Combined optimizations |

**Techniques:**

- Aggressive candidate nomination
- Parallel connectivity checks
- STUN response caching
- Optimized candidate pair sorting

### Memory Optimization

Real-time systems benefit from predictable memory behavior.

**Goals:**

- Minimize allocations in hot paths
- Use buffer pools for packet buffers
- Pre-allocate data structures where possible
- Reduce memory fragmentation

**Tracking:**

```rust
// Example: Using dhat for allocation profiling
#[global_allocator]
static ALLOC: dhat::Alloc = dhat::Alloc;

#[test]
fn test_allocations_in_hot_path() {
    let _profiler = dhat::Profiler::new_heap();

    // Run hot path code
    for _ in 0..10000 {
        process_rtp_packet(&packet);
    }

    // Analyze allocation count and sizes
}
```

### Continuous Performance Monitoring

**CI integration:**

- Run benchmarks on every PR
- Track performance regressions
- Publish benchmark results
- Alert on significant regressions

**Planned dashboard metrics:**

- Throughput trends over time
- Latency percentiles
- Memory usage patterns
- CPU efficiency

---

## Focus 3: Test Coverage

### Current State

The codebase has growing test coverage, but there's room for improvement:

| Category | Current | Target |
|----------|---------|--------|
| Unit tests | Partial | 80%+ line coverage |
| Integration tests | 28 test files | Comprehensive scenarios |
| Interop tests | Chrome, Firefox | All major browsers |
| Fuzz tests | Limited | Critical parsers |

### Unit Test Expansion

**Priority areas for unit testing:**

1. **SDP parsing and generation** — Complex edge cases in offer/answer
2. **ICE state machine** — All state transitions and error conditions
3. **DTLS handshake** — Certificate validation, cipher negotiation
4. **SCTP association** — Stream management, congestion control
5. **Interceptor logic** — NACK timing, RTCP report generation

**Example test structure:**

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_ice_state_transitions() {
        // Test all valid state transitions
        // Test invalid transition handling
        // Test event emission
    }

    #[test]
    fn test_nack_timing_accuracy() {
        // Verify NACK generation timing
        // Test RTT-based retransmit intervals
    }
}
```

### Integration Test Expansion

**Planned integration test scenarios:**

- [ ] Multi-party conferencing (3+ peers)
- [ ] Renegotiation (adding/removing tracks mid-session)
- [ ] Network condition simulation (packet loss, delay, reordering)
- [ ] Long-running sessions (stability over hours)
- [ ] ICE restart scenarios
- [ ] TURN relay failover
- [ ] Simulcast layer switching
- [ ] DataChannel flow control under load

### Fuzz Testing

Critical parsers should be fuzz-tested:

```rust
// Using cargo-fuzz or libfuzzer
fuzz_target!(|data: &[u8]| {
    let _ = RtpPacket::unmarshal(data);
});

fuzz_target!(|data: &[u8]| {
    let _ = StunMessage::unmarshal(data);
});
```

**Priority fuzz targets:**

- RTP/RTCP packet parsing
- STUN message parsing
- SDP parsing
- SCTP chunk parsing
- DTLS record parsing

---

## Focus 4: Code Quality and Tech Debt

### TODO/FIXME Cleanup

The codebase currently contains **104 TODO/FIXME comments** that need to be addressed:

```bash
$ grep -r "TODO\|FIXME" --include="*.rs" | wc -l
104
```

**Categorization plan:**

| Category | Action |
|----------|--------|
| Missing features | Implement or document as "won't fix" |
| Performance notes | Create benchmark, then optimize |
| Error handling | Improve error messages and recovery |
| Code cleanup | Refactor or remove dead code |
| Documentation | Add missing docs |

**Tracking approach:**

1. Create GitHub issues for each significant TODO
2. Prioritize by impact (user-facing vs internal)
3. Address in dedicated cleanup sprints
4. Add CI check to prevent new untracked TODOs

### Documentation Improvements

- [ ] Complete rustdoc coverage for public APIs
- [ ] Add architecture decision records (ADRs)
- [ ] Improve inline code comments for complex algorithms
- [ ] Create troubleshooting guide
- [ ] Add performance tuning guide

### API Refinements

Some APIs may benefit from refinement based on user feedback:

- Error types consolidation
- Builder pattern consistency
- Event handling ergonomics
- Configuration validation

---

## Community Contribution Opportunities

We welcome contributions in these areas:

### Good First Issues

- Documentation improvements
- Adding missing unit tests
- Resolving simple TODO comments
- Example improvements

### Intermediate

- Integration test scenarios
- Benchmark implementations
- Bug fixes with clear reproduction

### Advanced

- Performance optimizations
- New interceptor implementations
- Protocol extensions

See [CONTRIBUTING.md](https://github.com/webrtc-rs/rtc/blob/master/CONTRIBUTING.md) for guidelines.

---

## Conclusion

The `rtc` crate has achieved its primary goal: a complete, W3C-compliant WebRTC implementation using sans-I/O principles. With feature parity established, the focus now shifts to making it **faster**, **more reliable**, and **easier to use**.

The sans-I/O architecture provides a solid foundation for these improvements. By separating protocol logic from I/O, we can:

- Benchmark and optimize without network variability
- Test deterministically with controlled time
- Profile precisely with no background noise

We're excited about the road ahead and welcome community participation in shaping the future of Rust WebRTC.

---

## Get Involved

- **GitHub**: https://github.com/webrtc-rs/rtc
- **Discord**: https://discord.gg/4Ju8UHdXMs
- **Crates.io**: https://crates.io/crates/rtc
- **Documentation**: https://docs.rs/rtc

Have ideas for performance improvements or want to contribute tests? Open an issue or join the discussion on Discord!

---

## References

- [W3C WebRTC 1.0 Specification](https://www.w3.org/TR/webrtc/)
- [W3C WebRTC Statistics API](https://www.w3.org/TR/webrtc-stats/)
- [Building WebRTC's Pipeline with sansio::Protocol](/blog/2026/01/04/building-webrtc-pipeline-with-sansio.html)
- [Interceptor Design Principle](/blog/2026/01/09/interceptor-design-principle-sansio.html)
- [Stats Collector Design](/blog/2026/01/17/stats-collector-design-sansio.html)
