# Announcing `rtc` 0.5.0: Simulcast Support and API Refinements 🎉

I'm excited to announce **`rtc` 0.5.0**, a major release that brings **simulcast support**, API improvements, and enhanced documentation to our sans-I/O WebRTC implementation.

## What's New in 0.5.0

### Simulcast Support 📹

The headline feature is **simulcast support**. Simulcast enables sending multiple quality levels (spatial or temporal layers) of the same video stream simultaneously, allowing adaptive bitrate streaming and improved user experience across varying network conditions.

**Key capabilities:**
- Multiple spatial layers per video track (e.g., quarter/half/full resolution)
- RTP Stream ID (RID) for layer identification
- Dynamic layer selection and switching
- Proper RTP header extension handling (`mid`, `rtp-stream-id`, `repaired-rtp-stream-id`)

**Example use cases:**
- Video conferencing with quality adaptation
- Live streaming with bandwidth-aware layer selection
- Multi-party calls with selective forwarding

### New Examples

Both examples demonstrate the sans-I/O event loop pattern and follow the same architectural principles.

#### Simulcast Example

The [simulcast example](https://github.com/webrtc-rs/rtc/tree/master/examples/examples/simulcast) shows how to send video with three quality layers and dynamically forward different layers based on bandwidth or CPU constraints:

```rust
// Configure three spatial layers
let layers = vec![
    RTCRtpEncodingParameters {
        rtp_coding_parameters: RTCRtpCodingParameters {
            rid: "q".to_string(),  // quarter resolution
            ssrc: Some(100),
            ..Default::default()
        },
        codec: vp8_codec.clone(),
        ..Default::default()
    },
    RTCRtpEncodingParameters {
        rtp_coding_parameters: RTCRtpCodingParameters {
            rid: "h".to_string(),  // half resolution
            ssrc: Some(200),
            ..Default::default()
        },
        codec: vp8_codec.clone(),
        ..Default::default()
    },
    RTCRtpEncodingParameters {
        rtp_coding_parameters: RTCRtpCodingParameters {
            rid: "f".to_string(),  // full resolution
            ssrc: Some(300),
            ..Default::default()
        },
        codec: vp8_codec.clone(),
        ..Default::default()
    },
];
```

The example handles RID-based routing, sends periodic keyframe requests (PLI), and demonstrates proper header extension configuration.

#### Swap Tracks Example

The [swap-tracks example](https://github.com/webrtc-rs/rtc/tree/master/examples/examples/swap-tracks) demonstrates smooth switching between multiple incoming video tracks:

- Automatic keyframe requests when switching tracks
- Proper timestamp and sequence number handling
- Continuous output stream without artifacts

This pattern is essential for implementing features like speaker switching in video conferences.

---

## API Changes

To properly support simulcast, several APIs have been refined. These are **breaking changes** that make the API more powerful and consistent with multi-encoding scenarios.

### MediaStreamTrack Constructor

The constructor now accepts a vector of decoding parameters instead of individual fields:

**Before:**
```rust
MediaStreamTrack::new(
    stream_id,
    track_id,
    label,
    kind,
    rid: Option<String>,
    ssrc: u32,
    codec: RTCRtpCodec,
)
```

**After:**
```rust
MediaStreamTrack::new(
    stream_id,
    track_id,
    label,
    kind,
    codings: Vec<RTCRtpEncodingParameters>,
)
```

This design naturally supports multiple encodings:

```rust
let track = MediaStreamTrack::new(
    "stream-id".to_string(),
    "track-id".to_string(),
    "Video Track".to_string(),
    RtpCodecKind::Video,
    vec![RTCRtpEncodingParameters {
        rtp_coding_parameters: RTCRtpCodingParameters {
            ssrc: Some(12345),
            rid: "q".to_string(),
            ..Default::default()
        },
        codec,
        ..Default::default()
    }],
);
```

### Single Value → Iterator Methods

Several methods now return iterators to support multiple values in simulcast:

```rust
// Old API
let ssrc = track.ssrc();
let codec = track.codec();

// New API - iterate over all encodings
for ssrc in track.ssrcs() {
    if let Some(codec) = track.codec(ssrc) {
        println!("SSRC {}: {}", ssrc, codec.mime_type);
    }
}

// Or get first value
let ssrc = track.ssrcs().next().unwrap_or(0);
let codec = track.codecs().next().unwrap();
```

**New simulcast-aware methods:**

```rust
// Get RID for specific SSRC
if let Some(rid) = track.rid(ssrc) {
    println!("Layer: {}", rid);
}

// Get codec for specific SSRC
if let Some(codec) = track.codec(ssrc) {
    println!("Codec: {}", codec.mime_type);
}
```

### Simplified RTCRtpReceiver

The receiver API is now simpler - each receiver has exactly one track:

```rust
// Old API
let track = receiver.track(&track_id)?;
if let Some(track) = track {
    // use track
}

// New API
let track = receiver.track()?;
// use track directly
```

### New Types

- **`RtpStreamId`** - Type alias for RTP stream identifiers (e.g., "q", "h", "f")
- **`RepairedStreamId`** - Type alias for redundancy stream identifiers

Both follow [RFC 8852](https://www.rfc-editor.org/rfc/rfc8852.html) semantics.

---

## Documentation Improvements

Comprehensive documentation has been added for all simulcast-related APIs:

- **222+ passing doc tests** (up from 218 in v0.3.0)
- Detailed examples for every changed API
- Clear migration guides with before/after code
- Proper RFC and W3C specification references
- Real-world usage patterns

**Newly documented types:**
- `RTCRtpEncodingParameters` - Complete simulcast layer configuration
- `RTCRtpCodingParameters` - RTP-level parameters (SSRC, RID, RTX, FEC)
- `RTCRtpRtxParameters` - Retransmission stream configuration
- `RTCRtpFecParameters` - Forward error correction configuration
- `RTCTrackEventInit::rid` - Simulcast stream identification in track events

Each type includes practical examples demonstrating single-encoding and multi-encoding scenarios.

---

## Migration Guide

### Updating MediaStreamTrack Construction

```rust
// Old code (v0.3.x)
use rtc::media_stream::MediaStreamTrack;

let track = MediaStreamTrack::new(
    stream_id,
    track_id,
    "Video".to_string(),
    RtpCodecKind::Video,
    None,   // rid
    12345,  // ssrc
    codec,
);

// New code (v0.4.0)
use rtc::media_stream::MediaStreamTrack;
use rtc::rtp_transceiver::rtp_sender::{RTCRtpEncodingParameters, RTCRtpCodingParameters};

let track = MediaStreamTrack::new(
    stream_id,
    track_id,
    "Video".to_string(),
    RtpCodecKind::Video,
    vec![RTCRtpEncodingParameters {
        rtp_coding_parameters: RTCRtpCodingParameters {
            ssrc: Some(12345),
            ..Default::default()
        },
        codec,
        ..Default::default()
    }],
);
```

### Updating Method Calls

```rust
// ssrc() → ssrcs()
let ssrc = track.ssrc();  // Old
let ssrc = track.ssrcs().next().unwrap_or(0);  // New

// codec() → codecs() or codec(ssrc)
let codec = track.codec();  // Old
let codec = track.codecs().next().unwrap();  // New
// Or for specific SSRC:
let codec = track.codec(ssrc).unwrap();  // New

// rid() now requires SSRC
let rid = track.rid();  // Old
let ssrc = track.ssrcs().next().unwrap();
let rid = track.rid(ssrc);  // New

// receiver.track() simplified
let track = receiver.track(&track_id)?.unwrap();  // Old
let track = receiver.track()?;  // New
```

### Working with Simulcast

```rust
// Iterate over all layers
for ssrc in track.ssrcs() {
    if let Some(rid) = track.rid(ssrc) {
        if let Some(codec) = track.codec(ssrc) {
            println!("Layer {}: SSRC {} using {}", 
                rid, ssrc, codec.mime_type);
        }
    }
}
```

---

## Architecture & Implementation

### Sans-I/O Event Loop

All examples follow the consistent sans-I/O pattern described in the [v0.3.0 announcement](../04/announcing-rtc-v0.3.0.md):

```rust
'EventLoop: loop {
    // 1. Send outgoing packets
    while let Some(msg) = pc.poll_write() {
        socket.send_to(&msg.message, msg.transport.peer_addr).await?;
    }

    // 2. Handle state changes
    while let Some(event) = pc.poll_event() {
        match event {
            RTCPeerConnectionEvent::OnTrack(track_event) => {
                // Handle incoming tracks with simulcast info
                if let Some(rid) = &track_event.rid {
                    println!("Simulcast layer: {}", rid);
                }
            }
            _ => {}
        }
    }

    // 3. Process RTP/RTCP/DataChannel messages
    while let Some(message) = pc.poll_read() {
        match message {
            RTCMessage::RtpPacket(track_id, packet) => {
                // Process RTP packets
            }
            _ => {}
        }
    }

    // 4. Multiplex I/O
    tokio::select! {
        _ = tokio::time::sleep(delay) => {
            pc.handle_timeout(Instant::now())?;
        }
        Ok((n, peer_addr)) = socket.recv_from(&mut buf) => {
            pc.handle_read(tagged_bytes)?;
        }
    }
}
```

### Internal Improvements

- **Enhanced endpoint handler** - Proper mid/rid header extension processing
- **Better track lifecycle management** - Robust handling of simulcast track events
- **Improved SSRC-to-codec mapping** - Efficient lookups for multi-encoding tracks
- **Header extension support** - Full implementation of [RFC 8285](https://www.rfc-editor.org/rfc/rfc8285.html)

---

## What's Next: Interceptors 🚀

The next major milestone is **Interceptor support** for advanced RTCP handling and quality control.

**Planned capabilities:**
- Custom RTCP packet processing pipeline
- Real-time statistics collection (packet loss, jitter, RTT)
- Bandwidth estimation algorithms (GCC, Transport-CC)
- Adaptive bitrate control
- Custom QoS policies and congestion control

**Use cases:**
- Network quality monitoring and diagnostics
- Adaptive simulcast layer selection based on bandwidth
- Custom retransmission strategies
- Performance analytics and observability

The interceptor framework will be implemented using the same `sansio::Protocol` pattern, maintaining transport independence and testability. See the [pipeline architecture article](../04/building-webrtc-pipeline-with-sansio.md) for details on how handlers compose.

---

## Getting Started

### Installation

```toml
[dependencies]
rtc = "0.5.0"
```

### Quick Example

```rust
use rtc::peer_connection::RTCPeerConnection;
use rtc::peer_connection::configuration::RTCConfigurationBuilder;
use rtc::sansio::Protocol;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let config = RTCConfigurationBuilder::new().build();
    let mut pc = RTCPeerConnection::new(config)?;

    // Create and set local description
    let offer = pc.create_offer(None)?;
    pc.set_local_description(offer)?;

    // Sans-I/O event loop
    loop {
        while let Some(msg) = pc.poll_write() {
            // Send to network
        }
        
        while let Some(event) = pc.poll_event() {
            // Handle state changes
        }
        
        while let Some(message) = pc.poll_read() {
            // Process application messages
        }
        
        // Handle I/O and timeouts
        // ...
    }
}
```

Check out the [examples directory](https://github.com/webrtc-rs/rtc/tree/master/examples) for complete working code!

---

## Feature Parity Update

Progress toward full feature parity with the `webrtc` crate:

✅ **Complete:**
- ICE, DTLS, SRTP/SRTCP, SCTP
- Data Channels (reliable & unreliable)
- RTP/RTCP, Media Tracks, SDP
- Peer Connection API
- **Simulcast** ← New in 0.5.0!

🚧 **In Progress:**
- RTCP Interceptors (next release)

---

## Links

- **GitHub**: https://github.com/webrtc-rs/rtc
- **Crate**: https://crates.io/crates/rtc
- **Docs**: https://docs.rs/rtc
- **Discord**: https://discord.gg/4Ju8UHdXMs
- **Examples**: https://github.com/webrtc-rs/rtc/tree/master/examples

---

## Full Changelog

### Added
- ✨ Full simulcast support with multiple spatial layers
- ✨ RTP stream identifier (RID) support per RFC 8852
- ✨ Header extension handling (mid, rtp-stream-id, repaired-rtp-stream-id)
- ✨ Simulcast example with three quality layers
- ✨ Swap-tracks example with smooth track switching
- ✨ `MediaStreamTrack::codec(ssrc)` method for per-encoding codec lookup
- ✨ `RtpStreamId` and `RepairedStreamId` type aliases

### Changed
- 💥 `MediaStreamTrack::new()` - Consolidated into `Vec<RTCRtpEncodingParameters>`
- 💥 `MediaStreamTrack::ssrc()` → `ssrcs()` - Returns iterator
- 💥 `MediaStreamTrack::codec() - Now requires SSRC parameter
- 💥 `MediaStreamTrack::rid()` - Now requires SSRC parameter
- 💥 `RTCRtpReceiver::track()` - Removed track_id parameter, simplified return

### Improved
- 📚 222+ doc tests with comprehensive examples
- 📚 Complete documentation for RTCRtpEncodingParameters
- 📚 Complete documentation for RTCRtpCodingParameters
- 📚 Enhanced RTCTrackEventInit::rid documentation
- 🏗️ Improved endpoint handler for RTP header extensions
- 🏗️ Better internal track lifecycle management

### Fixed
- 🐛 Receiver tracks hashmap handling
- 🐛 SSRC-to-codec mapping in simulcast scenarios
- 🐛 Header extension processing

---

## Relationship with `webrtc` Crate

As stated in the [v0.3.0 announcement](../04/announcing-rtc-v0.3.0.md), `rtc` (sans-I/O) and `webrtc` (async) are **complementary**:

- **Use `webrtc`** for quick start with Tokio and async/await
- **Use `rtc`** for runtime independence, custom I/O, or maximum control

Both crates are actively maintained and share protocol implementations where possible.

---

*Thanks to everyone who contributed feedback, bug reports, and feature requests! Special thanks to the Rust WebRTC community for making this release possible.* 🦀

Feedback and contributions welcome on [GitHub](https://github.com/webrtc-rs/rtc)!
