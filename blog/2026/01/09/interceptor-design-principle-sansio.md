# Interceptor Design Principle: Composable RTP/RTCP Processing with `sansio::Protocol`

## Introduction

In the [previous article](/blog/2026/01/04/building-webrtc-pipeline-with-sansio.html), we explored how WebRTC can be modeled as a **pure protocol pipeline** using the sans-I/O pattern. Each protocol layer—ICE, DTLS, SCTP, SRTP—became a composable handler implementing the `sansio::Protocol` trait. This article continues that journey by examining a critical component of `RTC`: the **Interceptor**.

Interceptors sit between the SRTP layer and the application endpoint, processing RTP and RTCP packets to implement features like:

- **NACK** (Negative Acknowledgement, RFC 4585) for packet loss recovery
- **RTX** (Retransmission, RFC 4588) for packet retransmission
- **TWCC** (Transport-Wide Congestion Control) for bandwidth estimation
- **RTCP Reports** (Sender/Receiver Reports, RFC 3550) for quality metrics
- Custom feedback or experimental extensions

The challenge is designing an interceptor framework that is both **composable** and **efficient**. The original async-based interceptor in [webrtc-rs/webrtc](https://github.com/webrtc-rs/webrtc/tree/master/interceptor) served its purpose but had architectural limitations that became apparent as the project evolved.

This article explains the re-design of the interceptor framework using the `sansio::Protocol` trait and generic composition, as implemented in [rtc-interceptor](https://github.com/webrtc-rs/rtc/tree/master/rtc-interceptor).

### Prerequisites

This article builds on concepts introduced in:

- **[Building WebRTC's Pipeline with `sansio::Protocol`](/blog/2026/01/04/building-webrtc-pipeline-with-sansio.html)** — Covers the `sansio::Protocol` trait, the polling model (`handle_*`/`poll_*` methods), and how WebRTC's protocol layers are composed as a pipeline.

Familiarity with the following is helpful but not required:

- RTP/RTCP packet structure and SSRC identifiers
- RTCP feedback mechanisms (NACK, Sender/Receiver Reports)
- Rust generics and trait bounds

---

## The Old Async-Based Interceptor

Before diving into the new design, let's examine the original async-based interceptor architecture to understand its limitations.

### Architecture Overview

The old interceptor used a **wrapper/decorator chain pattern** with async traits:

```rust
#[async_trait]
pub trait Interceptor: Send + Sync {
    async fn bind_rtcp_reader(
        &self,
        reader: Arc<dyn RTCPReader + Send + Sync>,
    ) -> Arc<dyn RTCPReader + Send + Sync>;

    async fn bind_rtcp_writer(
        &self,
        writer: Arc<dyn RTCPWriter + Send + Sync>,
    ) -> Arc<dyn RTCPWriter + Send + Sync>;

    async fn bind_local_stream(
        &self,
        info: &StreamInfo,
        writer: Arc<dyn RTPWriter + Send + Sync>,
    ) -> Arc<dyn RTPWriter + Send + Sync>;

    async fn bind_remote_stream(
        &self,
        info: &StreamInfo,
        reader: Arc<dyn RTPReader + Send + Sync>,
    ) -> Arc<dyn RTPReader + Send + Sync>;

    async fn close(&self) -> Result<()>;
}
```

Each interceptor wrapped readers and writers during a one-time binding phase. The chain was built by sequential wrapping:

```
Original Reader/Writer
    ↓ wrapped by
Interceptor A
    ↓ wrapped by
Interceptor B
    ↓
Application Code
```

### Limitations

#### 1. Async-Trait Overhead

The `#[async_trait]` macro converts every method call to `Pin<Box<dyn Future>>`, adding runtime allocation overhead:

```rust
// What async-trait generates internally
fn bind_rtcp_reader<'a>(
    &'a self,
    reader: Arc<dyn RTCPReader + Send + Sync>,
) -> Pin<Box<dyn Future<Output = Arc<dyn RTCPReader + Send + Sync>> + Send + 'a>>
```

While this overhead is acceptable for one-time binding operations, it permeates the entire design philosophy.

#### 2. Background Task Spawning

Features like NACK generation and RTCP report transmission required spawning background tasks with `tokio::spawn`:

```rust
async fn bind_rtcp_writer(
    &self,
    writer: Arc<dyn RTCPWriter + Send + Sync>,
) -> Arc<dyn RTCPWriter + Send + Sync> {
    let internal = Arc::clone(&self.internal);
    let writer2 = Arc::clone(&writer);

    tokio::spawn(async move {
        let mut ticker = tokio::time::interval(internal.interval);
        loop {
            tokio::select! {
                _ = ticker.tick() => {
                    // Generate and send reports
                    let nacks = internal.generate_nacks().await;
                    for nack in nacks {
                        writer2.write(&[Box::new(nack)], &Attributes::new()).await;
                    }
                }
                _ = close_rx.recv() => return,
            }
        }
    });

    writer
}
```

This creates several problems:
- **Runtime dependency**: Requires tokio or async-std at the protocol layer
- **Cancellation safety**: `tokio::select!` branches may not be cancel-safe
- **Shutdown coordination**: Requires channels and WaitGroups for cleanup
- **Testing complexity**: Unit tests must spin up async runtimes

#### 3. Shared Mutable State via Mutex

Per-stream state was managed with `Mutex<HashMap<u32, Arc<Stream>>>`:

```rust
struct GeneratorInternal {
    streams: Mutex<HashMap<u32, Arc<GeneratorStream>>>,
    close_rx: Mutex<Option<mpsc::Receiver<()>>>,
}
```

Every periodic poll required acquiring the async lock, and the common workaround was to clone all Arc pointers under the lock, then process outside:

```rust
let streams: Vec<Arc<Stream>> = {
    let m = internal.streams.lock().await;
    m.values().cloned().collect()
};
for stream in streams {
    stream.generate_report(now).await;
}
```

#### 4. Type Erasure and Dynamic Dispatch

The `Arc<dyn RTCPReader>` pattern erases type information at every layer:

```rust
async fn read(&self, buf: &mut [u8], a: &Attributes)
    -> Result<(Vec<Box<dyn rtcp::Packet>>, Attributes)>;
```

This means:
- Every packet passes through multiple vtable lookups
- No compile-time optimization across interceptor boundaries
- Lost type information cannot be recovered

#### 5. Error Handling Strategy

Errors in background tasks were typically logged and swallowed:

```rust
if let Err(err) = rtcp_writer.write(&[Box::new(nack)], &a).await {
    log::warn!("failed sending nack: {err}");
}
```

While pragmatic for reliability, this prevents error propagation to callers who might want to react to failures.

---

## The New Sans-I/O Interceptor Framework

The new interceptor framework addresses these limitations by building on top of `sansio::Protocol` and using **generic composition** instead of trait objects.

### Core Design Principle

Every interceptor is:

1. **A state machine** implementing `sansio::Protocol<TaggedPacket, TaggedPacket, ()>`
2. **Generic over its inner interceptor** `P: Interceptor`
3. **Composable at compile time** via type nesting

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
        Error = Error,
    > + Sized
{
    fn with<O, F>(self, f: F) -> O
    where
        F: FnOnce(Self) -> O,
        O: Interceptor;

    fn bind_local_stream(&mut self, info: &StreamInfo);
    fn unbind_local_stream(&mut self, info: &StreamInfo);
    fn bind_remote_stream(&mut self, info: &StreamInfo);
    fn unbind_remote_stream(&mut self, info: &StreamInfo);
}
```

The `TaggedPacket` type carries both the packet and metadata:

```rust
pub enum Packet {
    Rtp(rtp::Packet),
    Rtcp(Vec<Box<dyn rtcp::Packet>>),
}

pub type TaggedPacket = TransportMessage<Packet>;

pub struct TransportMessage<T> {
    pub now: Instant,                 // Packet received/sent timestamp
    pub transport: TransportContext,  // Source/destination info
    pub message: T,                   // The actual RTP/RTCP packet
}
```

### Generic Composition

The key insight is that each interceptor **wraps** another interceptor as a generic type parameter:

```rust
pub struct NackGeneratorInterceptor<P: Interceptor> {
    inner: P,
    size: u16,
    interval: Duration,
    receive_logs: HashMap<u32, ReceiveLog>,
    write_queue: VecDeque<TaggedPacket>,
    // ...
}
```

Composition happens through a **Registry** that builds nested types:

```rust
pub struct Registry<P> {
    inner: P,
}

impl Registry<NoopInterceptor> {
    pub fn new() -> Self {
        Registry { inner: NoopInterceptor::new() }
    }
}

impl<P: Interceptor> Registry<P> {
    pub fn with<O, F>(self, f: F) -> Registry<O>
    where
        F: FnOnce(P) -> O,
        O: Interceptor,
    {
        Registry { inner: f(self.inner) }
    }

    pub fn build(self) -> P {
        self.inner
    }
}
```

Each interceptor provides a builder that returns a closure:

```rust
impl<P: Interceptor> NackGeneratorBuilder<P> {
    pub fn build(self) -> impl FnOnce(P) -> NackGeneratorInterceptor<P> {
        move |inner| NackGeneratorInterceptor::new(
            inner,
            self.size,
            self.interval,
            self.skip_last_n,
            self.max_nacks_per_packet,
        )
    }
}
```

Usage is fluent and type-safe:

```rust
let chain = Registry::new()
    .with(NackGeneratorBuilder::new()
        .with_size(512)
        .with_interval(Duration::from_millis(100))
        .build())
    .with(NackResponderBuilder::new()
        .with_size(1024)
        .build())
    .with(SenderReportBuilder::new()
        .with_interval(Duration::from_secs(1))
        .build())
    .build();

// Type: SenderReportInterceptor<NackResponderInterceptor<NackGeneratorInterceptor<NoopInterceptor>>>
```

The resulting type is fully known at compile time, enabling aggressive optimization.

### The Polling Model

Instead of spawning background tasks, interceptors use the **polling model**:

```rust
impl<P: Interceptor> sansio::Protocol<TaggedPacket, TaggedPacket, ()>
    for NackGeneratorInterceptor<P>
{
    fn handle_read(&mut self, msg: TaggedPacket) -> Result<()> {
        // Track incoming RTP sequence numbers
        if let Packet::Rtp(ref pkt) = msg.message {
            if let Some(log) = self.receive_logs.get_mut(&pkt.header.ssrc) {
                log.add(pkt.header.sequence_number);
            }
        }
        self.inner.handle_read(msg)
    }

    fn poll_read(&mut self) -> Option<TaggedPacket> {
        self.inner.poll_read()
    }

    fn handle_timeout(&mut self, now: Instant) -> Result<()> {
        if self.eto <= now {
            self.eto = now + self.interval;
            self.generate_nacks(now);  // Detect gaps, queue NACKs
        }
        self.inner.handle_timeout(now)
    }

    fn poll_write(&mut self) -> Option<TaggedPacket> {
        // First drain our queue, then delegate to inner
        if let Some(pkt) = self.write_queue.pop_front() {
            return Some(pkt);
        }
        self.inner.poll_write()
    }

    fn poll_timeout(&mut self) -> Option<Instant> {
        let inner_timeout = self.inner.poll_timeout();
        match inner_timeout {
            Some(t) => Some(t.min(self.eto)),
            None => Some(self.eto),
        }
    }
}
```

The caller (usually the RTCPeerConnection orchestrator) drives the state machine:

```rust
// Caller loop
loop {
    // Process incoming packets
    while let Some(pkt) = receive_from_network() {
        chain.handle_read(pkt)?;
    }

    // Handle timeouts
    let now = Instant::now();
    chain.handle_timeout(now)?;

    // Drain outputs
    while let Some(pkt) = chain.poll_read() {
        deliver_to_application(pkt);
    }
    while let Some(pkt) = chain.poll_write() {
        send_to_network(pkt);
    }

    // Schedule next wake
    if let Some(deadline) = chain.poll_timeout() {
        sleep_until(deadline);
    }
}
```

### Direction-Agnostic Processing

**Important:** Unlike PeerConnection's pipeline where `read` and `write` have opposite processing direction orders, interceptors have **no direction concept**.

In PeerConnection's pipeline, handlers are traversed in opposite orders:

```text
Read:  Network → HandlerA → HandlerB → HandlerC → Application
Write: Application → HandlerC → HandlerB → HandlerA → Network
       (reversed order)
```

In Interceptor chains, **all operations flow in the same direction**:

```text
handle_read:    Outer → Inner (A.handle_read calls B.handle_read calls C.handle_read)
handle_write:   Outer → Inner (A.handle_write calls B.handle_write calls C.handle_write)
handle_event:   Outer → Inner (A.handle_event calls B.handle_event calls C.handle_event)
handle_timeout: Outer → Inner (A.handle_timeout calls B.handle_timeout calls C.handle_timeout)

poll_read:    Outer → Inner (A.poll_read calls B.poll_read calls C.poll_read)
poll_write:   Outer → Inner (A.poll_write calls B.poll_write calls C.poll_write)
poll_event:   Outer → Inner (A.poll_event calls B.poll_event calls C.poll_event)
poll_timeout: Outer → Inner (A.poll_timeout calls B.poll_timeout calls C.poll_timeout)
```

This means interceptors are **symmetric**—they process `read`, `write`, and `event` in the same structural order. The distinction between "inbound" and "outbound" is **semantic** (based on message content), not **structural** (based on call order).

This design simplifies reasoning about interceptor behavior:

- Each interceptor sees packets in consistent order regardless of direction
- NACK generator can track incoming packets (`handle_read`) and emit feedback (`poll_write`) without special direction handling
- TWCC sender adds sequence numbers on `handle_write` while TWCC receiver tracks arrivals on `handle_read`—both using the same outer-to-inner flow

The PeerConnection's `InterceptorHandler` is responsible for placing the interceptor at the correct position in the bidirectional pipeline, but the interceptor itself remains direction-agnostic.

---

## Benefits of the Generic-Based Design

### 1. Zero Runtime Overhead for Composition

With the old design:
```rust
Arc<dyn Interceptor> → Arc<dyn RTPWriter> → Arc<dyn RTPWriter> → ...
```

Each arrow is a vtable lookup at runtime.

With the new design:
```rust
SenderReportInterceptor<NackResponderInterceptor<NackGeneratorInterceptor<NoopInterceptor>>>
```

The compiler sees the entire chain as a single type and can inline across boundaries.

### 2. No Async Runtime Dependency

The `sansio::Protocol` trait is completely synchronous:

```rust
fn handle_read(&mut self, msg: Rin) -> Result<(), Self::Error>;
fn poll_read(&mut self) -> Option<Self::Rout>;
fn handle_timeout(&mut self, now: Self::Time) -> Result<(), Self::Error>;
fn poll_timeout(&mut self) -> Option<Self::Time>;
```

This means:
- **Works anywhere**: sync code, async code, WASM, embedded systems
- **No allocation per call**: no Future boxing
- **Deterministic timing**: caller controls when timeouts fire

### 3. Testability Without Ceremony

Testing is straightforward—no async blocks, no runtime setup:

```rust
#[test]
fn test_nack_generator_detects_packet_loss() {
    let mut chain = Registry::new()
        .with(NackGeneratorBuilder::new()
            .with_size(512)
            .with_interval(Duration::from_millis(100))
            .build())
        .build();

    let ssrc = 0x12345678;
    chain.bind_remote_stream(&nack_stream_info(ssrc));

    // Simulate packets: 0, 1, 2, [gap: 3-5], 6, 7
    for seq in [0u16, 1, 2, 6, 7] {
        let pkt = create_rtp_packet(ssrc, seq);
        chain.handle_read(pkt).unwrap();
    }

    while chain.poll_read().is_some() {}

    // Advance time past the NACK interval
    let now = Instant::now() + Duration::from_millis(150);
    chain.handle_timeout(now).unwrap();

    // Poll for generated NACK
    let nack_pkt = chain.poll_write().expect("should generate NACK");
    if let Packet::Rtcp(rtcp_packets) = &nack_pkt.message {
        let nack = rtcp_packets[0]
            .as_any()
            .downcast_ref::<TransportLayerNack>()
            .unwrap();
        assert_eq!(nack.nacks[0].packet_id, 3);  // First missing seq
    }
}
```

### 4. Type-Safe Feature Detection

Each interceptor can inspect `StreamInfo` during binding to decide whether to activate:

```rust
fn stream_supports_nack(info: &StreamInfo) -> bool {
    info.rtcp_feedback
        .iter()
        .any(|fb| fb.typ == "nack" && fb.parameter.is_empty())
}

fn stream_supports_twcc(info: &StreamInfo) -> Option<u8> {
    info.rtp_header_extensions
        .iter()
        .find(|ext| ext.uri == TRANSPORT_CC_URI)
        .map(|ext| ext.id as u8)
}

impl<P: Interceptor> Interceptor for NackGeneratorInterceptor<P> {
    fn bind_remote_stream(&mut self, info: &StreamInfo) {
        if stream_supports_nack(info) {
            self.receive_logs.insert(info.ssrc, ReceiveLog::new(self.size));
        }
        self.inner.bind_remote_stream(info);
    }
}
```

This ensures interceptors only consume resources for streams that actually need them.

### 5. Explicit Data Flow

Every packet transformation is visible in the code:

```rust
fn handle_write(&mut self, msg: TaggedPacket) -> Result<()> {
    // TWCC: Add sequence number to outgoing RTP
    if let Packet::Rtp(ref mut pkt) = msg.message {
        if let Some(stream) = self.streams.get(&pkt.header.ssrc) {
            let seq = self.next_sequence_number;
            self.next_sequence_number = self.next_sequence_number.wrapping_add(1);

            let tcc_ext = TransportCcExtension { transport_sequence: seq };
            pkt.header.set_extension(stream.hdr_ext_id, tcc_ext.marshal()?);
        }
    }
    self.inner.handle_write(msg)
}
```

No hidden channels, no background mutations—just straightforward state machine transitions.

---

## Integrating Interceptors into the RTC Crate

The [rtc crate](https://github.com/webrtc-rs/rtc/tree/master/rtc) integrates the new interceptor framework throughout its architecture.

### Generic RTCPeerConnection

The `RTCPeerConnection` struct is parametrized with the interceptor type:

```rust
pub struct RTCPeerConnection<I = NoopInterceptor>
where
    I: Interceptor,
{
    pub(crate) configuration: RTCConfiguration<I>,
    pub(crate) rtp_transceivers: Vec<RTCRtpTransceiver<I>>,
    pub(crate) pipeline_context: PipelineContext,
    // ...
}
```

This type parameter flows through the entire call stack:
- `RTCConfiguration<I>` stores the built interceptor
- `RTCRtpTransceiver<I>` carries the same type for consistency
- Handler types use the parameter: `InterceptorHandler<'_, I>`, `EndpointHandler<'_, I>`

### Configuration Builder

The configuration builder supports type-safe registry building:

```rust
impl RTCConfigurationBuilder<NoopInterceptor> {
    pub fn new() -> Self { /* ... */ }
}

impl<I: Interceptor> RTCConfigurationBuilder<I> {
    pub fn with_interceptor_registry<P>(
        self,
        interceptor_registry: Registry<P>,
    ) -> RTCConfigurationBuilder<P>
    where
        P: Interceptor,
    {
        RTCConfigurationBuilder {
            interceptor: interceptor_registry.build(),
            // ... transfer other fields
        }
    }
}
```

Usage:

```rust
let config = RTCConfigurationBuilder::new()
    .with_ice_servers(vec![ice_server])
    .with_interceptor_registry(
        Registry::new()
            .with(NackGeneratorBuilder::new().build())
            .with(NackResponderBuilder::new().build())
            .with(SenderReportBuilder::new().build())
    )
    .build()?;

let peer_connection = RTCPeerConnection::new(config);
```

### Interceptor Handler in the Pipeline

The `InterceptorHandler` bridges between the RTC message format and the interceptor's packet format:

```rust
pub(crate) struct InterceptorHandler<'a, I: Interceptor> {
    ctx: &'a mut InterceptorHandlerContext,
    interceptor: &'a mut I,
}

impl<'a, I: Interceptor> sansio::Protocol<TaggedRTCMessageInternal, TaggedRTCMessageInternal, RTCEventInternal>
    for InterceptorHandler<'a, I>
{
    fn handle_read(&mut self, msg: TaggedRTCMessageInternal) -> Result<()> {
        match msg.message {
            RTCMessageInternal::Rtp(rtp_message) => {
                let tagged_packet = TaggedPacket {
                    now: msg.now,
                    transport: msg.transport,
                    message: Packet::Rtp(rtp_message.packet),
                };
                self.interceptor.handle_read(tagged_packet)?;
            }
            RTCMessageInternal::Rtcp(rtcp_packets) => {
                let tagged_packet = TaggedPacket {
                    now: msg.now,
                    transport: msg.transport,
                    message: Packet::Rtcp(rtcp_packets),
                };
                self.interceptor.handle_read(tagged_packet)?;
            }
            _ => { /* Forward non-RTP/RTCP unchanged */ }
        }
        Ok(())
    }

    fn poll_read(&mut self) -> Option<TaggedRTCMessageInternal> {
        // Convert interceptor output back to RTC format
        self.interceptor.poll_read().map(|pkt| {
            TaggedRTCMessageInternal {
                now: pkt.now,
                transport: pkt.transport,
                message: match pkt.message {
                    Packet::Rtp(rtp) => RTCMessageInternal::Rtp(RTPMessage { packet: rtp }),
                    Packet::Rtcp(rtcp) => RTCMessageInternal::Rtcp(rtcp),
                },
            }
        })
    }

    // ... similar for handle_write, poll_write, handle_timeout, poll_timeout
}
```

### Stream Binding Lifecycle

Stream binding occurs at specific lifecycle points:

1. **Local stream binding** happens when `start_rtp_senders()` is called:

```rust
// In RTCRtpSender
pub(crate) fn interceptor_local_streams_op(
    &mut self,
    media_engine: &MediaEngine,
    interceptor: &mut I,
    is_binding: bool,
) {
    for coding in self.track().codings() {
        let stream_info = create_stream_info(
            coding.ssrc,
            coding.rtx_ssrc,
            coding.fec_ssrc,
            codec.payload_type,
            rtx_payload_type,
            fec_payload_type,
            &codec.rtp_codec,
            &header_extensions,
        );

        if is_binding {
            interceptor.bind_local_stream(&stream_info);
        } else {
            interceptor.unbind_local_stream(&stream_info);
        }
    }
}
```

2. **Remote stream binding** happens when processing the remote description:

```rust
// In RTCRtpReceiver
pub(crate) fn interceptor_remote_stream_op(
    interceptor: &mut I,
    is_binding: bool,
    ssrc: SSRC,
    payload_type: PayloadType,
    rtp_codec: &RTCRtpCodec,
    header_extensions: &[RTCRtpHeaderExtensionParameters],
) {
    let stream_info = create_stream_info(/* ... */);

    if is_binding {
        interceptor.bind_remote_stream(&stream_info);
    } else {
        interceptor.unbind_remote_stream(&stream_info);
    }
}
```

### Default Interceptor Configuration

The RTC crate provides helper functions to register common interceptors:

```rust
pub fn register_default_interceptors<P: Interceptor>(
    media_engine: &mut MediaEngine,
    registry: Registry<P>,
) -> Result<Registry<impl Interceptor>> {
    let registry = configure_nack(media_engine, registry)?;
    let registry = configure_rtcp_reports(registry)?;
    let registry = configure_twcc_receiver(media_engine, registry)?;
    Ok(registry)
}

fn configure_nack<P: Interceptor>(
    media_engine: &mut MediaEngine,
    registry: Registry<P>,
) -> Result<Registry<impl Interceptor>> {
    // Register RTCP feedback in media engine
    media_engine.register_feedback(
        RTCPFeedback { typ: "nack".to_string(), parameter: "".to_string() },
        RTPCodecType::Video,
    );

    Ok(registry
        .with(NackGeneratorBuilder::new().build())
        .with(NackResponderBuilder::new().build()))
}
```

---

## Concrete Interceptor Implementations

### NACK Generator

Tracks incoming RTP sequence numbers and generates NACK requests for missing packets:

```rust
pub struct NackGeneratorInterceptor<P> {
    inner: P,
    interval: Duration,
    eto: Instant,  // Expected timeout
    receive_logs: HashMap<u32, ReceiveLog>,  // Per-SSRC tracking
    nack_counts: HashMap<u32, HashMap<u16, u16>>,  // Retransmit limits
    write_queue: VecDeque<TaggedPacket>,
}

impl<P: Interceptor> NackGeneratorInterceptor<P> {
    fn generate_nacks(&mut self, now: Instant) {
        for (&ssrc, log) in &self.receive_logs {
            let missing = log.missing_seq_numbers(self.skip_last_n);
            if missing.is_empty() { continue; }

            // Filter by retransmit count limits
            let filtered = self.filter_by_nack_count(ssrc, missing);
            if filtered.is_empty() { continue; }

            let nack = TransportLayerNack {
                sender_ssrc: self.sender_ssrc,
                media_ssrc: ssrc,
                nacks: build_nack_pairs(filtered),
            };

            self.write_queue.push_back(TaggedPacket {
                now,
                transport: TransportContext::default(),
                message: Packet::Rtcp(vec![Box::new(nack)]),
            });
        }
    }
}
```

### NACK Responder

Buffers outgoing packets and retransmits on NACK requests (with RFC 4588 RTX support):

```rust
pub struct NackResponderInterceptor<P> {
    inner: P,
    streams: HashMap<u32, LocalStream>,
    write_queue: VecDeque<TaggedPacket>,
}

struct LocalStream {
    send_buffer: SendBuffer,
    ssrc_rtx: Option<u32>,
    payload_type_rtx: Option<u8>,
    rtx_sequence_number: u16,
}

impl<P: Interceptor> NackResponderInterceptor<P> {
    fn handle_nack(&mut self, now: Instant, nack: &TransportLayerNack) {
        let Some(stream) = self.streams.get_mut(&nack.media_ssrc) else { return };

        for seq in nack.iter_sequence_numbers() {
            let Some(original) = stream.send_buffer.get(seq) else { continue };

            let packet = if let (Some(rtx_ssrc), Some(rtx_pt)) =
                (stream.ssrc_rtx, stream.payload_type_rtx)
            {
                // RFC 4588: Create RTX packet with original seq in payload
                let mut rtx_payload = Vec::with_capacity(2 + original.payload.len());
                rtx_payload.extend_from_slice(&seq.to_be_bytes());
                rtx_payload.extend_from_slice(&original.payload);

                rtp::Packet {
                    header: rtp::header::Header {
                        ssrc: rtx_ssrc,
                        payload_type: rtx_pt,
                        sequence_number: stream.next_rtx_seq(),
                        timestamp: original.header.timestamp,
                        marker: original.header.marker,
                        ..Default::default()
                    },
                    payload: rtx_payload.into(),
                }
            } else {
                // No RTX: retransmit original packet
                original.clone()
            };

            self.write_queue.push_back(TaggedPacket {
                now,
                transport: TransportContext::default(),
                message: Packet::Rtp(packet),
            });
        }
    }
}
```

### TWCC Sender

Adds transport-wide sequence numbers to outgoing RTP packets:

```rust
pub struct TwccSenderInterceptor<P> {
    inner: P,
    next_sequence_number: u16,  // Shared across all streams
    streams: HashMap<u32, LocalStream>,
}

impl<P: Interceptor> sansio::Protocol<TaggedPacket, TaggedPacket, ()>
    for TwccSenderInterceptor<P>
{
    fn handle_write(&mut self, mut msg: TaggedPacket) -> Result<()> {
        if let Packet::Rtp(ref mut pkt) = msg.message {
            if let Some(stream) = self.streams.get(&pkt.header.ssrc) {
                let seq = self.next_sequence_number;
                self.next_sequence_number = self.next_sequence_number.wrapping_add(1);

                let tcc_ext = TransportCcExtension { transport_sequence: seq };
                pkt.header.set_extension(stream.hdr_ext_id, tcc_ext.marshal()?);
            }
        }
        self.inner.handle_write(msg)
    }
}
```

### TWCC Receiver

Tracks incoming packet arrival times and generates TWCC feedback:

```rust
pub struct TwccReceiverInterceptor<P> {
    inner: P,
    interval: Duration,
    start_time: Option<Instant>,
    recorder: Option<Recorder>,
    streams: HashMap<u32, RemoteStream>,
    write_queue: VecDeque<TaggedPacket>,
    next_timeout: Option<Instant>,
}

impl<P: Interceptor> sansio::Protocol<TaggedPacket, TaggedPacket, ()>
    for TwccReceiverInterceptor<P>
{
    fn handle_read(&mut self, msg: TaggedPacket) -> Result<()> {
        if let Packet::Rtp(ref pkt) = msg.message {
            if let Some(stream) = self.streams.get(&pkt.header.ssrc) {
                if let Some(ext) = pkt.header.get_extension(stream.hdr_ext_id) {
                    let tcc = TransportCcExtension::unmarshal(&mut ext.as_ref())?;

                    let arrival_time = self.start_time
                        .map(|start| msg.now.duration_since(start).as_micros() as i64)
                        .unwrap_or(0);

                    if let Some(recorder) = self.recorder.as_mut() {
                        recorder.record(pkt.header.ssrc, tcc.transport_sequence, arrival_time);
                    }
                }
            }
        }
        self.inner.handle_read(msg)
    }

    fn handle_timeout(&mut self, now: Instant) -> Result<()> {
        if self.next_timeout.map_or(false, |t| now >= t) {
            self.generate_feedback(now);
            self.next_timeout = Some(now + self.interval);
        }
        self.inner.handle_timeout(now)
    }
}
```

---

## Summary: Old vs. New Design

| Aspect | Old Async Design | New Sans-I/O Design |
|--------|------------------|---------------------|
| **Composition** | Runtime trait objects | Compile-time generics |
| **Runtime Dependency** | Requires tokio/async-std | None (works anywhere) |
| **Per-Call Overhead** | Future allocation | Zero-cost |
| **Background Tasks** | `tokio::spawn` | Polling model |
| **State Sharing** | `Mutex<HashMap>` | Direct ownership |
| **Timeout Handling** | `tokio::select!` + ticker | `handle_timeout` + `poll_timeout` |
| **Testability** | Requires async runtime | Pure synchronous tests |
| **Error Propagation** | Log and swallow | `Result` return values |
| **Type Information** | Erased at boundaries | Preserved through chain |

---

## Conclusion

The interceptor redesign is not just a refactor—it is a **re-alignment with sansio::Protocol fundamentals**. Protocol logic is fundamentally **single-threaded state evolution**, and the sans-I/O pattern embraces this directly.

By combining:
- **Sans-I/O** state machines
- **Generic composition** instead of dynamic dispatch
- **Poll-driven** timeout handling

The re-designed interceptor framework achieves:

1. **Generic composition** replaces runtime polymorphism, enabling zero-cost abstraction across interceptor boundaries.

2. **The polling model** eliminates async runtime dependencies while providing explicit control over timing and I/O integration.

3. **Compile-time type nesting** preserves type information, enabling the optimizer to inline across the entire interceptor chain.

4. **Deterministic state machines** make testing straightforward and protocol behavior reproducible.

5. **Seamless integration** with the RTCPeerConnection through consistent generic parametrization.

This design philosophy extends naturally to other protocol components. Whether implementing bandwidth estimation, simulcast layer switching, or custom RTCP feedback mechanisms, the `sansio::Protocol` trait provides a solid foundation for building composable, testable, and efficient protocol handlers.

---

**Key Takeaways:**

- Sans-I/O separates protocol logic from I/O concerns
- Generic composition enables zero-overhead interceptor chaining
- The polling model integrates naturally with any I/O framework
- Stream binding provides type-safe feature negotiation
- Testability comes for free without async ceremony

---

## References

- [rtc-interceptor](https://github.com/webrtc-rs/rtc/tree/master/rtc-interceptor) — The new interceptor framework
- [rtc](https://github.com/webrtc-rs/rtc/tree/master/rtc) — RTCPeerConnection integration
- [Building WebRTC's Pipeline with `sansio::Protocol`](/blog/2026/01/04/building-webrtc-pipeline-with-sansio.html) — Previous article on the sans-I/O pipeline
- [RFC 3550](https://datatracker.ietf.org/doc/html/rfc3550) — RTP/RTCP specification
- [RFC 4585](https://datatracker.ietf.org/doc/html/rfc4585) — Extended RTP Profile for RTCP-Based Feedback (NACK)
- [RFC 4588](https://datatracker.ietf.org/doc/html/rfc4588) — RTP Retransmission Payload Format (RTX)
- [draft-holmer-rmcat-transport-wide-cc-extensions](https://datatracker.ietf.org/doc/html/draft-holmer-rmcat-transport-wide-cc-extensions) — Transport-Wide Congestion Control (TWCC)
