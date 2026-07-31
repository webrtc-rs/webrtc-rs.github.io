# Bring Your Own Async Runtime: Closing the Seam in `webrtc`'s Runtime Abstraction

The `webrtc` crate has advertised runtime independence since the `v0.20` rewrite began. It shipped a
`Runtime` trait, a `TokioRuntime`, a `SmolRuntime`, and a feature flag to pick between them. What it
did *not* have was a way for you to supply your own.

That sounds like a missing feature. It was worse than that: the abstraction had a **seam running
through the middle of it**, and the seam had a panic on the other side.

This post is about finding that seam, the single question that turned out to resolve the whole
design, and what it took to make "bring your own runtime" true rather than aspirational.

---

## The abstraction was two abstractions

Everything the stack needed from a runtime went through one of two mechanisms — and only one of them
was replaceable.

| | Mechanism | Members | Pluggable? |
|---|---|---|---|
| **Layer 1** | `dyn Runtime`, injected as a value | `spawn`, `spawn_reactor`, `wrap_udp_socket`, `wrap_tcp_listener`, `connect_tcp` | ✅ |
| **Layer 2** | `#[cfg(feature = …)]` + type aliases | `sleep`, `interval`, `timeout`, `yield_now`, `channel`, `Sender`, `Receiver`, `broadcast_channel`, `Mutex`, `Notify`, `block_on`, `resolve_host` | ❌ |

Layer 2 was resolved at compile time by **fourteen `#[cfg]`-gated type aliases** naming `tokio::*` or
`smol::*` concretely. There is no `#[cfg]` arm for "a third party". So a custom runtime — supposing
you could write one — would have had its `spawn` and its sockets used, while its **timers, channels,
mutexes and notifications were silently still tokio's or smol's**.

The traits that looked like they existed for extensibility (`AsyncMutex`, `AsyncNotify`,
`AsyncSender`, `AsyncReceiver`) were not reachable through `Runtime` at all. They described a shape
without providing a way in. `AsyncMutex` could not have been reached that way even in principle: its
generic associated type `Guard<'a>` made it non-object-safe.

### The seam had a panic behind it

`default = ["runtime-tokio"]`, and the documented rule when both features were on was "tokio takes
precedence". But `smol_runtime()` returned a `SmolRuntime`, whose `spawn` runs tasks on **smol's**
executor — while `sleep` and `interval` resolved to `tokio::time::*`, which **panic when polled
outside an ambient tokio reactor**.

Cargo features are additive. Any crate anywhere in your dependency graph depending on `webrtc` with
default features silently re-enables `runtime-tokio` for a smol user. You did not have to do anything
wrong to reach this.

The tell was in the call graph: `smol_runtime()` had **zero callers and zero tests** in the entire
crate. Its only reachable purpose was this failure mode.

---

## One question resolves the design

The temptation with an abstraction like this is to make *everything* pluggable — put channels,
mutexes and notifications on the trait alongside timers and sockets. That instinct is wrong, and
seeing why is what unlocked the rest.

Ask a single question of each primitive: **does it touch the reactor?**

| Category | Members | Treatment |
|---|---|---|
| **Reactor-bound** | spawn, timers, UDP, TCP, DNS, `block_on` | **Inject** through `Runtime` |
| **Executor-agnostic** | channels, broadcast, mutex, notify | **One implementation**; delete the abstraction |
| **Derivable** | `timeout`, `yield_now` | Build generically on injected `sleep` |

Two facts make that partition hold.

**Channels, mutexes and notify never appear in the public API.** Searching every `pub` item outside
`src/runtime/` turns up nothing — they are pure implementation details. Users cannot observe which
implementation is used, so they do not need to be pluggable.

**They are just data structures driven by wakers.** `async-channel`, `futures::lock::Mutex`,
`event-listener` and `async-broadcast` work on any executor. Only things that must register with a
*reactor* — timers and I/O — are genuinely runtime-specific.

And the payoff is what keeps the whole design viable: `fn channel<T>(&self, …)` is a **generic
method**, so putting it on the trait would make `Runtime` non-object-safe, which would force a
`<R: Runtime>` parameter viral through `PeerConnection`, the driver, transports, data channels and
transceivers — on top of the `<I: Interceptor>` parameter already there. Keeping executor-agnostic
primitives *off* the trait is what lets injection stay `Arc<dyn Runtime>`.

The result: `tokio.rs` and `smol.rs` are now about **400 lines each**, containing spawn, timers and
socket readiness — and nothing else.

---

## The trait

**Eight required methods, three defaulted.**

```rust
pub trait Runtime: Send + Sync + Debug + 'static {
    // Task execution
    fn spawn(&self, future: Pin<Box<dyn Future<Output = ()> + Send>>) -> Box<dyn JoinHandle>;

    // Timers — reactor-bound, which is why they cannot be free functions
    fn sleep(&self, duration: Duration) -> Pin<Box<dyn Future<Output = ()> + Send + 'static>>;
    fn interval(&self, period: Duration) -> Box<dyn AsyncInterval>;

    // Networking — reactor-bound
    fn wrap_udp_socket(&self, socket: std::net::UdpSocket) -> io::Result<Arc<dyn AsyncUdpSocket>>;
    fn wrap_tcp_listener(&self, l: std::net::TcpListener) -> io::Result<Arc<dyn AsyncTcpListener>>;
    fn connect_tcp<'a>(&'a self, remote: SocketAddr) -> /* boxed future */;
    fn resolve_host<'a>(&'a self, host: &'a str) -> /* boxed future */;

    // Synchronous entry point for `main` and test harnesses
    fn block_on(&self, future: Pin<Box<dyn Future<Output = ()> + '_>>);

    // Defaulted: spawn_reactor (falls back to spawn), yield_now, name
}
```

Object-safe by construction: `&self` receivers, no generic method parameters, no `where Self: Sized`.

Three details worth calling out.

**`spawn_reactor` defaults to `spawn`.** The built-in runtimes pin each connection's driver to a
thread from a bounded pool — the fix for the RSS blowup of one-OS-thread-per-connection. A custom
runtime that does not want to implement thread confinement simply inherits the default and forgoes
the benefit, rather than being unable to compile.

**`block_on` takes a non-`Send` future.** It runs on the calling thread and never moves the future
across threads, matching what tokio, smol and `futures::executor` all do. Over-constraining this to
`Send` was a real bug caught during implementation — it broke `examples/save-to-disk-av1`.

**`JoinHandle` is a trait**, so a runtime returns its own handle type boxed. This one is
retrospectively embarrassing: in the old code `JoinHandle` was a *struct* with a private field, a
private inner trait and no constructor. **No out-of-tree runtime could return one from `spawn`.**
Before any of the timer work, before the type aliases, the abstraction was already unpluggable for
that reason alone — and nobody had noticed, because nobody had tried.

---

## Features became additive, and the panic became unrepresentable

```toml
[features]
default       = ["runtime-tokio"]
runtime-tokio = ["dep:tokio"]   # provides TokioRuntime
runtime-smol  = ["dep:smol"]    # provides SmolRuntime
runtime-mock  = []              # provides MockRuntime
```

Features now gate exactly one thing: **whether a built-in runtime type exists**. They never select
which primitives the library uses. Consequences:

- The "tokio takes precedence" rule is **deleted** — nothing is cfg-selected any more.
- Enabling several at once is legitimate. One process can drive tokio connections and smol
  connections simultaneously.
- The panic is **impossible by construction**, not by documentation: timers come from the runtime
  *value*, so a `SmolRuntime` can never hand you a tokio timer.
- `default-features = false` with no built-in runtime at all is a supported configuration.

This is the provider pattern `rustls` uses for crypto and `sqlx` for database drivers.

---

## Injected per connection — and deliberately no global

```rust
let runtime: Arc<dyn Runtime> = Arc::new(MyRuntime::new());

let pc = PeerConnectionBuilder::new()
    .with_runtime(runtime.clone())
    .with_udp_addrs(vec!["0.0.0.0:0"])
    .build()
    .await?;
```

An earlier draft of the design had a settable process-global `set_default_runtime()`, so
applications and tests would not have to thread a handle around. It was cut, for five reasons:

1. **Per-connection is strictly more expressive.** A global is emulated by holding one `Arc` and
   cloning it per builder. The converse — "this connection on the mock runtime, that one on tokio" —
   is impossible with a global.
2. **It reintroduces exactly the bug being fixed:** a second source of truth for "which runtime",
   with precedence rules and an initialisation-order hazard.
3. **It is hostile to libraries.** A mid-stack crate calling it silently steals the choice from the
   application, and the "already set" error surfaces as a startup failure somewhere unrelated.
4. **It breaks parallel testing.** A process-global forces mock-runtime tests to serialise; with
   per-connection injection each `#[test]` owns an independent virtual clock.
5. **Nothing needs it.** Every timer call site in library code — four hits across three files —
   already had a runtime handle in scope.

`default_runtime()` remains as a **non-settable factory** returning the compiled-in built-in for
callers with no preference. There is no way to overwrite what it returns.

The rule this preserves: **the library has no ambient runtime.** Every internal call site takes an
explicit handle. The convenience helpers live in the test and example trees only, where they cannot
reintroduce a mismatch into library code.

---

## Derived rather than injected

`timeout` is not a trait method. It is a free function built generically on `sleep`, so every
runtime gets it — including yours — with no per-runtime implementation:

```rust
pub async fn timeout<T>(
    runtime: &dyn Runtime,
    duration: Duration,
    future: impl Future<Output = T>,
) -> Result<T, Elapsed> {
    match select(Box::pin(future), runtime.sleep(duration)).await {
        Either::Left((value, _)) => Ok(value),
        Either::Right(_) => Err(Elapsed),
    }
}
```

The evidence that this was the right call was already in the tree: the old smol implementation
built `timeout` from `sleep` + `or`, using no smol-specific timer API at all.

It also fixed an error type. `timeout` used to return `Result<T, ()>` — an error carrying no
information and not implementing `std::error::Error`. It now returns `Elapsed`.

---

## The proof: a runtime we do not ship

A trait is only pluggable if someone outside the crate can implement it. So the acceptance test is
an example that does exactly that — `examples/custom-runtime/`, ~350 lines implementing `Runtime`
over [`async-executor`](https://crates.io/crates/async-executor) and
[`async-io`](https://crates.io/crates/async-io), neither of which is tokio or smol:

```shell
cargo run --no-default-features --example custom-runtime
```

```text
running on a custom runtime: my-runtime
sleep(50ms) took 51.041458ms
tick 0 / tick 1 / tick 2
timeout on a pending future => Err(Elapsed)
hello from a spawned task
udp: received "ping" from 127.0.0.1:56527
resolved localhost:3478 -> [[::1]:3478, 127.0.0.1:3478]

All runtime capabilities exercised without tokio or smol.
```

`--no-default-features` is the load-bearing part: **neither built-in runtime is compiled in**. If
anything in the library were still reaching for tokio's timers, this would not link.

An example can only show one runtime, though. `tests/custom_runtime_interop.rs` closes the loop with
the thing that actually proves per-connection injection: **two peer connections on two different
runtimes, interoperating in one process** — the answerer on the custom `async-executor` runtime, the
offerer on a built-in. Every packet between them crosses that boundary, so a completed connection
proves both runtimes are independently live. A design with a process-global registry could not
express this test.

---

## The other prize: deterministic time

Once timers are injected, a runtime can lie about time. `MockRuntime` (behind `runtime-mock`)
implements the same trait but drives timers from a manually advanced virtual clock and performs no
I/O:

```rust
let rt = Arc::new(MockRuntime::new());
let clock = rt.clock();

let sleep = rt.sleep(Duration::from_secs(30));
// ...nothing fires until time is advanced...
clock.advance(Duration::from_secs(30));   // instant, deterministic
```

This finally extends up into the async layer the property the Sans-I/O `rtc` core has always had:
timing-dependent behaviour — ICE timeouts, DTLS retransmits, SCTP RTO, back-pressure — becomes
testable instantly and without flakiness. Each `MockRuntime` owns an independent clock, so tests
using one run in parallel; that is the concrete payoff of having refused the global registry.

---

## The hot path, and what it cost

One claim in the old module docs was simply false: "zero-cost abstraction without dynamic dispatch in
the hot path". The Layer-2 type aliases were indeed zero-cost, but `AsyncUdpSocket::send_to` and
`recv_from` were `dyn` calls returning `Pin<Box<dyn Future>>` — **one heap allocation per datagram**.

`AsyncUdpSocket` is now poll-based, mirroring `quinn`'s socket trait. Two primitives:

```rust
fn poll_send(&self, cx: &mut Context<'_>, transmit: &Transmit<'_>) -> Poll<io::Result<usize>>;

fn poll_recv(
    &self,
    cx: &mut Context<'_>,
    bufs: &mut [IoSliceMut<'_>],
    meta: &mut [RecvMeta],
) -> Poll<io::Result<usize>>;
```

The driver polls these directly and allocates nothing per datagram; `send_to` and `recv_from` remain
as single-datagram conveniences for the control plane.

`poll_recv` takes slices because that is the shape the kernel offers: on Linux one `recvmmsg` returns
up to 32 datagrams **from different peers**, and each of those may itself carry several GRO-coalesced
datagrams from one flow. A minimal implementation fills `bufs[0]`, returns `Ok(1)`, and is done —
which is what the custom-runtime example does, and what non-Linux platforms do regardless.

There is no wrapper layer around any of this. `Transmit`, `RecvMeta`, `EcnCodepoint`, `UdpSockRef`,
`UdpSocketState` and `BATCH_SIZE` are re-exported from `quinn-udp` directly, so a runtime that
already knows quinn's socket API has nothing new to learn and there is nothing for us to keep in
sync. The deliberate cost: **part of the public API is now tied to `quinn-udp`'s major version**, and
its syscall bindings compile even for a build that supplies its own socket type.

---

## What the plan got wrong

The implementation plan deliberately ordered "write a third-party runtime" as a phase that *gates*
acceptance of the trait design. That ordering earned its keep — writing the example surfaced three
defects that review had not:

1. **`JoinHandle` was unconstructible from outside the crate.** As above: the abstraction was not
   pluggable at all, independent of every other problem on the list.
2. **`block_on` was over-constrained to `Send`**, which broke a working example.
3. **The broadcast channel had two bugs**, one of them pre-existing: dropping the initial receiver
   permanently closes an `async-broadcast` channel and `subscribe` cannot reopen it — which means the
   old smol path's `send` would *always* have returned `Err(Closed)`. And `send` must stay
   synchronous, because an `async` version blocks forever with zero subscribers, wedging a publisher
   that has no viewers.

Each of these would have shipped as a "supported" extension point that nobody could actually use.

---

## Honest gaps

- **`MockRuntime` has no in-memory sockets yet.** It covers timers and task execution; socket methods
  return `io::ErrorKind::Unsupported`. End-to-end connection tests on virtual time need transports.
- **The driver does not yet exploit batched receive.** `poll_recv` accepts up to `BATCH_SIZE`
  messages, but the driver currently asks for one per call, and the burst-drain loop still makes N
  syscalls where one `recvmmsg` would do. The API is in place; the buffer rework is a follow-up.
- **The hot-path work is unmeasured.** Removing per-datagram allocations is correct by construction,
  but the magnitude has not been benchmarked.
- **`quinn-udp` in the public API** is a semver coupling taken deliberately, in exchange for not
  maintaining a parallel set of wrapper types.

---

## Try it

```toml
# Tokio, the default
webrtc = "0.20"

# smol
webrtc = { version = "0.20", default-features = false, features = ["runtime-smol"] }

# Neither — bring your own
webrtc = { version = "0.20", default-features = false }
```

Then read `examples/custom-runtime/`, which is deliberately small enough to read in one sitting and
is the template for any runtime we do not ship. If your executor is not tokio or smol, a backend for
it is now a crate *you* can publish — no `#[cfg]` edits, no fork, no upstream PR.

- **GitHub**: https://github.com/webrtc-rs/webrtc
- **Sans-I/O core (`rtc`)**: https://github.com/webrtc-rs/rtc
- **Discord**: https://discord.gg/4Ju8UHdXMs

---

## Further Reading

- [Building Async-Friendly webrtc on Sans-I/O rtc: Architecture Design and Roadmap](/blog/2026/01/31/async-friendly-webrtc-architecture)
- [From 13 Mbps to Beating Pion: How We Made webrtc-rs Data Channels Fast](/blog/2026/07/18/from-13-mbps-to-beating-pion)
- [Type-Erase the Interceptor Chain, Not Your Application](/blog/2026/07/28/boxed-interceptor-type-erasure)
