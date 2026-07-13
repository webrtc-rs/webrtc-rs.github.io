# Announcing SFU v0.20.0-rc.3: A Sans-I/O Selective Forwarding Unit in Rust

We're excited to introduce a new member of the WebRTC.rs family: **[`sfu`](https://github.com/webrtc-rs/sfu)**, released today as **v0.20.0-rc.3** and versioned in lockstep with the `rtc` and `webrtc` crates.

Up to now, webrtc-rs has been about *endpoints*: the Sans-I/O `rtc` protocol core, and the async `webrtc` crate built on top of it. `sfu` is the first *server* in the family — a **Selective Forwarding Unit** that terminates WebRTC from many browsers in a room and forwards each publisher's media to every subscriber, without decoding or re-encoding a single frame.

And, like everything else we've built since the Sans-I/O shift: **the library crate contains no sockets, no threads, and no clock of its own.**

---

## What Is an SFU, and Why Sans-I/O?

A peer-to-peer mesh does not scale: with *N* participants, every browser uploads *N-1* copies of its own video. An SFU fixes the upload problem by putting a server in the middle — each client uploads **once**, and the server selectively forwards those RTP streams to everyone else. It is the architecture behind essentially every modern group-calling product.

Traditionally, an SFU is written *as a server*: sockets, thread pools, timers, and protocol logic all tangled together. That makes the interesting part — the forwarding state machine — the hardest part to test, and it welds your media core to one runtime, one threading model, and one deployment shape.

`sfu` takes the opposite approach. The library is a **pure state machine**. The caller owns all I/O: it feeds datagrams and signaling in, and drains datagrams and signaling out. The bundled [`chat`](https://github.com/webrtc-rs/sfu/blob/master/examples/chat.rs) example provides one such I/O layer — an HTTPS / WebSocket signaling server plus a UDP media socket — but it is *an* I/O layer, not *the* I/O layer. Swap it for `tokio`, for `smol`, for a thread-per-core io_uring loop, or for a deterministic simulated network in a test, and the forwarding core never notices.

---

## The Sans-I/O Core

`Sfu` is the public entry point, and it implements [`sansio::Protocol<TaggedBytesMut, Infallible, SFUEvent>`](https://docs.rs/sansio) — the same `Protocol` trait the `rtc` crate is built on:

| Method                                                           | Plane     | Meaning                                                                            |
|------------------------------------------------------------------|-----------|------------------------------------------------------------------------------------|
| `handle_read(TaggedBytesMut)` / `poll_write() -> TaggedBytesMut` | media     | push an incoming UDP datagram in / drain an outgoing datagram                      |
| `handle_event(SFUEvent)` / `poll_event() -> SFUEvent`            | signaling | push a signaling request in / drain a signaling response or server-initiated event |
| `handle_timeout(Instant)` / `poll_timeout() -> Instant`          | clock     | advance the caller-supplied clock / ask for the next deadline                      |

Three planes, six methods, no I/O. The read/write planes carry raw datagrams: an inbound datagram is demultiplexed to a client and fed into that client's `RTCPeerConnection`, and the datagrams every client produces are drained back out.

The application plane is deliberately **unused** (`Rout = Win = Infallible`). There is no payload *above* an SFU: inbound media (RTP) and keyframe requests (RTCP PLI/FIR) are forwarded *internally* between clients, never surfaced to the caller. Encoding that in the type system is one of the small pleasures of Sans-I/O — `poll_read()` returning `Option<Infallible>` is a compile-time statement that an SFU forwards, it does not consume.

**All signaling is the event plane.** `SFUEvent` is the unified currency: `Join`, `SessionDescription`, `IceCandidate`, `Leave`, plus `Ok` / `Err` replies. Every variant carries a `request_id`, `room_id`, and `client_id`, which is how the engine routes it.

---

## Architecture: Three Nested Sans-I/O Layers

`Sfu`, `Room`, and `Client` each implement the *same* `Protocol` trait, so signaling is simply routed down and events bubble back up:

```
Sfu   ── owns rooms: HashMap<RoomId, Room>, a Demuxer, the local_addr, transmits/events
 └─ Room   ── owns clients: HashMap<ClientId, Client> and its own Demuxer
     └─ Client ── wraps exactly one rtc RTCPeerConnection
```

`Sfu::poll_write()` is nothing more than a drain of every room's `poll_write()`; `Sfu::poll_timeout()` is the `min` of every room's deadline. The composition is uniform all the way down, and each layer is independently testable in memory.

Three pieces do the real work:

### `Demuxer` — one socket, many clients

Every client is multiplexed over the same UDP socket, so the SFU has to tell datagrams apart before ICE has even completed. Each client is built in **ICE-lite** mode with a *local ufrag that encodes its address*: `"{room_id}/{client_id}+{random}"`. A browser's STUN binding request carries `USERNAME = local_ufrag:remote_ufrag`, so the `Demuxer` recovers `(room_id, client_id)` straight out of the ufrag, then caches the 4-tuple for the DTLS/SRTP phase that follows. `Sfu` holds one demuxer to pick the room; `Room` holds one to pick the client.

### `ForwardTable` — the media routing graph

Owned by `Room`, keyed by `ForwardKey { publisher, mid }`. Note what the key is **not**: it is not the SSRC. The publisher's m-line **mid** is the stable dedup key across renegotiations, with a separate SSRC index used for routing inbound RTP. `Room::reconcile` diffs the desired forwarding matrix whenever membership or published tracks change — adding a sendonly transceiver per new subscriber, tearing down the stale ones — which is what turns "someone joined" into "everyone sees them".

Because a subscriber leg is a *different* peer connection from the publisher leg, forwarding a packet is not a memcpy. The negotiated payload types and RTP header extension IDs can differ between the two legs, so `sfu` translates them per hop — remapping the payload type to the subscriber's codec and rewriting extension IDs into the subscriber's namespace — before the packet goes out.

### `RtcpForwarderInterceptor` — keyframes on demand

Installed as the outermost interceptor on every client, so a subscriber's keyframe requests (RTCP **PLI/FIR**) about a forwarded stream reach the application instead of dying in the receiver. `Room` relays them to the publisher, which is what makes a newly-subscribed track light up promptly rather than after the next scheduled keyframe.

---

## What the Caller Actually Writes

Here's the shape of the media loop from the `chat` example — this is the *entire* contract between the SFU and the network:

```rust
let mut sfu = Sfu::new(random(), local_addr);

loop {
    // 1. drain outbound datagrams to the socket
    while let Some(transmit) = sfu.poll_write() {
        socket.send_to(&transmit.message, transmit.transport.peer_addr)?;
    }

    // 2. push signaling in (join / offer / candidate / leave from the WebSocket)
    if let Ok(command) = rx.try_recv() {
        handle_command(&mut sfu, command, &mut subscribers);
    }

    // 3. drain signaling out (answers and server-initiated subscribe re-offers)
    drain_sfu_events(&mut sfu, &mut subscribers);

    // 4. wait until the SFU's next deadline, or until a datagram arrives
    let eto = sfu.poll_timeout().unwrap_or(Instant::now() + Duration::from_millis(100));
    socket.set_read_timeout(Some(eto.saturating_duration_since(Instant::now())))?;
    if let Some(packet) = recv_from(&socket, &mut buf) {
        sfu.handle_read(packet)?;
    }

    // 5. advance the caller-supplied clock
    sfu.handle_timeout(Instant::now())?;
}
```

That's it. No async, no tasks, no channels required — the example happens to use a blocking `UdpSocket` and a plain thread per media port, precisely because it *can*. Your deployment is free to make a completely different choice.

The full flow it drives: an `SFUEvent::Join` creates the room and client; an `SFUEvent::SessionDescription` offer is answered (`set_remote_description` → add the host candidate synthesized from `local_addr` → `create_answer` → `set_local_description`) and the answer is emitted back through `poll_event`; publishing a track prompts **server-initiated subscribe re-offers** to the other clients in the room; an `SFUEvent::Leave` tears the client down, prunes its forwarding entries, and reaps the room once empty.

---

## Testing: The Sans-I/O Dividend

Because the core has no I/O, `cargo test --lib` drives join → offer/answer → publish → forward **entirely in memory**, with no sockets and no wall clock. Those are the tests you want to be fast and deterministic, and they are.

On top of that, `tests/` drives the whole thing end-to-end: each integration test is a headless WebRTC client — built on the async [`webrtc`](https://github.com/webrtc-rs/webrtc) crate, which is a nice demonstration that the family composes — speaking the chat server's TLS-WebSocket signaling protocol. `data_channel_test` covers register/connect/data-channel; `media_test` publishes VP8 tracks and asserts the SFU forwards RTP, intact and in order, to every other peer in the room; and there are dedicated tests for payload-type and RTP header-extension-ID translation across legs. CI boots the `chat` server and runs the suite against it in a container.

---

## Try It Out

The `chat` example is **deployed and live at [https://sfu.rs](https://sfu.rs)** — open it in a couple of browser tabs (or send the link to a friend), join a room, and you are talking through a Sans-I/O SFU written in Rust. No build required.

To run it yourself:

```bash
git clone https://github.com/webrtc-rs/sfu && cd sfu
git submodule update --init --recursive

# HTTPS with a self-signed cert, on loopback
cargo run --example chat -- --force-local-loop
```

Or depend on the library and bring your own I/O:

```toml
[dependencies]
sfu = "0.20.0-rc.3"
```

---

## Roadmap

This is an **rc.3**, and it is honest about what it is: the architecture is fully operational end-to-end — signaling, datagram routing, RTP fan-out, RTCP keyframe relay — and there is real work left before it is the SFU we want to run in production.

### In the media engine

1. **Simulcast support** — the `ForwardTable` is already keyed by mid rather than SSRC, so multiple encodings of one track bind to the same key. Layer selection and per-subscriber switching build on that.
2. **A full publish/subscribe model** — today a room forwards every publisher to every subscriber. The next step is explicit subscription control over SDP m-line directions, so a large call forwards only the tiles each client actually renders.
3. **Better congestion control** — TWCC on both the uplink and the downlink, so the SFU can react to what each leg can actually carry, instead of forwarding hopefully.
4. **RTX / NACK per hop** — retransmission handled between client and server on each leg independently, rather than letting loss on one hop propagate end-to-end.
5. **Quality improvements** — demuxer affinity expiry and eviction policies, more interop coverage, and performance work.

### In signaling: AppRTC becomes the front door

The `chat` example carries its own WebSocket signaling, which is fine for a demo and not what you want to build a product on. The plan is to give the family **one signaling plane that serves both P2P and SFU calls**, and that plane is the new **[`apprtc`](https://github.com/webrtc-rs/apprtc)** crate — the Rust replacement for the legacy Python Room Server and Go Collider.

The shape we're building toward:

- **≤ 2 participants stay peer-to-peer.** A 1:1 call never touches the SFU — there is no reason to pay for a hop that buys nothing.
- **On the 3rd joiner, the room upgrades P2P → SFU.** Each client publishes its tracks, and the SFU offers back the others': browser ⇄ SFU ⇄ browser.
- **The SFU boundary never changes.** How signaling bytes reach the state machine — an in-process channel, a localhost WebSocket, a cross-host WebSocket — is a *driver* detail. That is precisely what Sans-I/O buys us here.

A demo is already online at **[https://appr.tc](https://appr.tc)**, though today it only speaks the **P2P** half of that story. Wiring it to the SFU for group calls is the next big milestone.

If any of this is the itch you've been meaning to scratch, this is a very good moment to jump in: the codebase is small (the core library is under 3,000 lines), the boundaries are clean, and Sans-I/O means you can write a test for your change without a network.

---

## Get Involved

- **Try it**: open [https://sfu.rs](https://sfu.rs), or point your own signaling at the SFU state machine
- **Break it**: browser interop, lossy networks, big rooms — we want the bug reports
- **Build it**: simulcast, pub/sub, TWCC, RTX/NACK, and the AppRTC signaling plane are all open and all well-scoped
- **Discuss it**: tell us what an SFU needs to be useful to *you*

Join our community:

- **GitHub**: https://github.com/webrtc-rs/sfu
- **Discord**: https://discord.gg/4Ju8UHdXMs
- **Discussions**: https://github.com/webrtc-rs/webrtc/discussions

---

## Conclusion

`sfu` v0.20.0-rc.3 is the first release of a Selective Forwarding Unit built the way we've been building everything since the Sans-I/O shift:

- **A pure state machine** — no sockets, no threads, no clock in the library
- **Three nested Sans-I/O layers** — `Sfu` → `Room` → `Client`, each implementing the same `Protocol`
- **Real media forwarding** — RTP fan-out with per-hop payload-type and header-extension translation, plus RTCP PLI/FIR relay
- **You own the I/O** — the `chat` example is one deployment shape, not the only one
- **Tests without a network** — the interesting logic is testable in memory

The Sans-I/O core made the async `webrtc` crate possible. It turns out it makes a SFU server possible too. 🦀

---

## Links

- **GitHub**: https://github.com/webrtc-rs/sfu
- **Homepage / live demo**: https://sfu.rs
- **Docs**: https://docs.rs/sfu
- **Crate**: https://crates.io/crates/sfu
- **rtc (Sans-I/O core)**: https://github.com/webrtc-rs/rtc
- **webrtc (async)**: https://github.com/webrtc-rs/webrtc
- **apprtc (signaling)**: https://github.com/webrtc-rs/apprtc — live demo at https://appr.tc
- **Discord**: https://discord.gg/4Ju8UHdXMs

---

## Further Reading

- [WebRTC v0.20.0-rc.1: Toward Stable Async-Friendly WebRTC Built on Sans-I/O](/blog/2026/06/30/webrtc-v0.20.0-rc.1-toward-stable-async-webrtc)
- [WebRTC v0.20.0-alpha.1: Async-Friendly WebRTC Built on Sans-I/O](/blog/2026/03/01/webrtc-v0.20.0-alpha.1-async-webrtc-on-sansio)
- [Building a WebRTC Pipeline with Sans-I/O](/blog/2026/01/04/building-webrtc-pipeline-with-sansio)
