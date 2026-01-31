# Quinn Architecture Analysis: Lessons for webrtc-rs

## Key Discovery

Quinn does NOT create separate crates like `quinn-tokio`, `quinn-async-std`, `quinn-smol`. Instead, they use a **single `quinn` crate** with **runtime trait abstraction** and **feature flags**.

## Quinn's Architecture

### Crate Structure

```
quinn/
├── quinn-proto/          # Sans-I/O protocol implementation (no I/O, no async)
├── quinn-udp/            # Platform-specific UDP optimizations
└── quinn/                # Async API (runtime-agnostic via traits)
    └── src/
        └── runtime/
            ├── mod.rs    # Runtime trait definitions
            ├── tokio.rs  # Tokio implementation
            └── smol.rs   # smol implementation
```

### Runtime Trait Abstraction

Quinn defines a `Runtime` trait in `quinn/src/runtime/mod.rs`:

```rust
pub trait Runtime: Send + Sync + Debug + 'static {
    fn new_timer(&self, i: Instant) -> Pin<Box<dyn AsyncTimer>>;
    fn spawn(&self, future: Pin<Box<dyn Future<Output = ()> + Send>>);
    fn wrap_udp_socket(&self, t: std::net::UdpSocket) -> io::Result<Box<dyn AsyncUdpSocket>>;
    fn now(&self) -> Instant;
}
```

### Feature Flags for Runtime Selection

From `quinn/Cargo.toml`:

```toml
[features]
default = ["runtime-tokio", "rustls-ring"]
runtime-tokio = ["tokio"]
runtime-smol = ["smol", "async-io"]

[dependencies]
tokio = { workspace = true }
smol = { workspace = true, optional = true }
async-io = { workspace = true, optional = true }
```

### Concrete Implementations

**Tokio Runtime** (`runtime/tokio.rs`):
```rust
#[derive(Debug)]
pub struct TokioRuntime;

impl Runtime for TokioRuntime {
    fn spawn(&self, future: Pin<Box<dyn Future<Output = ()> + Send>>) {
        tokio::spawn(future);
    }
    
    fn wrap_udp_socket(&self, sock: std::net::UdpSocket) -> io::Result<Box<dyn AsyncUdpSocket>> {
        Ok(Box::new(tokio::net::UdpSocket::from_std(sock)?))
    }
}
```

**smol Runtime** (`runtime/smol.rs`):
```rust
#[derive(Debug)]
pub struct SmolRuntime;

impl Runtime for SmolRuntime {
    fn spawn(&self, future: Pin<Box<dyn Future<Output = ()> + Send>>) {
        ::smol::spawn(future).detach();
    }
    
    fn wrap_udp_socket(&self, sock: std::net::UdpSocket) -> io::Result<Box<dyn AsyncUdpSocket>> {
        Ok(Box::new(Async::new_nonblocking(sock)?))
    }
}
```

## Why This Approach is Better

### 1. Single Crate, Multiple Runtimes
- Users install `quinn = { version = "...", features = ["runtime-tokio"] }`
- OR `quinn = { version = "...", features = ["runtime-smol"] }`
- No need for separate `quinn-tokio`, `quinn-async-std`, etc.

### 2. Shared Code
- All runtime-agnostic code lives in `quinn/`
- Only runtime-specific adapters differ
- Less code duplication
- Easier maintenance

### 3. Type Erasure via Trait Objects
- `Box<dyn Runtime>` allows runtime selection at instantiation
- `Box<dyn AsyncUdpSocket>` hides runtime-specific socket types
- Zero-cost at runtime (vtable indirection is minimal)

### 4. Simpler API
```rust
// User code - runtime-agnostic!
let endpoint = Endpoint::builder()
    .runtime(TokioRuntime)  // or SmolRuntime
    .bind(...)?;
```

## Applying This to webrtc-rs

### Current Roadmap Issue

The roadmap proposed:
```
webrtc/                  # Re-export from webrtc-tokio
webrtc-core/             # Runtime-agnostic types
webrtc-tokio/            # Tokio runtime adapter
webrtc-async-std/        # async-std runtime adapter  
webrtc-smol/             # smol runtime adapter
webrtc-embassy/          # Embassy runtime adapter
```

**Problem**: Too many crates, fragmentation, duplication.

### Better Approach (Quinn-style)

```
rtc/                     # Sans-I/O core (already exists!)
webrtc/                  # Async-friendly API with runtime abstraction
    └── src/
        └── runtime/
            ├── mod.rs         # Runtime trait
            ├── tokio.rs       # Tokio impl
            ├── async_std.rs   # async-std impl
            ├── smol.rs        # smol impl
            └── embassy.rs     # embassy impl
```

### Feature Flags

```toml
[features]
default = ["runtime-tokio"]
runtime-tokio = ["tokio"]
runtime-async-std = ["async-std"]
runtime-smol = ["smol"]
runtime-embassy = ["embassy-executor"]
```

### Runtime Trait for WebRTC

```rust
pub trait Runtime: Send + Sync + Debug + 'static {
    // Task spawning
    fn spawn(&self, future: Pin<Box<dyn Future<Output = ()> + Send>>);
    
    // UDP socket abstraction
    fn wrap_udp_socket(&self, sock: std::net::UdpSocket) -> io::Result<Box<dyn AsyncUdpSocket>>;
    
    // Timer abstraction
    fn sleep(&self, duration: Duration) -> Pin<Box<dyn Future<Output = ()> + Send>>;
    
    // Time
    fn now(&self) -> Instant;
}

pub trait AsyncUdpSocket: Send + Sync + Debug {
    fn poll_recv(&mut self, cx: &mut Context, bufs: &mut [IoSliceMut]) -> Poll<io::Result<usize>>;
    fn poll_send(&mut self, cx: &mut Context, buf: &[u8]) -> Poll<io::Result<usize>>;
    fn local_addr(&self) -> io::Result<SocketAddr>;
}
```

### User API

```rust
use webrtc::{PeerConnection, runtime::TokioRuntime};

#[tokio::main]
async fn main() {
    let pc = PeerConnection::builder()
        .runtime(TokioRuntime)
        .ice_servers(servers)
        .build()
        .await?;
    
    // All operations are now async-friendly!
    let offer = pc.create_offer().await?;
    pc.set_local_description(offer).await?;
}
```

Or with async-std:

```rust
use webrtc::{PeerConnection, runtime::AsyncStdRuntime};

#[async_std::main]
async fn main() {
    let pc = PeerConnection::builder()
        .runtime(AsyncStdRuntime)  // Different runtime, same API!
        .build()
        .await?;
}
```

## Advantages

### 1. Single Crate
- Users only need `webrtc` crate
- Select runtime via feature flags
- Less confusion, easier documentation

### 2. Less Maintenance
- Shared code in one place
- Only runtime-specific adapters differ
- Bug fixes benefit all runtimes

### 3. Better Testing
- Test with different runtimes using feature flags
- Catch runtime-specific issues early

### 4. Cleaner Dependencies
```toml
# User's Cargo.toml
[dependencies]
webrtc = { version = "0.20", features = ["runtime-tokio"] }
```

NOT:
```toml
# Avoid fragmentation
webrtc-tokio = "0.20"
webrtc-async-std = "0.20"
webrtc-smol = "0.20"
```

## Implementation Strategy

### Phase 1: Define Runtime Trait
```rust
// webrtc/src/runtime/mod.rs
pub trait Runtime { ... }
```

### Phase 2: Implement for Tokio
```rust
// webrtc/src/runtime/tokio.rs
#[cfg(feature = "runtime-tokio")]
pub struct TokioRuntime;

impl Runtime for TokioRuntime { ... }
```

### Phase 3: Wrap rtc Sans-I/O Core
```rust
// webrtc/src/peer_connection.rs
pub struct PeerConnection {
    runtime: Box<dyn Runtime>,
    inner: rtc::peer_connection::RTCPeerConnection,
}

impl PeerConnection {
    pub async fn create_offer(&self) -> Result<SessionDescription> {
        // Drive sans-I/O protocol with runtime I/O
        loop {
            // Poll rtc core
            if let Some(msg) = self.inner.poll_write() {
                self.runtime.send_udp(msg).await?;
            }
            
            // Handle timeouts
            self.runtime.sleep(timeout).await;
        }
    }
}
```

### Phase 4: Add More Runtimes
- async-std
- smol  
- embassy (for embedded)

## Comparison with Original Plan

| Aspect | Original Plan | Quinn-style Approach |
|--------|--------------|---------------------|
| Crate count | 5+ crates | 2 crates (rtc + webrtc) |
| Code duplication | High | Minimal |
| User experience | Choose crate | Choose feature flag |
| Maintenance | Complex | Simpler |
| Testing | Per-crate | Feature-flag based |
| Documentation | Scattered | Centralized |

## Conclusion

Quinn's approach is **significantly simpler and more maintainable**:

1. **Single `webrtc` crate** with runtime trait abstraction
2. **Feature flags** for runtime selection
3. **Trait objects** for runtime polymorphism
4. **Minimal code duplication** - only adapters differ

This should be the model for webrtc-rs v0.20.0!

## References

- Quinn repository: https://github.com/quinn-rs/quinn
- Quinn runtime abstraction: https://github.com/quinn-rs/quinn/tree/main/quinn/src/runtime
- Sans-I/O pattern: https://sans-io.readthedocs.io/
