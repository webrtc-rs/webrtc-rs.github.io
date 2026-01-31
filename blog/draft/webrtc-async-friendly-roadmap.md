# Async-Friendly WebRTC Crate Roadmap

## Vision

Build a modern, async-friendly WebRTC API on top of the Sans-I/O `rtc` crate that provides:

- 🚀 **Runtime Agnostic** - First-class support for Tokio, async-std, smol, and embassy
- 🎯 **Clean API** - Ergonomic async/await interface with trait-based event handling
- 🔒 **Memory Safe** - No callback leaks, clear ownership semantics
- ⚡ **High Performance** - Zero-cost abstractions over the Sans-I/O core
- 🧪 **Well Tested** - Comprehensive test coverage and browser interoperability

---

## Current State (v0.17.x)

### Maintenance Phase
- **Status**: Feature freeze, bug fixes only
- **Branch**: `v0.17.x` 
- **Issues**: Memory leaks, Tokio coupling, callback-based API
- **Support**: Critical bug fixes through 2026

---

## Development Phases

### Phase 1: Foundation (Q1 2026) ✅ In Progress

**Goal**: Establish Sans-I/O core and experimental async wrappers

#### Completed
- [x] `rtc` crate reaches feature parity with `webrtc`
- [x] Comprehensive W3C WebRTC API compliance (95%+)
- [x] Sans-I/O protocol implementations (ICE, DTLS, SRTP, SCTP, RTP/RTCP)
- [x] Interceptor framework with sansio::Protocol
- [x] WebRTC Stats API implementation
- [x] mDNS support for privacy-preserving connections

#### In Progress
- [ ] Runtime trait abstraction design (inspired by Quinn)
- [ ] Basic async API design and prototyping
- [ ] Trait-based event handling design

**Deliverables:**
- `rtc` v0.8.x with complete feature set
- Design RFC for async-friendly API with runtime trait abstraction
- Proof-of-concept implementation with Tokio

---

### Phase 2: API Design & Runtime Adapters (Q2 2026)

**Goal**: Finalize async API design and create runtime adapters

#### Core API Design

**Trait-Based Event Handling**
```rust
#[async_trait]
pub trait PeerConnectionHandler: Send {
    async fn on_connection_state_change(&mut self, state: RTCPeerConnectionState) {}
    async fn on_ice_candidate(&mut self, candidate: Option<RTCIceCandidate>) {}
    async fn on_track(&mut self, track: Track) {}
    async fn on_data_channel(&mut self, channel: DataChannel) {}
    // ... other events
}
```

**Modern Async API**
```rust
// Create peer connection with handler
let pc = PeerConnection::builder()
    .with_handler(MyHandler::new())
    .with_ice_servers(ice_servers)
    .build()
    .await?;

// Async operations
let offer = pc.create_offer().await?;
pc.set_local_description(offer).await?;

// Add tracks
let track = pc.add_track(media_stream).await?;

// Data channels
let dc = pc.create_data_channel("label").await?;
dc.send("Hello".as_bytes()).await?;
```

#### Runtime Trait Abstraction (Quinn-style)

**Runtime Trait Design**
```rust
pub trait Runtime: Send + Sync + Debug + 'static {
    fn spawn(&self, future: Pin<Box<dyn Future<Output = ()> + Send>>);
    fn wrap_udp_socket(&self, sock: std::net::UdpSocket) -> io::Result<Box<dyn AsyncUdpSocket>>;
    fn new_timer(&self, instant: Instant) -> Pin<Box<dyn AsyncTimer>>;
    fn now(&self) -> Instant;
}
```

**Runtime Implementations (within single `webrtc` crate)**

- [ ] **Tokio Runtime** - `runtime::TokioRuntime`
  - I/O driver for Tokio UDP sockets
  - Tokio task spawning integration
  - Tokio timer integration
  - Feature flag: `runtime-tokio` (default)

- [ ] **async-std Runtime** - `runtime::AsyncStdRuntime`
  - I/O driver for async-std UDP sockets
  - async-std task spawning
  - Feature flag: `runtime-async-std`

- [ ] **smol Runtime** - `runtime::SmolRuntime`
  - I/O driver for smol UDP sockets
  - Lightweight runtime integration
  - Feature flag: `runtime-smol`

- [ ] **Embassy Runtime** - `runtime::EmbassyRuntime` (future)
  - no_std compatible I/O driver
  - Embassy executor integration
  - Feature flag: `runtime-embassy`

#### API Improvements

- [ ] **Builder pattern** for configuration (RTCConfigurationBuilder)
- [ ] **Result types** with comprehensive error information
- [ ] **Graceful shutdown** - proper resource cleanup
- [ ] **Backpressure** - flow control for data channels and media
- [ ] **Stats API** - async-friendly statistics collection

**Deliverables:**
- API design RFC (finalized)
- Runtime trait specification
- `webrtc` v0.20.0-alpha with Tokio runtime support
- Documentation and migration guide (draft)

---

### Phase 3: Core Implementation (Q3 2026)

**Goal**: Implement the new async-friendly webrtc crate

#### New Crate Structure (Quinn-style)

```
rtc/                     # Sans-I/O protocol core (already exists!)
webrtc/                  # Async-friendly API with runtime abstraction
    ├── Cargo.toml       # Feature flags for runtime selection
    └── src/
        ├── lib.rs
        ├── peer_connection.rs
        ├── data_channel.rs
        ├── track.rs
        └── runtime/
            ├── mod.rs         # Runtime trait definitions
            ├── tokio.rs       # Tokio implementation (default)
            ├── async_std.rs   # async-std implementation
            ├── smol.rs        # smol implementation
            └── embassy.rs     # Embassy implementation (future)
```

**Cargo.toml Structure**
```toml
[features]
default = ["runtime-tokio"]
runtime-tokio = ["tokio"]
runtime-async-std = ["async-std"]
runtime-smol = ["smol", "async-io"]
runtime-embassy = ["embassy-executor"]

[dependencies]
rtc = { version = "0.8", path = "../rtc" }
tokio = { version = "1", optional = true }
async-std = { version = "1", optional = true }
smol = { version = "2", optional = true }
async-io = { version = "2", optional = true }
embassy-executor = { version = "0.6", optional = true }
```

#### Implementation Tasks

**1. Runtime Trait Abstraction**
- [ ] Define `Runtime` trait in `webrtc/src/runtime/mod.rs`
- [ ] Define `AsyncUdpSocket` trait for UDP I/O abstraction
- [ ] Define `AsyncTimer` trait for timer abstraction
- [ ] Type-erased wrappers: `Box<dyn Runtime>`, `Box<dyn AsyncUdpSocket>`

**2. Tokio Runtime Implementation**
- [ ] Implement `Runtime` for `TokioRuntime`
- [ ] Wrap `tokio::net::UdpSocket` as `AsyncUdpSocket`
- [ ] Wrap `tokio::time::Sleep` as `AsyncTimer`
- [ ] Feature-gated with `runtime-tokio` (default)

**3. Async-Friendly API Wrapping rtc Core**
- [ ] PeerConnection wrapping `rtc::peer_connection::RTCPeerConnection`
- [ ] I/O loop driving Sans-I/O protocol with runtime
- [ ] DataChannel with backpressure and async send/recv
- [ ] Track sender/receiver with media pipeline
- [ ] Proper shutdown and cleanup
- [ ] Memory leak tests and validation

**4. Additional Runtime Implementations**
- [ ] `AsyncStdRuntime` for async-std
- [ ] `SmolRuntime` for smol
- [ ] Documentation for adding custom runtimes

**4. Testing Infrastructure**
- [ ] Unit tests for all components
- [ ] Integration tests with rtc core
- [ ] Memory leak detection tests
- [ ] Browser interoperability tests (Chrome, Firefox)
- [ ] Cross-runtime compatibility tests

**Deliverables:**
- `webrtc` v0.20.0-alpha with runtime trait abstraction
- Tokio, async-std, and smol runtime implementations
- Comprehensive test suite
- Runtime selection examples

---

### Phase 4: Browser Interoperability & Testing (Q3-Q4 2026)

**Goal**: Ensure production-ready quality and browser compatibility

#### Browser Interoperability Testing

**Test Matrix:**
| Browser | Data Channels | Audio | Video | Simulcast | Status |
|---------|--------------|-------|-------|-----------|--------|
| Chrome | ☐ | ☐ | ☐ | ☐ | Planned |
| Firefox | ☐ | ☐ | ☐ | ☐ | Planned |
| Safari | ☐ | ☐ | ☐ | ☐ | Planned |
| Edge | ☐ | ☐ | ☐ | ☐ | Planned |

**Test Scenarios:**
- [ ] Peer-to-peer data channel communication
- [ ] Audio streaming (Opus codec)
- [ ] Video streaming (VP8, VP9, H.264)
- [ ] Simultaneous audio + video
- [ ] Simulcast with layer switching
- [ ] ICE restart mid-session
- [ ] Renegotiation (adding/removing tracks)
- [ ] Perfect negotiation pattern
- [ ] TURN relay connectivity
- [ ] Network condition handling (packet loss, delay)

#### Automated Testing Infrastructure

**CI/CD Pipeline:**
- [ ] GitHub Actions workflow for browser tests
- [ ] Selenium/Playwright integration
- [ ] Cross-platform testing (Linux, macOS, Windows)
- [ ] Performance benchmarking
- [ ] Memory leak detection (valgrind, ASAN)
- [ ] Nightly builds and testing

**Testing Tools:**
- [ ] Automated browser test harness
- [ ] Network condition simulator
- [ ] Signaling server for tests
- [ ] Test result dashboard

#### Performance Validation

- [ ] DataChannel throughput benchmarks (target: >500 Mbps)
- [ ] Media latency benchmarks (target: <100ms)
- [ ] Memory usage profiling
- [ ] CPU usage optimization
- [ ] Connection establishment time (target: <1s)

**Deliverables:**
- Comprehensive browser compatibility report
- Automated test suite in CI
- Performance benchmark results
- `webrtc` v0.20.0-beta

---

### Phase 5: Production Release & Documentation (Q4 2026)

**Goal**: Stable v0.20.0 release with comprehensive documentation

#### Documentation

**User Guides:**
- [ ] Getting Started guide
- [ ] Migration guide from v0.17.x
- [ ] Runtime selection guide
- [ ] API reference (rustdoc)
- [ ] Architecture overview
- [ ] Performance tuning guide
- [ ] Troubleshooting guide

**Examples:**
- [ ] Simple peer-to-peer data channel
- [ ] Audio streaming (microphone to speaker)
- [ ] Video streaming (camera to display)
- [ ] Screen sharing
- [ ] Multi-party conferencing (SFU integration)
- [ ] File transfer over data channel
- [ ] Game networking with unreliable channels
- [ ] IoT sensor data streaming

**Tutorials:**
- [ ] Building a video chat application
- [ ] Implementing a WebRTC gateway
- [ ] Creating custom interceptors
- [ ] Runtime adapter implementation guide

#### Migration Support

**Compatibility Layer:**
- [ ] Shim layer for v0.17.x callback API
- [ ] Migration tool (automated code updates)
- [ ] Deprecation warnings with guidance
- [ ] Side-by-side comparison examples

**Migration Guide:**
- [ ] Step-by-step migration instructions
- [ ] API mapping table (old → new)
- [ ] Common pitfalls and solutions
- [ ] Performance comparison

#### Community & Ecosystem

- [ ] Blog posts announcing v0.20.0
- [ ] Conference talks and presentations
- [ ] Community feedback incorporation
- [ ] Third-party adapter support (Actix, Warp, Axum)
- [ ] Integration examples with popular frameworks

**Deliverables:**
- `webrtc` v0.20.0 (stable)
- Complete documentation site
- Migration guide and tools
- 10+ working examples
- Blog post series

---

## Phase 6: Advanced Features & Optimization (2027+)

**Goal**: Enhanced features and performance optimizations

### Advanced Features

**1. Media Processing Pipeline**
- [ ] Jitter buffer improvements
- [ ] Custom codec support
- [ ] Media processing interceptors
- [ ] Hardware acceleration (when available)

**2. Advanced RTCP Features**
- [ ] Advanced congestion control
- [ ] Enhanced TWCC feedback
- [ ] Custom RTCP reports
- [ ] RTP header extensions

**3. Simulcast & SVC**
- [ ] Full simulcast support
- [ ] SVC (Scalable Video Coding) support
- [ ] Dynamic layer switching
- [ ] Bandwidth estimation improvements

**4. DataChannel Enhancements**
- [ ] Stream-based API (AsyncRead/AsyncWrite)
- [ ] Zero-copy data transfer
- [ ] Custom reliability profiles
- [ ] Message prioritization

### Performance Optimizations

**1. Memory Optimization**
- [ ] Buffer pooling
- [ ] Zero-copy packet processing
- [ ] Reduced allocations in hot paths
- [ ] Memory usage profiling

**2. CPU Optimization**
- [ ] SIMD optimizations for crypto
- [ ] Lazy parsing where possible
- [ ] Batch packet processing
- [ ] Lock-free data structures

**3. Network Optimization**
- [ ] UDP GSO (Generic Segmentation Offload)
- [ ] Socket option tuning
- [ ] Multi-socket support
- [ ] Custom transport protocols

### Ecosystem Integration

- [ ] Actix Web integration
- [ ] Axum/Tokio integration examples
- [ ] Warp integration examples
- [ ] Rocket integration
- [ ] Next-gen signaling protocols
- [ ] WebRTC-HTTP ingestion protocol (WHIP)
- [ ] WebRTC-HTTP egress protocol (WHEP)

---

## Migration Path

### For v0.17.x Users

**Phase 1: Preparation (Q2 2026)**
1. Pin to `webrtc = "0.17.x"` in Cargo.toml
2. Review deprecation warnings
3. Read migration guide (draft)
4. Experiment with `webrtc = "0.20.0-alpha"` in test branch

**Phase 2: Gradual Migration (Q3-Q4 2026)**
1. Update to `webrtc = "0.18.0-beta"`
2. Migrate one module at a time
3. Replace callbacks with trait-based handlers
4. Test thoroughly with browser interop tests
5. Update to `webrtc = "0.18.0"` stable

**Phase 3: Cleanup (2027)**
1. Remove compatibility shims
2. Adopt new patterns and APIs
3. Optimize for new architecture
4. Remove v0.17.x dependencies

### Compatibility Promise

- **v0.17.x**: Supported with bug fixes through 2026
- **v0.20.0**: Migration guide and tooling provided
- **Compatibility layer**: Available for 6 months after v0.20.0 release
- **Breaking changes**: Well documented with upgrade path

---

## Success Metrics

### Technical Metrics

| Metric | v0.17.x | v0.20.0 Target |
|--------|---------|---------------|
| Memory leak per connection | ~109 KiB | 0 KiB |
| Supported runtimes | 1 (Tokio) | 4+ (Tokio, async-std, smol, embassy) |
| Browser interop | Partial | 100% |
| Connection setup time | ~2s | <1s |
| DataChannel throughput | ~300 Mbps | >500 Mbps |
| Test coverage | ~60% | >80% |

### Community Metrics

- 100+ GitHub stars for new async adapters
- 10+ community-contributed examples
- 5+ production deployments reported
- Active Discord community discussions
- Conference talks and blog posts

---

## Risk Mitigation

### Technical Risks

| Risk | Impact | Mitigation |
|------|--------|------------|
| API complexity | High | Extensive user testing, iterative design |
| Performance regression | Medium | Continuous benchmarking, profiling |
| Browser compatibility issues | High | Automated browser testing, early feedback |
| Runtime abstraction overhead | Low | Zero-cost abstractions, benchmarks |

### Project Risks

| Risk | Impact | Mitigation |
|------|--------|------------|
| Extended timeline | Medium | Phased approach, MVP first |
| Community fragmentation | Medium | Clear migration path, support v0.17.x |
| Maintainer bandwidth | High | Community contributions, clear roadmap |
| Breaking changes pushback | High | Gradual migration, compatibility layer |

---

## Contributing

### How to Help

**Phase 2-3 (Current Focus):**
- Review API design RFC and provide feedback
- Test experimental runtime adapters
- Contribute runtime adapter implementations
- Write examples and documentation
- Report bugs in v0.17.x for fixes

**Phase 4-5 (Near Future):**
- Browser interoperability testing
- Write migration guides
- Create tutorials and examples
- Performance testing and optimization

### Contributor Guide

See [CONTRIBUTING.md](https://github.com/webrtc-rs/webrtc/blob/master/CONTRIBUTING.md) for:
- Code style guidelines
- PR submission process
- Testing requirements
- Documentation standards

---

## Communication

### Updates

- **Monthly progress updates** - GitHub Discussions
- **Blog posts** - Major milestone announcements
- **Discord** - Real-time discussions and support
- **GitHub Issues** - Feature requests and bug reports

### Feedback Channels

- **GitHub Discussions**: https://github.com/webrtc-rs/webrtc/discussions
- **Discord**: https://discord.gg/4Ju8UHdXMs
- **RFC Process**: For major API changes

---

## References

- [Sans-I/O Architecture](https://sans-io.readthedocs.io/)
- [W3C WebRTC Specification](https://www.w3.org/TR/webrtc/)
- [rtc Crate Documentation](https://docs.rs/rtc)
- [Perfect Negotiation Pattern](https://w3c.github.io/webrtc-pc/#perfect-negotiation-example)

---

**Last Updated**: January 31, 2026  
**Status**: Living document - updates as phases progress

---

## Architecture Decision: Quinn-style Runtime Abstraction

### Why Single Crate Instead of Multiple Runtime Crates?

After analyzing [Quinn](https://github.com/quinn-rs/quinn) (a mature Sans-I/O QUIC implementation), we've decided to adopt their proven architecture pattern:

**Quinn's Approach:**
- Single `quinn` crate with runtime trait abstraction
- Runtime selection via feature flags
- Implementations live in `quinn/src/runtime/`

**Benefits:**
1. **Simpler for users** - One crate, not multiple
2. **Less maintenance** - Shared code, minimal duplication
3. **Better testing** - Test all runtimes with feature flags
4. **Cleaner docs** - Everything in one place

### Comparison: Multiple Crates vs Single Crate

| Aspect | Multiple Crates | Single Crate (Quinn-style) |
|--------|----------------|---------------------------|
| User choice | `webrtc-tokio = "0.20"` | `webrtc = { version = "0.20", features = ["runtime-tokio"] }` |
| Crate count | 5+ crates | 2 crates (rtc + webrtc) |
| Code duplication | High (each crate has wrapper code) | Minimal (only adapters differ) |
| Documentation | Scattered across crates | Centralized in one crate |
| Testing | Per-crate test suites | Feature-flag based in CI |
| Maintenance burden | High | Low |
| API consistency | Risk of divergence | Always consistent |

### Implementation Details

**Runtime Trait**
```rust
// webrtc/src/runtime/mod.rs
pub trait Runtime: Send + Sync + Debug + 'static {
    fn spawn(&self, future: Pin<Box<dyn Future<Output = ()> + Send>>);
    fn wrap_udp_socket(&self, sock: std::net::UdpSocket) 
        -> io::Result<Box<dyn AsyncUdpSocket>>;
    fn new_timer(&self, instant: Instant) -> Pin<Box<dyn AsyncTimer>>;
    fn now(&self) -> Instant;
}
```

**Tokio Implementation**
```rust
// webrtc/src/runtime/tokio.rs
#[cfg(feature = "runtime-tokio")]
#[derive(Debug)]
pub struct TokioRuntime;

impl Runtime for TokioRuntime {
    fn spawn(&self, future: Pin<Box<dyn Future<Output = ()> + Send>>) {
        tokio::spawn(future);
    }
    
    fn wrap_udp_socket(&self, sock: std::net::UdpSocket) 
        -> io::Result<Box<dyn AsyncUdpSocket>> 
    {
        Ok(Box::new(tokio::net::UdpSocket::from_std(sock)?))
    }
}
```

**User Code**
```rust
use webrtc::{PeerConnection, runtime::TokioRuntime};

#[tokio::main]
async fn main() -> Result<()> {
    let pc = PeerConnection::builder()
        .runtime(TokioRuntime)
        .ice_servers(servers)
        .build()
        .await?;
    
    let offer = pc.create_offer().await?;
    pc.set_local_description(offer).await?;
    
    Ok(())
}
```

**Switching Runtimes**
```rust
// Just change the runtime and feature flag!
use webrtc::{PeerConnection, runtime::AsyncStdRuntime};

#[async_std::main]
async fn main() -> Result<()> {
    let pc = PeerConnection::builder()
        .runtime(AsyncStdRuntime)  // Same API, different runtime
        .build()
        .await?;
    
    // Everything else is identical!
}
```

### Feature Flags

```toml
[features]
default = ["runtime-tokio"]

# Runtime options (mutually exclusive in practice)
runtime-tokio = ["tokio"]
runtime-async-std = ["async-std"]
runtime-smol = ["smol", "async-io"]
runtime-embassy = ["embassy-executor"]

[dependencies]
rtc = { version = "0.8" }
tokio = { version = "1", optional = true }
async-std = { version = "1", optional = true }
smol = { version = "2", optional = true }
async-io = { version = "2", optional = true }
embassy-executor = { version = "0.6", optional = true }
```

### References

- **Quinn**: https://github.com/quinn-rs/quinn
- **Quinn runtime abstraction**: https://github.com/quinn-rs/quinn/tree/main/quinn/src/runtime
- **Analysis document**: See `quinn-architecture-analysis.md` in this directory

