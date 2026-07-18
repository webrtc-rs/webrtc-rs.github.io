---
title: "From 13 Mbps to beating Pion: how we made webrtc-rs data channels fast (with AI on a short leash)"
date: 2026-07-18
author: Stefano Di Martino (StefanoD)
---

# From 13 Mbps to beating Pion

When [issue #101](https://github.com/webrtc-rs/webrtc/issues/101) was opened, the numbers were
brutal. The exact same `data-channels-flow-control` example ran at **~570 Mbps on Pion** and
**~13 Mbps on webrtc-rs**. A 44× gap on a data channel — the one part of WebRTC where you'd hope a
systems language would shine.

That gap is gone. On my machine today, webrtc-rs data-channel throughput **beats Pion's** — decisively
in multi-connection aggregate (roughly 2–3×), and single-connection too **once you enable the opt-in
dedicated reactor thread**. And for the *same bytes transferred*, webrtc-rs burns **50–75% fewer CPU
cycles, instructions, and cache/branch misses than Pion** across every case I measured — that
per-byte efficiency is the most robust part of the result. I'll show the full `poop` tables below,
including the case where webrtc-rs's *plain default* still loses to Pion, because that nuance is the
honest version of the story. The maintainer
[asked me](https://github.com/webrtc-rs/webrtc/issues/101#issuecomment-5008312490) whether I'd write
up how we got here. This is that write-up.

I want to be honest about one thing up front, because it's the more interesting story: **most of
this work was done with an AI coding agent.** Not "I asked a chatbot for tips" — the agent read the
code, formed hypotheses, wrote the patches, and drafted the PRs. What made that produce real,
mergeable, correct optimizations instead of plausible-sounding garbage was **the harness I built
around it**: a set of hard, external ground-truth checks the AI was not allowed to argue with.
`poop`, `perf`, and `bpftrace` weren't just how I measured — they were how I kept the AI honest.

Let me tell both stories at once: what we changed, and how we worked.

---

## The rule that mattered most: measure, don't reason

An AI agent is a fluent, confident hypothesis machine. Ask it why some code is slow and it will give
you a beautifully-argued answer. It might even be right. The problem is you can't tell the difference
between "right" and "confidently wrong" from the prose — and in performance work, confidently wrong
is the default. The received wisdom about what's slow is almost always out of date or was measured on
someone else's hardware.

So the first harness was simple: **no optimization is real until `poop` says it is.**

[`poop`](https://github.com/andrewrk/poop) ("Performance Optimizer Observation Platform") runs two
binaries head-to-head and reports the deltas in wall-time, instructions, cycles, cache-misses,
branch-misses and peak RSS — *with statistical significance markers*, so it tells you when a
difference is just noise. Every single change in this project had to survive an A/B `poop` run
against the exact commit before it. If the agent claimed "this removes an allocation on the hot
path," fine — prove it: build both, `poop` them, show me the instruction-count delta and a `perf`
profile where that allocation symbol actually disappears.

This killed a *lot* of good-sounding ideas. It also produced the single most useful discovery of the
whole effort, which I'll get to.

A concrete example of the discipline: early on, the agent "knew" (from training data) that
`sendmmsg`/UDP GSO batching was the obvious win for UDP throughput. It sounded right. I made it
measure instead. It instrumented the driver's send path and found that on the 1 KB-message
benchmark, **83% of send bursts were a single packet** — GSO had nothing to batch. We *declined*
the optimization and wrote down why. (Weeks later, once other changes had started coalescing sends,
GSO *did* become a win — but only because we had the measurement to know when the world had changed.
More on that below.)

---

## The second rule: the loopback lies to you

The very first thing `perf` told us contradicted the existing diagnosis. There was a handoff
document from earlier macOS investigation describing a 90-second linear throughput ramp and an SCTP
receive-window ceiling. On Linux, none of it reproduced. `perf` showed the machine was ~90% *idle* —
only ~1.6 of 16 cores busy. This wasn't a bandwidth problem. It was a **latency** problem: the
in-process ping-pong between the application task and the driver task was spending tens of
milliseconds per round trip in the tokio scheduler, not in SCTP.

That distinction — latency-bound vs CPU-bound vs memory-bound — turned out to be the thing you have
to get right before any number means anything, and it's exactly where a single benchmark will fool
you. So the third harness was **A/B testing across deliberately different benchmark regimes**, each
stressing a different resource:

- **Single-connection loopback** — latency-bound. Great for finding scheduling and round-trip
  overhead; *useless* for measuring per-byte CPU wins (the CPU is mostly idle, so a 12% instruction
  cut shows up as ~0% throughput).
- **N-connection aggregate (10+ pairs)** — CPU-bound once the cores saturate. This is where per-byte
  efficiency (allocations, copies, crypto, hashing) actually shows up as throughput.
- **Flood / bulk-transfer** — send-bound with buffers under pressure. This is where memory (RSS) and
  send back-pressure behavior show up, and where a slow allocator hurts.
- **Large-message vs 1 KB-message** — changes whether syscall batching (GSO) can engage at all.

The number of times a change looked like a win in one regime and a wash (or a regression) in another
is the reason this list exists. A 12% instruction reduction that's "sub-noise on throughput" is
still worth shipping for an SFU that runs thousands of connections and pays the energy bill — but you
only know that if you measure both the single-connection wall-clock *and* the CPU-bound aggregate.
"Instruction-count deltas overstate wall-clock when the removed work was high-IPC" is now a rule I
have tattooed on my brain.

---

## The third rule: never quote the RFC from memory

This is the harness I'd recommend to anyone doing protocol work with an AI, and it's the one that
caught the most subtle bug.

LLMs will happily "cite" RFC 4960 §6.2 or tell you how Chrome handles a particular SCTP edge case,
in complete, authoritative-sounding sentences, entirely from memory. Sometimes it's right.
Sometimes the section number is off by one, or the requirement is subtly inverted, or the browser
behavior is what the model *expects* rather than what ships. In a protocol stack that has to
interoperate with real browsers, that's how you get a bug that passes every unit test and fails
against Firefox.

So the rule was absolute: **whenever a change touched a spec requirement or an interop assumption,
the agent had to fetch the primary source** — `curl` the actual RFC text from rfc-editor.org, or
read the actual Pion / libwebrtc / Firefox source — and quote *that*, not its own recollection.

Here's what that caught. We rewrote SCTP's FORWARD-TSN generation (more below), and the agent's first
framing was "this is behavior-preserving." I made it fetch RFC 3758. Reading the actual text of
[§3.2](https://www.rfc-editor.org/rfc/rfc3758.txt) revealed that the *original* code had a latent
conformance violation: the Stream-Sequence field "MUST NOT report TSN's corresponding to DATA chunks
that are marked as unordered," and the old code reported them anyway (harmless only because the
receiver happened to ignore them). Our rewrite, which only tracked ordered streams, was the
*conformant* behavior. The change went from "a refactor we hope is equivalent" to "a fix for a spec
violation, with the paragraph quoted in the PR." Same for the SACK-piggyback work, where the correct
requirement lives in RFC 4960 §6.1, not the §6.2 the model first reached for. When you're asking a
maintainer to trust a protocol change, "here is the RFC paragraph, verbatim" is worth ten paragraphs
of confident explanation.

---

## Knowing the enemy: profiling Pion and rustrtc

You can't beat what you don't understand, so I pointed the same tools at the competition. This turned
out to be one of the best uses of the AI: I'd `perf`/`bpftrace` Pion and
[rustrtc](https://github.com/restsend/rustrtc) (a from-scratch Rust WebRTC stack that advertises
~2.8× webrtc-rs), capture *what* they did differently at the syscall and CPU level, and then hand the
agent their actual source code and ask "we do X and pay for it — what do they do here, and why is it
cheaper?"

That comparison is where several of our best changes came from:

- **`bpftrace` on Pion and rustrtc** showed they emitted far fewer SACK-carrying datagrams per byte
  than we did. Reading rustrtc's source (with the agent) showed *why*: it batch-drains its receive
  channel and piggybacks the SACK onto outgoing DATA instead of sending each SACK as its own
  encrypted datagram. That directly inspired our batch-drain work.
- **`perf` normalized per delivered byte** (not per wall-clock second — that's confounded when two
  stacks deliver different volumes in a fixed window) showed the honest gap to rustrtc was
  **CPU-work-per-byte, not syscalls**: ~1.6× the instructions per byte, from allocations, copies and
  crypto. That reframed the whole roadmap away from "clever syscall tricks" and toward "stop copying
  and allocating on the hot path."
- **Reading Pion's allocator behavior** made it clear that Go's per-thread allocator was a real part
  of its low memory footprint, and that our thread-per-connection reactor model was inflating RSS via
  glibc per-thread arenas — which set up the reactor-pool and back-pressure work.

Two honest notes from that comparison, because a blog post that only reports wins is marketing, not
engineering:

1. **rustrtc is still leaner than us.** Its from-scratch, single-runtime, tight-buffer architecture
   uses a fraction of our memory (tens of MB vs hundreds) and is still a bit faster per byte. We
   closed the throughput gap from ~1.9× to near parity in the bulk regime and cut the per-byte
   instruction gap from 1.74× to ~1.31×, but "we beat Pion" is the honest headline, not "we beat
   everything."
2. **Some of what looked like a Pion advantage was a benchmark artifact.** Pion's original 570 Mbps
   was on faster 2021 hardware; on a level playing field (same box, same config, steady-state rather
   than ramp-diluted cumulative averages) the picture is very different, as the numbers below show.

---

## The work, PR by PR

Eighteen PRs landed across the two repos (`webrtc-rs/webrtc`, the runtime/driver, and
`webrtc-rs/rtc`, the sans-I/O protocol core). They group into five themes.

### 1. Kill the scheduler overhead — [webrtc#813](https://github.com/webrtc-rs/webrtc/pull/813)

The latency problem `perf` surfaced. Two changes: a **coalesced wake** (replace a per-message
blocking channel send that woke the driver once per 1 KB message with an atomic flag that fires at
most one wakeup per batch — the same trick Pion's `awakeWriteLoop` uses), and an **opt-in dedicated
reactor thread** that pins a connection's driver to one thread so the tokio scheduler stops migrating
it across workers on every wake (context-switches and CPU-migrations were exploding 50–100× at ≥4
workers). Together: **178 → ~300 Mbps** at the default runtime, closing the Pion gap from ~2× to
~1.2×. This is the PR that turned "webrtc-rs is hopelessly behind" into "it's a race."

### 2. Batch-drain the receive path — [webrtc#815](https://github.com/webrtc-rs/webrtc/pull/815) + [rtc#124](https://github.com/webrtc-rs/rtc/pull/124)

The change directly inspired by profiling rustrtc. The driver now **burst-reads** the UDP socket
(drains up to 64 already-ready datagrams per wakeup instead of one per `select!` iteration), and the
SCTP handler **defers its transmit flush** to one pass per event-loop iteration, so N ingested DATA
chunks produce *one* SACK instead of one per two. Neither does much alone; together they roughly
**doubled** throughput and flipped webrtc-rs from behind Pion to ahead of it. This is the change the
[earlier A/B comment](https://github.com/webrtc-rs/webrtc/issues/101#issuecomment-4950542200)
documented.

### 3. Stop copying and allocating — the `rtc` per-byte rounds ([#104](https://github.com/webrtc-rs/rtc/pull/104), [#105](https://github.com/webrtc-rs/rtc/pull/105), [#106](https://github.com/webrtc-rs/rtc/pull/106), [#107](https://github.com/webrtc-rs/rtc/pull/107), [#108](https://github.com/webrtc-rs/rtc/pull/108), [#111](https://github.com/webrtc-rs/rtc/pull/111), [#113](https://github.com/webrtc-rs/rtc/pull/113), [#121](https://github.com/webrtc-rs/rtc/pull/121), [#125](https://github.com/webrtc-rs/rtc/pull/125), [#130](https://github.com/webrtc-rs/rtc/pull/130))

The long tail, and where the "CPU-per-byte" insight from the rustrtc comparison paid off. Highlights:

- **Zero-copy payloads** on the SCTP send *and* receive hot paths — the datachannel send path was
  doing an `alloc + memcpy` per fragment for a buffer it already owned; the receive path did two
  copies where one would do (#121, #106).
- **Hardware CRC-32C and in-place AEAD**, and running DTLS **AES-GCM on `ring`** with ARMv8 AES/PMULL
  enabled for the RustCrypto path (#107, #125) — crypto was ~4–6% of per-byte CPU and inherent, so
  making it hardware-accelerated mattered.
- **Algorithmic fixes**, which are the ones I trust most because they're not micro-tuning:
  O(N²)→O(N) datachannel queues (#106); and the big one, **FORWARD-TSN generation from O(rwnd) to
  O(streams)** (#113). `perf` had flagged `Association::poll_transmit` at ~9% self-time; annotating
  the hot addresses showed it was almost entirely hashmap probes from a per-SACK linear rescan of the
  whole in-flight window — with PR-SCTP (no-retransmit) that's the *entire* receive window, ~99
  million probes to build a one-entry map on a 512 MB transfer. We replaced it with an incrementally
  maintained map: **−12.3% instructions**, and `poll_transmit` dropped from 8.9% to 0.7%. (Pion has
  the identical O(window) scan, so this is a place we're now *ahead* of it.) This is also the change
  the RFC-3758 primary-source rule turned into a conformance fix.
- **Marshalling without allocations** for SACK / FORWARD-TSN control chunks (#130) and bulk-copy /
  preallocate on the RTCP/media/NACK serialization paths (#105, #108).

### 4. Bound the memory — [webrtc#817](https://github.com/webrtc-rs/webrtc/pull/817) + [rtc#127](https://github.com/webrtc-rs/rtc/pull/127), [webrtc#819](https://github.com/webrtc-rs/webrtc/pull/819) + [rtc#129](https://github.com/webrtc-rs/rtc/pull/129)

Profiling rustrtc and Pion made clear that our peak RSS scaled badly, for two reasons: an unbounded
send path, and one reactor thread per connection (glibc caches a per-thread arena on each). So:

- **Opt-in send back-pressure** (#817/#127): `writable().await` / `try_send()` on the data channel,
  so an application flooding a channel faster than the network drains it blocks (or fails fast)
  instead of growing memory without bound. In a naive-flood stress test this cut peak RSS by ~50–78%
  depending on the configured limit, with the happy path unchanged.
- **A bounded, shared reactor pool** plus a **configurable SCTP receive window** (#819/#129):
  replace thread-per-connection with a pool of `N` reactor threads (default `available_parallelism`),
  so thread count — and the RSS that rides on it — stops scaling with connection count. This is a
  **scale-dependent** tradeoff, and I measured the crossover honestly: below ~cores connections it's
  a mild loss (correctly off by default), but at SFU scale (50–100 connections) it wins *both* axes —
  ~+21% throughput and ~−28% RSS at 50 connections, with RSS that stops growing. Combined with a
  smaller receive window it's ~−70% RSS versus the default at that scale.

### 5. Batch the syscalls, finally — [webrtc#820](https://github.com/webrtc-rs/webrtc/pull/820)

The GSO idea we *declined* early, revived once the batch-drain work meant sends actually coalesced.
UDP **GSO on send** and **GRO on receive** via `quinn-udp`. The measurement story here is a good
capstone for the whole "measure everything" theme, because the sign of the result *flips* by regime:
on a CPU-bound 10-pair run it's a clean win (−24% instructions, −35% wall on large messages); on a
single latency-bound connection, send-side batching turns a smooth 1-in-1-out ping-pong into bursty
N-at-once and *regressed*. `bpftrace` root-caused it, and the fix was an **adaptive threshold** —
only take the GSO `sendmsg` path when a run has ≥16 datagrams to batch — which makes it a win (or
neutral) across every regime we measured. No single benchmark would have told us that.

---

## The numbers today

Fresh A/B, run for this post on the same box (AMD Ryzen 7 1700, 8c/16t, Linux, loopback), current
`webrtc-rs` master versus **Pion v4.2.16**, both driven through the same flow-control harness: 1 KB
messages, 512 KB / 1 MB buffered-amount watermarks, per-interval **steady-state** throughput (warmup
skipped — no ramp-diluted cumulative averages). Two configurations: **unordered / no-retransmit**
(Pion's own `data-channels-flow-control` example ships exactly this) and **ordered / reliable** (the
config the earlier issue-101 comment used). N=10 is ten concurrent connections, aggregate throughput.

**Is this apples-to-apples?** I checked, from source rather than memory, because it's the first thing
that invalidates a benchmark. The application config is identical on both stacks (ordering,
reliability, watermarks, 1 KB messages). The parameter that actually caps loopback throughput — the
SCTP receive window — is **1 MB on both** (`INITIAL_RECV_BUF_SIZE` in rtc-sctp; `initialRecvBufSize`
in Pion's `sctp` v1.10.3, the version v4.2.16 pins). The one webrtc-rs-specific knob I enable is the
**opt-in dedicated reactor thread** — so I report webrtc-rs *both* ways (plain tokio default and
`+dedicated reactor`), because the difference is large and hiding it would be dishonest. Pion has no
equivalent knob; its Go runtime provides that scheduling for free, which is rather the point.
(`poop` needs kernel perf counters, so these were captured with `perf_event_paranoid` lowered; the
Pion harness does a non-trickle full-SDP handshake to match the webrtc-rs one exactly.)

### Steady-state throughput (Mbps, median of runs; ratio vs Pion in parens)

| configuration | Pion v4.2.16 | webrtc-rs (plain default) | webrtc-rs (+dedicated reactor) |
|---|---|---|---|
| Unordered / no-rtx, **N=1**  | 392  | 259 (0.66×) | **689 (1.76×)** |
| Unordered / no-rtx, **N=10** | 1681 | **2863 (1.70×)** | **5453 (3.24×)** |
| Ordered / reliable, **N=1**  | 385  | 184 (0.48×) | **575 (1.49×)** |
| Ordered / reliable, **N=10** | 1848 | **4297 (2.33×)** | **5296 (2.87×)** |

Read that honestly: **single-connection, webrtc-rs's *plain default* loses to Pion** (0.48–0.66×) —
that regime is latency-bound (the CPU sits ~90% idle) and Pion's Go scheduler beats plain tokio on
round-trip latency. Turn on the one-line dedicated reactor and webrtc-rs pulls ahead (1.49–1.76×). In
**multi-connection aggregate — the regime that saturates cores — webrtc-rs wins even at its plain
default** (1.70–2.33×) and by ~2.9–3.2× with the reactor, because there per-byte CPU efficiency, not
scheduling, decides it.

### The `poop` counter tables

`poop` runs each binary head-to-head over **fixed work** (16 MB warmup + 48 MB measured per
connection, setup/teardown included) and reports deltas with significance markers (⚡ = significant
improvement, 💩 = significant regression, no marker = within noise). Benchmark 1 is Pion (baseline);
Benchmark 2 is webrtc-rs **+dedicated reactor**.

**Unordered / no-retransmit, N=1** (64 MB fixed work):

| measurement | Pion v4.2.16 | webrtc-rs +reactor | Δ (webrtc vs Pion) |
|---|---|---|---|
| wall_time        | 1.52 s  | 1.43 s  | ⚡ −5.9% ± 4.0% |
| peak_rss         | 23.6 MB | 11.6 MB | ⚡ −50.9% ± 2.8% |
| cpu_cycles       | 11.0 G  | 2.81 G  | ⚡ −74.5% ± 0.8% |
| instructions     | 8.94 G  | 3.69 G  | ⚡ −58.7% ± 0.5% |
| cache_references | 1.09 G  | 405 M   | ⚡ −62.7% ± 1.2% |
| cache_misses     | 269 M   | 88.2 M  | ⚡ −67.2% ± 1.9% |
| branch_misses    | 72.2 M  | 17.8 M  | ⚡ −75.4% ± 1.3% |

**Unordered / no-retransmit, N=10** (640 MB fixed work):

| measurement | Pion v4.2.16 | webrtc-rs +reactor | Δ (webrtc vs Pion) |
|---|---|---|---|
| wall_time        | 3.40 s  | 1.74 s  | ⚡ −48.8% ± 2.0% |
| peak_rss         | 136 MB  | 111 MB  | ⚡ −18.4% ± 4.5% |
| cpu_cycles       | 106 G   | 31.7 G  | ⚡ −70.1% ± 0.6% |
| instructions     | 79.0 G  | 35.6 G  | ⚡ −54.9% ± 0.5% |
| cache_references | 7.51 G  | 3.03 G  | ⚡ −59.6% ± 0.4% |
| cache_misses     | 1.28 G  | 410 M   | ⚡ −68.0% ± 0.7% |
| branch_misses    | 370 M   | 111 M   | ⚡ −70.0% ± 0.7% |

**Ordered / reliable, N=1** (64 MB fixed work):

| measurement | Pion v4.2.16 | webrtc-rs +reactor | Δ (webrtc vs Pion) |
|---|---|---|---|
| wall_time        | 1.48 s  | 1.97 s  | 💩 +33.2% ± 21.3% |
| peak_rss         | 23.6 MB | 21.5 MB | −8.7% ± 24.9% |
| cpu_cycles       | 11.1 G  | 2.88 G  | ⚡ −73.9% ± 1.0% |
| instructions     | 8.87 G  | 3.72 G  | ⚡ −58.1% ± 0.6% |
| cache_references | 1.08 G  | 415 M   | ⚡ −61.7% ± 1.1% |
| cache_misses     | 269 M   | 86.8 M  | ⚡ −67.7% ± 2.4% |
| branch_misses    | 72.2 M  | 18.1 M  | ⚡ −74.9% ± 1.5% |

**Ordered / reliable, N=10** (640 MB fixed work):

| measurement | Pion v4.2.16 | webrtc-rs +reactor | Δ (webrtc vs Pion) |
|---|---|---|---|
| wall_time        | 3.49 s  | 3.23 s  | −7.5% ± 16.1% |
| peak_rss         | 106 MB  | 191 MB  | 💩 +79.5% ± 22.8% |
| cpu_cycles       | 102 G   | 31.3 G  | ⚡ −69.4% ± 0.7% |
| instructions     | 71.5 G  | 35.7 G  | ⚡ −50.1% ± 0.6% |
| cache_references | 7.59 G  | 3.35 G  | ⚡ −55.9% ± 1.0% |
| cache_misses     | 1.42 G  | 523 M   | ⚡ −63.1% ± 1.6% |
| branch_misses    | 416 M   | 126 M   | ⚡ −69.7% ± 0.9% |

Three things to take from those tables, in order of how much I trust them:

1. **Per-byte CPU efficiency is a decisive, robust win for webrtc-rs.** Every cell, every counter:
   **−50% to −75% cycles, instructions, cache references, cache misses and branch misses** versus
   Pion for the *same bytes transferred*, with tight error bars. This is the Rust-vs-Go per-byte
   story, and it's the part I'd stake the post on. For an SFU paying an energy bill, this is the
   number that matters.
2. **`wall_time` is the noisy one, and it's not throughput.** It includes per-connection *setup*
   (DTLS handshake + reactor-thread spawn, which webrtc-rs pays more of) and, at N=1, it's
   latency-bound so scheduling jitter dominates — the ordered N=1 `wall_time 💩 +33%` is driven by a
   few outlier runs up to 4.2 s, while that same cell's CPU counters are all clean ⚡ wins. That's why
   the steady-state throughput table above (setup excluded, measured window only) is the honest
   throughput figure, and it shows webrtc-rs ahead everywhere except the plain-default single
   connection.
3. **Memory is a split decision.** webrtc-rs is leaner on the unordered path (peak_rss ⚡ −51% at N=1,
   ⚡ −18% at N=10) but *heavier* on ordered/reliable at scale (💩 +79.5%, 191 MB vs 106 MB) — that's
   the reliable retransmit buffer, and it's exactly the cost the configurable receive window and
   reactor pool from theme #4 exist to bound.

For completeness and honesty, the bulk-transfer / flood regime — the one that most favors rustrtc —
puts webrtc-rs at roughly parity-to-ahead of Pion on throughput and near rustrtc, while rustrtc still
wins decisively on memory (tens of MB vs our hundreds). We beat Pion; we've closed most of the gap to
the leanest from-scratch Rust stack; we're not the lightest thing in the room yet. That's the true
state of it.

---

## What I actually learned about building with an AI

If you take one thing from this, let it be the shape of the collaboration, because it's replicable.

The AI was extraordinary at the parts humans are slow at: reading an unfamiliar 100k-line codebase,
holding the SCTP state machine in its head, generating a dozen plausible hypotheses for a hotspot,
writing the patch *and* the tests *and* the PR description, and reading a competitor's source to
explain what they do differently. It compressed weeks of work into days.

It was also, left unchecked, a confident fabricator — of benchmark intuitions that were a decade out
of date, of RFC section numbers, of "this is obviously the bottleneck" claims that `perf` flatly
contradicted. The value wasn't the AI or the tools alone. It was **wiring the tools in as the AI's
reality check**:

- **`poop` as the acceptance test.** No change merges until an A/B run shows a significant delta in
  the direction claimed. The tool's significance markers are doing real work here — they're what stop
  "it felt faster" from becoming a commit.
- **`perf` and `bpftrace` as the hypothesis oracle.** Don't ask the AI *why* it's slow; make it
  attribute the cost to a symbol and a syscall count, then optimize *that*, then prove the symbol
  shrank.
- **Primary sources as the anti-hallucination rule.** For anything a browser or an RFC has an opinion
  about, the model quotes the fetched text, never its memory.
- **Multiple benchmark regimes as the anti-overfit rule.** A win has to survive latency-bound,
  CPU-bound, and memory-bound framing, or it ships with an explicit note about which regime it's for.

Adversarial review had the same flavor: for each substantive change I had a second agent try to
*refute* it — find the correctness bug, the missing test, the regressed regime — rather than confirm
it. That's how the FORWARD-TSN clear-logic test and the send back-pressure close/drop liveness fix
got found before a maintainer had to find them.

Huge thanks to the maintainer for the reviews, for merging eighteen of these, and for pushing back where it
mattered (the send back-pressure API went through three redesigns before it was right). The result is
that the boring, honest goal of issue #101 — "why is our data channel slower than Pion's?" — now has
a boring, honest answer: **it isn't.**

The benchmark harnesses, the competitor comparisons, and the A/B scripts are all reproducible; if
you want to check the numbers on your own hardware, that's the whole point of building it this way.
