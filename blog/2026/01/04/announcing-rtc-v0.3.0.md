# Announcing `rtc` 0.3.0: Sans-I/O WebRTC Stack for Rust 🎉

I'm excited to announce the first public release of **[rtc](https://github.com/webrtc-rs/rtc)**, a pure Rust WebRTC implementation built on a **sans-I/O architecture**.

## What is Sans-I/O?

Sans-I/O (without I/O) is a design pattern where the library handles all protocol logic, but **you** control the I/O operations. Instead of the library directly performing network reads and writes, you feed it data and it tells you what to send back.

Think of it like a state machine: you drive the I/O loop while the library handles WebRTC's complex protocol details (ICE, DTLS, SRTP, SCTP, SDP, etc.).

## Why Sans-I/O for WebRTC?

The existing [`webrtc`](https://github.com/webrtc-rs/webrtc) crate (async/await based) is excellent, but it has some inherent limitations:

**Limitations of Async-Based Approach:**
- 🔒 **Runtime Lock-in** - Tightly coupled to Tokio
- 🧵 **Hidden Threading** - Internal task spawning you can't control  
- 🎭 **Black Box I/O** - Can't intercept or customize network behavior
- 🧪 **Testing Challenges** - Requires actual network for protocol tests
- 🔌 **Integration Friction** - Hard to embed in existing event loops

**Benefits of Sans-I/O:**
- 🚀 **Runtime Independent** - Works with tokio, async-std, smol, or even blocking I/O
- 🎯 **Full Control** - You control threading, scheduling, and I/O multiplexing
- 🧪 **Testable** - Protocol logic testable without real network I/O
- 🔌 **Flexible** - Easy integration with existing networking code
- 📊 **Observable** - Complete visibility into protocol state and events
- ⚡ **Zero Copy** - Efficient buffer management with `bytes` crate

## Simple API Example

The core API is straightforward - a simple event loop with six core methods:

1. **`poll_write()`** - Get outgoing network packets to send via UDP
2. **`poll_event()`** - Process connection state changes and notifications
3. **`poll_read()`** - Get incoming application messages (RTP, RTCP, data)
4. **`poll_timeout()`** - Get next timer deadline for retransmissions/keepalives
5. **`handle_read()`** - Feed incoming network packets into the connection
6. **`handle_timeout()`** - Notify about timer expiration

Additional methods for external control:

* handle_write() - Queue application messages (RTP/RTCP/data) for sending
* handle_event() - Inject external events into the connection

```rust
use rtc::peer_connection::RTCPeerConnection;
use rtc::peer_connection::configuration::RTCConfigurationBuilder;
use rtc::peer_connection::event::{RTCPeerConnectionEvent, RTCTrackEvent};
use rtc::peer_connection::state::RTCPeerConnectionState;
use rtc::peer_connection::message::RTCMessage;
use rtc::peer_connection::sdp::RTCSessionDescription;
use rtc::shared::{TaggedBytesMut, TransportContext, TransportProtocol};
use rtc::sansio::Protocol;
use std::time::{Duration, Instant};
use tokio::net::UdpSocket;
use bytes::BytesMut;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Setup peer connection
    let config = RTCConfigurationBuilder::new().build();
    let mut pc = RTCPeerConnection::new(config)?;

    // Signaling: Create offer and set local description
    let offer = pc.create_offer(None)?;
    pc.set_local_description(offer.clone())?;

    // TODO: Send offer.sdp to remote peer via your signaling channel
    // signaling_channel.send_offer(&offer.sdp).await?;

    // TODO: Receive answer from remote peer via your signaling channel
    // let answer_sdp = signaling_channel.receive_answer().await?;
    // let answer = RTCSessionDescription::answer(answer_sdp)?;
    // pc.set_remote_description(answer)?;

    // Bind UDP socket
    let socket = UdpSocket::bind("0.0.0.0:0").await?;
    let local_addr = socket.local_addr()?;
    let mut buf = vec![0u8; 2000];

    'EventLoop: loop {
        // 1. Send outgoing packets
        while let Some(msg) = pc.poll_write() {
            socket.send_to(&msg.message, msg.transport.peer_addr).await?;
        }

        // 2. Handle events
        while let Some(event) = pc.poll_event() {
            match event {
                RTCPeerConnectionEvent::OnConnectionStateChangeEvent(state) => {
                    println!("Connection state: {state}");
                    if state == RTCPeerConnectionState::Failed {
                        return Ok(());
                    }
                }
                RTCPeerConnectionEvent::OnTrack(RTCTrackEvent::OnOpen(init)) => {
                    println!("New track: {}", init.track_id);
                }
                _ => {}
            }
        }

        // 3. Handle incoming messages
        while let Some(message) = pc.poll_read() {
            match message {
                RTCMessage::RtpPacket(track_id, packet) => {
                    println!("RTP packet on track {track_id}");
                }
                RTCMessage::DataChannelMessage(channel_id, msg) => {
                    println!("Data channel message");
                }
                _ => {}
            }
        }

        // 4. Handle timeouts
        let timeout = pc.poll_timeout()
            .unwrap_or(Instant::now() + Duration::from_secs(86400));
        let delay = timeout.saturating_duration_since(Instant::now());

        if delay.is_zero() {
            pc.handle_timeout(Instant::now())?;
            continue;
        }

        // 5. Multiplex I/O
        tokio::select! {
            _ = stop_rx.recv() => {
                break 'EventLoop,
            } 
            _ = tokio::time::sleep(delay) => {
                pc.handle_timeout(Instant::now())?;
            }
            Ok(message) = message_rx.recv() => {
                pc.handle_write(message)?;
            }
            Ok(event) = event_rx.recv() => {
                pc.handle_event(event)?;
            }
            Ok((n, peer_addr)) = socket.recv_from(&mut buf) => {
                pc.handle_read(TaggedBytesMut {
                    now: Instant::now(),
                    transport: TransportContext {
                        local_addr,
                        peer_addr,
                        ecn: None,
                        transport_protocol: TransportProtocol::UDP,
                    },
                    message: BytesMut::from(&buf[..n]),
                })?;
            }
        }
    }

    pc.close()?;

    Ok(())
}
```

## Feature Parity Status

The `rtc` crate is nearly feature-complete compared to the `webrtc` crate:

✅ **Complete:**
- ICE (Interactive Connectivity Establishment)
- DTLS (Datagram Transport Layer Security)  
- SRTP/SRTCP (Secure RTP/RTCP)
- SCTP (Stream Control Transmission Protocol)
- Data Channels (reliable & unreliable)
- RTP/RTCP (Real-time Transport Protocol)
- Media Tracks (audio & video)
- SDP (Session Description Protocol)
- Peer Connection API
- Media Streams API

🚧 **In Progress:**
- Simulcast support
- RTCP feedback handling (interceptors)

## Architecture Highlights

- **14+ workspace crates** - Modular design (rtc-ice, rtc-dtls, rtc-srtp, etc.)
- **Zero unsafe** - Pure safe Rust implementation
- **Comprehensive docs** - 215+ passing doc tests
- **W3C compliant** - Follows WebRTC and Media Capture specs
- **RFC compliant** - Implements ICE, DTLS, SRTP, SCTP standards

## Use Cases

Sans-I/O architecture shines when you need:

- **Custom networking** - Non-standard transports, custom protocols
- **Embedded systems** - No runtime overhead, precise control
- **Game engines** - Integration with existing game loops
- **High performance** - Fine-tuned I/O scheduling and batching
- **Testing infrastructure** - Deterministic protocol testing
- **Special environments** - WebAssembly, no_std (future), embedded

## Getting Started

```toml
[dependencies]
rtc = "0.3.0"
```

Check out the [documentation](https://docs.rs/rtc) and [examples](https://github.com/webrtc-rs/rtc/tree/master/examples) to get started!

## Relationship with `webrtc` Crate

The `rtc` (sans-I/O) and `webrtc` (async) crates are **complementary, not competitive**:

- **Use `webrtc`** if you want async/await, Tokio integration, and quick start
- **Use `rtc`** if you need runtime independence, custom I/O, or maximum control

Both are actively maintained by the WebRTC.rs project and share the same underlying protocol implementations where possible.

## Future Plans

- Complete simulcast support
- RTCP interceptor framework  
- Performance optimizations
- More examples and documentation
- Potential `no_std` support for embedded systems

## Links

- **GitHub**: https://github.com/webrtc-rs/rtc
- **Crate**: https://crates.io/crates/rtc
- **Docs**: https://docs.rs/rtc
- **Discord**: https://discord.gg/4Ju8UHdXMs

Feedback, questions, and contributions are welcome! 🦀

---

*This release represents months of work redesigning WebRTC for maximum flexibility. Special thanks to all contributors and the Rust WebRTC community!*
