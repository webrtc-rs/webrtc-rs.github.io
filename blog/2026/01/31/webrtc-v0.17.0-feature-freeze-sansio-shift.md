# WebRTC v0.17.0: Feature Freeze and Shifting to Build Async-Friendly API on Sans-I/O rtc Crate

We're announcing a significant transition for the webrtc-rs project. Version **0.17.0** marks the **final feature release** of the Tokio-coupled async WebRTC implementation. This isn't an ending—it's a strategic evolution toward a more sustainable, flexible, and robust architecture.

## What This Means

**v0.17.x branch**: Will receive **bug fixes only**. No new features will be added.

**Master branch**: Will transition to a **Sans-I/O architecture** built on top of [webrtc-rs/rtc](https://github.com/webrtc-rs/rtc).

This new architecture will:

- ✅ Support multiple async runtimes (Tokio, smol, async-std, embassy, etc.)
- ✅ Provide a clean, protocol-centric Sans-I/O core via webrtc-rs/rtc
- ✅ Enable a truly runtime-agnostic, async-friendly WebRTC implementation in Rust

---

## Why This Fundamental Shift?

Over several years of building and maintaining webrtc-rs, we've learned valuable lessons from production deployments. While the current Tokio-based implementation has served the community well, we've encountered **fundamental architectural challenges** that cannot be adequately addressed through incremental improvements.

### Critical Issue: Resources Leak

One of the most serious problems plaguing the current architecture is **systematic resource leakage through callback handlers** ([#772](https://github.com/webrtc-rs/webrtc/issues/772), [#406](https://github.com/webrtc-rs/webrtc/issues/406)).

The current callback-based event handling API looks like this:

```rust
pc.on_peer_connection_state_change(Box::new(move |s: RTCPeerConnectionState| {
    Box::pin(async { /* ... */ })
}));
```

**The Problem**: When a `PeerConnection` is dropped and closed, these `Box`ed callbacks are **not properly released**. Community testing has revealed the severity:

- **~109 KiB leaked per connection** in typical usage scenarios
- Callbacks remain in memory for minutes (or indefinitely) after peer connections are closed
- No API exists to manually clear handlers or force cleanup
- The `is_closed: Arc<AtomicBool>` flag prevents connection reuse, forcing memory leaks with each new connection

Community member [@Ddystopia](https://github.com/Ddystopia) created a [test repository](https://github.com/Ddystopia/webrtc_leak) demonstrating the leak with empirical data:

| Connections | Memory Leaked |
|------------|---------------|
| 1 | 283 KiB |
| 4 | 620 KiB |
| 21 | 2.5 MiB |
| 40 | 4.6 MiB |

Linear regression analysis: **`leak = 111 KiB × connections + 172 KiB`**

This demonstrates a **systematic, linear memory leak pattern** that accumulates with every peer connection created. For long-running servers handling hundreds or thousands of connections, this becomes a critical production issue.

### Deep Architectural Problems

The callback memory leak is just a symptom of deeper structural issues:

#### 1. Unclear Ownership and Lifecycle Management

The `Box<dyn Fn>` callback pattern creates ambiguous ownership semantics:

- Who owns the callback? The caller? The library?
- When should it be freed? On connection close? Never?
- How do circular references get broken?

The lack of explicit lifecycle management has led to resource leaks that are extremely difficult to debug and fix within the current design. These aren't simple bugs—they're architectural problems baked into the foundation.

#### 2. Tight Tokio Coupling

The current implementation deeply integrates with Tokio's runtime:

```rust
// Current webrtc crate - Tokio everywhere
async fn do_something(&self) -> Result<()> {
    tokio::spawn(async move { ... });  // Hidden task spawning
    tokio::time::sleep(duration).await; // Tokio-specific timers
    // ...
}
```

**Consequences:**

- **No other runtimes**: Cannot use with async-std, smol, or embedded runtimes like embassy
- **Hidden behavior**: Background tasks and timers you don't control
- **Testing complexity**: Requires Tokio runtime even for protocol-only tests
- **Platform limitations**: Limits deployment to platforms where Tokio works well

For embedded systems, game engines with custom schedulers, or applications already using a different runtime, this coupling is a dealbreaker.

#### 3. Missed Opportunity: Async Traits

The callback-based API predates Rust's stable `async fn in trait` (stabilized in 1.75). As noted in [#522](https://github.com/webrtc-rs/webrtc/issues/522), a trait-based event handling system would provide:

- **Better ergonomics**: Single trait instead of multiple callback registrations
- **Fewer allocations**: No `Box::new()` for every handler
- **Clearer ownership**: Explicit lifecycle tied to trait implementor
- **Modern Rust patterns**: Alignment with ecosystem evolution

Example of what could have been:

```rust
trait PeerConnectionEventHandler {
    async fn on_connection_state_change(&mut self, state: RTCPeerConnectionState) {}
    async fn on_ice_candidate(&mut self, candidate: Option<RTCIceCandidate>) {}
    async fn on_track(&mut self, track: Arc<RTCTrack>) {}
    // ...
}

// Single registration, clear ownership
pc.with_handler(MyHandler { ... });
```

#### 4. Deadlock Potential

[#121](https://github.com/webrtc-rs/webrtc/issues/121) highlights how `Box<dyn Fn>` callbacks can lead to deadlocks. The callbacks hold locks, call async functions, and can create cycles. Moving to `Arc<dyn Fn>` helps, but it's another band-aid on a fundamentally flawed design.

#### 5. Protocol Logic Entangled with I/O

The current codebase mixes WebRTC protocol state machines with Tokio-specific I/O operations:

```rust
// Protocol logic mixed with I/O - hard to test, hard to port
async fn handle_rtcp(&self, conn: &Arc<UdpSocket>) -> Result<()> {
    let mut buf = vec![0u8; 1500];
    let n = conn.recv(&mut buf).await?;  // Tokio-specific I/O
    // Process RTCP packet...
    // More Tokio I/O...
}
```

This makes it difficult to:

- Test protocol logic in isolation (requires mocking Tokio primitives)
- Port to different runtimes (Tokio assumptions everywhere)
- Reason about correctness (I/O errors mixed with protocol errors)
- Optimize independently (protocol and I/O performance coupled)

---

## The Sans-I/O Solution

Sans-I/O (without I/O) is a proven architectural pattern that **separates protocol logic from I/O operations**. This isn't a new idea—it's successfully used in:

- **Python**: `h11` (HTTP/1.1), `h2` (HTTP/2), `wsproto` (WebSocket)
- **Rust**: `quinn` (QUIC), various embedded networking stacks
- **Embedded**: Many RTOS networking implementations

### How Sans-I/O Works

**Traditional approach** (protocol + I/O coupled):

```rust
// OLD: Protocol + I/O mixed together
async fn handle_rtcp(conn: &TokioUdpSocket) -> Result<()> {
    let mut buf = vec![0u8; 1500];
    let n = conn.recv(&mut buf).await?;  // Tokio-specific I/O
    // Process RTCP packet... (protocol logic)
}
```

**Sans-I/O approach** (protocol and I/O separated):

```rust
// NEW: Pure protocol logic (webrtc-rs/rtc)
fn process_rtcp(packet: &[u8]) -> Result<ProtocolAction> {
    // Pure protocol logic, no I/O, no async
    // Returns what actions to take (send packets, notify app, etc.)
}

// Separate: Tokio-specific I/O wrapper (webrtc-rs/webrtc)
async fn io_loop_tokio(socket: TokioUdpSocket, protocol: &mut RtcState) {
    let mut buf = vec![0u8; 2000];
    loop {
        // Your Tokio I/O
        let (n, peer_addr) = socket.recv_from(&mut buf).await?;
        
        // Drive protocol state machine
        protocol.handle_read(&buf[..n], peer_addr, Instant::now())?;
        
        // Send outgoing packets
        while let Some(msg) = protocol.poll_write() {
            socket.send_to(&msg.data, msg.addr).await?;
        }
    }
}

// Alternative: async-std I/O wrapper (same protocol core!)
async fn io_loop_async_std(socket: AsyncStdUdpSocket, protocol: &mut RtcState) {
    // async-std I/O, but identical protocol logic
}
```

The protocol core (`rtc` crate) knows **nothing about I/O**. It just processes bytes and tells you what to send. You control all I/O operations.

### Concrete Benefits

#### ✅ Multiple Async Runtime Support

**First-class support for:**

- **Tokio** — Production servers, web services
- **async-std** — Alternative async runtime
- **smol** — Lightweight async runtime
- **embassy** — Embedded async runtime (no_std)
- **Synchronous I/O** — Blocking sockets for special use cases

The same protocol core works with all of them. Just write a different I/O adapter.

#### ✅ Clean Protocol-Centric Core

- **Pure Rust protocol logic** in [webrtc-rs/rtc](https://github.com/webrtc-rs/rtc)
- **No I/O dependencies** — easier to audit, test, and reason about
- **Better performance** — optimize protocol logic and I/O independently
- **Easier to maintain** — clear separation of concerns

#### ✅ Proper Resource Management

```rust
// Sans-I/O: Clear ownership, explicit lifecycle
struct MyApp {
    pc: RTCPeerConnection,  // You own it
    socket: UdpSocket,      // You own it
}

impl Drop for MyApp {
    fn drop(&mut self) {
        self.pc.close();  // Explicit cleanup
        // Everything is freed properly - no leaks
    }
}
```

- **Clear ownership semantics** — you own the state machine
- **Explicit lifecycle** — you control when things are created and destroyed
- **No hidden callbacks** — no leaked closures or circular references
- **Predictable memory usage** — allocations are visible and controllable

#### ✅ Modern Rust Patterns

- Leverage stable `async fn in trait` for event handling
- Use modern zero-cost abstractions
- Better alignment with Rust ecosystem evolution
- No legacy design decisions holding us back

#### ✅ Superior Testing

**Protocol logic without I/O:**

```rust
#[test]
fn test_ice_state_machine() {
    let mut pc = RTCPeerConnection::new(config)?;
    
    // Pure protocol test - no networking, no async runtime
    let packet = create_stun_binding_request();
    pc.handle_read(&packet, peer_addr, Instant::now())?;
    
    // Check what to send
    let response = pc.poll_write().unwrap();
    assert!(is_stun_binding_response(&response.data));
}
```

**Deterministic time:**

```rust
#[test]
fn test_ice_timeout() {
    let fixed_time = Instant::now();
    
    pc.handle_timeout(fixed_time)?;
    pc.handle_timeout(fixed_time + Duration::from_secs(5))?;
    
    // Fully deterministic, no flaky timing
}
```

- **No network required** — test protocol logic in isolation
- **Deterministic** — control time, no race conditions
- **Faster** — no async overhead in tests
- **Easier fuzzing** — pure functions are easy to fuzz

#### ✅ Future-Proof

- Not locked to any runtime's evolution
- Can adopt new async patterns as they emerge
- Easier to optimize for specific platforms
- Ready for `no_std` embedded systems

---

## What's New in v0.17.0

Despite being the final feature release, v0.17.0 includes important improvements:

**Features:**

- Multi-codec negotiation support ([#741](https://github.com/webrtc-rs/webrtc/pull/741))
- H.264 High Profile codec support ([#768](https://github.com/webrtc-rs/webrtc/pull/768))
- AES CM 256 crypto profiles for SRTP ([#764](https://github.com/webrtc-rs/webrtc/pull/764))
- Async-capable PSK callbacks ([#751](https://github.com/webrtc-rs/webrtc/pull/751))

**Performance:**

- Improved RR/SR ticker behavior to prevent catchup bursts ([#745](https://github.com/webrtc-rs/webrtc/pull/745))

**Quality:**

- Better SDP parsing per RFC 8866 ([#770](https://github.com/webrtc-rs/webrtc/pull/770))
- DTLS refactoring replacing bincode with rkyv ([#767](https://github.com/webrtc-rs/webrtc/pull/767))
- Removed `Seek` requirement for some writers ([#743](https://github.com/webrtc-rs/webrtc/pull/743))

**Documentation:**

- Feature flags documented in lib.rs ([#759](https://github.com/webrtc-rs/webrtc/pull/759))
- Basic WebRTC explanatory markdown files ([#756](https://github.com/webrtc-rs/webrtc/pull/756))

**Full changelog**: [v0.14.0...v0.17.0](https://github.com/webrtc-rs/webrtc/compare/v0.14.0...v0.17.0)

---

## Timeline and Migration Path

### Q1 2026 (Now)

- ✅ v0.17.0 released with feature freeze
- ✅ v0.17.x branch created for bug fixes
- 🔄 Master branch begins Sans-I/O refactoring
- 🔄 Early adopters can experiment with [webrtc-rs/rtc](https://github.com/webrtc-rs/rtc)

### Q2-Q3 2026

- Core protocol implementations stabilize in `rtc` crate
- Runtime adapters for Tokio, smol, async-std published
- Migration guide and examples published
- Browser interoperability testing

### Q4 2026+

- v0.20.0 release with Sans-I/O architecture
- Comprehensive documentation and tutorials
- Deprecation timeline for v0.17.x (with ample notice)

---

## For Current Users

### Should I upgrade to v0.17.0?

**Yes, if you're on an older version.** v0.17.0 includes important fixes and features.

**Pin to the `v0.17.x` branch** if you need stability and critical bug fixes.

### Will v0.17.x be supported?

**Yes**, with critical bug fixes for a reasonable transition period. The exact timeline will be communicated as v0.20.0 stabilizes.

We're committed to giving users ample time to migrate.

### What about my existing code?

- Code using v0.17.x will continue to work
- Migration guides will be provided **well in advance** of v0.20.0
- We're committed to making the transition as smooth as possible
- Breaking changes will be clearly documented

### Why not just fix the memory leaks?

We tried. The memory leak issue is a symptom of deeper architectural problems—unclear ownership, tight runtime coupling, and protocol/I/O entanglement. Band-aid fixes would just create more technical debt. A clean architectural foundation is necessary for long-term sustainability.

---

## Get Involved

This is an exciting evolution for webrtc-rs, and we'd love your help:

- **Try the new architecture**: Experiment with [webrtc-rs/rtc](https://github.com/webrtc-rs/rtc)
- **Provide feedback**: Share your use cases and requirements
- **Contribute**: Help build runtime adapters for your preferred async runtime
- **Report issues**: Help us identify and fix bugs in v0.17.x

Join our discussions:

- **GitHub**: https://github.com/webrtc-rs/webrtc
- **Discord**: https://discord.gg/4Ju8UHdXMs
- **Discussions**: https://github.com/webrtc-rs/webrtc/discussions

---

## Conclusion

The shift to Sans-I/O architecture represents webrtc-rs **growing up**. We've learned from years of production use, listened to community feedback about memory leaks and runtime coupling, and are making the hard but necessary architectural changes to build a truly world-class WebRTC implementation in Rust.

**Version 0.17.0** marks the culmination of the Tokio-coupled era—a solid foundation that served us well. **Version 0.20.0** will open the door to a more flexible, efficient, and maintainable future.

The Sans-I/O architecture provides:

- **No memory leaks** — clear ownership, no hidden callbacks
- **Runtime freedom** — use any async runtime or none at all
- **Better testing** — deterministic, fast, no network required
- **Future-proof** — ready for new patterns and platforms

Thank you to all contributors who made v0.17.0 possible, and to everyone who will help shape the Sans-I/O future of webrtc-rs. 🦀

---

## Links

- **GitHub**: https://github.com/webrtc-rs/webrtc
- **rtc (Sans-I/O)**: https://github.com/webrtc-rs/rtc
- **Discussions**: https://github.com/webrtc-rs/webrtc/discussions
- **Docs**: https://docs.rs/webrtc
- **Discord**: https://discord.gg/4Ju8UHdXMs

---

## Further Reading

- [Announcing rtc 0.3.0: Sans-I/O WebRTC Stack for Rust](/blog/2026/01/04/announcing-rtc-v0.3.0)
- [Building WebRTC's Pipeline with sansio::Protocol](/blog/2026/01/04/building-webrtc-pipeline-with-sansio)
- [RTC Feature Complete: What's Next](/blog/2026/01/18/rtc-feature-complete-whats-next)
