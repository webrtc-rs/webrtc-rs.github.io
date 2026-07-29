# Type-Erase the Interceptor Chain, Not Your Application: `RTCPeerConnection<BoxedInterceptor>`

Composing behaviour by wrapping — each layer holding the next as a generic field — is one of Rust's best zero-cost patterns. It is also one of the easiest ways to accidentally push a type into your users' data structures that they never asked for and cannot name.

`rtc`'s interceptor chain is a textbook case of both halves, and on [master](https://github.com/webrtc-rs/rtc) you can now opt out of the second half:

```rust
struct Session {
    peer_connection: RTCPeerConnection<BoxedInterceptor>,   // no type parameter
}
```

Whatever chain that connection was built with — defaults, a custom layer, something chosen from a config file at startup — the struct holding it looks like that. This post is about why it took a change to a trait bound to make it possible, and about the general shape of the problem, which is not specific to WebRTC: what a generic composition parameter costs the code that has to *hold* the composed value, why `impl Trait` doesn't rescue it, the two places you can erase the type and when each is the right one, and what a trait needs in order to make the second possible at all.

---

## A pattern and its receipt

The wrapper chain is simple. Each layer is generic over the next one, holds it by value, and delegates:

```rust
pub struct SomeLayer<P> {
    next: P,
    // …layer state
}

impl<P: Interceptor> Protocol<…> for SomeLayer<P> {
    fn handle_read(&mut self, msg: TaggedPacket) -> Result<(), Self::Error> {
        // …do the layer's work
        self.next.handle_read(msg)
    }
}
```

Stack a few of those and the compiler monomorphizes the whole thing into straight-line code. No vtables, no indirection, and the optimizer can see through every layer at once. For a packet pipeline that runs per RTP packet, that is exactly the right trade — it's why we [built the interceptor framework this way](/blog/2026/01/09/interceptor-design-principle-sansio) in the first place.

The receipt arrives when you ask what the composed value's *type* is. Composition by generics means the type is **structural**: it spells out the entire recipe, in order.

```rust
let registry = register_default_interceptors(Registry::new(), &mut media_engine)?;
let registry = registry.with(RtcpForwarderBuilder::new().build());
```

```text
RtcpForwarderInterceptor<TwccReceiverInterceptor<SenderReportInterceptor<
    ReceiverReportInterceptor<NackResponderInterceptor<NackGeneratorInterceptor<
        NoopInterceptor>>>>>>
```

And because the pipeline that runs the chain is itself generic over it —

```rust
pub struct RTCPeerConnection<I = NoopInterceptor> where I: Interceptor { … }
```

— that recipe is now part of the type of everything that owns a peer connection.

---

## Composition is not a capability

It helps to separate two kinds of type parameter.

Some parameters are part of the caller's **interface**: `Vec<T>`, `HashMap<K, V>`, `impl Read`. The caller picked `T` on purpose, knows what it is, and wants it in the signature. Making the holder generic over it is correct — it *is* the caller's choice.

Others are the residue of a **configuration decision**. The user of an interceptor chain chose a policy — "the defaults, plus my RTCP forwarder" — and a type fell out of it as an implementation detail of how composition happens to be encoded. Nobody wants that type. Nobody can reasonably write it down. It exists only because wrapping-with-generics is how the layers were spelled.

The trouble is that Rust treats both identically, so the second kind propagates exactly like the first. And it propagates *upward*, along ownership:

```rust
struct Session<I: Interceptor> { pc: RTCPeerConnection<I>, … }
struct Room<I: Interceptor>    { members: HashMap<Id, Session<I>>, … }
struct Server<I: Interceptor>  { rooms: HashMap<RoomId, Room<I>>, … }
//                              …and now it is in your public API
```

Three structs deep, a parameter representing "which RTCP layers did we enable" is a required part of constructing a server. Worse than the noise: it also *unifies* the choice. Every room in that `HashMap` must have the identical chain, because it is one type parameter — even though per-session policy is the obvious thing to want.

---

## Why `impl Trait` doesn't rescue it

The reflex is to hide the type behind `impl Trait`:

```rust
fn build_chain() -> Registry<impl Interceptor> { … }
fn create_peer()  -> RTCPeerConnection<impl Interceptor> { … }
```

That compiles, and it genuinely helps at the boundary where the chain is built. But `impl Trait` is not erasure — it is an **opaque alias for one concrete type, fixed per return site**. The type still exists, is still unique, and is still chosen by the callee; you just can't spell it. So it solves the "I don't want to write that type" problem while leaving the "that type is in my struct" problem completely intact:

| You want to…                                                    | With `impl Trait` |
|-----------------------------------------------------------------|-------------------|
| put the value in a struct field                                   | you must name a type; you can't, so the struct takes a parameter — infection continues |
| hold values from two builders in one `Vec`/`HashMap`              | impossible: different sites, different opaque types |
| choose the chain in an `if`/`match` at runtime                    | impossible: arms must agree on one type |
| write a non-generic function that takes one                       | impossible |
| return one from two different functions and treat them uniformly  | impossible |

Every row is a thing ordinary programs do. `impl Trait` postpones the naming problem by exactly one layer.

---

## Two places to erase

Sooner or later, someone reaches for `dyn`. The interesting question is *what* to make dynamic — and there are two candidates. Which one is right depends less on the code than on who is writing it and why.

### Erasing the holder (the facade)

Leave the generic type alone and hide the *whole composed object* behind a trait you write yourself.

```rust
trait PeerConnection {                    // your own mirror of the library's API
    fn create_offer(&mut self, o: Option<RTCOfferOptions>) -> Result<RTCSessionDescription>;
    fn set_local_description(&mut self, sdp: RTCSessionDescription) -> Result<()>;
    // …one line per method you need, forever
}

impl<I: Interceptor> PeerConnection for RTCPeerConnection<I> {
    fn create_offer(&mut self, o: Option<RTCOfferOptions>) -> Result<RTCSessionDescription> {
        RTCPeerConnection::create_offer(self, o)          // …and one forward for each
    }
}

struct Session { pc: Box<dyn PeerConnection> }            // the parameter is gone
```

#### When this is exactly right

Sometimes the facade **is the product**. The `webrtc` crate is built this way on top of `rtc`, deliberately:

```rust
/// Object-safe trait exposing all public PeerConnection operations.
///
/// Object-safe trait exposing all public PeerConnection operations, hiding the
/// generic interceptor type so callers can store and share connections easily.
#[async_trait::async_trait]
pub trait PeerConnection: Send + Sync + 'static {
    async fn create_offer(&self, options: Option<RTCOfferOptions>) -> Result<RTCSessionDescription>;
    async fn set_local_description(&self, desc: RTCSessionDescription) -> Result<()>;
    …
}
```

Look at what that trait is doing. It is not a mirror — it is a **translation**. `rtc`'s core is Sans-I/O: synchronous, `&mut self`, driven by a poll loop the caller owns. `webrtc`'s trait is `async`, takes `&self`, is `Send + Sync + 'static`, and is object safe, so a connection can be wrapped in an `Arc<dyn PeerConnection>` and shared across tasks; a driver task and an event-handler trait sit behind it. The trait object is the seam between two programming models, and that is precisely the job the `webrtc` crate exists to do.

Three things follow from being a real API boundary rather than a workaround:

- **The facade earns its keep on its own merits.** If the interceptor parameter vanished tomorrow, `webrtc` would still want this trait, because the async translation is the point.
- **It is written once, upstream, by people who own both sides.** Every `webrtc` user gets a parameter-free connection handle — `impl PeerConnection`, wrappable as `Arc<dyn PeerConnection>` — without writing a line of it.
- **The dispatch cost is noise in context.** One vtable hop next to a mutex acquisition, a channel send, and an `.await` is not measurable.

That the generic interceptor type disappears along the way is a welcome side effect — not the motive.

#### When it is a workaround

The trouble starts when a facade has no reason to exist except to launder a type parameter — when its methods are one-line forwards with the same names, same arguments and same semantics as the thing they wrap. Then you are not presenting an API; you are paying for a type parameter in boilerplate. None of the three benefits above apply, and three taxes do instead:

**The mirror is only as complete as you remembered to make it.** Every API you later want is another signature plus another forward, and every upstream release can add methods your facade doesn't have. The abstraction is not "a peer connection" — it is "the subset of a peer connection someone needed on a Tuesday."

**Dyn-safety flattens the signatures.** A trait object cannot carry generic methods, and it cannot hand back a type that mentions the erased parameter. So borrowing views and lazy iterators have to be lowered into owned snapshots:

```rust
// library:  fn get_receivers(&self) -> impl Iterator<Item = ReceiverId> + use<'_, I>
fn get_receivers(&self) -> Vec<ReceiverId>;              // → allocate

// library:  fn rtp_receiver(&mut self, id) -> Option<RTCRtpReceiver<'_, I>>
fn receiver_track(&mut self, id) -> Option<MediaStreamTrack>;         // → clone
fn receiver_parameters(&mut self, id) -> Option<RTCRtpReceiveParameters>;  // → clone
```

A borrowing handle like `RTCRtpReceiver<'_, I>` mentions `I` in its type, so it can *never* cross the facade. The only way through is to copy data out — which turns a free read into an allocation, and puts it wherever the caller happens to be, including hot loops.

**And you pay it in every application.** The facade lives downstream, so each project that hits the problem writes its own, with its own subset and its own set of forced clones — and unlike the `webrtc` case, nobody else ever benefits from the work.

The test is a single question: **would this trait still exist if the type parameter went away?** For `webrtc`, obviously yes. For a mirror, obviously no — and that is the signal that you are erasing the wrong thing.

### Erasing the parameter

The other option is to make the *parameter itself* erasable, and substitute the erased type back into the same generic slot:

```rust
type BoxedInterceptor = Box<dyn Interceptor>;

struct Session { pc: RTCPeerConnection<BoxedInterceptor> }   // still the real type
```

Everything downstream of the parameter is untouched. `RTCPeerConnection<I>`'s entire API is defined for all `I: Interceptor`, so if `BoxedInterceptor: Interceptor`, the erased peer connection keeps **the whole API**, including the generic methods and the borrowing views the facade could never express. Nothing needs re-declaring, so nothing can drift.

This is the general rule the two approaches illustrate:

> **Erase a holder when the erased API is one you actually want to present. Otherwise erase the smallest type that removes the parameter, in the crate that owns the trait.**

A mirror facade erases something enormous — a peer connection, with dozens of methods and several borrowing view types — in order to get rid of something small: a chain. Erasing the chain directly removes the same parameter at a fraction of the surface area, and, being upstream, does it once for everybody. `webrtc` sits on the other side of that line: it erases a holder, but the thing it presents is a different API, not a copy of the one underneath.

---

## What it takes to erase a parameter

For `Box<dyn Trait>` to be a drop-in for `T: Trait`, the trait needs two properties. Only the first is famous.

### 1. The trait must be dyn-compatible

The rules for object safety (recently renamed *dyn compatibility*) come down to: everything in the vtable must be callable through a pointer without knowing `Self`. In practice you check three things.

**No `Sized` supertrait.** `trait Foo: Sized` is an outright veto — `dyn Foo` cannot exist, because a trait object is unsized by construction. This is the one that bit us, and it is worth dwelling on *why* someone writes it. Usually it's to support a by-value combinator:

```rust
pub trait Interceptor: Protocol<…> + Sized + Send + Sync + 'static {
    fn with<O, F>(self, f: F) -> O where F: FnOnce(Self) -> O, O: Interceptor { f(self) }
    //           ^^^^ takes self by value, so it needs a Sized bound somewhere
}
```

The bound is real — `with` genuinely requires `Self: Sized` — but it was put in the wrong place. Hoisting one method's requirement onto the entire trait vetoes trait objects for every user of the trait, forever. The fix is to state it where it applies:

```rust
fn with<O, F>(self, f: F) -> O
where
    Self: Sized,          // ← the requirement, scoped to the method that has it
    F: FnOnce(Self) -> O,
    O: Interceptor,
{ f(self) }
```

A method with `where Self: Sized` is simply **excluded from the vtable**. It stays fully available on every concrete type — the ergonomic chaining API is unchanged — and it stops blocking `dyn`. The same trick covers generic methods and by-value receivers.

**All associated types must be nameable.** `dyn Iterator` is illegal; `dyn Iterator<Item = u32>` is fine. If a trait's associated types are already pinned by its own supertrait bound, this is free:

```rust
pub trait Interceptor:
    sansio::Protocol<TaggedPacket, TaggedPacket, (),
        Rout = TaggedPacket, Wout = TaggedPacket, Eout = (),
        Time = Instant, Error = shared::error::Error> + …
```

Every associated type is already fixed, so `dyn Interceptor` is complete with no extra annotations.

**Auto traits come along for free if they're supertraits.** `trait Foo: Send + Sync + 'static` means `dyn Foo` is `Send + Sync + 'static` — no `Box<dyn Foo + Send + Sync>` noise at the use site.

### 2. The erased type must implement the trait

This is the step that's easy to miss, and it's the one that makes erasure a *drop-in* rather than a parallel universe. `Box<dyn Interceptor>` being constructible is useless if it can't go back into `RTCPeerConnection<I>`'s `I` slot. It needs to satisfy the bound itself:

```rust
impl<P: Interceptor + ?Sized> Interceptor for Box<P> {
    fn bind_local_stream(&mut self, info: &StreamInfo)   { (**self).bind_local_stream(info) }
    fn unbind_local_stream(&mut self, info: &StreamInfo) { (**self).unbind_local_stream(info) }
    fn bind_remote_stream(&mut self, info: &StreamInfo)  { (**self).bind_remote_stream(info) }
    fn unbind_remote_stream(&mut self, info: &StreamInfo){ (**self).unbind_remote_stream(info) }
}
```

The `+ ?Sized` matters: without it the impl doesn't cover `P = dyn Interceptor`, which is the only case anyone cares about. And note how little there is to write — `sansio` already blanket-implements `Protocol` for `Box<P>`, so the supertrait half was done. **Libraries that ship a `Box<T>` forwarding impl for their traits make themselves erasable by anyone downstream**; libraries that don't force every consumer to write a newtype.

### 3. Make it opt-in, and give it a name

Erasure should be a choice at the point of construction, not a change to the default:

```rust
pub type BoxedInterceptor = Box<dyn Interceptor>;

impl<P: Interceptor> Registry<P> {
    pub fn boxed(self) -> Registry<BoxedInterceptor> {
        Registry { inner: Box::new(self.inner) }
    }
}
```

The alias is not cosmetic — it's the thing users put in struct fields, so it should read like a type, not like machinery. And `boxed()` on the builder is where the two worlds meet: everything before it is statically composed, everything after it is one type.

That is the whole change: move one bound, add one blanket impl, add one alias, add one method. Nothing else in `rtc` moved — the peer connection, the transceivers, the derive macros and the built-in interceptors all compiled untouched, and it is not a breaking change, because `T: Interceptor` in generic position still implies `Sized` by default.

---

## What it costs

Dynamic dispatch is not free, so it matters *where* the indirection lands.

Erasing the chain's outer handle puts one virtual call at each **entry point of the chain** — `handle_read`, `poll_write`, `handle_timeout` and friends — once per pipeline drive. The chain's **interior is unchanged**: layer *n* still calls layer *n+1* through a statically known type and still inlines exactly as before. You are not paying a vtable hop per layer per packet; you are paying one hop to get into the stack.

That asymmetry is the reason erasing at the boundary is cheap and erasing per-layer would not be. It's also why a mirror facade can end up *slower* than erasing the parameter: its forced clones are real allocations on real hot paths, while the virtual call it avoided was one indirection amortized over an entire drive.

There is a second, quieter benefit. Every distinct chain type instantiates its own copy of the generic pipeline. Collapsing *N* configurations to one erased type collapses *N* monomorphizations to one — less code to compile and less binary to ship.

The rule of thumb:

> **Keep the generic when the chain is fixed at compile time. Erase when the chain is a runtime decision, or when the composed value has to live inside someone else's data structures.**

---

## The worked example

In `rtc`, that's `BoxedInterceptor`. Master also carries a runnable demonstration of all three things the generic form couldn't do.

**A chain chosen at runtime.** The two branches build different types; `.boxed()` is where they unify:

```rust
fn build_peer_connection(
    forward_rtcp: bool,
    mut media_engine: MediaEngine,
) -> Result<RTCPeerConnection<BoxedInterceptor>> {
    let registry = register_default_interceptors(Registry::new(), &mut media_engine)?;
    let registry = if forward_rtcp {
        registry.with(RtcpForwarderBuilder::new().build()).boxed()
    } else {
        registry.boxed()
    };
    …
}
```

**A struct with no type parameter**, and therefore an ordinary `impl` block:

```rust
struct RtcpSession {
    peer_connection: RTCPeerConnection<BoxedInterceptor>,
    socket: Arc<UdpSocket>,
    …
}
```

**A collection of peers whose chains differ.** `tests/rtcp_processing_boxed_interop.rs` puts two peers built with *different* chains in one `Vec` and drives them from one non-generic loop: the offerer streams RTP without the RTCP forwarder installed, the answerer has it. The assertion is that the behaviour differs accordingly — the answerer surfaces the offerer's Sender Reports to the application, and the offerer never surfaces a single RTCP packet.

> **Try it:** `cargo run --example rtcp-processing-boxed`, then the same command with
> `-- --no-rtcp-forwarding`. Same peer connection type, same code path, different chain —
> and the RTCP output disappears.

---

## In practice: the `sfu` crate

`sfu` is the reason this got fixed, and it had gone down the mirror-facade road first — the wrong side of the line drawn above.

An SFU owns rooms, each room owns a `HashMap<ClientId, Client>`, and every client's chain is assembled at join time from runtime configuration — so `I` had nowhere to stop propagating, and would have ended up in the crate's public API. The workaround was a hand-written `trait PeerConnection` with thirty methods and a thirty-method forwarding impl: 250 lines that did nothing but restate `rtc`'s API, complete with a commented-out `create_data_channel` stub for the parts nobody had gotten around to mirroring.

Switching the field to `RTCPeerConnection<BoxedInterceptor>` deleted all of it. `client.rs` went from 1,209 lines to 965 (+87 / −320 including `room.rs`), and **`room.rs` needed no edits at all** — it reached the peer connection through `Deref`, and the `Deref` target simply changed from `Box<dyn PeerConnection>` to the real type underneath.

The tax that actually mattered was the third one. `Room::forward_rtp` runs per RTP packet, and to find which receiver owns a packet's SSRC it had to go through the facade — which meant cloning a `MediaStreamTrack` and an `RTCRtpReceiveParameters` per candidate receiver, then throwing most of them away, because the only thing it wanted was `ssrcs()`. With the real type in hand it holds the borrowing view instead:

```rust
let Some(mut receiver) = self.peer_connection.rtp_receiver(receiver_id) else {
    continue;
};
if !receiver.track().ssrcs().any(|track_ssrc| track_ssrc == ssrc) {
    continue;                                     // nothing was cloned to learn this
}
if let Some(codec) = Client::codec_for_payload_type(receiver.get_parameters(), payload_type) {
    return Some(codec);
}
```

Same logic, no clones, on the per-packet path. That performance was never lost to `dyn` — it was lost to erasing the wrong layer.

---

## Takeaways

If you are writing a library whose users compose behaviour through a generic chain:

- **Assume the composed type will end up in someone's struct.** Design for it.
- **Never put `Sized` on the trait** to satisfy one by-value combinator. Put `where Self: Sized` on the combinator.
- **Ship a `impl<P: Trait + ?Sized> Trait for Box<P>`.** It's a few lines, and it's what makes your trait erasable by anyone else.
- **Ship the alias and the `boxed()` constructor.** Users should not have to discover that erasure is possible; they should find a type they can put in a field.
- **Keep static dispatch the default.** Erasure is a choice for the cases that need it, not a tax on the cases that don't.

And if you are on the consuming side: before writing a facade trait over somebody else's API, ask whether it would still be worth having if the type parameter disappeared. If yes — you are presenting a genuinely different API, the way `webrtc` presents an async one over `rtc`'s Sans-I/O core — then write it, and the erased parameter is a bonus. If no, you are about to wrap something enormous to get rid of something small. Check the size of the type you actually want gone; the fix usually belongs upstream, and it is usually four lines long.

🦀

---

## Links

- **rtc (Sans-I/O core)**: https://github.com/webrtc-rs/rtc
- **`rtc-interceptor`**: https://github.com/webrtc-rs/rtc/tree/master/rtc-interceptor
- **sfu**: https://github.com/webrtc-rs/sfu — live demo at https://sfu.rs
- **webrtc (async)**: https://github.com/webrtc-rs/webrtc
- **apprtc (signaling)**: https://github.com/webrtc-rs/apprtc — live demo at https://appr.tc
- **Discord**: https://discord.gg/4Ju8UHdXMs

---

## Further Reading

- [Interceptor Design Principle: Composable RTP/RTCP Processing with `sansio::Protocol`](/blog/2026/01/09/interceptor-design-principle-sansio)
- [Announcing `sfu` v0.20.0-rc.3: A Sans-I/O Selective Forwarding Unit in Rust](/blog/2026/07/13/announcing-sfu-v0.20.0-rc.3)
- [AppRTC Goes Group: Automatic P2P → SFU Upgrade at appr.tc](/blog/2026/07/22/apprtc-p2p-to-sfu-upgrade)
