# Building WebRTC’s Pipeline with `sansio::Protocol`: A Transport-Agnostic Approach

## Introduction

WebRTC is not a single protocol but a *stack* of tightly interrelated protocols: ICE for connectivity, DTLS for
security, SCTP for reliable data channels, and SRTP for media transport. Each layer has its own state machine, timers,
retransmission logic, and failure modes. Traditionally, these protocols are implemented alongside socket I/O and async
runtimes, leading to deeply intertwined code that is difficult to test, reason about, or adapt to new transports.

This article explores an alternative approach: building WebRTC as a **pure protocol pipeline** using the *sans-I/O*
pattern. By separating protocol logic from all I/O concerns, we can model WebRTC as a sequence of composable handlers,
each acting as a deterministic state machine. The result is a transport-agnostic, testable, and extensible WebRTC
implementation.

The design is built on the `sansio::Protocol` trait from [`webrtc-rs/sansio`](https://github.com/webrtc-rs/sansio), and
the concrete pipeline implementation lives in the WebRTC peer connection handlers under [
`webrtc-rs/rtc/src/peer_connection/handler`](https://github.com/webrtc-rs/rtc/tree/master/rtc/src/peer_connection/handler).

---

## The Sans-I/O Model

A *sans-I/O* protocol does not perform any input/output operations. Instead, it:

- **Consumes inputs** (messages, events, timestamps)
- **Mutates internal state**
- **Emits outputs** that the runtime may choose to send, schedule, or drop

All side effects are explicit and observable.

This separation gives us three key properties:

- **Determinism** – protocol logic is pure and replayable
- **Testability** – no sockets, timers, or async runtimes required
- **Transport independence** – UDP, TCP, QUIC, or in-memory buffers all work

---

## The `sansio::Protocol` Trait

At the core of this approach is the `sansio::Protocol` trait:

```rust
trait Protocol<Rin, Rout, Eout> {
    type Rout;
    type Wout;
    type Eout;
    type Error;
    type Time;

    fn handle_read(&mut self, msg: Rin) -> Result<(), Self::Error>;
    fn poll_read(&mut self) -> Option<Self::Rout>;

    fn handle_write(&mut self, msg: Rout) -> Result<(), Self::Error>;
    fn poll_write(&mut self) -> Option<Self::Wout>;

    fn handle_event(&mut self, evt: Eout) -> Result<(), Self::Error>;
    fn poll_event(&mut self) -> Option<Self::Eout>;

    fn handle_timeout(&mut self, now: Self::Time) -> Result<(), Self::Error>;
    fn poll_timeout(&mut self) -> Option<Self::Time>;

    fn close(&mut self) -> Result<(), Self::Error>;
}
```

Each protocol handler is a **state machine** with a uniform interface. It does not block, does not allocate sockets, and
does not depend on an async runtime. Instead, it transforms messages and queues outputs for the next layer in the
pipeline.

---

## A Pipeline, Not a Call Stack

WebRTC is naturally layered, which makes it an excellent fit for a pipeline model.

Think of the stack as two conveyor belts:

- **Read path**: bytes → validated, decrypted, structured messages
- **Write path**: structured messages → encrypted bytes

### Read Path

```
Raw Bytes → Demuxer → ICE → DTLS → SCTP → DataChannel → SRTP → Interceptor → Endpoint → Application
```

### Write Path

```
Application → Endpoint → Interceptor → SRTP → DataChannel → SCTP → DTLS → ICE → Demuxer → Raw Bytes
```

Each handler has **exactly one responsibility** and never calls another handler directly.

---

## Handler Breakdown

### 1. Demuxer Handler

The Demuxer is the entry point for all incoming packets. Following RFC 7983, it inspects the first byte to classify
packets as STUN, DTLS, or RTP/RTCP.

```rust
fn match_dtls(b: &[u8]) -> bool {
    match_range(20, 63, b)
}

fn match_srtp(b: &[u8]) -> bool {
    match_range(128, 191, b)
}
```

On read, it transforms `Raw` bytes into categorized messages:

- `[20..63]` → `DTLSMessage::Raw`
- `[128..191]` → `RTPMessage::Raw`
- Everything else → `STUNMessage::Raw`

On write, it simply unwraps all protocol messages back to raw bytes.

This handler is intentionally **stateless**. Given the same input bytes, it always produces the same classification,
making it ideal for fuzzing and offline testing.

---

### 2. ICE Handler

The ICE handler manages connectivity establishment using STUN.

Before candidate nomination, it actively consumes STUN messages and drives the ICE agent. After a candidate pair is
selected, the handler undergoes a **phase transition** and becomes a transparent transport adaptor, forwarding all
non-STUN traffic unchanged.

All address selection and connectivity checks are encapsulated within this handler, keeping transport concerns isolated
from higher protocol layers.

---

### 3. DTLS Handler

The DTLS handler provides authentication, encryption, and key agreement.

On the read path, it processes DTLS records and advances the handshake state machine. When the handshake completes,
keying material is exported using the TLS exporter and transformed into SRTP contexts.

```rust
fn handle_read(&mut self, msg: TaggedRTCMessageInternal) -> Result<()> {
    if let RTCMessageInternal::Dtls(DTLSMessage::Raw(dtls_message)) = msg.message {
        //...
        EndpointEvent::HandshakeComplete => {
            let (local_srtp_context, remote_srtp_context) =
                DtlsHandler::update_srtp_contexts(state, replay_protection)?;
            self.ctx.event_outs.push_back(
                RTCEventInternal::DTLSHandshakeComplete(peer_addr, Some(local_srtp_context), Some(remote_srtp_context))
            );
        }
        //...
    }
    Ok(())
}
```

Until this event occurs, the SRTP handler remains inactive. DTLS therefore forms a strict dependency boundary in the
pipeline.

---

### 4. SCTP Handler

The SCTP handler implements reliable and unordered delivery for data channels.

It manages:

- SCTP **associations** (roughly equivalent to connections)
- Multiple **streams** per association
- Flow control and retransmissions
- DCEP (Data Channel Establishment Protocol)

WebRTC data channels map directly to SCTP streams, not to separate connections.

---

### 5. DataChannel Handler

The DataChannel handler introduces WebRTC-specific semantics.

It maps SCTP streams to WebRTC data channels, handles open and close negotiation, and interprets messages based on their
PPI (Payload Protocol Identifier).

Below this layer, messages are generic framed bytes. Above it, messages have meaning: text vs binary payloads, lifecycle
events, and application-level semantics.

---

### 6. SRTP Handler

The SRTP handler encrypts and decrypts RTP and RTCP packets using keys derived from DTLS.

```rust
let mut decrypted = context.decrypt_rtp( & message) ?;
let rtp_packet = rtp::Packet::unmarshal( & mut decrypted) ?;
```

Replay protection, rollover counters, and packet authentication are fully encapsulated within this handler. Because it
is sans-I/O, packet reordering and duplication can be tested deterministically.

---

### 7. Interceptor Handler

The Interceptor handler is an intentional extension point.

RTCP packets terminate here, allowing future interceptors to implement:

- NACK, PLI, and FIR feedback
- Congestion control algorithms
- Bandwidth estimation
- Statistics and observability

This keeps experimental or policy-driven logic out of core protocol handlers.

---

### 8. Endpoint Handler

The Endpoint handler bridges protocol logic and application logic.

It maps SSRCs to tracks, emits `OnTrack` and `OnDataChannel` events, and routes RTP packets to the correct media tracks.
This is the final stage where protocol messages become application-visible events.

---

## Orchestrating the Pipeline

`RTCPeerConnection` itself implements `sansio::Protocol` and acts as the **orchestrator** for all handlers. Handlers never
call each other directly. The macros `for_each_handler!` with `forward` and `reverse` parameters ensure messages flow
through the pipeline in the correct order.

```rust
for_each_handler!(forward: process_handler!(self, handler, {
    while let Some(msg) = intermediate.pop_front() {
        handler.handle_read(msg)?;
    }
    while let Some(msg) = handler.poll_read() {
        intermediate.push_back(msg);
    }
}));
```

The write path runs the same handlers in reverse order. This symmetry ensures that every transformation on the read path
has a clear inverse on the write path.

---

## Benefits of the sansio Approach

### 1. **Transport Agnostic**

The same protocol handlers work with UDP, TCP, QUIC, or even in-memory buffers for testing.

### 2. **Testable**

Each handler can be tested in isolation by feeding it messages and verifying outputs:

```rust
#[test]
fn test_demuxer() {
    let mut ctx = DemuxerHandlerContext::default();
    let mut demuxer = DemuxerHandler::new(&mut ctx);

    // Test DTLS packet demuxing
    let dtls_packet = vec![22, /* ... */]; // Content Type = 22 (handshake)
    demuxer.handle_read(TaggedRTCMessageInternal {
        message: RTCMessageInternal::Raw(BytesMut::from(&dtls_packet[..])),
        ..
    })?;

    let output = demuxer.poll_read().unwrap();
    assert!(matches!(output.message, RTCMessageInternal::Dtls(_)));
}
```

### 3. **Composable**

Handlers can be reordered, replaced, or extended without affecting others. Want to add custom encryption? Insert a
handler between DTLS and SCTP.

### 4. **Clear Separation of Concerns**

Each handler has a single responsibility and well-defined interfaces.

### 5. **Event-Driven**

The polling model (`poll_read`, `poll_write`, `poll_event`, `poll_timeout`) integrates naturally with async runtimes
without requiring async/await in protocol logic.

## Message Flow Example

Let's trace a data channel message from application to wire:

1. **Application** writes: `RTCMessage::DataChannelMessage(id, data)`
2. **Endpoint** converts to: `DTLSMessage::DataChannel(ApplicationMessage)`
3. **Interceptor** passes through (no-op for data channels)
4. **SRTP** passes through (not RTP/RTCP)
5. **DataChannel** converts to: `DTLSMessage::Sctp(DataChannelMessage)` with PPI
6. **SCTP** fragments and adds headers: `DTLSMessage::Raw(sctp_packet)`
7. **DTLS** encrypts: `DTLSMessage::Raw(encrypted_packet)`
8. **ICE** adds addresses: same packet with transport context
9. **Demuxer** unwraps to: `RTCMessageInternal::Raw(bytes)`
10. **Transport** sends `TaggedBytesMut` to network

And on the read path, the process reverses!

---

## Conclusion

The `sansio::Protocol` trait provides an elegant foundation for building complex protocol stacks like WebRTC. By
separating protocol logic from I/O, we achieve:

- Clean, testable code
- Transport independence
- Easy extensibility
- Clear architectural boundaries

This approach transforms WebRTC from a monolithic implementation into a composable pipeline of handlers, each focused on
a specific protocol layer. Whether you're building a WebRTC implementation, a custom signaling protocol, or any layered
network stack, the sansio pattern is worth considering.

The full implementation can be explored in the [rtc repository](https://github.com/webrtc-rs/rtc), demonstrating how
these principles scale to production-quality WebRTC.

---

**Key Takeaways:**

- sansio separates protocol logic from I/O operations
- Each handler implements a simple, uniform interface
- Messages flow through a pipeline of transformations
- The architecture is testable, composable, and transport-agnostic
- WebRTC's complexity becomes manageable through clear layering
