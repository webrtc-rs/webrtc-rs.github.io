# Building Async-Friendly `webrtc` on Sans-I/O `rtc`: Architecture Design and Roadmap

The webrtc-rs project is embarking on a significant architectural evolution. With `webrtc` v0.17.0 marking the final feature release of the Tokio-coupled implementation, we're now designing the next generation: an async-friendly API built on top of our Sans-I/O `rtc` crate, supporting multiple async runtimes while maintaining clean, ergonomic APIs.

This post details the **proposed architecture design**, explains our approach, and presents the roadmap for webrtc v0.20.0. **This is a design discussion—feedback and suggestions are welcome!**

---

## The Challenge: Moving Beyond Tokio Coupling

The current `webrtc` crate (v0.17.x) faces fundamental architectural limitations:

### Callback Hell and Ergonomics Issues

The callback-based event handling API creates significant ergonomic challenges:

```rust
// Current v0.17.x API - Callback hell
let pc = Arc::new(api.new_peer_connection(config).await?);

// Problem 1: Excessive cloning and Arc wrapping
let pc_clone1 = Arc::clone(&pc);
pc.on_peer_connection_state_change(Box::new(move |s: RTCPeerConnectionState| {
    let pc = Arc::clone(&pc_clone1);  // Clone inside closure
    Box::pin(async move {
        println!("State: {s}");
        // Need to capture 'pc' if we want to call methods
    })
}));

// Problem 2: Each callback requires new clones
let pc_clone2 = Arc::clone(&pc);
pc.on_ice_candidate(Box::new(move |candidate: Option<RTCIceCandidate>| {
    let pc = Arc::clone(&pc_clone2);  // More cloning...
    Box::pin(async move {
        // Send candidate to peer
    })
}));

// Problem 3: Verbose nested closures
let pc_clone3 = Arc::clone(&pc);
let shared_state = Arc::clone(&my_state);
pc.on_track(Box::new(move |track: Arc<TrackRemote>| {
    let shared_state = Arc::clone(&shared_state);
    Box::pin(async move {
        // Triple nesting: Box, closure, async block
    })
}));
```

**Key Problems**:

1. **Arc explosion**: Every callback requires `Arc::clone(&pc)`, then clones inside the closure
2. **Triple wrapping**: `Box::new(move |...| Box::pin(async move { ... }))`—verbose and hard to read
3. **Scattered event handling**: Each event type needs separate callback registration
4. **Difficult to share state**: Need to wrap everything in `Arc<Mutex<>>` for mutable state
5. **No structured cleanup**: Callbacks live forever, no way to coordinate their lifecycle
6. **Error handling complexity**: Each callback must handle errors independently

**Real-world impact**:

```rust
// Managing multiple events becomes unwieldy
let pc = Arc::new(api.new_peer_connection(config).await?);
let state = Arc::new(Mutex::new(MyState::new()));

// 10+ lines per callback * 6 event types = 60+ lines of boilerplate!
let pc1 = Arc::clone(&pc);
let state1 = Arc::clone(&state);
pc.on_track(Box::new(move |track| {
    let state = Arc::clone(&state1);
    Box::pin(async move { /* ... */ })
}));

let pc2 = Arc::clone(&pc);
let state2 = Arc::clone(&state);
pc.on_data_channel(Box::new(move |dc| {
    let state = Arc::clone(&state2);
    Box::pin(async move { /* ... */ })
}));

// ... 4 more similar blocks
```

This callback pattern also has resource management issues—when `PeerConnection` is dropped, these `Box`ed callbacks may not be properly cleaned up, potentially leading to memory leaks ([#772](https://github.com/webrtc-rs/webrtc/issues/772)).

### Tight Runtime Coupling

The current implementation is deeply integrated with Tokio:

```rust
// Hidden Tokio dependencies everywhere
async fn internal_method(&self) -> Result<()> {
    tokio::spawn(async move { ... });      // Hidden task spawning
    tokio::time::sleep(duration).await;     // Tokio-specific timers
    // ... more Tokio coupling
}
```

**Consequences**:
- Cannot use async-std, smol, or embedded runtimes like embassy
- Hidden background tasks you don't control
- Testing requires Tokio even for protocol-only tests
- Platform limitations where Tokio doesn't work optimally

### Missed Opportunity: Modern Async Patterns

The callback API predates modern Rust async patterns. Modern approaches (Rust 1.75+ with `async fn in trait`) could provide:

- **Stream-based APIs**: Events as `impl Stream<Item = Event>` instead of callbacks
- **Trait-based handlers**: Single trait implementation vs. multiple callback registrations
- **Better ergonomics**: No `Box::new(move |...| Box::pin(async move { ... }))` dance
- **Clearer ownership**: Direct ownership without Arc explosion
- **Ecosystem alignment**: Works naturally with tokio-stream, async-std streams, etc.

---

## The Foundation: Sans-I/O Protocol Core

The `rtc` crate provides a complete Sans-I/O WebRTC implementation:

### What is Sans-I/O?

Sans-I/O (without I/O) separates protocol logic from I/O operations:

```rust
// Sans-I/O: Pure protocol logic
fn process_packet(packet: &[u8]) -> Result<ProtocolAction> {
    // No I/O, no async, just protocol state machine
}

// Runtime-specific I/O wrapper
async fn io_loop(socket: UdpSocket, protocol: &mut RtcState) {
    let (n, peer_addr) = socket.recv_from(&mut buf).await?;
    protocol.handle_read(&buf[..n], peer_addr, Instant::now())?;
    
    while let Some(msg) = protocol.poll_write() {
        socket.send_to(&msg.data, msg.addr).await?;
    }
}
```

### The rtc Crate: Feature Complete

The `rtc` crate (v0.8.x) has achieved:

- ✅ **Full feature parity** with `webrtc` v0.17.x
- ✅ **W3C compliance**: 95%+ WebRTC API compliance
- ✅ **Complete protocol stack**: ICE, DTLS, SRTP, SCTP, RTP/RTCP
- ✅ **Interceptor framework** with `sansio::Protocol` trait
- ✅ **WebRTC Stats API** implementation
- ✅ **mDNS support** for privacy-preserving connections

**Key advantage**: All protocol logic is testable without networking, deterministic with controlled time, and runtime-agnostic.

---

## Learning from Quinn: Runtime Abstraction Done Right

Before designing our approach, we studied [Quinn](https://github.com/quinn-rs/quinn), a mature Sans-I/O QUIC implementation in Rust.

### Quinn's Architecture

Quinn does **NOT** create separate crates like `quinn-tokio`, `quinn-async-std`, etc. Instead:

```
quinn/
├── quinn-proto/          # Sans-I/O protocol (like our rtc crate)
└── quinn/                # Async API with runtime trait
    └── src/
        └── runtime/
            ├── mod.rs    # Runtime trait
            ├── tokio.rs  # Tokio implementation
            └── smol.rs   # smol implementation
```

**Key insight**: Single crate with runtime trait abstraction.

### Runtime Trait Pattern

Quinn defines a `Runtime` trait:

```rust
pub trait Runtime: Send + Sync + Debug + 'static {
    fn new_timer(&self, i: Instant) -> Pin<Box<dyn AsyncTimer>>;
    fn spawn(&self, future: Pin<Box<dyn Future<Output = ()> + Send>>);
    fn wrap_udp_socket(&self, sock: UdpSocket) -> io::Result<Box<dyn AsyncUdpSocket>>;
    fn now(&self) -> Instant;
}
```

Concrete implementations:

```rust
// Tokio runtime
pub struct TokioRuntime;

impl Runtime for TokioRuntime {
    fn spawn(&self, future: Pin<Box<dyn Future<Output = ()> + Send>>) {
        tokio::spawn(future);
    }
    
    fn wrap_udp_socket(&self, sock: UdpSocket) -> io::Result<Box<dyn AsyncUdpSocket>> {
        Ok(Box::new(tokio::net::UdpSocket::from_std(sock)?))
    }
}

// smol runtime
pub struct SmolRuntime;

impl Runtime for SmolRuntime {
    fn spawn(&self, future: Pin<Box<dyn Future<Output = ()> + Send>>) {
        ::smol::spawn(future).detach();
    }
    
    fn wrap_udp_socket(&self, sock: UdpSocket) -> io::Result<Box<dyn AsyncUdpSocket>> {
        Ok(Box::new(Async::new_nonblocking(sock)?))
    }
}
```

**Runtime selection via feature flags**:

```toml
[features]
default = ["runtime-tokio"]
runtime-tokio = ["tokio"]
runtime-smol = ["smol", "async-io"]
```

**User code**:

```rust
// Works with any runtime!
let endpoint = Endpoint::builder()
    .runtime(TokioRuntime)  // or SmolRuntime
    .bind(...)?;
```

### Why This Approach is Superior

| Aspect | Multiple Crates | Quinn-style Single Crate |
|--------|----------------|--------------------------|
| Crate count | 6+ separate crates | 2 crates (proto + async) |
| User choice | `webrtc-tokio = "0.20"` | `webrtc = { features = ["runtime-tokio"] }` |
| Code duplication | High | Minimal |
| Maintenance | Complex | Simple |
| Documentation | Scattered | Centralized |
| API consistency | Risk of divergence | Always consistent |
| Testing | Per-crate test suites | Feature-flag based |

**Decision**: We're adopting Quinn's architecture pattern for webrtc-rs.

---

## Proposed Architecture Design: webrtc v0.20.0

### Crate Structure

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
            └── embassy.rs     # embassy implementation (embedded)
```

### Runtime Trait Design

```rust
/// Abstracts async runtime operations for runtime independence
pub trait Runtime: Send + Sync + Debug + 'static {
    /// Spawn a background task
    fn spawn(&self, future: Pin<Box<dyn Future<Output = ()> + Send>>);
    
    /// Wrap a standard UDP socket as an async socket
    fn wrap_udp_socket(&self, sock: std::net::UdpSocket) 
        -> io::Result<Box<dyn AsyncUdpSocket>>;
    
    /// Create a timer that expires at the given instant
    fn new_timer(&self, instant: Instant) -> Pin<Box<dyn AsyncTimer>>;
    
    /// Get current time
    fn now(&self) -> Instant;
}

/// Async UDP socket abstraction
pub trait AsyncUdpSocket: Send + Sync + Debug + 'static {
    fn poll_recv(&mut self, cx: &mut Context, bufs: &mut [IoSliceMut]) 
        -> Poll<io::Result<usize>>;
    
    fn poll_send(&mut self, cx: &mut Context, transmit: &Transmit) 
        -> Poll<io::Result<()>>;
    
    fn local_addr(&self) -> io::Result<SocketAddr>;
}

/// Async timer abstraction
pub trait AsyncTimer: Send + Debug + 'static {
    fn reset(self: Pin<&mut Self>, instant: Instant);
    fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<()>;
}
```

### Modern Async API: Design Choices

WebRTC v0.20.0 **plans to use a push-based trait handler approach** for event handling. While there are two primary patterns used in the Rust async ecosystem, WebRTC's complex state management and multiple interconnected event types make trait handlers the most suitable choice.

Below, we explore both options and explain the rationale for proposing trait-based handlers:

#### Option 1: Push-Based Handler Trait (Proposed for v0.20.0)

Events are delivered via trait callbacks, similar to libp2p's `NetworkBehaviour`:

```rust
pub trait PeerConnectionEventHandler: Send {
    async fn on_connection_state_change(&mut self, state: RTCPeerConnectionState) {}
    async fn on_ice_candidate(&mut self, candidate: Option<RTCIceCandidate>) {}
    async fn on_track(&mut self, track: Track) {}
    async fn on_data_channel(&mut self, channel: DataChannel) {}
}

struct MyHandler {
    // Your state
}

impl PeerConnectionEventHandler for MyHandler {
    async fn on_track(&mut self, track: Track) {
        println!("New track: {}", track.id());
        // Handle track with clean ownership
    }
}
```

**Pros:**
- ✅ **Single coordination point**: All events handled in one place
- ✅ **Familiar to WebRTC users**: Similar to JavaScript WebRTC API
- ✅ **Centralized state**: Easy to maintain handler state across events
- ✅ **No missed events**: Framework ensures all events are delivered

**Cons:**
- ⚠️ **Trait boilerplate**: Requires implementing traits
- ⚠️ **Less composable**: Harder to use combinators or share logic
- ⚠️ **Framework-style**: More opinionated, less flexible
- ⚠️ **Testing complexity**: Requires mocking entire trait

#### Option 2: Pull-Based Stream API (Alternative Pattern)

Following Quinn's pattern, events are modeled as async streams that users pull from. While this works well for QUIC's simpler event model, it becomes complex for WebRTC:

```rust
use tokio_stream::StreamExt;

// Accept incoming tracks
let mut tracks = conn.tracks();
while let Some(track) = tracks.next().await {
    println!("New track: {}", track.id());
    // Handle track with clean ownership
}

// Pull ICE candidates
let mut candidates = conn.ice_candidates();
while let Some(candidate) = candidates.next().await {
    send_to_peer(candidate).await?;
}

// Monitor connection state
let mut states = conn.connection_states();
while let Some(state) = states.next().await {
    match state {
        RTCPeerConnectionState::Connected => println!("Connected!"),
        RTCPeerConnectionState::Failed => break,
        _ => {}
    }
}

// Note: Managing 6+ event streams separately is complex!
```

**Pros:**
- ✅ **Ecosystem integration**: Works naturally with `Stream` combinators (`filter`, `map`, `merge`)
- ✅ **Composable**: Easy to combine multiple streams with `tokio::select!` or `StreamExt`
- ✅ **Clear ownership**: No trait implementations, clean borrow semantics
- ✅ **Testable**: Streams are easy to mock and test
- ✅ **Modern Rust idiom**: Matches patterns in tokio-tungstenite, async-std

**Cons:**
- ⚠️ **Multiple event loops**: Users must spawn tasks or use `select!` for multiple event types
- ⚠️ **Potential backpressure**: Slow consumers could buffer events
- ⚠️ **Complexity**: Coordinating 5+ streams can be verbose
- ⚠️ **State coordination difficulty**: Sharing state across 6+ separate tasks/loops is complex

### Comparison: Push vs Pull

| Aspect | Push-Based (Traits) | Pull-Based (Streams) |
|--------|-------------------|---------------------|
| **Composability** | ⭐⭐ Limited | ⭐⭐⭐⭐⭐ Stream combinators |
| **Simplicity** | ⭐⭐⭐⭐ Single handler | ⭐⭐⭐ Multiple loops needed |
| **Flexibility** | ⭐⭐⭐ Framework-bound | ⭐⭐⭐⭐⭐ Very flexible |
| **Testing** | ⭐⭐⭐ Trait mocking | ⭐⭐⭐⭐⭐ Easy to mock |
| **Ecosystem fit** | ⭐⭐⭐ Less common | ⭐⭐⭐⭐⭐ Tokio/async-std idiomatic |
| **Learning curve** | ⭐⭐⭐⭐ Familiar to WebRTC users | ⭐⭐⭐⭐ Familiar to Rust devs |
| **State management** | ⭐⭐⭐⭐⭐ Centralized | ⭐⭐⭐ Distributed across tasks |

### Design Decision: Push-Based Traits for WebRTC v0.20.0

After careful consideration of the trade-offs, **we propose the push-based trait handler approach** as the primary API design for v0.20.0. Here's the reasoning behind this design decision:

#### WebRTC's Complex Event Model

Unlike Quinn (QUIC), which has a simpler event model focused on accepting streams and datagrams, WebRTC has **multiple interconnected event types** that often need coordinated handling:

```rust
// WebRTC events are diverse and interconnected:
- ICE candidate gathering (continuous during connection)
- Connection state changes (affects all other operations)
- Track additions/removals (media streams)
- Data channel events (application data)
- DTLS state changes (security layer)
- SCTP events (data channel transport)
- Negotiation needed (renegotiation triggers)
- Signaling state changes
```

**The problem with pull-based streams for WebRTC:**

```rust
// Pull-based: Need to coordinate 6+ event streams!
tokio::select! {
    Some(track) = tracks.next() => { /* handle track */ },
    Some(candidate) = candidates.next() => { /* handle ICE */ },
    Some(state) = states.next() => { /* handle state */ },
    Some(dc) = data_channels.next() => { /* handle data channel */ },
    Some(negotiation) = negotiations.next() => { /* handle renegotiation */ },
    // ... more branches
}

// Or spawn 6+ separate tasks:
tokio::spawn(async move { /* handle tracks */ });
tokio::spawn(async move { /* handle ICE */ });
tokio::spawn(async move { /* handle state */ });
// Coordinating state across tasks becomes complex!
```

**With trait-based handlers:**

```rust
impl PeerConnectionEventHandler for MyHandler {
    // All events in one place, easy to coordinate
    async fn on_track(&mut self, track: Track) {
        self.tracks.push(track);  // Direct state access
        self.check_ready().await;  // Easy coordination
    }
    
    async fn on_connection_state_change(&mut self, state: RTCPeerConnectionState) {
        self.state = state;
        self.check_ready().await;  // Same coordination logic
    }
    
    async fn check_ready(&mut self) {
        if self.state == Connected && !self.tracks.is_empty() {
            // Coordinated logic across multiple event types
        }
    }
}
```

#### Key Advantages for WebRTC

1. **Centralized State Management**: WebRTC applications typically need to coordinate state across multiple event types. A single handler trait makes this natural.

2. **Familiar Mental Model**: WebRTC developers from JavaScript, Go, C++ are used to callback/handler patterns. This reduces friction.

3. **Simpler for Common Cases**: Most WebRTC applications need to handle all event types. Writing one trait implementation is simpler than managing 6+ streams.

4. **Clear Event Ordering**: Events are delivered in order through a single handler, making it easier to reason about state transitions.

5. **Better Match for Protocol Complexity**: WebRTC's state machine involves complex interactions between ICE, DTLS, SCTP, and RTP layers. Centralized handling helps manage this complexity.

#### Future Flexibility

The proposed design for v0.20.0 uses trait-based handlers as the primary API, but we could still provide stream-based access for users who prefer it:

```rust
// Optional: Stream-based access built on top of trait system
impl PeerConnection {
    pub fn tracks(&self) -> impl Stream<Item = Track> {
        // Internally uses trait handler to broadcast to stream
    }
}
```

This gives us the best of both worlds: a simple, cohesive API by default, with opt-in flexibility for advanced use cases.

### Example API Design

The proposed primary API for v0.20.0 uses the push-based trait handler approach:

#### Trait-Based Handler Example (v0.20.0 Proposed Primary API)

```rust
use webrtc::{PeerConnection, PeerConnectionEventHandler, runtime::TokioRuntime};

struct MyHandler {
    // Your application state
}

impl PeerConnectionEventHandler for MyHandler {
    async fn on_track(&mut self, track: Track) {
        println!("New track: {}", track.id());
        // Handle track
    }
    
    async fn on_ice_candidate(&mut self, candidate: Option<RTCIceCandidate>) {
        // Send to peer
        send_to_peer(candidate).await;
    }
    
    async fn on_connection_state_change(&mut self, state: RTCPeerConnectionState) {
        println!("State: {state}");
    }
}

#[tokio::main]
async fn main() -> Result<()> {
    let pc = PeerConnection::builder()
        .runtime(TokioRuntime)
        .handler(MyHandler::new())
        .ice_servers(vec![ice_server])
        .build()
        .await?;
    
    // All events handled via trait methods
    let offer = pc.create_offer().await?;
    pc.set_local_description(offer).await?;
    
    Ok(())
}
```

#### Stream-Based Alternative (Optional, for Advanced Use Cases)

For users who prefer stream-based patterns, we can provide optional stream access:

```rust
use webrtc::{PeerConnection, runtime::TokioRuntime};
use tokio_stream::StreamExt;

#[tokio::main]
async fn main() -> Result<()> {
    // Create peer connection with Tokio runtime
    let pc = PeerConnection::builder()
        .runtime(TokioRuntime)
        .ice_servers(vec![ice_server])
        .build()
        .await?;
    
    // Clean async operations
    let offer = pc.create_offer().await?;
    pc.set_local_description(offer).await?;
    
    // Handle events via streams
    tokio::spawn({
        let pc = pc.clone();
        async move {
            let mut tracks = pc.tracks();
            while let Some(track) = tracks.next().await {
                println!("New track: {}", track.id());
                // Handle track
            }
        }
    });
    
    tokio::spawn({
        let pc = pc.clone();
        async move {
            let mut candidates = pc.ice_candidates();
            while let Some(candidate) = candidates.next().await {
                // Send to peer
                send_to_peer(candidate).await?;
            }
        }
    });
    
    Ok(())
}
```

**Note**: Stream-based access may be provided as a convenience wrapper, but the trait-based handler is the proposed and recommended primary API for v0.20.0. **This design is still being finalized and is subject to community feedback.**

### Runtime Switching

The trait-based handler approach works with any runtime by changing the runtime parameter:

```rust
// Switch to async-std - just change runtime and feature flag!
use webrtc::{PeerConnection, runtime::AsyncStdRuntime};

#[async_std::main]
async fn main() -> Result<()> {
    let pc = PeerConnection::builder()
        .runtime(AsyncStdRuntime)  // Different runtime, same API!
        .build()
        .await?;
    
    // Everything else is identical
}
```

### Feature Flags

```toml
# webrtc/Cargo.toml
[features]
default = ["runtime-tokio"]

# Runtime options
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

**User dependencies**:

```toml
# Tokio (default)
[dependencies]
webrtc = "0.20"

# async-std
[dependencies]
webrtc = { version = "0.20", default-features = false, features = ["runtime-async-std"] }

# smol
[dependencies]
webrtc = { version = "0.20", default-features = false, features = ["runtime-smol"] }
```

### Wrapping Sans-I/O Core

The proposed async API would wrap the `rtc` Sans-I/O core using the trait-based handler pattern:

```rust
pub struct PeerConnection {
    runtime: Box<dyn Runtime>,
    inner: rtc::peer_connection::RTCPeerConnection,
    handler: Arc<Mutex<dyn PeerConnectionEventHandler>>,
}

impl PeerConnection {
    pub async fn create_offer(&self) -> Result<SessionDescription> {
        // Synchronously create offer via Sans-I/O core
        let offer = self.inner.create_offer(None)?;
        
        // Drive I/O loop with runtime
        self.drive_io().await?;
        
        Ok(offer)
    }
    
    async fn drive_io(&self) -> Result<()> {
        loop {
            // Send outgoing packets
            while let Some(msg) = self.inner.poll_write() {
                self.runtime.send_udp(msg).await?;
            }
            
            // Process events and dispatch to handler
            while let Some(event) = self.inner.poll_event() {
                let mut handler = self.handler.lock().await;
                match event {
                    Event::Track(track) => {
                        handler.on_track(track).await;
                    }
                    Event::IceCandidate(candidate) => {
                        handler.on_ice_candidate(candidate).await;
                    }
                    Event::StateChange(state) => {
                        handler.on_connection_state_change(state).await;
                    }
                    Event::DataChannel(dc) => {
                        handler.on_data_channel(dc).await;
                    }
                }
            }
                match event {
                    Event::DataChannel(dc) => {
                        handler.on_data_channel(dc).await;
                    }
                }
            }
            
            // Handle timeouts
            if let Some(timeout) = self.inner.poll_timeout() {
                self.runtime.sleep_until(timeout).await;
                self.inner.handle_timeout(Instant::now())?;
            }
            
            // Receive incoming packets
            tokio::select! {
                Ok((packet, addr)) = self.runtime.recv_udp() => {
                    self.inner.handle_read(packet, addr, Instant::now())?;
                }
                // ... other branches
            }
        }
    }
}
```

**Key aspects of the wrapper**:

1. **Sans-I/O core** (`rtc::peer_connection::RTCPeerConnection`) handles all protocol logic
2. **Runtime abstraction** (`Box<dyn Runtime>`) provides async I/O operations
3. **Event handler trait** dispatches events from Sans-I/O core to user-implemented handler
4. **I/O loop** (`drive_io`) coordinates:
   - Sending packets: `poll_write()` → `runtime.send_udp()`
   - Processing events: `poll_event()` → handler trait method dispatch
   - Timer management: `poll_timeout()` → `runtime.sleep_until()`
   - Receiving packets: `runtime.recv_udp()` → `handle_read()`

This proposed architecture provides clean separation between protocol logic (Sans-I/O), I/O operations (runtime abstraction), and application logic (trait handler).

---

## Benefits of This Proposed Architecture

### 1. Clean, Ergonomic APIs

**No more callback hell**:

```rust
// Trait-based approach - clean and simple
impl PeerConnectionEventHandler for MyHandler {
    async fn on_track(&mut self, track: Track) {
        handle_track(track).await;
    }
    
    async fn on_ice_candidate(&mut self, candidate: Option<RTCIceCandidate>) {
        send_to_peer(candidate).await;
    }
    
    // All events in one place, easy state coordination
}

// No Arc cloning, no Box::new, no triple nesting!
```

Benefits:
- Direct ownership without `Arc` explosion
- Clear, linear code flow
- Centralized state management with `&mut self`
- Predictable resource cleanup
- Natural coordination across multiple event types

### 2. Runtime Independence

**Support for multiple runtimes**:

- ✅ **Tokio** - Production servers, web services
- ✅ **async-std** - Alternative async runtime
- ✅ **smol** - Lightweight runtime
- ✅ **embassy** - Embedded systems (no_std)
- ✅ **Custom runtimes** - Implement `Runtime` trait

**Same protocol core, different I/O**:

```rust
// The rtc core doesn't change
let mut rtc_pc = rtc::RTCPeerConnection::new(config)?;

// Just wrap with different runtime
let pc_tokio = webrtc::PeerConnection::new(rtc_pc.clone(), TokioRuntime);
let pc_smol = webrtc::PeerConnection::new(rtc_pc.clone(), SmolRuntime);
```

### 3. Superior Testing

**Pure protocol tests**:

```rust
#[test]
fn test_ice_state_machine() {
    let mut pc = rtc::RTCPeerConnection::new(config)?;
    
    // No networking, no async runtime needed
    let packet = create_stun_binding_request();
    pc.handle_read(&packet, peer_addr, Instant::now())?;
    
    let response = pc.poll_write().unwrap();
    assert!(is_stun_binding_response(&response.data));
}
```

**Deterministic time**:

```rust
#[test]
fn test_ice_timeout() {
    let fixed_time = Instant::now();
    
    pc.handle_timeout(fixed_time)?;
    pc.handle_timeout(fixed_time + Duration::from_secs(5))?;
    
    // Fully deterministic, no flaky timing
}
```

### 4. Better Performance

**Optimizations at two levels**:

- **Protocol layer**: Optimize state machines, packet processing
- **I/O layer**: Optimize for specific runtime characteristics

**Zero-cost abstractions**:

- Trait object vtable indirection is minimal
- No hidden allocations in hot paths
- Compiler can optimize through trait boundaries

### 5. Ecosystem Alignment

**Modern Rust patterns**:

- ✅ Stable `async fn in trait`
- ✅ Builder pattern for configuration
- ✅ Type-safe error handling
- ✅ Zero-cost abstractions

**Framework integration**:

- Works with Actix Web, Axum, Warp, Rocket
- Compatible with any Tokio-based framework
- Integrates with async-std ecosystem

---

## Development Roadmap

### Phase 1: Foundation (Q1 2026) ✅ In Progress

**Goal**: Complete Sans-I/O core and design async API

**Completed:**
- [x] `rtc` crate feature parity with `webrtc`
- [x] W3C WebRTC API compliance (95%+)
- [x] Complete protocol stack
- [x] Interceptor framework
- [x] Stats API

**In Progress:**
- [ ] Runtime trait abstraction design
- [ ] Event handling API design (trait-based vs stream-based discussion)
- [ ] API design RFC and community feedback
- [ ] Proof-of-concept with Tokio

**Deliverables:**
- Design RFC for async-friendly API
- Community feedback and design iteration
- Proof-of-concept implementation

---

### Phase 2: API Design & Runtime Implementation (Q2 2026)

**Goal**: Finalize API design and implement runtime adapters

**Tasks:**

1. **Runtime Trait Finalization**
   - [ ] Define `Runtime` trait
   - [ ] Define `AsyncUdpSocket` trait
   - [ ] Define `AsyncTimer` trait
   - [ ] Type-erased wrappers

2. **Tokio Runtime** (default)
   - [ ] Implement `TokioRuntime`
   - [ ] UDP socket wrapper
   - [ ] Timer wrapper
   - [ ] Examples and tests

3. **Additional Runtimes**
   - [ ] `AsyncStdRuntime`
   - [ ] `SmolRuntime`
   - [ ] Documentation for custom runtimes

4. **API Design**
   - [ ] Trait-based event handlers
   - [ ] Builder pattern
   - [ ] Error types
   - [ ] Configuration types

**Deliverables:**
- Finalized API design
- Runtime trait specification
- `webrtc` v0.20.0-alpha

---

### Phase 3: Core Implementation (Q3 2026)

**Goal**: Complete async-friendly webrtc crate

**Tasks:**

1. **PeerConnection API**
   - [ ] Wrap `rtc::RTCPeerConnection`
   - [ ] I/O loop implementation
   - [ ] Async operations: `create_offer`, `create_answer`, etc.
   - [ ] Track management
   - [ ] Data channel creation

2. **DataChannel**
   - [ ] Async send/recv
   - [ ] Backpressure handling
   - [ ] Stream-based API (future)

3. **Media Tracks**
   - [ ] Track sender with media pipeline
   - [ ] Track receiver
   - [ ] RTP/RTCP handling

4. **Event Handling**
   - [ ] Trait-based handler dispatch
   - [ ] Proper lifetime management
   - [ ] No memory leaks

**Deliverables:**
- `webrtc` v0.20.0-beta
- Complete API implementation
- Examples for all runtimes

---

### Phase 4: Browser Interoperability & Testing (Q3-Q4 2026)

**Goal**: Production-ready quality

**Test Matrix:**

| Browser | Data Channels | Audio | Video | Simulcast |
|---------|--------------|-------|-------|-----------|
| Chrome | ☐ | ☐ | ☐ | ☐ |
| Firefox | ☐ | ☐ | ☐ | ☐ |
| Safari | ☐ | ☐ | ☐ | ☐ |
| Edge | ☐ | ☐ | ☐ | ☐ |

**Automated Testing:**
- [ ] GitHub Actions workflow
- [ ] Selenium/Playwright tests
- [ ] Cross-platform (Linux, macOS, Windows)
- [ ] Performance benchmarks
- [ ] Memory leak detection

**Deliverables:**
- Browser compatibility report
- Automated test suite in CI
- `webrtc` v0.20.0-rc

---

### Phase 5: Production Release & Documentation (Q4 2026)

**Goal**: Stable release with comprehensive docs

**Documentation:**
- [ ] Getting Started guide
- [ ] Migration guide from v0.17.x
- [ ] Runtime selection guide
- [ ] API reference (rustdoc)
- [ ] Architecture overview
- [ ] Performance tuning guide

**Examples:**
- [ ] Simple peer-to-peer data channel
- [ ] Audio/video streaming
- [ ] Multi-party conferencing
- [ ] File transfer
- [ ] Game networking

**Deliverables:**
- `webrtc` v0.20.0 (stable)
- Complete documentation
- 10+ working examples
- Migration guide and tools

---

## Migration from v0.17.x

### Timeline

**Q2 2026**: Preparation
- Pin to `webrtc = "0.17.x"`
- Review migration guide
- Experiment with v0.20.0-alpha

**Q3-Q4 2026**: Gradual migration
- Update to v0.20.0-beta
- Migrate one module at a time
- Replace callbacks with async streams
- Test thoroughly

**2027**: Adopt new architecture
- Update to v0.20.0 stable
- Remove compatibility shims
- Optimize for new patterns

### API Comparison

**v0.17.x (old)**:
```rust
let api = APIBuilder::new().build();
let pc = Arc::new(api.new_peer_connection(config).await?);

let pc_clone = pc.clone();
pc.on_peer_connection_state_change(Box::new(move |s| {
    let pc = pc_clone.clone();  // More cloning...
    Box::pin(async move {
        println!("State: {s}");
    })
}));
```

**v0.20.0 (new)**:
```rust
use tokio_stream::StreamExt;

let pc = PeerConnection::builder()
    .runtime(TokioRuntime)
    .build()
    .await?;

// Handle events via streams - no cloning needed!
tokio::spawn(async move {
    let mut states = pc.connection_states();
    while let Some(state) = states.next().await {
        println!("State: {state}");
    }
});
```

### Compatibility Promise

- **v0.17.x**: Bug fixes through 2026
- **v0.20.0**: Migration guide provided
- **Compatibility layer**: 6 months after release
- **Breaking changes**: Well documented

---

## Success Metrics

### Technical Goals

| Metric | v0.17.x | v0.20.0 Target |
|--------|---------|---------------|
| Memory leak per connection | ~109 KiB | 0 KiB |
| Supported runtimes | 1 (Tokio) | 4+ |
| Browser interop | Partial | 100% |
| Connection setup time | ~2s | <1s |
| DataChannel throughput | ~300 Mbps | >500 Mbps |
| Test coverage | ~60% | >80% |

### Community Goals

- 100+ GitHub stars for async webrtc
- 10+ community-contributed examples
- 5+ production deployments
- Active Discord discussions
- Conference talks and blog posts

---

## Get Involved

We're actively working on this and would love your input:

**Current Phase (Q1 2026):**
- Review API design RFC
- Provide feedback on architecture
- Test experimental implementations
- Report bugs in v0.17.x

**How to Contribute:**
- **Design**: Comment on API design discussions
- **Implementation**: Help implement runtime adapters
- **Testing**: Browser interoperability testing
- **Documentation**: Write guides and examples

**Communication:**
- **GitHub Discussions**: https://github.com/webrtc-rs/webrtc/discussions
- **Discord**: https://discord.gg/4Ju8UHdXMs
- **Blog**: Regular progress updates

---

## Conclusion

The shift to an async-friendly architecture represents the next evolution for webrtc-rs. This design proposal:

1. **Learns from Quinn** - Adopts proven runtime abstraction patterns
2. **Builds on Sans-I/O** - Leverages our complete `rtc` protocol core
3. **Supports multiple runtimes** - Via clean trait abstraction
4. **Proposes the right pattern** - Trait-based handlers fit WebRTC's complex event model
5. **Uses modern async patterns** - Native async trait methods, builder APIs
6. **Eliminates callback hell** - Clean ownership and lifecycle management

We're designing a truly world-class WebRTC implementation in Rust that will be:
- **Runtime agnostic** - Works anywhere Rust runs (Tokio, async-std, smol, embassy)
- **Memory safe** - No leaks, clear ownership semantics
- **Performant** - Zero-cost abstractions over Sans-I/O core
- **Ergonomic** - Clean, intuitive APIs for complex WebRTC use cases
- **Well tested** - Comprehensive browser interop and deterministic testing

**This is a design proposal—we welcome your feedback!** Join the discussion on [GitHub](https://github.com/webrtc-rs/webrtc) or our [Discord](https://discord.gg/4Ju8UHdXMs).

The future of webrtc-rs will be async-friendly, runtime-agnostic, and built on solid Sans-I/O foundations with APIs designed specifically for WebRTC's complexity. 🦀

---

## References

- **Quinn**: https://github.com/quinn-rs/quinn
- **rtc crate**: https://github.com/webrtc-rs/rtc
- **Sans-I/O pattern**: https://sans-io.readthedocs.io/
- **W3C WebRTC**: https://www.w3.org/TR/webrtc/
- **webrtc v0.17.0 release**: https://github.com/webrtc-rs/webrtc/releases/tag/v0.17.0

---

*Follow webrtc-rs development on [GitHub](https://github.com/webrtc-rs/webrtc) and join our [Discord](https://discord.gg/4Ju8UHdXMs) community!*
