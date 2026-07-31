# The Path to `webrtc` v1.0

[`webrtc` v0.20.0 shipped yesterday](/blog/2026/07/31/announcing-webrtc-v0.20.0) — the first non-prerelease of the Sans-I/O architecture. This post is about the next number, what it will mean, and what still has to happen first.

**Treat v0.20.0 as the proving ground for v1.0.** The largest architectural rough edges are fixed: the runtime abstraction no longer has a [seam through the middle of it](/blog/2026/07/30/pluggable-async-runtime), the interceptor type parameter no longer [leaks into your data structures](/blog/2026/07/28/boxed-interceptor-type-erasure), and the callback-lifetime problems of `v0.17.x` are gone with the callbacks. What remains is a short pre-1.0 API checklist and, more importantly, validation by applications outside this repository. This is the window in which feedback can still reshape the API cheaply. After v1.0 we can extend and deprecate it compatibly, but a breaking correction would wait for the next major release.

That framing is the one [Tokio used before its own 1.0](https://tokio.rs/blog/2020-10-tokio-0-3), and we are adopting the substance of it too: **v1.0 is a commitment to API stability, not a claim of feature completeness.** Those are different promises, and conflating them is how projects either ship a v1.0 they have to break, or never ship one at all.

So this post says plainly what we are promising, what v1.0 will not require, and the handful of things still in the way.

---

## What v1.0 will mean

**A stable API.** Once v1.0 ships, the `webrtc` and `rtc` public APIs evolve under normal Semantic Versioning compatibility rules. Additive change continues — new methods, new interceptors, new runtimes — while breaking change waits for a new major version.

**What it will not mean:** that every W3C feature is implemented, or that every subsystem a production media server needs is present. The gaps are named below. None of them gates the release, and any that are finished in time will be in it.

We think this is the honest framing. The alternative — waiting until the feature list is complete — would push v1.0 out by a long way while the API sits in a `v0.x` that signals "may break at any time" to everyone evaluating it. The architecture and API direction are ready; the remaining freeze preparation is named below. Finish that work, then keep shipping features against a stable surface.

---

## W3C compliance today

The distinction above — stable API versus complete feature list — needs a concrete inventory, not an adjective. The best existing measurement is the compliance audit of the core.

The [compliance summary](/blog/2026/01/24/full-webrtc-api-compliance-analysis) published in January scored the Sans-I/O `rtc` core against the [W3C WebRTC 1.0 spec](https://www.w3.org/TR/webrtc/) interface by interface. Eight interfaces came out at 100%:

`RTCPeerConnection` · `RTCSessionDescription` · `RTCIceCandidate` · `RTCDataChannel` · `RTCRtpReceiver` · `RTCRtpTransceiver` · `RTCStatsReport` · `RTCCertificate`

with `RTCRtpSender` at 95% (the missing piece is DTMF), and the three transport interfaces at 85–90%.

Two honest qualifications on that number, because it is widely quoted as "95%+ compliant" and the details matter:

- **It measured `rtc`, not `webrtc`.** The transports score 85–90% because the *core* `rtc` implements them. The async `webrtc` crate does not currently expose `RTCIceTransport`, `RTCDtlsTransport`, or `RTCSctpTransport` at all — see “Transport objects” below for why that does not gate the release. The async layer maps many core operations into object-safe handles, but the core's percentage should not be transferred to that surface without a separate audit.

- **It predates this release.** The analysis is from January; `v0.20.0` is a rewrite of the async layer that has since added `RtpSender::set_parameters`, `RtpTransceiver::set_direction`, and the `RtpReceiver` CSRC/SSRC accessors — all W3C members that the January table predates. We will refresh the table against `v0.20.0` before v1.0 rather than keep citing a six-month-old figure.

What that leaves is the useful part: **the known residual has names.** The refresh may find more, and if it does, we will publish those too.

| Known gap | Status for v1.0 |
|---|---|
| Transport interface accessors | Candidate — additive once the extensibility pass lands |
| `RTCDTMFSender` | Out of scope |
| Identity provider / assertion | Out of scope for the current v1.0 plan |

---

## Decisions now final

Several API questions have been open since the [January architecture post](/blog/2026/01/31/async-friendly-webrtc-architecture). They are settled. Recording them here so nobody is waiting on a change that is not coming.

### 1. The event handler takes `&self`

The January design promised `&mut self` on handler methods and "no `Arc` explosion". What shipped is `&self` on an `Arc<dyn PeerConnectionEventHandler>`, so mutable handler state goes behind a lock:

```rust
struct MyHandler {
    state: Mutex<MyState>,   // interior mutability, once
}
```

**This is the final design.** It is still a large improvement on `v0.17.x`, where every callback needed an `Arc::clone` before the closure and another inside it — you now write one `Arc` and one lock for the whole connection. But it is not the `&mut self` story the January post described, and we would rather close the question than leave it looking like an unfinished migration.

### 2. No peer-connection-level stream API

January floated `pc.tracks()` and `pc.ice_candidates()` as an optional stream layer over the handler. We are not planning that layer. The shipped API splits event consumption by responsibility: connection-level lifecycle and discovery go through `PeerConnectionEventHandler`, while `DataChannel`, `TrackLocal`, and `TrackRemote` each expose `poll()` for their own events. Applications therefore pull media and data per object instead of multiplexing everything through one peer-connection stream.

Treat the January proposal as superseded rather than outstanding.

### 3. `#[async_trait]`, and why native `async fn` in traits does not work here

The object-facing async traits in the crate use `#[async_trait]`. This gets read as legacy — Rust stabilised `async fn` in traits in 1.75, so why the macro? The answer is not maturity. It is two specific limitations in stable Rust, and the first is a hard blocker for this API.

`async fn foo(&self)` in a trait desugars to `fn foo(&self) -> impl Future<Output = ()>` — a return-position `impl Trait` in trait (RPITIT).

- **RPITIT is not `dyn`-compatible.** Each implementation returns a different anonymous future type, of a size no vtable can describe, so a trait with an RPITIT method cannot become a trait object. The current API deliberately uses objects such as `Arc<dyn PeerConnectionEventHandler>`, `Arc<dyn DataChannel>`, `Arc<dyn TrackRemote>`, and `Arc<dyn RtpSender>` throughout. Adopting native async methods would mean either giving those objects up or threading generic parameters through `PeerConnection`, the driver, transports, data channels, and transceivers — the exact viral-parameter problem [the runtime design was shaped to avoid](/blog/2026/07/30/pluggable-async-runtime).

- **AFIT gives no `Send` bound.** The driver awaits handler methods inside its task, and that task is spawned through `Runtime::spawn(Pin<Box<dyn Future<Output = ()> + Send>>)`. The returned futures must therefore be `Send` — and `async fn` in a trait has no syntax to say so. A caller only knows it receives `impl Future`.

The common alternatives each solve only part of the problem:

| Approach | Result |
|---|---|
| Hand-desugar to `-> impl Future<Output = ()> + Send` | Fixes (2). Still RPITIT, still not `dyn`-compatible. Loses `async fn` ergonomics for implementors. |
| `trait_variant::make` | Fixes (2) only. |

For the public shape we want, the practical result is still a boxed `dyn Future + Send`. The cost is one future allocation per async trait call. The lower socket and protocol-driving paths remain poll-based, but this cost is not exclusively control-plane: object-safe application methods such as data-channel sends also cross an async trait boundary. The v0.20.0 throughput results include that cost; it is a known trade rather than a hidden one.

We will revisit when native dyn-compatible async trait methods can express the bounds this API needs. Until then this is a language constraint, not a preference for macro-generated futures.

---

## What has to land before v1.0

Three items. The first is tiny, the second is a broad compatibility review, and the third is the largest implementation change. None is a research project.

### 1. Stop advertising ULPFEC by default

The default media engine registers `video/ulpfec`, but the receive path does not recover media from ULPFEC packets. FlexFEC MIME constants exist but are not in the default codec list; RTX is registered separately and is implemented, so neither belongs in this cleanup.

This is the only *correctness* item on the list: stop offering ULPFEC by default until recovery exists. The change is small, reversible, and safer than negotiating a capability the receive path cannot provide.

### 2. Make the API extensible before freezing it

This is the least glamorous item and probably the most consequential, because it decides how much of everything else has to happen before v1.0 rather than after.

Across the two top-level crate source trees, there are currently **40 directly declared public enums, none marked `#[non_exhaustive]`, and no sealed-trait pattern.** That count deliberately excludes enums re-exported from the protocol subcrates; the compatibility audit must include those re-exports as well. Frozen as-is, that means:

- Adding a variant to an exhaustively matched public enum — a new ICE candidate type, a new DTLS role, a new stats entry — breaks downstream `match` expressions.
- Adding a required method to a public trait breaks external implementors. A default method can preserve source compatibility, but not every future operation has a meaningful default.

The work is mechanical only after the ownership decisions are made. For each public enum: can this ever gain a variant? If yes, mark it `#[non_exhaustive]`. For each public trait: is this an extension point for users, or an interface implemented only by the library? Keep user extension points such as `Runtime`, `PeerConnectionEventHandler`, and application-provided tracks open; seal implementation-owned traits where doing so buys compatible evolution.

Those choices have to be made before the freeze, because adding `#[non_exhaustive]` or sealing a trait later is itself a breaking change for code that already relies on exhaustive matches or downstream implementations.

### 3. Crypto provider unification

[`rtc` #128](https://github.com/webrtc-rs/rtc/issues/128) proposes unifying crypto across DTLS, SRTP, STUN, and the top-level `rtc` crate behind an `RTCCryptoProvider` trait. It touches public API, which makes it dramatically cheaper before v1.0 than after. This is the largest of the three and the one whose cost grows most sharply if deferred.

Getting this right is what makes the next section possible.

---

## What does not gate v1.0

The implementation work in this section is a **candidate for v1.0, not a release requirement.** None of it will hold the release — but the release is not a cut-off either. Anything here that is finished before v1.0 ships goes into v1.0; anything that is not can follow in a compatible point release, which the extensibility pass above is there to make possible.

We are naming them because "the API is stable" should not be read as "everything is present", and because knowing what is absent today is more useful than a promise about when it arrives.

### 1. Congestion control

This is the honest one, and the most important thing in this post for anyone shipping video.

We have TWCC sender and receiver interceptors, so feedback is generated and received. What we do not have is a **bandwidth estimator or a pacer**: `target_bitrate` appears in our stats structures and nowhere in any control path. There is no send-rate adaptation.

For data channels this specific gap does not apply — SCTP runs its own congestion control, and [that path is fast](/blog/2026/07/18/from-13-mbps-to-beating-pion). For **media**, it is the difference between "can send video" and "can send video over a congested link without making it worse". Production browser stacks include media congestion controllers; we do not yet have an equivalent.

Under a v1.0 that promises API stability rather than feature completeness, this not gating the release is consistent — and if an estimator and pacer land before v1.0 ships, they ship with it. What we will not do is delay the API freeze waiting for them. **If you are shipping media over unpredictable networks today, this is the gap to plan around.**

### 2. Transport objects

The W3C API exposes `RTCPeerConnection.sctp`, `RTCRtpSender.transport`, and `RTCIceTransport` with `getSelectedCandidatePair()`. The Sans-I/O core owns all three; the async layer surfaces none of them. `SctpTransportStats` is also missing from our stats set.

We had these as release-gating and have moved them out, because the architecture makes them a design job rather than a patch. In the core they are `pub(crate)` state machines living inside handler contexts, holding live `Agent`, `dtls::Endpoint`, and `sctp::Endpoint` state; the async layer keeps the whole core behind a mutex owned by the driver. Exposing them properly likely means three new handle types reaching back through that mutex, plus a coherent transport-event model. The handler already reports ICE connection and gathering state, but it has no DTLS- or SCTP-transport handle events.

That is worth doing carefully rather than quickly, which is why it is not gating. With item 2 above done first it is purely additive — a `#[non_exhaustive]` stats enum accepts a new variant, and sealed traits accept new methods — so it can land in v1.0 if it is ready, or in a point release if it is not. **The prerequisite is the extensibility pass, not the release.**

### 3. Also candidates

Jitter buffering, ULPFEC/FlexFEC recovery, an FFI C API, embedded/`no_std`, and TURN over TCP in the async layer are all possible follow-on work. None blocks a stable API, and each can ship in v1.0 if it is done in time.

Two W3C areas are **out of scope for the current v1.0 plan** rather than candidates. `RTCDTMFSender`: `audio/telephone-event` is registered in the media engine, but there is no browser-style tone-generation API; applications that need it can generate telephone-event RTP themselves. Identity provider and assertion: we have not seen enough server-side demand to justify adding them to the freeze checklist.

---

## Where we diverge from the Tokio playbook

Tokio's path to 1.0 included removing non-1.0 crates — `bytes`, `mio` — from its public API, so that a breaking change in a dependency could not force a breaking change in Tokio.

**We are doing the opposite, deliberately.** `webrtc::runtime` re-exports `quinn-udp`'s `Transmit`, `RecvMeta`, `EcnCodepoint`, `UdpSockRef`, and `UdpSocketState`, and the first two appear in `AsyncUdpSocket`'s method signatures. An incompatible `quinn-udp` change therefore reaches our public API unless we carry a compatibility layer.

The reasoning: the alternative is maintaining a parallel set of wrapper types that add no information and must be kept in sync forever. We tried that — an earlier design had a `UdpBatchState` wrapper — and removed it, because a runtime author who already knows quinn's socket API had to learn our restatement of it instead. Re-exporting means there is nothing to drift.

It is a real trade and we would rather name it than have someone discover it. If `quinn-udp` releases a major version, we will need a major version or a compatibility shim. We think that is a better problem than a permanent wrapper layer, but reasonable people would call it differently — Tokio did.

---

## Sequencing

We are not naming a date. The critical path is clear: remove ULPFEC from the defaults, complete the extensibility audit across the top-level crates and their re-exported protocol types, then finish the crypto-provider work in [`rtc` #128](https://github.com/webrtc-rs/rtc/issues/128). v1.0 ships when those are done — we would rather move when the work is finished than announce a month and then move it.

Alongside them, the work that does **not** gate the release but is where help matters most:

- **Live-browser interop in CI.** There is no Playwright, Selenium, or equivalent job running real browser engines. The repository does contain useful protocol-level fixtures — including a captured Firefox 152 simulcast offer and Safari 26.5.2 SDP — but a recorded offer is not a live browser. Chrome, Firefox, Safari, and Edge should all become repeatable CI targets.
- **Continuous fuzzing.** Fuzz targets already exist for SDP, STUN, RTP, RTCP, SCTP, and DTLS parsers. What is missing is persistent execution in CI or a hosted service, corpus management, and a regular triage loop. The Sans-I/O design makes that unusually tractable because the targets are pure byte-oriented protocol machinery with no network environment to construct.
- **Congestion control.** The largest single piece of work in the project. If you have worked on GCC, BBR, or pacing, this is the highest-leverage thing you could contribute.

---

## What we are asking for

The point of announcing a path rather than just shipping is that **API feedback is most useful before the API freezes.** After v1.0 we can add alternatives and deprecate old paths, but correcting a mistake without preserving compatibility becomes major-version work.

So: if something in `webrtc` or `rtc` is awkward to use, now is when saying so changes anything. Particularly valuable —

- **Porting reports from `v0.17.x`.** What was mechanical, what forced a redesign.
- **Anything you had to work around** rather than express directly in the API.
- **Missing surface you expected to find**, especially the transport objects above — tell us if the shape we are planning is the shape you need.

Open a [GitHub issue](https://github.com/webrtc-rs/webrtc/issues), or come to [Discord](https://discord.gg/4Ju8UHdXMs).

---

## Links

- **v0.20.0 release**: [Announcing `webrtc` v0.20.0](https://webrtc.rs/blog/2026/07/31/announcing-webrtc-v0.20.0.html)
- **Repo**: [github.com/webrtc-rs/webrtc](https://github.com/webrtc-rs/webrtc)
- **Sans-I/O core (`rtc`)**: [github.com/webrtc-rs/rtc](https://github.com/webrtc-rs/rtc)
- **Docs**: [docs.rs/webrtc](https://docs.rs/webrtc)
- **Discord**: [discord.gg/4Ju8UHdXMs](https://discord.gg/4Ju8UHdXMs)

---

## Further Reading

- [Announcing webrtc v0.20.0](/blog/2026/07/31/announcing-webrtc-v0.20.0)
- [Bring Your Own Async Runtime](/blog/2026/07/30/pluggable-async-runtime)
- [How We Made webrtc-rs Data Channels Fast](/blog/2026/07/18/from-13-mbps-to-beating-pion)
- [Building Async-Friendly webrtc on Sans-I/O rtc](/blog/2026/01/31/async-friendly-webrtc-architecture)
- [Announcing Tokio 0.3 and the path to 1.0](https://tokio.rs/blog/2020-10-tokio-0-3)
