# Announcing `webrtc` v0.20.0: Async-Friendly, Runtime-Agnostic WebRTC on Sans-I/O Core `rtc`

We're delighted to announce **webrtc v0.20.0**.

This is the **first non-prerelease version of the new architecture** — the one we sketched back in January in [Building Async-Friendly `webrtc` on Sans-I/O `rtc`](/blog/2026/01/31/async-friendly-webrtc-architecture). It supersedes the Tokio-coupled `v0.17.x` line, which moves to bug-fix-only maintenance.

Three things make this release worth writing about beyond the version number:

1. **What landed since `v0.20.0-rc.1`** — the RC promised "bug fixing, browser polish, performance tuning". That turned out to be the most consequential month of the whole cycle, because it is where the data-channel path went from *correct* to *fast*.
2. **What landed in the Sans-I/O `rtc` core underneath it** — more than 70 commits, and the larger half of the engineering. Most of the perf wins and nearly all the spec-compliance fixes live down there.
3. **How the shipped crate compares to the January design** — including the parts of that design we did **not** build. A roadmap post is only useful if someone later checks it honestly.

---

## What's New in the Async `webrtc` Crate Since rc.1

Between `v0.20.0-rc.1` (June 30) and the end of the stabilization cycle, the async crate took nearly 50 commits. Top-level integration-test targets grew from **19 to more than 30**, and examples from 33 to **36**.

### ✅ The Data-Channel Path Got Dramatically Faster

The headline work of the RC period was performance, told in full in [From 13 Mbps to Beating Pion](/blog/2026/07/18/from-13-mbps-to-beating-pion). Four changes did the heavy lifting:

| PR | Change |
|---|---|
| [#813](https://github.com/webrtc-rs/webrtc/pull/813) | Eliminate Tokio scheduler overhead on the data-channel **send** path |
| [#815](https://github.com/webrtc-rs/webrtc/pull/815) | **Burst-read** the UDP socket to batch the SCTP receive path |
| [#820](https://github.com/webrtc-rs/webrtc/pull/820) | **UDP GSO send + GRO receive** batching via `quinn-udp` |
| [#819](https://github.com/webrtc-rs/webrtc/pull/819) | **Bounded shared reactor pool** + configurable SCTP receive window |

Steady-state throughput on the `data-channels-flow-control` benchmark, median of runs, with the ratio against Pion v4.2.16 in parentheses:

| configuration | Pion v4.2.16 | webrtc-rs (plain default) | webrtc-rs (+dedicated reactor) |
|---|---|---|---|
| Unordered / no-rtx, **N=1**  | 392  | 259 (0.66×) | **689 (1.76×)** |
| Unordered / no-rtx, **N=10** | 1681 | **2863 (1.70×)** | **5453 (3.24×)** |
| Ordered / reliable, **N=1**  | 385  | 184 (0.48×) | **575 (1.49×)** |
| Ordered / reliable, **N=10** | 1848 | **4297 (2.33×)** | **5296 (2.87×)** |

Read that honestly, as the perf post does: at **N=1 the plain default still loses to Pion** (0.48–0.66×) — that regime is latency-bound and Go's scheduler beats plain Tokio on round-trip latency. Turn on the one-line dedicated reactor and we lead (1.49–1.76×). In multi-connection aggregate — the regime that saturates cores — we win **even at the plain default** (1.70–2.33×), and by ~2.9–3.2× with the reactor, because per-byte CPU efficiency decides it there. Under `poop` with fixed work at N=1 we also use **−50.9% peak RSS** and **−74.5% CPU cycles** versus Pion.

The reactor pool deserves a specific note, because it replaced a design that did not scale. Enabling `with_dedicated_reactor_thread(true)` used to spawn **one OS thread per `PeerConnection`**. It now draws from a process-global pool of at most *N* threads, so thread count — and the RSS that rides on it — stops scaling with connection count. `tests/dedicated_reactor_drop.rs` proves the bound directly on Linux by counting `webrtc-rx*` threads in `/proc/self/task`.

New tuning knobs on the builder:

```rust
let pc = PeerConnectionBuilder::new()
    .with_configuration(config)
    .with_handler(Arc::new(MyHandler))
    .with_udp_addrs(vec!["0.0.0.0:0"])
    // Pin the driver to a pooled reactor thread — the big single-connection win
    .with_dedicated_reactor_thread(true)
    .with_reactor_pool_size(4)
    // Trade memory for window: smaller = much lower peak RSS at high connection counts
    .with_sctp_receive_buffer_size(256 * 1024)
    .build()
    .await?;
```

### ✅ Data-Channel Send Back-Pressure (Opt-In)

[#817](https://github.com/webrtc-rs/webrtc/pull/817) added real back-pressure to the data-channel send path, so a fast producer can no longer grow the queue without bound:

```rust
// Bound the outbound queue when building the connection
let pc = PeerConnectionBuilder::new()
    .with_data_channel_send_buffer_limit(4 * 1024 * 1024)
    /* ... */ .build().await?;

// Then either wait for room ...
dc.writable().await?;
dc.send(payload).await?;

// ... or fail fast instead of queueing
dc.try_send(payload).await?;
```

The default remains unbounded, preserving the previous `send()` behavior. Once a limit is configured, `send()` / `send_text()` wait for capacity, `writable()` lets an application pace sends explicitly, and `try_send()` / `try_send_text()` fail fast with `ErrSendBufferFull`. In a naive-flood stress test the bounded queue cut peak RSS by **~50–78%**. Five new test binaries cover the semantics — `data_channel_send_backpressure`, `..._blocking`, `..._unbounded`, `..._after_close`, and `data_channel_close_wakes_send` (a closing channel must wake a blocked sender, not hang it).

### ✅ An Async Media API for Local Tracks

`TrackLocal` gained a pull-based event interface — `TrackLocal::poll() -> Option<TrackLocalEvent>` alongside `bind(ctx, evt_rx)` — so applications can react to bind/unbind and RTCP-driven events on a local track without a callback. `SampleWriter` also learned to carry a payload type. The new `tests/rtcp_processing_webrtc2webrtc.rs` exercises the whole loop between two async peers.

### ✅ Correctness Fixes

- **[#825](https://github.com/webrtc-rs/webrtc/pull/825)** — `on_data_channel` now fires **only** for channels the *remote* peer opened. Previously locally-created channels were announced back to you, which is not what the W3C event means and led to double-handling. Regression test: `tests/on_data_channel_only_for_remote.rs`.
- **[#828](https://github.com/webrtc-rs/webrtc/pull/828)** — the stats structs returned by `get_stats()` are now exported under the names `get_stats` reports, so you can actually name the types you receive.
- **[#832](https://github.com/webrtc-rs/webrtc/pull/832)** — ICE server updates are applied on `set_configuration()`; previously a post-construction change to the server list was ignored.
- **[#808](https://github.com/webrtc-rs/webrtc/issues/808)** — `SettingEngine::set_dtls_cipher_suites()` now lets applications restrict DTLS negotiation to suites compatible with their certificate, and the async crate re-exports `CipherSuiteId` and `SrtpProtectionProfile` so callers do not need a version-locked direct `rtc` dependency merely to name the arguments.
- **[#776](https://github.com/webrtc-rs/webrtc/issues/776)** — deterministic regression coverage verifies that an unordered data channel with `max_retransmits: Some(0)` remains open after an individual socket-send failure; a dropped unreliable message is not itself a terminal channel event.
- **Remote certificate fingerprints** are now covered by `tests/remote_certificate_fingerprint.rs`.

### ✅ Pluggable Runtimes: the Part That Outgrew the Design

The late-cycle change that most reshaped what the crate *is* was pluggable runtimes. Until now, "runtime-agnostic" meant "pick one of our two backends at compile time". It now means the `Runtime` trait is a genuine extension point. The built-ins provide convenient defaults, while a custom implementation is selected through the same per-connection interface.

Four things changed together:

**1. Runtime-neutral primitives.** The library's channels, broadcast, mutexes, and notifications are now plain waker-driven data structures (`async-channel`, `async-broadcast`, `event-listener`, `futures::lock::Mutex`) rather than a runtime's. They are *not* feature-gated, because they need no reactor. Only genuinely reactor-bound operations — timers, I/O, spawning — go through the `Runtime` trait. That single distinction is what made everything else possible, and it shrank the Tokio and smol backends to ~400 lines each.

**2. Features became purely additive.** Each runtime feature now only makes an implementation *available*; none of them selects the primitives the library uses. Enabling both is safe, and a single process can drive different connections on different runtimes:

```rust
let pc = PeerConnectionBuilder::new()
    .with_runtime(my_runtime.clone())   // per connection, not per binary
    .with_udp_addrs(vec!["0.0.0.0:0"])
    .build()
    .await?;
```

**3. A deterministic `MockRuntime`.** Behind `runtime-mock`, it implements the same `Runtime` trait but drives timers from a manually advanced virtual clock and performs no I/O — so a test can advance thirty seconds instantly and assert on what fired, with no sleeping and no flakiness. This finally extends the Sans-I/O layer's deterministic-time property up into the async layer.

The proof that the abstraction is real is the new **`custom-runtime` example**: it implements `Runtime` over `async-executor` + `async-io` — neither Tokio nor smol, defined entirely outside the crate — and runs with **`--no-default-features`**, so neither built-in is even compiled in. I ran it:

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

And `tests/custom_runtime_interop.rs` closes the loop with the thing an example cannot show on its own — **two peer connections on two different runtimes, interoperating in one process**, the answerer on the custom `async-executor` runtime and the offerer on a built-in. Every packet between them crosses that boundary, so completion proves both runtimes are independently live. The example and interop test both pass.

**4. A contract with nothing invented in it.** Once third parties were expected to implement `Runtime`, its surface stopped being an internal detail, and the final commits in that runtime series spent their time removing things from it rather than adding them. The socket types are now `quinn-udp`'s own — `Transmit`, `RecvMeta`, `EcnCodepoint`, `UdpSockRef`, `UdpSocketState` are re-exported rather than re-declared — so a runtime that already knows quinn's socket API has nothing new to learn, and there is no wrapper layer left to drift out of sync with it. The two packet primitives are poll-based:

```rust
fn poll_send(&self, cx: &mut Context<'_>, transmit: &Transmit<'_>) -> Poll<io::Result<usize>>;

fn poll_recv(
    &self,
    cx: &mut Context<'_>,
    bufs: &mut [IoSliceMut<'_>],
    meta: &mut [RecvMeta],
) -> Poll<io::Result<usize>>;
```

That contract does not force runtime implementations to allocate a boxed future for every packet-readiness check. `poll_recv` takes slices because that is the shape the kernel offers: on Linux one `recvmmsg` returns up to 32 datagrams **from different peers**, and each of those may itself carry several GRO-coalesced datagrams from one flow. A minimal implementation fills `bufs[0]`, returns `Ok(1)`, and is done — which is exactly what the `custom-runtime` example does. Implementations can use wider batches where the platform exposes `recvmmsg`, `recvmsg_x`, or an equivalent facility.

Two honest notes on that. The driver currently asks for one message per call, so the multi-message path is available to implementors but not yet exploited internally; wiring it through is a follow-up, and a throughput-under-load one rather than a latency win. And re-exporting `quinn-udp`'s types means a major bump there is a breaking change here — a deliberate trade for not maintaining a parallel set of wrappers.

The practical consequence of all four: **adding a runtime no longer requires us**. An embedded, in-house, or not-yet-written executor is an implementation of one trait in your own crate — no `#[cfg]` edits, no fork, no upstream PR.

### ✅ Documentation and Test Infrastructure

The README was rewritten around a working quick-start, a feature-flag table, and a migration section. The `rtc` workspace went through a full inline-documentation pass: `#![warn(missing_docs)]` is now enabled across all 16 library crates, with crate-level documentation throughout, and `cargo doc --workspace --no-deps` builds cleanly with warnings denied. `codecov.yml` landed so coverage is tracked from here on.

---

## What's New in the Sans-I/O `rtc` Core Since rc.1

The async crate is a thin driver over `rtc`, so most of the engineering in this cycle happened in the core. Over the same period the `rtc` workspace took **more than 70 commits** and roughly **15,000 lines of substantive change** across the protocol crates. A workspace refactor also moved the `rtc` crate to the repository root, producing hundreds of rename-only paths, so raw changed-file totals would be misleading.

### ✅ A Sustained Hot-Path Optimization Campaign

The throughput numbers above are *mostly earned here*. The async layer contributed batching and scheduling; the core contributed per-byte CPU. Eleven PRs across SCTP, SRTP, and RTP/RTCP:

| PR | Change |
|---|---|
| [#104](https://github.com/webrtc-rs/rtc/pull/104) | Eliminate per-packet allocations and byte-by-byte copies on the SCTP/SRTP hot paths |
| [#105](https://github.com/webrtc-rs/rtc/pull/105) | Bulk-copy and preallocate on RTCP / media / NACK serialization |
| [#106](https://github.com/webrtc-rs/rtc/pull/106) | Receive-path copy elimination; **O(N²) → O(N)** SCTP data-channel queues |
| [#107](https://github.com/webrtc-rs/rtc/pull/107) | Hardware CRC-32C, zero-copy send path, O(1) queue pops, in-place AEAD |
| [#108](https://github.com/webrtc-rs/rtc/pull/108) | SCTP send-path and RTP/RTCP marshalling allocation/CPU reductions |
| [#111](https://github.com/webrtc-rs/rtc/pull/111) | SCTP/ICE data-channel hot-path allocation and CPU reductions |
| [#113](https://github.com/webrtc-rs/rtc/pull/113) | **FORWARD-TSN generation O(streams) instead of O(rwnd)** |
| [#121](https://github.com/webrtc-rs/rtc/pull/121) | Zero-copy payload on the SCTP send **and** receive hot path |
| [#124](https://github.com/webrtc-rs/rtc/pull/124) | Defer the transmit flush to coalesce SACKs on read bursts |
| [#125](https://github.com/webrtc-rs/rtc/pull/125) | AES-GCM on `ring`; ARMv8 AES/PMULL enabled for the RustCrypto path |
| [#130](https://github.com/webrtc-rs/rtc/pull/130) | Reduce control-chunk marshal allocations (SACK / FORWARD-TSN) |

Two of these are complexity fixes rather than constant-factor work — the O(N²) data-channel queues (#106) and FORWARD-TSN generation that scaled with the receive window rather than the stream count (#113). Those are the ones that made high-connection-count and large-window configurations viable at all.

### ✅ Flow Control Down to the Association

The async crate's `writable()` / `try_send()` API needs the core to report what the association has actually released:

- **[#127](https://github.com/webrtc-rs/rtc/pull/127)** plumbs **per-stream released-bytes** through SCTP and the data channel, which is the signal the async back-pressure layer is built on.
- **[#129](https://github.com/webrtc-rs/rtc/pull/129)** makes the **SCTP receive buffer (`a_rwnd`) configurable**, the knob `with_sctp_receive_buffer_size()` exposes.

### ✅ Interoperability and Spec Compliance — the Long Tail

This is the least glamorous and arguably most valuable category. A selection:

**SDP / negotiation**

- **[#114](https://github.com/webrtc-rs/rtc/pull/114)** — accept any RFC 8866 `<proto>` token in `m=` lines instead of a hard-coded allowlist.
- **[#115](https://github.com/webrtc-rs/rtc/pull/115)** — allow `extmap` IDs up to 255, per RFC 8285.
- **[#118](https://github.com/webrtc-rs/rtc/pull/118)** — order simulcast layers by the `a=simulcast` attribute (RFC 8853 §5.2), plus a **real-Firefox interop test** (more on this below).
- **[#120](https://github.com/webrtc-rs/rtc/pull/120)** — **negotiate RTX (RFC 4588) by default** and de-encapsulate received RTX packets. This one materially improves loss resilience for anyone streaming video.
- **JSEP rollback** now restores the transceiver `mid` and removes the remote transceiver, and several spurious/missing `negotiation-needed` transitions were fixed (direction changes, post-`stable` triggers, and a false trigger while already `Stable`).
- `a=ice-lite` handling fixed for SFU mode; `codecs()` added to `MediaDescription`.

**ICE / DTLS**

- **[#100](https://github.com/webrtc-rs/rtc/pull/100)** — fix ICE agent crashes from stale candidate indices.
- **[#136](https://github.com/webrtc-rs/rtc/pull/136)** — send srflx/prflx connectivity checks from the **candidate base**, not the mapped address.
- **[#110](https://github.com/webrtc-rs/rtc/pull/110)** — stamp transmits with the selected candidate pair's transport protocol (the ICE TCP correctness fix that let the async layer simplify its write path).
- **[#135](https://github.com/webrtc-rs/rtc/pull/135)** — ignore DTLS timeouts before the transport starts.
- **[#116](https://github.com/webrtc-rs/rtc/pull/116)** — default `rtc::SettingEngine` mDNS mode to `QueryOnly` **so Safari candidates resolve**.
- **[#137](https://github.com/webrtc-rs/rtc/pull/137)** — actually honor `disable_certificate_fingerprint_verification`.

**Data channels**

- **[#101](https://github.com/webrtc-rs/rtc/pull/101)** — open the SCTP stream for *negotiated* data channels.
- **[#132](https://github.com/webrtc-rs/rtc/pull/132)** — open data channels immediately when SCTP is already connected.
- **[#131](https://github.com/webrtc-rs/rtc/pull/131)** — fix receiving an empty message payload.
- **[#140](https://github.com/webrtc-rs/rtc/pull/140)** — default `ordered` to `true`, as documented.
- **[#138](https://github.com/webrtc-rs/rtc/pull/138)** — reject sends the write path cannot carry out, rather than silently dropping them.

**Media**

- **[#99](https://github.com/webrtc-rs/rtc/pull/99)** — fix AV1 LEB128 encoding for values > 127.
- **[#122](https://github.com/webrtc-rs/rtc/pull/122)** — validate a reassembled fragmented AV1 OBU size against the buffer length.
- **[#123](https://github.com/webrtc-rs/rtc/pull/123)** — return an error when `write_rtp` finds no matching RTP header extension id; `write_rtp` was also tightened to accept only sender payload types, removing an unsafe base-codec fallback.

### ✅ Crypto Backends, External Signers, and Platform Reach

- **[#95](https://github.com/webrtc-rs/rtc/pull/95)** — **`aws-lc-rs` support alongside `ring`**, so you can pick your crypto backend by feature flag. (A follow-up fixed the test suite under `--no-default-features --features aws-lc-rs` by passing the rustls `CryptoProvider` explicitly instead of letting rustls infer it from unified crate features.)
- **[#94](https://github.com/webrtc-rs/rtc/pull/94)** — a **`CustomSigner` trait** for external signing providers, so DTLS signing can be delegated to an HSM, TPM, or cloud KMS instead of requiring raw private-key bytes in memory. This unlocks TPM-backed device certificates.
- **[#126](https://github.com/webrtc-rs/rtc/pull/126)** — `wasm32-wasip2` can now be targeted (fallback interface enumeration for non-Unix/non-Windows targets).
- `rkyv` was bumped to 0.8.17 to avoid known advisories, alongside a general dependency refresh.

### ✅ A Type-Erasable Interceptor Chain

`Interceptor` dropped its `Sized` and `'static` supertraits, so `Box<dyn Interceptor>` is now a legal interceptor and `RTCPeerConnection<BoxedInterceptor>` is a concrete, nameable type. Applications that build their chain at runtime no longer need to hand-write a facade trait to hide the type parameter — the SFU crate deleted ~250 lines of boilerplate doing exactly that. Full story in [Type-Erase the Interceptor Chain, Not Your Application](/blog/2026/07/28/boxed-interceptor-type-erasure).

A few API-surface papercuts were fixed alongside it: `RTCCertificate` members and `DTLSParameters` are now public.

---

## Scorecard: the January Design vs. the Shipped Crate

In January we published an architecture proposal with a five-phase roadmap and a table of success metrics. Here is where each piece actually landed.

### What We Achieved

**Quinn-style single crate with a runtime trait — and then past it.** No `webrtc-tokio` / `webrtc-async-std` crate explosion. One `webrtc` crate and a `Runtime` abstraction over task spawning, timers, and sockets. The default path is a feature flag:

```toml
# Tokio (default)
webrtc = "0.20"

# smol
webrtc = { version = "0.20", default-features = false, features = ["runtime-smol"] }
```

But the final shape of this went **further than the January design** — see *Pluggable Runtimes: the Part That Outgrew the Design* above.

**Push-based trait handler — as designed.** The January post argued that WebRTC's 8-ish interconnected event types make a single handler better than 6+ event streams, and picked `PeerConnectionEventHandler`. That is exactly what shipped, with default no-op methods so you implement only what you need.

**Sans-I/O foundation — as designed.** The WebRTC protocol state machines live in `rtc` and are testable with no sockets and no runtime; the async crate is a `PeerConnection` handle plus a background `PeerConnectionDriver` structured as `poll_writes` → `poll_events` → `poll_reads` → `poll_timeout`/`handle_timeout`.

**A tractable lifecycle.** The callback-registration pattern behind the `v0.17.x` lifetime problem ([#772](https://github.com/webrtc-rs/webrtc/issues/772)) is gone: one handler is owned with the connection instead of user-owned boxed closures being registered independently. Applications should still call `pc.close().await`. Explicit `close()` marks the driver as closing, wakes blocked data-channel senders, and terminates the driver task; on the dedicated-reactor path it also waits for a bounded clean shutdown before aborting any task that remains. Dropping a dedicated-reactor connection signals the driver so it cannot pin a reactor-pool slot, but `Drop` is not a substitute for explicit async shutdown on every runtime path.

**Performance well past the target.** The January metric was ">500 Mbps" data-channel throughput against a ~300 Mbps baseline. We shipped 689 Mbps single-connection with the reactor and 5.3–5.5 Gbps in 10-connection aggregate, at roughly a quarter of Pion's CPU cycles per byte.

**Deliverables beyond the plan.** The roadmap asked for "10+ working examples"; the crate ships **36**. It asked for a migration guide and getting-started docs; both are in the README.

| January success metric | Target | Shipped |
|---|---|---|
| DataChannel throughput | >500 Mbps | ✅ 689 Mbps (N=1, reactor); 5.3 Gbps (N=10) |
| Memory leak per connection | 0 KiB | ⚠️ Callback-registration leak pattern removed; exact 0 KiB target not independently measured |
| Supported runtimes | 4+ | ⚠️ 2 built-in + test-only mock; third-party `Runtime` implementations are supported |
| Browser interop | 100% | ⚠️ Chrome/Safari manual; Firefox frozen-SDP test in CI; Edge untested |
| Connection setup time | <1 s | ❔ Not measured or published |
| Test coverage | >80% | ❔ Not established; tracking just enabled |
| Working examples | 10+ | ✅ 36 |

### The Gaps — What We Did Not Build

**Two built-in runtimes, not the four we named — and two of those four aged badly.** `runtime-tokio` and `runtime-smol` ship as built-ins. Of the other two the January post promised:

- **[async-std is discontinued](https://github.com/async-rs/async-std).** Its maintainers recommend migrating to smol, so an adapter for it would be a liability rather than a feature. This roadmap item did not go unbuilt so much as expire; the correct action in 2026 is to drop it, and smol covers that constituency.
- **embassy remains genuinely out of reach.** It implies `no_std`, and the crate uses `std` types throughout. Listing it beside async-std as a comparable line item was the least realistic thing in the January post, and that is still true.

What changed is that the count matters much less than it did. With the `Runtime` trait now a real extension point, "supported runtimes" is no longer a number we control — which is the right answer to a roadmap whose named targets can be archived out from under it.

**The handler takes `&self`, not `&mut self`.** This is the most visible deviation. The January post promised "centralized state management with `&mut self`" and "no `Arc` explosion". What shipped is:

```rust
#[async_trait::async_trait]
impl PeerConnectionEventHandler for MyHandler {
    async fn on_track(&self, track: Arc<dyn TrackRemote>) { /* ... */ }
}
```

Two consequences. First, the handler is shared as `Arc<dyn PeerConnectionEventHandler>`, so **mutable handler state still needs interior mutability** (`Mutex`/`RwLock`) — you write one `Arc` and one lock instead of an `Arc::clone` per callback per event, which is a large improvement but not the `&mut self` story we described. Second, we use `#[async_trait]` rather than native `async fn in trait`, because the trait must remain `dyn`-compatible and native async fns in traits are not yet object-safe. Anyone reading the January post should adjust expectations accordingly.

**No stream-based API.** The January post offered `pc.tracks()` / `pc.ice_candidates()` as an optional convenience layer on top of the handler. It was never built. If you want stream semantics today, bridge them yourself: have your handler forward events into an `mpsc` channel.

**Browser interop is partly automated, but the matrix is not filled in.** The picture here is better than "manual testing" and worse than the roadmap's 100%:

- **Chrome and Safari** are exercised by hand against [appr.tc](https://appr.tc), which is itself built on this stack. Safari specifically drove a real fix — the `rtc::SettingEngine` mDNS mode now defaults to `QueryOnly` so Safari's candidates resolve ([#116](https://github.com/webrtc-rs/rtc/pull/116)).
- **Firefox has a genuine automated interop test**, but at the protocol layer: `rtc`'s `tests/firefox_simulcast_interop.rs` freezes a real simulcast offer captured from **Firefox 152** (via Marionette) and asserts the core parses and answers it correctly. That pinned down three Firefox-specific SDP shapes: directional `a=extmap:<id>/sendonly` qualifiers on RID header extensions (RFC 8285 §8), `a=msid:- <track>` with an empty stream id, and per-layer `a=ssrc-group:FID` alongside `a=rid` / `a=simulcast:send`.
- **What is still missing**: frozen-SDP tests are not live-browser tests. There is no Selenium/Playwright job in CI, **Edge is untested**, and the 4-browser × 4-capability matrix from the roadmap has no filled-in cells. Media-level interop (audio/video/simulcast actually flowing to a live browser) is validated ad hoc, not continuously.

This remains the most significant quality gap, and the best place for community help.

**Two metrics are simply unmeasured.** Connection setup time and test coverage have no published numbers. `codecov.yml` only just landed. We would rather say "unknown" than quote a figure we did not measure.

---

## Migrating from v0.17.x — Please Do

If you are on `v0.17.x` or earlier, this is the release to move to. `v0.17.x` is now bug-fix-only, and the rewrite directly addresses its central structural problems: callback registration, Tokio coupling, and protocol logic intertwined with I/O.

### Callback hell and `Arc` explosion → one handler

**v0.17.x** — an `Arc::clone` before the closure, another inside it, `Box::new(move |…| Box::pin(async move …))`, repeated per event type:

```rust
let pc = Arc::new(api.new_peer_connection(config).await?);

let pc_c1 = Arc::clone(&pc);
pc.on_peer_connection_state_change(Box::new(move |s| {
    let _pc = Arc::clone(&pc_c1);
    Box::pin(async move { println!("State: {s}"); })
}));

let pc_c2 = Arc::clone(&pc);
pc.on_ice_candidate(Box::new(move |c| {
    let _pc = Arc::clone(&pc_c2);
    Box::pin(async move { /* signal candidate */ })
}));
// ... four more blocks like this
```

**v0.20.0** — one type, one `Arc`, all events together, and only the methods you care about:

```rust
use std::sync::Arc;
use webrtc::peer_connection::{
    PeerConnection, PeerConnectionBuilder, PeerConnectionEventHandler, RTCConfigurationBuilder,
    RTCIceServer, RTCPeerConnectionIceEvent, RTCPeerConnectionState,
};
use webrtc::runtime::TokioRuntime;

struct MyHandler { /* your state, behind a Mutex if mutable */ }

#[async_trait::async_trait]
impl PeerConnectionEventHandler for MyHandler {
    async fn on_connection_state_change(&self, state: RTCPeerConnectionState) {
        println!("State: {state}");
    }

    async fn on_ice_candidate(&self, event: RTCPeerConnectionIceEvent) {
        // signal event.candidate to the remote peer
    }
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let config = RTCConfigurationBuilder::default()
        .with_ice_servers(vec![RTCIceServer {
            urls: vec!["stun:stun.l.google.com:19302".to_owned()],
            ..Default::default()
        }])
        .build();

    let pc = PeerConnectionBuilder::new()
        .with_configuration(config)
        // Runtime selection is per connection. Replace this with `SmolRuntime`
        // or your own `Runtime` implementation without changing the WebRTC API.
        .with_runtime(Arc::new(TokioRuntime))
        .with_handler(Arc::new(MyHandler { /* ... */ }))
        .with_udp_addrs(vec!["0.0.0.0:0"])
        .build()
        .await?;

    let offer = pc.create_offer(None).await?;
    pc.set_local_description(offer).await?;
    Ok(())
}
```

`build()` returns an opaque `impl PeerConnection`. `PeerConnection` is an object-safe trait, so to store the connection in a struct or share it across tasks, wrap it once:

```rust
let pc: Arc<dyn PeerConnection> = Arc::new(pc);
```

No runtime or interceptor type parameter leaks into your own types.

### The other v0.17.x pain points

| Migration concern | v0.20.0 |
|---|---|
| **Resource leaks in callbacks** ([#772](https://github.com/webrtc-rs/webrtc/issues/772)) | One connection-owned handler replaces independently registered boxed closures. Call `close().await` for deterministic driver and socket shutdown. |
| **Tight Tokio coupling** — hidden `tokio::spawn` / `tokio::time` inside the library | Async socket wrapping/polling, timers, spawning, and name resolution go through the `Runtime` trait. Switch to smol with a feature flag; the peer-connection API is unchanged. |
| **Untestable protocol logic** — needed a runtime and real sockets | Protocol lives in Sans-I/O `rtc`: feed bytes, read `poll_write()`, drive time by hand. Deterministic, no flaky timing. |
| **Scattered event handling** across 6+ registrations | One handler trait, ordered delivery, one coordination point. |
| **Unbounded send buffering** | Opt-in back-pressure: `with_data_channel_send_buffer_limit`, `writable()`, `try_send()`. |
| **Initial rewrite throughput** (~13 Mbps when the performance campaign began) | 689 Mbps single-connection with the dedicated reactor; multi-Gbps in aggregate. |

The rewritten async stack retains features such as TURN relay, mDNS candidates, and statistics while adding runtime-pluggable I/O, opt-in send back-pressure, ICE TCP active/passive support in the new driver, UDP GSO/GRO batching, the pooled reactor, and **RTX (RFC 4588) negotiated by default**. At the Sans-I/O layer, `rtc` additionally offers a choice of crypto backend (`ring` or `aws-lc-rs`), external DTLS signing via `CustomSigner` for HSM/TPM/KMS-held keys, and `wasm32-wasip2` as a build target.

### A pragmatic migration path

1. **Read the events you handle.** Every `on_*` callback becomes a method on one handler type. That is the bulk of the work, and it usually *shrinks* the code.
2. **Move mutable callback state behind one lock** in the handler struct, instead of per-callback `Arc<Mutex<_>>` captures.
3. **Delete your `Arc::clone` scaffolding.** If you were cloning the peer connection into closures just to call methods on it, hold `Arc<dyn PeerConnection>` in the handler instead.
4. **Keep Tokio at first.** `runtime-tokio` is the default, so runtime independence costs you nothing up front and is there when you want it.
5. **Then tune.** Turn on `with_dedicated_reactor_thread(true)`, size the reactor pool, and bound the SCTP receive window and data-channel send buffer for your connection count.

Expect it to be a real port, not a drop-in: the event-handler traits replace callbacks and the API is async throughout. In exchange you get an architecture where the protocol is testable without I/O and the runtime is your choice.

---

## Try It Out

```toml
[dependencies]
webrtc = "0.20"
```

Or on smol:

```toml
[dependencies]
webrtc = { version = "0.20", default-features = false, features = ["runtime-smol"] }
```

Or on neither — bring your own, and hand it to each connection:

```toml
[dependencies]
webrtc = { version = "0.20", default-features = false }
```

Then browse the [36 examples](https://github.com/webrtc-rs/webrtc/tree/master/examples). Good starting points:

- `data-channels` and `data-channels-flow-control` — the basics, then the fast path
- `data-channels-flow-control-multi` / `data-channels-stress-bench` — reproduce the benchmark numbers
- `play-from-disk-vpx` / `save-to-disk-h26x` — media in and out
- `trickle-ice-relay`, `ice-tcp` — connectivity in hostile networks
- `stats` — observability
- `rtcp-processing` — the new `TrackLocal::poll` flow
- `custom-runtime` — the whole stack on `async-executor` + `async-io`, with no built-in runtime compiled in

---

## Get Involved

The gaps above are an invitation, and the browser matrix is the most valuable one:

- **Browser interop**: a live-browser Playwright/Selenium job in CI, Edge coverage, and more captured-SDP fixtures in the style of `rtc`'s Firefox 152 simulcast test
- **Runtimes**: if your executor is not Tokio or smol, a backend is now a well-scoped crate *you* can publish — implement `Runtime`, copy the `custom-runtime` example's shape, and nothing upstream has to change
- **Streams**: if you want `pc.tracks()`, we would like to see a design
- **Migration reports**: tell us what was awkward coming from `v0.17.x` — that feedback shapes `v0.21`

Join us:

- **GitHub**: https://github.com/webrtc-rs/webrtc
- **Sans-I/O core (`rtc`)**: https://github.com/webrtc-rs/rtc
- **Discord**: https://discord.gg/4Ju8UHdXMs
- **Discussions**: https://github.com/webrtc-rs/webrtc/discussions

---

## Conclusion

`v0.20.0` delivers the architecture we proposed in January: a Sans-I/O protocol core, a Quinn-style runtime abstraction, a single push-based event handler, deterministic protocol tests, and — after the RC-period performance work — a data-channel path that beats Pion on per-byte CPU, memory, and aggregate throughput. On one axis it goes past the design: the runtime abstraction ended up a real extension point, so the stack runs on a runtime we have never heard of, chosen per connection.

Underneath it, the `rtc` core spent this cycle on the work that does not make headlines: eleven hot-path optimization PRs, two algorithmic complexity fixes, and a long tail of SDP, ICE, DTLS, data-channel, and media compliance fixes — plus RTX by default, a second crypto backend, external signing, and a new build target.

It still does not deliver everything we described. No stream layer, `&self` instead of `&mut self` on the handler, embassy out of reach behind `no_std`, and a browser matrix with one automated cell in it. Publishing that list next to the wins is the point: the design post was a promise, and this is the audit.

If you are still on `v0.17.x`, the callback hell, the `Arc` explosion, the leaked closures, and the Tokio lock-in are all fixed on the other side of this port. 🦀

---

## Links

- **GitHub**: https://github.com/webrtc-rs/webrtc
- **rtc (Sans-I/O)**: https://github.com/webrtc-rs/rtc
- **Examples**: https://github.com/webrtc-rs/webrtc/tree/master/examples
- **Docs**: https://docs.rs/webrtc
- **AppRTC**: https://appr.tc
- **Discord**: https://discord.gg/4Ju8UHdXMs

---

## Further Reading

- [Building Async-Friendly webrtc on Sans-I/O rtc: Architecture Design and Roadmap](/blog/2026/01/31/async-friendly-webrtc-architecture)
- [WebRTC v0.20.0-alpha.1: Async-Friendly WebRTC Built on Sans-I/O](/blog/2026/03/01/webrtc-v0.20.0-alpha.1-async-webrtc-on-sansio)
- [WebRTC v0.20.0-rc.1: Toward Stable Async-Friendly WebRTC](/blog/2026/06/30/webrtc-v0.20.0-rc.1-toward-stable-async-webrtc)
- [From 13 Mbps to Beating Pion: How We Made webrtc-rs Data Channels Fast](/blog/2026/07/18/from-13-mbps-to-beating-pion)
- [Type-Erase the Interceptor Chain, Not Your Application](/blog/2026/07/28/boxed-interceptor-type-erasure)
