# The Interceptor Chain Is a List, Not a Tower

`rtc` 0.21 takes the interceptors that ship with the stack from six to thirteen — congestion control with a full GCC estimator, a pacer, FlexFEC in both directions, a jitter buffer, RFC 8888 feedback, interval PLI — and rewrites the chain that holds them:

```rust
pub(crate) struct Chain {
    /// Ordered by distance from the wire: index 0 is closest to the network.
    interceptors: Vec<Box<dyn Interceptor>>,
    read_outs: VecDeque<TaggedPacket>,
    write_outs: VecDeque<TaggedPacket>,
}
```

Those two facts are the same fact. The nested generic tower we [described in January](/blog/2026/01/09/interceptor-design-principle-sansio) held the original six interceptors perfectly well, and could not hold these. The new ones are **bidirectional** — each acts on both the inbound and outbound path. They have to **see every byte that leaves**, including the packets other interceptors generate. And they have to **tell each other things**, often across directions, where what one learns from arriving packets steers what another does to departing ones. A tower of `A<B<C>>` fails all three, for reasons that turn out to be the same reason: nesting encodes one static containment relationship, and none of those three is one.

So this post is about two features rather than a refactor. Direction became a property of the walk instead of the structure, and packets grew a place to carry what the chain has learned about them.

---

## What 0.21 ships

| Slot | Interceptor | | Direction |
|---|---|---|---|
| 1\_000 | congestion control — send history, feedback ingest | **new** | read + write |
| 2\_000 | TWCC sender | | write |
| 3\_000 | pacer | **new** | write |
| 4\_000 | NACK responder | | read + write |
| 5\_000 | FlexFEC encoder | **new** | write |
| 6\_000 | FlexFEC decoder | **new** | read |
| 7\_000 | NACK generator | | read + write |
| 8\_000 | TWCC receiver | | read + write |
| 9\_000 | RFC 8888 feedback | **new** | read + write |
| 10\_000 | RTCP receiver reports | | read + write |
| 11\_000 | RTCP sender reports | | write |
| 12\_000 | interval PLI | **new** | write |
| 13\_000 | jitter buffer | **new** | read |

Plus a `BandwidthEstimator` trait with a Google Congestion Control implementation behind it — Kalman filter, adaptive overuse threshold, loss controller, AIMD rate control — and a `ConstantBitrate` estimator that is a legitimate configuration rather than a placeholder, for when the path is known and you would rather an algorithm did not second-guess you.

Look down the last column. Six of the thirteen work in both directions, and those are mostly the new arrivals. That is the pressure that broke the old design.

---

## Why a tower could not hold them

In the old design each interceptor owned the next as a generic field and delegated to it:

```rust
pub struct NackGeneratorInterceptor<P: Interceptor> {
    inner: P,
    // …
}
```

Composed, the value's type spelled out the whole recipe:

```text
TwccReceiverInterceptor<SenderReportInterceptor<ReceiverReportInterceptor<
    NackResponderInterceptor<NackGeneratorInterceptor<NoopInterceptor>>>>>
```

The compiler monomorphizes that into straight-line code with no vtables and inlining across every layer. For a per-RTP-packet pipeline it is the right instinct, and for a set of interceptors that mostly do one thing in one direction it works.

The trouble is that `A<B<C>>` encodes **containment**, and containment is a single, static relationship. The chain needs two.

### 1. An interceptor is not on one side of another one

A pacer sits between the application and the wire on the way out. The congestion controller that tells the pacer its rate sits between the wire and the pacer on the way *in*. Ask which one contains the other and the question has no answer — it depends on which direction you are asking about.

You can express this in a tower by building two of them, one per direction. The price is steep and lands exactly on the new interceptors: "closest to the wire" would mean opposite things in the two structures, every interceptor acting in both directions would exist twice, and any state shared between its two halves would need a lock or a cell. An arrival recorder that records in one direction and emits feedback in the other is then split across both towers, with the ordering constraints of each pulling on a different copy of it.

The alternative is what shipped: **one list, ordered by distance from the wire, and direction is how you walk it.**

```text
read   (network → application)   walk forward:  0 → 1 → 2 → … → N
write  (application → network)   walk reverse:  N → … → 2 → 1 → 0
```

One ordering serving both directions is the load-bearing part. It means "closest to the wire" names one thing, so the send history and the FEC decoder can sit next to each other near index 0 — one being the last thing a packet meets on the way out, the other the first on the way in — and an interceptor that acts in both directions is one object at one position, subject to the constraints of both.

The mechanism is a shared belt that each interceptor is fed from and emits back onto:

```rust
fn walk<'a, T>(
    interceptors: impl Iterator<Item = &'a mut Box<dyn Interceptor>>,
    mut belt: VecDeque<T>,
    handle: fn(&mut dyn Interceptor, T) -> Result<(), Error>,
    poll: fn(&mut dyn Interceptor) -> Option<T>,
) -> VecDeque<T> {
    for interceptor in interceptors {
        while let Some(next) = belt.pop_front() {
            if let Err(err) = handle(interceptor.as_mut(), next) {
                log::warn!("interceptor handle failed: {err}");
            }
        }
        // Whatever it has ready — passed through, transformed, generated, or released — is
        // what the next interceptor in the walk receives.
        while let Some(next) = poll(interceptor.as_mut()) {
            belt.push_back(next);
        }
    }
    belt
}
```

`handle` and `poll` pick the direction's method pair; the iterator picks the direction — `iter_mut()` for read, `iter_mut().rev()` for write. That is the whole of it, and both directions are the same function.

One subtlety, because it is easy to miss when reimplementing this: `poll_read` and `poll_write` walk with an **empty belt** when nothing is pending. A jitter buffer may have released a packet on `handle_timeout` with no inbound packet since to carry it forward; the pacer releases on a timer; generated RTCP appears with no outbound packet to ride along with. Walking only on arrival would strand all of it.

### 2. A packet generated inside the tower skipped everything between it and the wire

This is the old NACK responder, in full:

```rust
fn poll_write(&mut self) -> Option<Self::Wout> {
    // First drain retransmitted packets
    if let Some(pkt) = self.write_queue.pop_front() {
        return Some(pkt);
    }
    self.inner.poll_write()
}
```

Read the two branches. A packet it is *forwarding* comes from `self.inner` — the layers nested inside it, which on the write path are the ones between it and the network. A packet it *generated* is returned straight to the caller instead, and the caller is the layer **outside** it. The retransmission goes up and out. It never touches `self.inner`.

So a generated packet skips every interceptor between its author and the wire. Whether that matters depends entirely on what is down there.

Before 0.21, not much was: the default chain had the NACK generator and the terminus wire-ward of the responder, and a retransmission has no business with either. The bug was latent.

The 0.21 interceptors are precisely the ones it becomes fatal for, because they are the ones that must see **every byte that leaves**. The pacer meters departures. The TWCC sender assigns the transport-wide number the remote will report against. The congestion controller's send history records what actually went out. All three have to sit wire-ward of the NACK responder — that is the only place they can sit, since a retransmission has to reach them — so in a tower every retransmission would leave unpaced, unnumbered, and uncounted. The last one is the worst: an estimator that never sees retransmitted bytes believes the path is carrying less than it is, and raises the target during loss, which is the worst possible moment.

The flat chain has no such branch, because there is no "up and out". An interceptor emits onto the belt, and the belt carries whatever is on it to every interceptor still ahead in the walk:

> Nothing can bypass an interceptor by being generated past it — which is the class of bug the previous, nested design allowed, and why a retransmission used to escape the pacer, the transport-wide numbering and the send history.

A retransmission emitted at 4\_000 is seen by 3\_000, 2\_000 and 1\_000 — not because anyone arranged that, but because there is nowhere else for it to go. The same applies to FlexFEC repair packets at 5\_000 and to every generated RTCP report above them.

### 3. Interceptors have to inform each other

The second thing a tower cannot do is let two interceptors share what they know without one of them holding a reference to the other. With `Ein`/`Eout` left as `()`, the chain has no event channel either.

0.21 needs this constantly, because the new interceptors form a control loop rather than a sequence of independent filters. So packets grew somewhere to carry information:

```rust
/// A packet together with what the interceptors have learned about it.
pub struct AttributedPacket {
    /// What the interceptors have attached on the way, in the order they attached it.
    pub attributes: Vec<Attribute>,
    /// The packet itself.
    pub packet: Packet,
}

pub enum Attribute {
    RecoveredByFec,
    Retransmission,
    DeliverToApplication,
    TargetBitrateChanged { bits_per_second: f64 },
    Custom(Arc<dyn Any + Send + Sync>),
}
```

with `has`, `get`, `with`/`add`, and `custom::<T>()` for downcasting an application's own payload out of `Attribute::Custom`. Attributes travel in **both** directions: attached on the way in from the wire and read further up toward the application, or attached by the application and read on the way down to the wire.

Riding with the packet is more correct than a side channel would be, not merely more convenient. An interceptor that holds a packet and emits it later puts it back on the belt *behind* itself, where every interceptor ahead sees it again — so what was learned about that packet has to travel with it or be recomputed. A side channel cannot manage that: it has no way to say *which* packet it refers to once the packet has been held, reordered, or duplicated.

---

## The worked example: congestion control

This is the interaction the whole design exists for, and it uses both features at once.

Send-side congestion control is four interceptors in a loop. The TWCC sender at 2\_000 stamps every departing packet with a transport-wide sequence number. The congestion controller at 1\_000 records each departure in its send history. The remote reports back what arrived and when — as TWCC, or as RFC 8888 feedback from the recorder at 9\_000. And the pacer at 3\_000 releases packets at whatever rate the estimator currently believes.

Two wire formats, one estimator. The conversion layer flattens them:

```rust
if let Some(feedback) = payload.downcast_ref::<TransportLayerCc>() {
    for acknowledgement in convert_twcc(feedback) {
        self.history.on_twcc_feedback(now, acknowledgement);
    }
    return true;
}

if let Some(feedback) = payload.downcast_ref::<CcFeedbackReport>() {
    let (_report_delay, per_stream) = convert_ccfb(feedback);
    // …
}
```

Both become the same `PacketReport` — when the packet left here, whether it arrived, when it arrived on the receiver's clock, its size, its ECN marking — which is everything a delay-based or loss-based estimator needs, and nothing about which RFC delivered it. `BandwidthEstimator` implementors never learn the difference.

Now the interesting part. The controller has just computed a new target **while processing an inbound RTCP feedback packet**, and the thing that must act on that number is the pacer, which works on the *outbound* path. It attaches the estimate to the inbound packet it is holding:

```rust
let target = self.estimator.target_bitrate();
if target != self.last_target {
    self.last_target = target;
    // Onto *this* packet: the pacer is application-ward of here, so it sees this
    // packet after this interceptor does and reads the attribute on its way past.
    msg.message.add(Attribute::TargetBitrateChanged {
        bits_per_second: target,
    });
}
```

and the pacer picks it up as that same feedback packet continues its walk toward the application:

```rust
// It arrives on the **read** leg because that is the only one it can cross on. The
// controller is wire-most, so on the write leg it is the last interceptor to see a packet
// and anything it attached would already have gone past here. On the read leg it is the
// first, and this is downstream of it — so the feedback packet that produced the estimate
// carries it here on its way to the application.
//
// Observed, not consumed: the packet carries on with the attribute attached, so anything
// further along reads the same number.
if let Some(Attribute::TargetBitrateChanged { bits_per_second }) = msg.message.get(/* … */) {
    self.pacer.set_target_bitrate(*bits_per_second);
}
```

Read that comment again, because it is the argument for both features compressed into one paragraph. **The estimate can only cross on the read walk.** The controller is wire-most, so on the write walk it is the *last* interceptor to see a departing packet — anything it attached there would already be past the pacer. On the read walk it is the *first*, and the pacer is downstream of it. So a fact learned from inbound traffic reaches an outbound-path interceptor by riding an inbound packet up the chain.

You cannot express that in two separate per-direction pipelines, because the two ends of the handoff are in different pipelines. You cannot express it with a tower, because neither interceptor contains the other. And you cannot express it without attributes, because there is nothing else travelling between them.

The other attributes carry the same kind of cross-interceptor knowledge in whichever direction suits them. The NACK responder attaches `Retransmission` to a packet it is resending, on the write walk, so the send history counts it as new bytes on the wire rather than as an original — an estimator that misses those believes the path is carrying less than it is, and raises the rate during loss. The FlexFEC decoder attaches `RecoveredByFec` on the read walk, so anything that must distinguish *arrived* from *present* does not tell the remote that a packet turned up when in fact it was lost and rebuilt locally.

`RecoveredByFec` also shows the design working by placement rather than by inspection: the NACK generator does **not** read it, despite the obvious guess. The FEC decoder sits wire-ward of the generator, so a rebuilt packet reaches it on the read walk like any other arrival and fills the gap in its receive log. There is nothing left to ask for.

### Attributes cross to the application too

The boundary is per-attribute and deliberate. A connection-level fact reaches the application: `TargetBitrateChanged` is folded into the statistics as `targetBitrate` on each outbound RTP stream, so an application can drive its encoder from it. A per-packet fact does not: `RecoveredByFec` is chain business, and the tests pin both halves —

> the estimate must reach the stats, or nothing outside the chain ever learns it

> A per-packet attribute is chain business. `RecoveredByFec` tells the NACK generator not to ask for a packet again; an application has nothing to do with it, so it stops here.

Going the other way, an application attaches attributes to packets it writes, and `Attribute::Custom(Arc<dyn Any + Send + Sync>)` is the escape hatch that keeps the enum from being a bottleneck on what anyone can express. It is an `Arc` rather than a `Box` so an attributed packet stays cheap to clone — the NACK responder clones into its send buffer and the FEC encoder into its protection block, and neither should deep-copy an application's payload.

The last one, `DeliverToApplication`, is how the chain decides what an application sees at all. `Registry::build` appends a terminus that ends the inbound RTCP path, so control traffic the interceptors act on does not arrive mixed in with your media. What gets past is a per-packet judgement by whichever interceptor is qualified to make it — an SFU relaying keyframe requests marks those and leaves its own chain's receiver reports alone. A chain-wide switch could only have offered all of it or none.

One convention to know before writing an interceptor: a connection-level fact sometimes has to travel when no media is going that way. The carrier for those is an RTCP packet with an empty payload, `Packet::Rtcp(vec![])`, which is inert to every interceptor that does not look for attributes and is dropped at the crate boundary once its attributes are read. That makes an empty compound RTCP packet **reserved** — emit one meaning anything else and it is discarded with no error and no trace. `tests/empty_rtcp_is_reserved.rs` holds the generators to it.

---

## Position is data, not call order

If nesting no longer encodes order, something must. Every interceptor is added at a `Slot`, and `Registry::build` sorts by it:

```rust
let chain = Registry::new()
    .with(Slot::TwccSender, TwccSenderBuilder::new().build())
    .with(Slot::NackResponder, NackResponderBuilder::new().build())
    .with(Slot::NackGenerator, NackGeneratorBuilder::new().build())
    .build();
```

Call order does not matter, which is what makes the configuration helpers composable: `configure_twcc` places interceptors at 2\_000 and 8\_000, `configure_nack` at 4\_000 and 7\_000, and they interleave correctly however a caller sequences them. Under the old design they did not, and it had actually gone wrong — the nested registry added the *innermost* interceptor first, so `register_default_interceptors` assembled TWCC receiver → RTCP reports → NACK generator, the reverse of what the chain contract documented. Nothing caught it, because there was no artifact to compare against the contract.

Three rules generate most of the table above, and they are what to apply when placing one of your own:

- **It produces packets** — retransmissions, repair, RTCP, anything the wire has not seen. Put it **above the pacer**, or its output leaves unpaced and the send history never counts the bytes. This is the constraint people miss, because an interceptor that only *reports* on what it received still produces packets to report with.
- **It reads inbound sequence numbers or arrival times.** Put it **below the jitter buffer**, so it sees the order and timing the path produced. A recorder placed after the buffer reports local *playout* instants to the remote as arrival times, and the remote's controller reads this endpoint's own buffering depth as network delay variation — a delay signal manufactured locally and, at the far end, indistinguishable from a congested path.
- **It repairs inbound packets.** Put it below anything that would otherwise ask for them again, as the FEC decoder is below the NACK generator.

### Why the numbers go up by a thousand

The named slots are 1\_000, 2\_000, 3\_000 … 13\_000, and the gaps are the point. They belong to you.

An application interceptor rarely wants to be first or last. It wants to be at a *particular* position relative to the built-ins, because the three rules above are correctness constraints and they apply to your interceptor exactly as they apply to ours. An interceptor that emits probe packets has to be above the pacer or its probes leave unmetered and unrecorded — which is worse than not probing, since it corrupts the estimate it was meant to inform. One that inspects arrivals has to be below the jitter buffer. One that marks inbound RTCP for the application has to be application-ward of the interceptors whose control traffic it is deciding about. "Somewhere in the chain" is not a position; "between the FEC decoder and the NACK generator" is.

A thousand-wide gap between every pair of named slots gives you room to say that without touching anything of ours:

```rust
// A prober generates packets, so it must sit above the pacer (3_000) to be metered and
// counted by the send history. 3_500 is free, and says exactly that.
let registry = register_default_interceptors(Registry::new(), &mut media_engine)?
    .with(Slot::from(3_500), BandwidthProbe::new());
```

There is nothing to coordinate and nothing to renumber: 3\_500 is a position, it is free, and `build` puts your interceptor there. `a_custom_interceptor_fits_between_named_slots` pins the mechanism in the test suite, and its doc comment is the one-line version: *a bare number puts an interceptor between two named slots — the reason they are spaced.*

The alternative designs are worse in ways that show up later. Consecutive integers (`0, 1, 2, …`) mean inserting between two of them renumbers everything after, so a library upgrade that adds an interceptor silently moves yours somewhere else. A `before(Slot::NackGenerator)` / `after(Slot::FecDecoder)` API expresses relative position, but two independent applications each asking to sit "before the NACK generator" have declared a constraint the registry cannot satisfy with one answer. A bare number is total, stable across upgrades within a gap, and needs no negotiation: `Slot::from(6_500)` is after the FEC decoder and before the NACK generator, today and after we add something at 6\_000 or 7\_000.

Two details follow from wanting that to work properly.

You reach a custom position through `Slot::from(n)` rather than by naming `Slot::Custom(n)`, so the spelling survives the variant gaining a richer representation later.

And `Slot`'s `PartialEq` and `Ord` are written by hand rather than derived, because deriving them would break precisely this use case in two different ways. A derived `PartialEq` compares *variants*, so `Slot::from(2_000)` would not equal `Slot::TwccSender` even though both name the same distance from the wire — and "a slot holds one interceptor" would stop being true for anyone who spelled a position numerically. A derived `Ord` compares *declaration order*, so `Slot::from(1_500)` would sort after `JitterBuffer` rather than between `CongestionControl` and `TwccSender`, which is the whole point of allowing a custom number. Both come from `Slot::slot()` instead, so a custom position sorts and compares exactly where its number says.

And because order is a value now, it is inspectable — `Registry::slots()` returns each interceptor's slot and type name in the order `build` will compose them, which exists for exactly the failure above: the composition is the thing no single helper can check. The tests read as the contract: `call_order_does_not_decide_chain_order`, `read_runs_in_the_order_stages_were_added`, `write_runs_in_reverse`, `a_slot_holds_one_interceptor`.

---

## What it cost, and what it bought

January's post listed zero-cost composition as benefit number one. That is what we gave up, and it is worth saying plainly rather than reframing: the chain holds `Box<dyn Interceptor>`, so it is **one virtual call per interceptor per packet**, where the monomorphized tower had none and could inline across the whole pipeline.

We have not benchmarked that in isolation, and I would rather say so than imply a number we do not have. The reasoning for accepting it: an indirect call is a few nanoseconds with a warm predictor, and it sits beside SRTP doing AES and HMAC over the whole packet — about 1.7 µs per 1200-byte RTP packet on the machine in [`rtc-srtp/benches/README.md`](https://github.com/webrtc-rs/rtc/blob/master/rtc-srtp/benches/README.md). The default chain is five interceptors plus the terminus, so six dispatches; they are not visible against that. If it ever stops being true, the fix is to specialize inside the chain, not to put the type parameter back.

In exchange, **`RTCPeerConnection` has no type parameter at all**:

```rust
pub struct RTCPeerConnection {
    // …
}
```

[July's post](/blog/2026/07/28/boxed-interceptor-type-erasure) was about reaching `RTCPeerConnection<BoxedInterceptor>` so that a struct holding a connection did not have to name its chain. That machinery is unnecessary now, because there is no chain type to name: `Registry::build` returns one concrete type whatever it was built from, so both arms of an `if` produce the same type, two connections with different chains go in the same `Vec`, and a chain assembled from a config file at startup is no harder to hold than a default one.

---

## Writing one

Implement `sansio::Protocol` and `Interceptor`, then add it where it belongs. The contract is uniform in both directions:

| To | Do |
|---|---|
| pass a packet through | queue it in `handle_*`, return it from `poll_*` |
| transform it | queue the modified packet |
| drop or delay it | queue nothing; a delayed one is queued later, from `handle_timeout` |
| generate one | queue it whenever you like; it joins the walk from `poll_*` |
| act on a timer | `handle_timeout`, and report the deadline from `poll_timeout` |
| tell another interceptor something | attach an `Attribute` to a packet going its way |

```rust
impl Protocol<TaggedPacket, TaggedPacket, ()> for Counter {
    fn handle_write(&mut self, msg: TaggedPacket) -> Result<(), Self::Error> {
        self.sent += 1;
        self.write_queue.push_back(msg); // queueing nothing would swallow it
        Ok(())
    }

    fn poll_write(&mut self) -> Option<Self::Wout> {
        self.write_queue.pop_front()
    }
    // …and the read pair, and the associated types
}

let chain = Registry::new().with(Slot::from(6_500), Counter::default()).build();
```

Even a pass-through needs a queue, because the queue is what the next interceptor is fed from. Note the comment: queueing nothing is how you drop a packet, so *forgetting* to queue is how you silently swallow one. That is the sharpest edge in the contract, and the direct consequence of making emission the only way forward.

---

## Try it

```toml
[dependencies]
rtc = "0.21.0-rc.1"
```

Coming from `0.20`: the chain type parameter is gone from `RTCPeerConnection`, `Registry::with` takes a `Slot` as its first argument, and an interceptor written against the nested design needs its `inner` field removed and its delegation replaced by a queue. If yours generates packets, check where it sits now — it very likely belongs above the pacer, and under the old design it was probably below.

## Related posts

- [Bring Your Own Crypto Provider](/blog/2026/09/01/pluggable-crypto-provider) — the other half of 0.21
- [Type-Erase the Interceptor Chain, Not Your Application](/blog/2026/07/28/boxed-interceptor-type-erasure) — the intermediate step this replaces
- [Interceptor Design Principle: Composable RTP/RTCP Processing](/blog/2026/01/09/interceptor-design-principle-sansio) — the generic design
- [Building WebRTC's Pipeline with sansio::Protocol](/blog/2026/01/04/building-webrtc-pipeline-with-sansio)
