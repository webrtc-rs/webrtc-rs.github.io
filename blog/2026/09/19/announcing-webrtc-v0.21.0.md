# Announcing `webrtc` and `rtc` v0.21.0: The 1.0 Release Candidate

We're happy to announce **`webrtc` v0.21.0** and **`rtc` v0.21.0**.

Treat this release as **1.0.0-rc.1** in everything but its version number. The next release is intended to be **1.0.0**. Between here and there, the public API of both crates is meant to stay where it is. The only thing that would justify a breaking change is a significant bug that cannot be fixed any other way.

That is a change of plan, and it deserves an explanation first.

---

## Why there is a 0.21 at all

Seven weeks ago, [The Path to `webrtc` 1.0](/blog/2026/08/01/the-path-to-webrtc-1.0) said the number after 0.20.0 would be 1.0. It named three gating items and said 1.0 would ship when they were done.

All three were done within four days. ULPFEC left the default codecs on August 1, the extensibility audit merged the same day, and the crypto-provider rewrite landed on August 4.

Going straight to 1.0 would have kept that promise to the letter and missed its point. The same post listed a set of **candidates**: work that would not hold 1.0 back but could ship in it if finished in time. Congestion control, transport objects, jitter buffering, and FEC recovery were all on that list. Once the gating work was out of the way, most of the candidates were finished too, and several of them changed public API:

- Congestion control needed a new interceptor chain. The chain type parameter disappeared from `RTCPeerConnection`, and `Registry::with` now takes a `Slot`.
- The crypto provider replaced every default-resolving constructor across DTLS, SRTP, STUN, ICE, and TURN.
- Transport objects added three new handle traits to the async crate.
- One data-channel fix required changing the type of `RTCDataChannelId`.

Freezing an API the week it was written is exactly what the 1.0 post warned against: **1.0 is a commitment to API stability**, and you can only make that commitment for an API that people have had time to use. So we inserted one more release. v0.21.0 is the API we intend to freeze. It went through alpha, beta, and two release candidates in August and September, and it now needs time with real applications before we put a 1 in front of it.

The rule until 1.0:

> **v0.21.x → v1.0.0 keeps the public API of `webrtc` and `rtc` stable.** Bug fixes and additive changes are fine. A breaking change needs a significant bug that cannot be fixed compatibly, and if one is needed it will be called out as such.

The extensibility work described below is what makes that rule realistic. Most of the changes we can foresee can now be made without breaking anyone.

---

## The scorecard

A roadmap post is only useful if someone later checks it against what shipped. Here is everything [The Path to 1.0](/blog/2026/08/01/the-path-to-webrtc-1.0) promised or named, and where each item ended up.

### What had to land before 1.0 (gating)

| Item | Status in 0.21.0 |
|---|---|
| Stop advertising ULPFEC by default | ✅ Done ([#837](https://github.com/webrtc-rs/webrtc/issues/837)) |
| Make the API extensible before freezing it | ✅ Done ([#838](https://github.com/webrtc-rs/webrtc/issues/838)) |
| Crypto provider unification ([`rtc` #128](https://github.com/webrtc-rs/rtc/issues/128)) | ✅ Done ([#839](https://github.com/webrtc-rs/webrtc/issues/839)) |

### What did not gate 1.0 (candidates)

| Item | Status in 0.21.0 |
|---|---|
| Congestion control: bandwidth estimator + pacer | ✅ Shipped: GCC estimator, pacer, RFC 8888 feedback |
| Transport objects (`sctp`, `transport`, `IceTransport`) | ✅ Shipped in both crates |
| Jitter buffering | ✅ Shipped as an interceptor |
| FEC recovery | ✅ FlexFEC (draft-03) end to end. ULPFEC recovery is not supported, by design |
| FFI C API, embedded / `no_std` | Out of scope, as planned |
| `RTCDTMFSender`, identity provider | Out of scope, as planned |

### Where we asked for help

| Item | Status in 0.21.0 |
|---|---|
| Live-browser interop in CI | ✗ Not yet. Browser interop was fixed by hand this cycle (see below), but no CI job runs a real browser |
| Continuous fuzzing | ✗ Not yet. The fuzz targets exist, but nothing runs them on a schedule |
| Refresh the W3C compliance table | ✗ Not yet. The "95%+" figure is still the January analysis of `rtc` |

The last two tables need an honest reading. The API work is ahead of what we said. The **validation work is behind**: browser interop in CI, continuous fuzzing, and the refreshed compliance table did not happen. None of them changes the API, which is why they do not block the freeze. They are also a big part of why this is a release candidate and not 1.0, and they are where help is most valuable. See [What we are asking for](#what-we-are-asking-for).

---

## The three gating items

### 1. ULPFEC is no longer offered by default

`MediaEngine::register_default_codecs` no longer registers `video/ulpfec`. The receive path cannot recover media from ULPFEC packets, so offering the codec invited peers to send repair packets we then threw away. `MIME_TYPE_ULP_FEC` is still public for applications that register it deliberately.

**This changes the default offer's SDP:** payload type 116 is gone. ULPFEC recovery is not supported by design, so ULPFEC will not come back to the defaults. FlexFEC is the FEC scheme we support, with recovery on the receive path (see below).

### 2. The API can now evolve without breaking you

`#[non_exhaustive]` and sealed traits have one awkward property: **adding either one after 1.0 is itself a breaking change.** So this audit had to happen before the freeze. That is why it was on the gating list even though it was not glamorous work.

The 1.0 post counted 40 public enums across the two top-level crates, with none marked `#[non_exhaustive]` and no sealed traits. It also warned that the enums re-exported from the protocol subcrates had to be included. They were, because `rtc` re-exports all fifteen of those subcrates wholesale, which puts their public surface in `rtc`'s public API. The result:

| | `#[non_exhaustive]` | Kept exhaustive |
|---|---:|---:|
| `rtc` top-level enums | 34 | 0 |
| `rtc` protocol subcrate enums | 69 | 19 |
| `webrtc` enums | 3 | 3 |

The rule behind every decision is written down in `docs/semver.md` in both repositories. **An enum is `#[non_exhaustive]` when something outside this codebase defines its variants and can add more**: an IANA registry, a W3C enum, a protocol state machine, an event stream. **It stays exhaustive when the set of variants is closed by construction**: `Controlling`/`Controlled`, a CCM tag of 8 or 16 bytes, "full" and "closed" as the only ways a non-blocking send can fail. For an enum like that, being able to match every variant is worth more to callers than room for a variant that cannot exist.

Traits follow the same logic. **Library-implemented traits are sealed**: `PeerConnection`, `DataChannel`, `RtpSender`, `RtpReceiver`, `RtpTransceiver`, and the new `SctpTransport`, `DtlsTransport`, and `IceTransport`. Sealing lets us add required methods in a minor release. **Extension points stay open**: `Runtime` and its socket and timer traits, `PeerConnectionEventHandler`, the `Track` traits, and the new `RTCCryptoProvider`. These exist so that you can implement them. The trade-off is that a new method on one of them has to come with a default body.

Both repositories now run `cargo-semver-checks` in CI, so from here on an accidental break fails the build instead of reaching a release.

### 3. Crypto provider unification

We covered this in detail in [Bring Your Own Crypto Provider](/blog/2026/09/01/pluggable-crypto-provider). The short version:

- **All cryptography in the stack goes through one provider-neutral crate, `rtc-crypto`.** No protocol crate depends on `ring`, `aws-lc-rs`, or a RustCrypto primitive crate any more. `webrtc` depends on no crypto implementation at all, and a CI check keeps it that way.
- **A provider is a value, not a global.** `RTCCryptoProvider` bundles an `RTCCrypto` operations trait and an `RTCRandom` CSPRNG. You pass it per peer connection, so two connections in one process can use different providers. There is nothing to install at startup.
- **Bring your own.** OpenSSL, a FIPS-validated module, an HSM, a platform API, or a deterministic test double: implement three traits, then run `rtc_crypto::conformance::assert_provider` against your implementation. It is the same RFC-vector suite the built-in providers pass.
- **The `crypto-ring` and `crypto-aws-lc-rs` features are now additive.** The four `compile_error!` guards that broke builds under Cargo feature unification are gone. `ring` remains the default when both are enabled.
- **No performance regression.** Every per-packet and per-record benchmark on the default provider is at parity with the pre-migration baseline or faster. Keyed `Mac` objects moved the HMAC key schedule off the per-packet path, which made SRTCP about 40% faster per packet.

Selecting a provider for a peer connection is one line on the setting engine:

```rust
let setting_engine = SettingEngineBuilder::new()
    .with_crypto_provider(Arc::new(MyProvider::new()))
    .build();
```

This was the most expensive item to do after 1.0, and it is also where most of 0.21's breaking changes come from. The migration table is in the "Breaking changes" section of [that post](/blog/2026/09/01/pluggable-crypto-provider), and `rtc/docs/crypto-provider-migration.md` has before/after examples.

---

## The candidates that made it

### Congestion control, and the chain that holds it

The 1.0 post called this "the honest one": TWCC feedback was generated and received, but nothing estimated bandwidth and nothing paced. `target_bitrate` appeared in our stats structures and nowhere in any control path.

That has changed. We covered it in detail in [The Interceptor Chain Is a List, Not a Tower](/blog/2026/09/01/interceptor-chain-list-not-tower). The short version:

- **The built-in interceptors went from six to thirteen.** The new ones are a congestion controller, a pacer, a FlexFEC encoder and decoder, a jitter buffer, an RFC 8888 feedback recorder, and interval PLI.
- **The estimator is pluggable.** `BandwidthEstimator` is the extension point. `Gcc` is a full Google Congestion Control implementation (Kalman filter, adaptive overuse threshold, loss controller, AIMD rate control), and `ConstantBitrate` is a real option for when you already know the path. The estimate appears as `targetBitrate` on each outbound RTP stream's stats, so an application can drive its encoder from it.
- **TWCC or RFC 8888, one or the other.** Both are turned into the same per-packet reports before the estimator sees them.
- **The chain is now a flat list, not a nested generic tower.** The new interceptors act in both directions and need to share information across directions. They also have to see every packet that leaves, including retransmissions that other interceptors generate. A nested `A<B<C>>` could do none of these things. The flat list walks one way for reads and the other way for writes, and packets carry `Attribute`s that let one interceptor tell another something without either holding a reference to the other.
- **Order is data.** Each interceptor sits at a `Slot`. The named slots are 1,000 apart, and the gaps are there for your own interceptors: `Slot::from(3_500)` means "above the pacer" and keeps meaning that across upgrades.
- **`RTCPeerConnection` has no type parameter any more.** It costs one virtual call per interceptor per packet, which is small next to SRTP, and in exchange there is no chain type for your code to name.

Congestion control is opt-in, because pacing adds queueing delay and an application should not get that without asking:

```rust
let registry = configure_congestion_control(
    registry,
    Gcc::default(),
    CongestionFeedback::Twcc,
    &mut media_engine,
)?;
```

Three new examples cover this: `bandwidth-estimation-from-disk`, `play-from-disk-fec`, and `save-to-disk-fec`.

### Jitter buffer and FlexFEC

Both ship as interceptors, so both are opt-in and neither needed a change to the peer-connection API.

The **jitter buffer** releases packets on `handle_timeout`, in the caller's time base. That was an open design question in July, and the deterministic-time work below answered it. The buffer also sits in the right place relative to NACK: it is what creates the window in which a retransmission is still useful.

**FlexFEC (draft-03)** now works end to end: the encoder protects, packets are lost on the path, and the decoder rebuilds them. It is tested with induced loss, not just on loopback. A recovered packet reaches the NACK generator like any other arrival, so we do not ask the remote to resend a packet we have already rebuilt.

### Transport objects

The 1.0 post moved these out of the gating list because they were "a design job rather than a patch". The design took a few weeks, and it followed the W3C spec closely:

```rust
// Data: the only transport the spec puts on RTCPeerConnection.
let sctp = pc.sctp().await.expect("SCTP negotiated");
let dtls = sctp.transport();
let ice  = dtls.ice_transport();
let pair = ice.get_selected_candidate_pair().await?;

// Media: through a sender or receiver. This is the only way in on a
// connection without a data channel.
let dtls = sender.transport().await?.expect("sender is associated");
```

There is deliberately no `pc.dtls_transport()` or `pc.ice_transport()`, because the spec does not have them. An earlier draft added lookup by id, and we removed it as unfaithful to the spec. Transports are compared by `id()` rather than by reference. State is read, not delivered: there are no `onstatechange` events. `max_message_size` is always a finite number, because the value we report is the value we enforce.

Every place where the Rust API differs from the W3C Recommendation is listed with its reason in `docs/transport-objects.md`. One of those differences is a known deviation we have not fixed yet: `sctp()` does not go back to `None` after a renegotiation removes the data channel. Check `state()` if that matters to you.

---

## More than we promised

Several pieces of work were not on the 1.0 list at all.

### Deterministic time: the core is told the time, it never asks

> **Sans-I/O protocol code is told the time. It does not ask.**

A Sans-I/O core that calls `Instant::now()` is only partly Sans-I/O: you cannot replay it, and you cannot test a 30-second timeout without waiting 30 seconds. In 0.21 every protocol object in `rtc` makes its decisions against an instant its caller supplied, through `handle_timeout(now)`, the timestamps on inbound messages, or `now` fields in constructor configs. The only wall-clock reads left are the ones that are actually about the wall clock:

- the NTP timestamp in an RTCP sender report
- DTLS `gmt_unix_time`
- the SDP session version
- X.509 validity windows. A virtual instant would happily accept an expired certificate.

A CI script treats the remaining clock reads as a ratchet. The allow-list can only shrink, and a new `Instant::now()` in protocol code fails the build.

On the async side, `Runtime` gained a defaulted `now()`, and the driver takes its time from there. The new `runtime-mock` feature provides a `MockRuntime` whose virtual clock reaches ICE timeouts, DTLS retransmits, and SCTP RTO. An end-to-end test drives a whole connection on virtual time. Existing `Runtime` implementations keep working unchanged, because the default is the wall clock.

### No silent drops on internal channels

In 0.20, a slow consumer on a **reliable** data channel could lose messages: the driver used `try_send` on a bounded channel, logged an error when it was full, and dropped the message ([#858](https://github.com/webrtc-rs/webrtc/issues/858)).

The fix makes the driver keep the message instead of dropping it. It also doesn't wait for the consumer, because the driver loop also runs ICE consent, DTLS retransmits, and SCTP timers, and blocking it would drop the connection. Instead, the driver **stops pulling** data-channel messages from the core while any are waiting to be delivered. The backlog then builds up in SCTP's reassembly queue, `a_rwnd` shrinks, and the remote peer slows down. SCTP already had the flow control needed for this. The old code bypassed it by draining the queue immediately. Media keeps flowing while this happens, which a dedicated test checks.

All four internal channels now have a written overflow policy (`docs/internal-channel-overflow-policy.md`), and a CI check fails any send site that doesn't declare one. Media can still be dropped when its queue is full, because UDP has no flow control to push back on, but those drops are now counted.

### Browser interop, fixed by hand

Without a browser in CI, these were found the slow way, and each one is worth knowing about if you ran 0.20 against Chrome:

- **TWCC feedback left out packets on unbound SSRCs**, so Chrome read its own probe clusters as lost and its bandwidth estimate never ramped up ([rtc#211](https://github.com/webrtc-rs/rtc/issues/211)).
- **Answers renumbered the offer's payload types** (a violation of RFC 3264 §6.1) and could reorder codecs, so Chrome fell back from simulcast to single-stream VP9 ([rtc#213](https://github.com/webrtc-rs/rtc/issues/213)).
- **A send-direction transceiver without a sender** made `check_negotiation_needed` return true forever, which caused a renegotiation storm ([rtc#212](https://github.com/webrtc-rs/rtc/issues/212)).
- **RTX retransmissions** were not always RTP version 2, and their ordering was nondeterministic ([rtc#214](https://github.com/webrtc-rs/rtc/issues/214)).
- **The first RTP packet of a stream** went through the interceptor chain before its stream was bound ([rtc#207](https://github.com/webrtc-rs/rtc/issues/207)).
- **Answers duplicated `a=rid` lines and doubled the direction in `a=simulcast`** ([rtc#181](https://github.com/webrtc-rs/rtc/pull/181)).
- **When STUN and TURN shared a server address,** the TURN relay consumed the STUN Binding responses, so no server-reflexive candidates were gathered ([#890](https://github.com/webrtc-rs/webrtc/issues/890)).

### Data channels, done by the RFC

- **SCTP stream ids are assigned only after the DTLS role is known** ([rtc#199](https://github.com/webrtc-rs/rtc/issues/199)). RFC 8832 §6 says the stream-id parity depends on the DTLS role, and a channel created before the remote description is applied cannot know that role yet. So `RTCDataChannelId` is now a connection-local handle (`usize`), and the id on the wire is a separate `stream_id() -> Option<StreamId>`. **This is a breaking change**, and the kind we wanted to make before 1.0 rather than after.
- **A data channel opens only after the DCEP handshake completes** ([rtc#171](https://github.com/webrtc-rs/rtc/issues/171)), and an inbound channel is exposed only once SCTP is ready.
- **SCTP fixes**:
  - A permanent zero-window deadlock is fixed ([rtc#217](https://github.com/webrtc-rs/rtc/issues/217)).
  - The initial retransmission timeout now follows RFC 9260.
  - `maxPacketLifeTime` is now measured from when a message is queued, not when it is first sent.
  - Timed reliability is refreshed before retransmission.
  - A new `SettingEngineBuilder::with_sctp_mtu` supports paths with a non-default MTU.

### Surviving the network changing underneath you

- **ICE restart rebinds its UDP sockets** ([#868](https://github.com/webrtc-rs/webrtc/issues/868)). This fixes Android connections that stalled after about 10 seconds in the background.
- **Bind addresses are resolved again on every bind** ([#874](https://github.com/webrtc-rs/webrtc/issues/874)). A wildcard like `0.0.0.0` now means "every interface", with one socket per interface address, so an ICE restart after a Wi-Fi-to-cellular handover picks up the interfaces the device has now. An address that can't be bound is logged and skipped instead of stopping the driver.
- **The driver no longer hot-loops** on an expired timeout that doesn't advance ([#862](https://github.com/webrtc-rs/webrtc/issues/862)).
- **TURN allocation refresh** can be capped (`with_turn_allocation_refresh_interval_cap`), overdue refresh deadlines are rebased, and a zero-lifetime relay no longer breaks.

### Smaller things

- `SettingEngine` is now built with `SettingEngineBuilder`.
- DTLS server configs can hold both a PSK callback and certificates.
- RSA is accepted as a DTLS private key type.
- `nix` was upgraded to a version that supports building for `ohos` (OpenHarmony).

---

## Upgrading from 0.20

The 0.21 breaking changes you are most likely to hit:

| Area | Change |
|---|---|
| Settings | `SettingEngine` is built with `SettingEngineBuilder::new()…build()` |
| Crypto | Constructors take an `Arc<dyn RTCCryptoProvider>`. `CustomSigner` → implement `SigningKey`. `RTCCertificate::from_key_pair` → `RTCCertificate::generate`. The `openssl` features were removed. [Full table](/blog/2026/09/01/pluggable-crypto-provider) |
| Interceptors | `RTCPeerConnection` has no chain type parameter. `Registry::with(Slot, …)`. Custom interceptors lose their `inner` field and forward packets through a queue |
| Data channels | `RTCDataChannelId` is a `usize` handle. The wire id is `stream_id() -> Option<StreamId>` |
| Enums | Many public enums are now `#[non_exhaustive]`, so add a `_` arm to your `match` |
| Codecs | The default offer no longer includes ULPFEC (PT 116) |
| Bind addresses | `PeerConnectionBuilder::build` requires `A: Send + 'static` for configured addresses |

The last row only affects you if you pass borrowed, non-`'static` address strings. `String`, `SocketAddr`, and `&'static str` all still work.

---

## What did not make it

To keep the gaps visible, and to separate them from what is out of scope:

- **Live-browser CI, continuous fuzzing, and a refreshed compliance table.** See the scorecard. None of them affects the API, so none of them blocks 1.0 on its own. But they are how we would *find* the kind of significant bug that could justify breaking it, which is why this release is a release candidate.
- **Out of scope, as planned:** an FFI C API, embedded / `no_std`, `RTCDTMFSender`, and the identity provider. ULPFEC recovery is also not supported by design. FlexFEC is the supported FEC scheme.

The trade we described in the 1.0 post is also still in place: `webrtc::runtime` re-exports `quinn-udp` types in its public API. We are keeping that trade on purpose.

---

## What we are asking for

**This is the last point at which feedback can reshape the API cheaply.** After 1.0, a mistake can be deprecated and routed around, but fixing it properly means a new major version.

Especially valuable right now:

- **Upgrade reports from 0.20 to 0.21.** Tell us which changes were mechanical and which forced you to redesign something.
- **Anything you had to work around** instead of expressing it directly, especially in the new transport handles, the `Slot`/`Attribute` interceptor contract, and the crypto traits.
- **Congestion control on real networks.** The estimator has been tested against a deterministic path simulator with induced loss and jitter. Real networks will find problems that a simulator can't.
- **Browser interop CI and fuzzing.** If you have set up either for another project, these are the most useful things you could contribute before 1.0.

Open a [GitHub issue](https://github.com/webrtc-rs/webrtc/issues), or come to [Discord](https://discord.gg/4Ju8UHdXMs).

---

## Try it

```toml
[dependencies]
webrtc = "0.21.0"   # async, runtime-agnostic
# or
rtc = "0.21.0"      # Sans-I/O core
```

---

## Links

- **Repo**: [github.com/webrtc-rs/webrtc](https://github.com/webrtc-rs/webrtc)
- **Sans-I/O core (`rtc`)**: [github.com/webrtc-rs/rtc](https://github.com/webrtc-rs/rtc)
- **Docs**: [docs.rs/webrtc](https://docs.rs/webrtc) · [docs.rs/rtc](https://docs.rs/rtc)
- **Discord**: [discord.gg/4Ju8UHdXMs](https://discord.gg/4Ju8UHdXMs)

---

## Further Reading

- [The Path to `webrtc` 1.0](/blog/2026/08/01/the-path-to-webrtc-1.0): the plan this release is checked against
- [Bring Your Own Crypto Provider](/blog/2026/09/01/pluggable-crypto-provider)
- [The Interceptor Chain Is a List, Not a Tower](/blog/2026/09/01/interceptor-chain-list-not-tower)
- [Announcing `webrtc` v0.20.0](/blog/2026/07/31/announcing-webrtc-v0.20.0)
- [Bring Your Own Async Runtime](/blog/2026/07/30/pluggable-async-runtime)
