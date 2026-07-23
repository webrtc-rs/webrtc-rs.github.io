# AppRTC Goes Group: Automatic P2P → SFU Upgrade at appr.tc

Nine days ago, when we [announced the `sfu` crate](/blog/2026/07/13/announcing-sfu-v0.20.0-rc.3), we described the signaling milestone we were building toward:

> - **≤ 2 participants stay peer-to-peer.** A 1:1 call never touches the SFU — there is no reason to pay for a hop that buys nothing.
> - **On the 3rd joiner, the room upgrades P2P → SFU.** Each client publishes its tracks, and the SFU offers back the others': browser ⇄ SFU ⇄ browser.
> - **The SFU boundary never changes.** How signaling bytes reach the state machine … is a *driver* detail. That is precisely what Sans-I/O buys us here.

That milestone is now live. **[https://appr.tc](https://appr.tc)** no longer speaks only the P2P half of the story — open a room in three browser tabs and the call **upgrades itself from a direct peer-to-peer connection to a Selective Forwarding Unit**, mid-session, without a page reload, a second permission prompt, or a dropped frame.

This post is about the new **V2 P2P/SFU signaling server** that makes that happen: what it does, how the upgrade works, and why the whole thing is built on the same Sans-I/O core as everything else in the family.

---

## Two is a call, three is a conference

A peer-to-peer mesh is the right answer for a 1:1 call and the wrong answer for a group one. With *N* participants, every browser has to upload *N−1* copies of its own video — fine at *N=2*, quadratic misery beyond it. An SFU fixes that: each client uploads **once**, and a server in the middle forwards the streams to everyone else.

But an SFU is not free. It is a media hop with CPU, bandwidth, and an extra network round trip that a 1:1 call gains nothing from. So the policy AppRTC now implements is the obvious one, made automatic:

- **1 or 2 members → direct P2P.** No SFU worker ever sees the room, its SDP, or its media.
- **The 3rd member → the room upgrades P2P → SFU.** From then on it is `browser ⇄ SFU ⇄ browser` for everyone.

The user does nothing. They share a room link; the first two get a direct call; the third arrival quietly turns it into a conference. The room stays affine to its assigned worker until it empties.

> **Try it:** open [https://appr.tc](https://appr.tc) in two tabs for a P2P call, then add a third to watch it become an SFU conference. The room-selection page ships with the **V2 P2P/SFU** mode checked by default; unchecking it falls back to the legacy V1 two-party flow.

---

## Three processes, one authority

AppRTC is not one server. It is three cooperating processes plus the SFU workers, and the split is deliberate:

```
Browser ── HTTP ─────────────▶ appweb      (HTTP room API; gRPC client of signaling)
Browser ── WebSocket ────────▶ signaling   (Sans-I/O authority; owns all room state)
appweb  ── unary gRPC ───────▶ signaling
sfu     ── bidi gRPC stream ─▶ signaling   (one long-lived OpenSfuSession per worker)
Browser ── ICE/DTLS/SRTP ────▶ sfu         (media never touches appweb or signaling)
```

Two rules make the design tractable:

- **`signaling` is the single authority.** It owns room membership, mode, epochs, worker assignment, and the upgrade barrier. `appweb` holds *no* room state — it is a thin HTTP front door that round-trips every mutation to `signaling` over gRPC. The SFU owns only a *projection* of the members assigned to it; it never decides occupancy or initiates an upgrade.
- **Media never passes through the control plane.** Browser WebSocket signaling connects straight to `signaling`; browser media flows straight to the `sfu` worker over ICE/DTLS/SRTP. Neither ever transits `appweb`.

The workspace is four Rust crates:

| Crate | Role |
|---|---|
| `apprtc` | The runtime adapters and the standalone `appweb`, `signaling`, and `sfu` binaries: TLS listeners, WebSocket sessions, gRPC service/client, UDP media, graceful shutdown. |
| `appweb` | HTTP room API, templates, static web assets, and the reusable gRPC client for the authority. |
| `signaling` | The authoritative V1/V2 room model — **a pure Sans-I/O crate with no sockets, threads, clock, or entropy of its own.** |
| `signaling-proto` | The generated Protobuf/tonic contract shared by all three. |

---

## The authority is a Sans-I/O state machine

If you have read any of our earlier posts, you know the pattern by now. Just as `rtc` is a Sans-I/O WebRTC endpoint and `sfu` is a Sans-I/O media engine, the **`signaling` authority is a Sans-I/O state machine** — the same `sansio::Protocol` discipline, no I/O inside:

```
Collider          ── the authority: sessions, worker registry, SFU assignment
└── RoomTable      ── V1 and V2 room tables (separate numeric/string namespaces)
    └── Room       ── one room's members, mode, and signal epoch
        └── Client ── one member's registration, queue, and reconnect timer
```

Every layer implements `sansio::Protocol`, so the whole thing is deterministic and testable **in memory, without a socket or a wall clock.** The runtime (`apprtc`) is the only place sockets exist: `ws_server.rs` turns browser WebSocket frames into commands, `grpc_server.rs` adapts the private gRPC service, and `signaling_server.rs` owns a single serialized event loop that drives the `Collider` and fires its timeouts. Every browser and authority operation flows through that one loop, so there is no lock and no shared-state race — the state machine is advanced by exactly one owner.

What does V2 add over the legacy AppRTC contract? Numeric `u64` room/client IDs (validated with `BigInt` in the browser to dodge JS `Number` precision loss), signaling-issued **admission tokens** bound to `(room, client)`, an explicit `registered` acknowledgement, **signal epochs**, symmetric WebSocket relay for offers/answers/trickle-ICE, reconnect grace, and survivor promotion. V1 stays wire-compatible on its own routes and its own string-keyed table — a V1 room named `"42"` and a V2 room `42` are simply different rooms.

---

## The upgrade moment

Here is what happens the instant the third browser joins room 42, whose existing P2P members are A and B:

```
C  ── POST /v2/join/42 ────────────▶ appweb
appweb ── AdmitV2(42, C) ──────────▶ signaling
                                     signaling: pick a Ready worker with capacity,
                                                set room mode = Upgrading
signaling ── JoinMember(42, A) ────▶ sfu     ; sfu ─ MemberJoined(A) ─▶ signaling
signaling ── JoinMember(42, B) ────▶ sfu     ; sfu ─ MemberJoined(B) ─▶ signaling
signaling ── JoinMember(42, C) ────▶ sfu     ; sfu ─ MemberJoined(C) ─▶ signaling
                                     signaling: all three joined → commit mode = SFU,
                                                increment signal epoch 0 → 1
signaling ── {control:"sfu-upgrade", epoch:1} ─▶ A and B
appweb    ── join success (mode=sfu) ──────────▶ C   (joins directly in SFU mode)
```

Three details make this safe rather than merely hopeful:

**`Upgrading` is a real state, not a flag.** While the room is `Upgrading`, additional joins get a retryable `ROOM_TRANSITION` and browser signaling is held. The commit happens only after *every* member's `MemberJoined` barrier returns from the worker — so a browser can never send a publish offer to a worker that has not yet created its client. If any join fails, the provisional third member is removed and the original P2P pair is restored, untouched.

**Signal epochs make in-flight frames race-free.** Every V2 room carries a monotonic `signal_epoch` that increments on the P2P→SFU commit. Browsers stamp the epoch they know onto every `send`; the hub silently drops any frame whose epoch is stale or that arrives while the room is `Upgrading`. That is what stops a late pre-upgrade P2P renegotiation from being misrouted to the worker as a bogus publish offer.

**The transport handoff is make-before-break.** On `sfu-upgrade`, A and B create a *fresh* `RTCPeerConnection` to the SFU and attach the **same** local `MediaStreamTrack` instances — no recapture, no second camera prompt. The old P2P connection stays alive until the SFU connection's ICE state reaches `connected`, and only then does the UI cross-fade to the grid and close the old peer connection. A media gap never appears.

---

## Perfect negotiation at the SFU boundary

Once a browser is in SFU mode it does two things at once: it **publishes** its own tracks (browser offers, SFU answers) and it **subscribes** to everyone else's (SFU offers, browser answers). Two offerers on one connection is textbook glare, so the browser runs the [perfect-negotiation](/blog/2026/01/23/perfect-negotiation-webrtc-deep-dive) roles — **the browser is polite, the SFU is impolite.**

One rule turned out to matter more than any other, and it is worth stating plainly:

> **The SFU never makes the first offer. The client does.** Only after the client's first offer is answered and the initial negotiation completes does the SFU begin issuing its own subscribe re-offers.

Why? Because the browser establishes the RTP header-extension ID map in its first offer, and Chrome rejects a BUNDLE group where one extension ID maps to two different URIs. If the SFU raced in with a subscribe offer using its own extension namespace before the client had negotiated, the browser's `setLocalDescription` would fail with a codec/extension collision. Making the client's first offer authoritative — and having the SFU wait for a stable signaling state before it re-offers — removes the collision at the source instead of papering over it with SDP munging.

When glare does happen — a browser publish offer meets an outstanding SFU subscribe offer — the resolution is the polite peer's ordinary path: the browser rolls back its pending offer, answers the SFU's, and its `negotiationneeded` handler republishes once the state is stable again. The SFU-side rejected offer is silently dropped; nothing surfaces to the user.

---

## One application, two layouts

A guiding UI constraint shaped the browser code: **the only thing that should change between P2P and SFU is the remote view.** Self-view, mute buttons, hangup, device selection, status — all of it keeps its exact P2P position and behavior. P2P shows one remote participant full-screen; SFU shows a responsive grid of per-publisher tiles overlaid in the same stage, with the self-view still tucked in its corner.

It is one call session with two layouts, not two applications. A P2P→SFU transition never reloads the page, never replaces the signaling socket, and never reacquires the camera. Each SFU tile is keyed by the publisher's client id — recovered from the `peer-<clientid>` msid the SFU stamps on every forwarded track — so a peer's audio and video land in the same tile.

Two behaviors are worth calling out because they are exactly what users notice:

- **Tiles disappear when a peer leaves.** A departed peer's forwarded transceiver goes `inactive` rather than firing the track's `ended` event, so the browser reconciles against the negotiated transceivers after every SFU re-offer and drops any tile whose media is gone.
- **Closing the tab counts as leaving — immediately.** For an SFU member, a dropped WebSocket is treated as an instant leave: the authority issues the worker `LeaveMember` right away, the SFU stops forwarding that publisher, and the other participants' tiles prune within a beat. (P2P members keep a 10-second reconnect grace, so a refresh there rejoins the call instead of collapsing it.)

---

## Assignment, reconnect, and failure

For the initial three-member upgrade, `signaling` filters out disconnected, draining, or full workers and picks the least-loaded one — the minimum `(assigned_clients, assigned_rooms, instance_id)` tuple, with the id breaking ties deterministically. Later joins to that room do **not** re-run the selector; the room is affine to its worker until it empties.

Each SFU worker owns exactly one long-lived `OpenSfuSession` gRPC stream and reconnects with exponential backoff if `signaling` is not up yet. Membership commands (`JoinMember`/`LeaveMember`) are idempotent by `(room_id, client_id, lifecycle_id)` and carry a signaling-allocated `request_id` for replay, so a transient stream reconnect resynchronizes safely: the hub sends a `SyncRoom` roster, then replays unacknowledged commands, and the adapter returns cached results rather than applying anything twice. A *restarted* worker gets a new `instance_id` and is treated as a brand-new worker — it never inherits the old process's rooms. If an assigned worker stays gone past its grace period, the room is marked `Failed` and its members get a `room-failed` control.

---

## Tested without a network

Because the authority is Sans-I/O, its interesting logic — admission, the three-member upgrade barrier, stale-epoch drops, survivor promotion, same-instance worker reconnect/sync, immediate SFU-member leave — is driven entirely in memory by `cargo test --lib`, no sockets and no wall clock. On top of that, the `apprtc` integration suite boots real `appweb`, `signaling`, and `sfu` TLS processes and drives headless WebRTC clients through the whole flow: the third-member join barrier, browser upgrade controls, three-client data channels, and three-publisher RTP forwarding. CI runs the same sequence with a release build.

---

## Try it out

The easiest path is the live demo:

**Open [https://appr.tc](https://appr.tc) in three tabs, join the same room, and watch P2P become a group call.**

To run the whole stack yourself from the [`apprtc`](https://github.com/webrtc-rs/apprtc) repo — three processes, HTTPS/WSS with the bundled dev cert:

```bash
git clone https://github.com/webrtc-rs/apprtc && cd apprtc
git submodule update --init --recursive

# 1. signaling authority (browser WSS + private gRPC)
cargo run -p apprtc --bin signaling -- --host-ip 127.0.0.1 --port 8081 \
  --grpc-port 50051 --tls &

# 2. one SFU worker
cargo run -p apprtc --bin sfu -- --host-ip 127.0.0.1 \
  --media-port-min 35000 --media-port-max 35000 \
  --grpc-url https://127.0.0.1:50051 --insecure-tls &

# 3. AppWeb front door
cargo run -p apprtc --bin appweb -- --host-ip 127.0.0.1 --port 8080 --web-root appweb \
  --public-url https://127.0.0.1:8080 --ws-url wss://127.0.0.1:8081/ws \
  --grpc-url https://127.0.0.1:50051 --insecure-tls --tls &
```

Then open [https://127.0.0.1:8080](https://127.0.0.1:8080) (trust the self-signed cert first) in three tabs.

---

## What's next

This is the group-calling milestone, and it is honest about its edges:

- **SFU → P2P downgrade is deferred.** A room that drops back to two members stays on the SFU today. The reserved design is a break-before-make downgrade behind a dwell timer; it is specified but not implemented.
- **Browser reconnect is partial.** P2P members can re-register within the grace window, but the JavaScript `SignalingChannel` does not yet auto-reopen a closed socket.
- **Production hardening remains.** The private gRPC plane uses server-authenticated TLS plus firewalling today; mTLS identities for the AppWeb/SFU roles and browser `Origin` validation are the next security items.

And on the media side, the `sfu` roadmap is unchanged: simulcast, an explicit publish/subscribe model, TWCC, and per-hop RTX/NACK.

If wiring a Sans-I/O signaling authority to a Sans-I/O SFU sounds like your kind of problem, the boundaries are clean and the tests need no network — a very good place to jump in.

---

## Get Involved

- **Try it**: open [https://appr.tc](https://appr.tc) and turn a 1:1 call into a conference
- **Break it**: browser interop, flaky networks, fast rejoin/leave races — we want the reports
- **Build it**: SFU→P2P downgrade, browser reconnect, mTLS, and per-room capacity control are all open and well-scoped
- **Discuss it**: tell us what a P2P/SFU signaling server needs to be useful to *you*

Join our community:

- **GitHub**: https://github.com/webrtc-rs/apprtc
- **Discord**: https://discord.gg/4Ju8UHdXMs
- **Discussions**: https://github.com/webrtc-rs/webrtc/discussions

---

## Conclusion

`appr.tc` is now a **P2P/SFU signaling server** built the way we build everything since the Sans-I/O shift:

- **Two is a call, three is a conference** — direct P2P for 1:1, automatic upgrade to the SFU on the third joiner
- **One authority** — `signaling` owns room state; `appweb` is a thin HTTP front door; the SFU owns only its projection
- **A Sans-I/O core** — `Collider` → `RoomTable` → `Room` → `Client`, no sockets/threads/clock, testable in memory
- **Race-free transitions** — a real `Upgrading` state, a three-member join barrier, and signal epochs
- **One app, two layouts** — the grid replaces the full-screen remote view; self-view and controls never move

The Sans-I/O core made the async `webrtc` crate possible, then the `sfu` server, and now a signaling authority that ties them into a real group call. 🦀

---

## Links

- **GitHub**: https://github.com/webrtc-rs/apprtc
- **Live demo**: https://appr.tc
- **sfu (media engine)**: https://github.com/webrtc-rs/sfu — live demo at https://sfu.rs
- **rtc (Sans-I/O core)**: https://github.com/webrtc-rs/rtc
- **webrtc (async)**: https://github.com/webrtc-rs/webrtc
- **Discord**: https://discord.gg/4Ju8UHdXMs

---

## Further Reading

- [Announcing `sfu` v0.20.0-rc.3: A Sans-I/O Selective Forwarding Unit in Rust](/blog/2026/07/13/announcing-sfu-v0.20.0-rc.3)
- [Perfect Negotiation in WebRTC: A Deep Dive](/blog/2026/01/23/perfect-negotiation-webrtc-deep-dive)
- [WebRTC v0.20.0-rc.1: Toward Stable Async-Friendly WebRTC Built on Sans-I/O](/blog/2026/06/30/webrtc-v0.20.0-rc.1-toward-stable-async-webrtc)
