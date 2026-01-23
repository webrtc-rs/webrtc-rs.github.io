# Perfect Negotiation in WebRTC: A Deep Dive into `rtc` Implementation

## Introduction

WebRTC's offer/answer negotiation model is inherently asymmetric: one peer must initiate as the "offerer" while the other responds as the "answerer." This asymmetry creates complexity in peer-to-peer applications where either side might want to initiate renegotiation, and it becomes particularly problematic when both peers attempt to send offers simultaneously—a scenario known as **"glare"**.

**Perfect Negotiation** is a design pattern introduced by the W3C WebRTC Working Group that elegantly solves these problems. It makes negotiation appear symmetric from the application's perspective while handling edge cases like glare automatically behind the scenes.

In this deep dive, we'll explore:
- What Perfect Negotiation is and why it matters
- How it's implemented in browser-based WebRTC
- The primitives required to support Perfect Negotiation
- How `rtc` implements these primitives at the sans-I/O level
- A complete working example demonstrating the pattern
- Lessons learned and bugs discovered during implementation

---

## Understanding Perfect Negotiation

### The Problem: Asymmetric Negotiation

Traditional WebRTC applications have asymmetric roles:

```javascript
// Offerer side (Client A)
const offer = await pc.createOffer();
await pc.setLocalDescription(offer);
sendToServer(offer);

// Answerer side (Client B)  
const offer = await receiveFromServer();
await pc.setRemoteDescription(offer);
const answer = await pc.createAnswer();
await pc.setLocalDescription(answer);
sendToServer(answer);
```

This creates several problems:

1. **Role Coordination**: Applications must decide who calls whom before connecting
2. **Bidirectional Calling**: Hard to support "either peer can initiate"
3. **Renegotiation**: Only one peer can trigger renegotiation safely
4. **Glare Handling**: If both peers send offers simultaneously, one must be rejected

### The Solution: Perfect Negotiation Pattern

Perfect Negotiation eliminates these issues by:

#### 1. **Symmetric Application Code**

Both peers run identical code:

```javascript
// Same code on BOTH peers!
pc.onnegotiationneeded = async () => {
  makingOffer = true;
  await pc.setLocalDescription();
  signaler.send({ description: pc.localDescription });
  makingOffer = false;
};

signaler.onmessage = async ({ data: { description } }) => {
  const offerCollision = description.type === 'offer' && 
    (makingOffer || pc.signalingState !== 'stable');
  
  ignoreOffer = !polite && offerCollision;
  if (ignoreOffer) return;
  
  await pc.setRemoteDescription(description);
  if (description.type === 'offer') {
    await pc.setLocalDescription();
    signaler.send({ description: pc.localDescription });
  }
};
```

#### 2. **Automatic Glare Resolution**

When collision occurs:
- **Polite peer**: Backs off by rolling back its offer
- **Impolite peer**: Ignores the colliding offer and continues

#### 3. **Assigned Roles**

Despite symmetric code, peers have roles:
- **Polite**: Yields during collisions ("after you")
- **Impolite**: Wins collisions ("no, after YOU")

Role assignment is arbitrary (e.g., connection order) and doesn't affect functionality.

### Key Mechanisms

The pattern relies on four key mechanisms:

#### 1. **`makingOffer` Flag**

Tracks whether we're currently creating an offer:

```javascript
let makingOffer = false;

pc.onnegotiationneeded = async () => {
  makingOffer = true;
  try {
    await pc.setLocalDescription();
    signaler.send({ description: pc.localDescription });
  } finally {
    makingOffer = false;  // Always clear, even on error
  }
};
```

#### 2. **Collision Detection**

Determines if incoming offer collides with our state:

```javascript
const readyForOffer = 
  !makingOffer &&
  (pc.signalingState === "stable" || isSettingRemoteAnswerPending);

const offerCollision = description.type === "offer" && !readyForOffer;
```

A collision occurs when:
- We receive an offer, AND
- We're currently making an offer OR not in stable state

#### 3. **Rollback (Polite Peer)**

The polite peer automatically rolls back when accepting a colliding offer:

```javascript
// In browser, rollback is implicit when setting remote offer
// while in HaveLocalOffer state
await pc.setRemoteDescription(incomingOffer);
// ^ This automatically rolls back our pending local offer
```

#### 4. **Ignore Strategy (Impolite Peer)**

The impolite peer simply ignores colliding offers:

```javascript
ignoreOffer = !polite && offerCollision;
if (ignoreOffer) {
  return;  // Don't process this offer
}

// Also ignore ICE candidates for ignored offers
if (!ignoreOffer) {
  await pc.addIceCandidate(candidate);
}
```

### State Transitions

Perfect Negotiation leverages these signaling state transitions:

```
Normal offer/answer:
Stable → HaveLocalOffer → Stable → HaveRemoteOffer → Stable

Polite peer rollback (collision):
HaveLocalOffer → Stable (via SetLocal rollback)
              → HaveRemoteOffer → Stable

Offer rejection:
HaveRemoteOffer → Stable (via SetRemote rollback)
```

---

## WebRTC Primitives Required for Perfect Negotiation

To implement Perfect Negotiation, a WebRTC stack must provide:

### 1. Rollback SDP Type

Per RFC 8829 §5.7, rollback is a special SDP type that:

- Returns signaling state to Stable
- Discards pending local/remote description
- Typically has empty SDP content

```javascript
// Browser API
await pc.setLocalDescription({ type: 'rollback' });
```

### 2. Signaling State Machine

Must support these rollback transitions:

| From State | Operation | SDP Type | To State |
|------------|-----------|----------|----------|
| HaveLocalOffer | SetLocal | Rollback | Stable |
| HaveRemoteOffer | SetRemote | Rollback | Stable |
| Stable | SetLocal/SetRemote | Rollback | Error |

### 3. Negotiation Needed Event

Fires when negotiation is required:

```javascript
pc.onnegotiationneeded = () => {
  // Create and send offer
};
```

Must follow W3C spec algorithm to avoid spurious events.

### 4. Signaling State Access

Applications need to query current state:

```javascript
if (pc.signalingState === "stable") {
  // Safe to create offer
}
```

---

## How `rtc` Implements the Primitives

Now let's examine how our sans-I/O Rust implementation provides these primitives.

### ✅ Rollback SDP Type

The library explicitly defines rollback:

```rust
// rtc/src/peer_connection/sdp/sdp_type.rs
pub enum RTCSdpType {
    Offer,
    Pranswer,
    Answer,
    /// Rollback moves the SDP offer and answer back to what they were
    /// in the last stable state. This is useful for recovering from
    /// glare (both peers send offers simultaneously).
    Rollback,
}
```

Constructor for creating rollback descriptions:

```rust
// rtc/src/peer_connection/sdp/session_description.rs
impl RTCSessionDescription {
    /// Creates a rollback session description.
    ///
    /// Per WebRTC specification (RFC 8829 §5.7), rollback descriptions
    /// typically have empty SDP content. This is used to abort an in-progress
    /// negotiation, such as when implementing Perfect Negotiation collision
    /// resolution.
    pub fn rollback(sdp: Option<String>) -> Result<RTCSessionDescription> {
        let mut desc = RTCSessionDescription {
            sdp: if let Some(sdp) = sdp {
                sdp
            } else {
                "".to_string()
            },
            sdp_type: RTCSdpType::Rollback,
            parsed: None,
        };

        if !desc.sdp.is_empty() {
            let parsed = desc.unmarshal()?;
            desc.parsed = Some(parsed);
        }

        Ok(desc)
    }
}
```

### ✅ Signaling State Machine

Complete state machine implementation:

```rust
// rtc/src/peer_connection/state/signaling_state.rs
pub fn check_next_signaling_state(
    cur: RTCSignalingState,
    sdp_type: RTCSdpType,
    op: StateChangeOp,
    local_description_set: bool,
    remote_description_set: bool,
) -> Result<RTCSignalingState> {
    // Rollback transitions
    if sdp_type == RTCSdpType::Rollback {
        match cur {
            RTCSignalingState::HaveLocalOffer => {
                if op == StateChangeOp::SetLocal {
                    return Ok(RTCSignalingState::Stable);
                }
            }
            RTCSignalingState::HaveRemoteOffer => {
                if op == StateChangeOp::SetRemote {
                    return Ok(RTCSignalingState::Stable);
                }
            }
            RTCSignalingState::Stable => {
                return Err(Error::ErrSignalingStateCannotRollback);
            }
            // ... other states
        }
    }
    
    // Normal offer/answer transitions
    match (cur, sdp_type, op) {
        // Stable → HaveLocalOffer (offer created)
        (RTCSignalingState::Stable, RTCSdpType::Offer, StateChangeOp::SetLocal) 
            if !local_description_set => {
            Ok(RTCSignalingState::HaveLocalOffer)
        }
        
        // HaveLocalOffer → Stable (answer received)
        (RTCSignalingState::HaveLocalOffer, RTCSdpType::Answer, StateChangeOp::SetRemote)
            => Ok(RTCSignalingState::Stable),
        
        // ... other transitions
    }
}
```

### ✅ Negotiation Needed Event

Complete implementation following W3C spec:

```rust
// rtc/src/peer_connection/internal.rs
pub(super) fn trigger_negotiation_needed(&mut self) {
    if !self.do_negotiation_needed() {
        return;
    }
    let _ = self.negotiation_needed_op();
}

fn do_negotiation_needed(&mut self) -> bool {
    // https://w3c.github.io/webrtc-pc/#updating-the-negotiation-needed-flag
    
    if self.negotiation_needed_state == NegotiationNeededState::Run {
        self.negotiation_needed_state = NegotiationNeededState::Queue;
        return false;
    }
    
    if self.signaling_state != RTCSignalingState::Stable {
        return false;
    }
    
    // Check various conditions per spec...
    // Returns true if negotiation is actually needed
}
```

### ✅ Sans-I/O Architecture Benefits

The sans-I/O design provides complete control:

```rust
// Applications poll events explicitly
while let Some(event) = pc.poll_event() {
    match event {
        RTCPeerConnectionEvent::OnNegotiationNeeded => {
            // Handle when ready
        }
        // ... other events
    }
}

// Applications control when to apply descriptions
let offer = pc.create_offer(None)?;
pc.set_local_description(offer)?;

// Applications manage signaling
let state = pc.signaling_state();
if state == RTCSignalingState::Stable {
    // Safe to create offer
}
```

This flexibility is **perfect** for implementing custom negotiation strategies.

---

## Building Perfect Negotiation at the Application Level

With the primitives in place, let's build Perfect Negotiation at the application level.

### PerfectNegotiationHandler

Wrapper around RTCPeerConnection implementing the pattern:

```rust
pub struct PerfectNegotiationHandler {
    pc: RTCPeerConnection,
    polite: bool,
    is_making_offer: bool,
    ignore_offer: bool,
    signaling_state: RTCSignalingState,
    auto_answer: bool,
    suppress_negotiation_needed: bool,
}

impl PerfectNegotiationHandler {
    pub fn new(pc: RTCPeerConnection, polite: bool) -> Self {
        Self {
            pc,
            polite,
            is_making_offer: false,
            ignore_offer: false,
            signaling_state: RTCSignalingState::Stable,
            auto_answer: true,
            suppress_negotiation_needed: false,
        }
    }
}
```

### Collision Detection

Implement the readyForOffer logic:

```rust
impl PerfectNegotiationHandler {
    pub fn handle_remote_description_with_response(
        &mut self,
        description: RTCSessionDescription,
        local_addr: SocketAddr,
    ) -> Result<Option<RTCSessionDescription>> {
        let role = if self.polite { "POLITE" } else { "IMPOLITE" };
        
        // Detect offer collision
        let offer_collision = description.sdp_type == RTCSdpType::Offer &&
            (self.is_making_offer || self.signaling_state != RTCSignalingState::Stable);
        
        // Impolite peer ignores colliding offers
        self.ignore_offer = !self.polite && offer_collision;
        if self.ignore_offer {
            info!("[{}] Ignoring colliding offer", role);
            return Ok(None);
        }
        
        // Polite peer rolls back on collision
        if offer_collision && self.polite {
            warn!("[{}] Collision detected, rolling back local offer", role);
            
            let mut rollback = RTCSessionDescription::default();
            rollback.sdp_type = RTCSdpType::Rollback;
            rollback.sdp = String::new();
            
            // Rollback: HaveLocalOffer → Stable
            self.pc.set_local_description(rollback)?;
            self.signaling_state = RTCSignalingState::Stable;
            info!("[{}] Rolled back to Stable", role);
        }
        
        // Accept the remote description
        self.pc.set_remote_description(description.clone())?;
        self.signaling_state = self.pc.signaling_state();
        
        // Auto-create answer if this was an offer
        if description.sdp_type == RTCSdpType::Offer && self.auto_answer {
            info!("[{}] Creating answer", role);
            let answer = self.pc.create_answer(None)?;
            self.pc.set_local_description(answer.clone())?;
            
            // Add local ICE candidate after setting description
            self.add_local_host_candidate(local_addr)?;
            
            self.signaling_state = RTCSignalingState::Stable;
            return Ok(Some(answer));
        }
        
        Ok(None)
    }
}
```

### Offer Rejection

Support rejecting offers to test SetRemote(rollback):

```rust
impl PerfectNegotiationHandler {
    /// Reject a remote offer by rolling back to stable
    pub fn reject_remote_offer(&mut self) -> Result<()> {
        let role = if self.polite { "POLITE" } else { "IMPOLITE" };
        
        if self.signaling_state != RTCSignalingState::HaveRemoteOffer {
            warn!("[{}] Cannot reject - not in HaveRemoteOffer state", role);
            return Ok(());
        }
        
        info!("[{}] Rejecting remote offer via SetRemote(rollback)", role);
        
        let mut rollback = RTCSessionDescription::default();
        rollback.sdp_type = RTCSdpType::Rollback;
        rollback.sdp = String::new();
        
        // Reject: HaveRemoteOffer → Stable
        self.pc.set_remote_description(rollback)?;
        self.signaling_state = RTCSignalingState::Stable;
        
        info!("[{}] Successfully rejected offer, back to Stable", role);
        Ok(())
    }
}
```

### ICE Candidate Filtering

Ignore candidates for ignored offers:

```rust
impl PerfectNegotiationHandler {
    pub fn handle_remote_candidate(&mut self, candidate: RTCIceCandidateInit) -> Result<()> {
        if self.ignore_offer {
            trace!("Ignoring ICE candidate (ignoring offer)");
            return Ok(());
        }
        
        self.pc.add_remote_candidate(candidate)?;
        Ok(())
    }
}
```

---

## A Complete Working Example

Location: `examples/examples/perfect-negotiation/`

### Architecture

**Rust ↔ Rust** peer-to-peer with browser UI:

```
┌──────────┐                              ┌──────────┐
│ Browser  │ ◄─────── WebSocket ──────────┤  Polite  │
│  (UI)    │   (commands/status only)     │   Peer   │
└──────────┘                              └─────┬────┘
                                                │
                                          Relay Channel
                                          (mpsc channel)
                                                │
┌──────────┐                              ┌─────┴────┐
│ Browser  │ ◄─────── WebSocket ──────────┤ Impolite │
│  (UI)    │   (commands/status only)     │   Peer   │
└──────────┘                              └──────────┘
```

Both WebRTC peers run in Rust. Browsers are thin clients.

### Main Event Loop

```rust
async fn run_peer(
    role: &str,
    polite: bool,
    local_addr: SocketAddr,
    mut ws: WebSocketStream<TcpStream>,
    peer_tx: mpsc::Sender<String>,
    mut peer_rx: mpsc::Receiver<String>,
) -> Result<()> {
    let config = RTCConfiguration::default();
    let pc = RTCPeerConnection::new(config)?;
    let mut negotiation = PerfectNegotiationHandler::new(pc, polite);
    
    let udp_socket = UdpSocket::bind(local_addr).await?;
    let mut data_channel_created = false;
    
    loop {
        tokio::select! {
            // Handle peer connection timeouts
            _ = tokio::time::sleep(sleep_duration) => {
                negotiation.peer_connection().handle_timeout(Instant::now()).ok();
                
                // Poll events
                while let Some(event) = negotiation.peer_connection().poll_event() {
                    match event {
                        RTCPeerConnectionEvent::OnNegotiationNeeded => {
                            if negotiation.suppress_negotiation_needed {
                                info!("[{}] Ignoring negotiation (suppressed)", role);
                                continue;
                            }
                            
                            if !data_channel_created {
                                info!("[{}] Ignoring negotiation (no data channel)", role);
                                continue;
                            }
                            
                            // Create offer
                            negotiation.is_making_offer = true;
                            let offer = negotiation.peer_connection().create_offer(None)?;
                            negotiation.peer_connection().set_local_description(offer.clone())?;
                            negotiation.add_local_host_candidate(local_addr)?;
                            negotiation.signaling_state = RTCSignalingState::HaveLocalOffer;
                            negotiation.is_making_offer = false;
                            
                            // Send via relay
                            let msg = SignalingMessage::Description { description: offer };
                            peer_tx.send(serde_json::to_string(&msg)?).await?;
                        }
                        
                        RTCPeerConnectionEvent::OnIceCandidateEvent(event) => {
                            let msg = SignalingMessage::Candidate { 
                                candidate: event.candidate.to_json()? 
                            };
                            peer_tx.send(serde_json::to_string(&msg)?).await?;
                        }
                        
                        // ... other events
                    }
                }
            }
            
            // Handle WebSocket commands from browser
            Some(msg) = ws.next() => {
                if let Message::Text(text) = msg? {
                    match text.as_str() {
                        "connect" => {
                            negotiation.suppress_negotiation_needed = false;
                            if !data_channel_created {
                                let label = format!("data-{}", role.to_lowercase());
                                negotiation.peer_connection().create_data_channel(&label, None)?;
                                data_channel_created = true;
                            }
                        }
                        
                        "reject" => {
                            negotiation.reject_remote_offer()?;
                            
                            // Send reject notification to other peer
                            let msg = SignalingMessage::Reject;
                            peer_tx.send(serde_json::to_string(&msg)?).await?;
                        }
                        
                        // ... other commands
                    }
                }
            }
            
            // Handle signaling messages from other peer
            Some(peer_msg) = peer_rx.recv() => {
                let signal_msg: SignalingMessage = serde_json::from_str(&peer_msg)?;
                
                match signal_msg {
                    SignalingMessage::Description { description } => {
                        let response = negotiation.handle_remote_description_with_response(
                            description,
                            local_addr,
                        )?;
                        
                        if let Some(answer) = response {
                            let msg = SignalingMessage::Description { description: answer };
                            peer_tx.send(serde_json::to_string(&msg)?).await?;
                        }
                    }
                    
                    SignalingMessage::Candidate { candidate } => {
                        negotiation.handle_remote_candidate(candidate)?;
                    }
                    
                    SignalingMessage::Reject => {
                        // Other peer rejected our offer, rollback
                        if negotiation.signaling_state == RTCSignalingState::HaveLocalOffer {
                            let mut rollback = RTCSessionDescription::default();
                            rollback.sdp_type = RTCSdpType::Rollback;
                            rollback.sdp = String::new();
                            
                            negotiation.peer_connection().set_local_description(rollback)?;
                            negotiation.signaling_state = RTCSignalingState::Stable;
                            negotiation.suppress_negotiation_needed = true;
                        }
                    }
                }
            }
            
            // Handle UDP packets (DTLS/SRTP/SCTP)
            res = udp_socket.recv_from(&mut buf) => {
                if let Ok((n, src_addr)) = res {
                    negotiation.peer_connection().handle_read(TaggedBytesMut {
                        now: Instant::now(),
                        transport: TransportContext {
                            local_addr,
                            peer_addr: src_addr,
                            ecn: None,
                            transport_protocol: TransportProtocol::UDP,
                        },
                        message: &mut buf[..n],
                    })?;
                }
            }
        }
    }
}
```

### Running the Example

```bash
cd examples
cargo run --example perfect-negotiation
```

**Test Scenarios:**

1. **Normal Connection**
   - Open http://localhost:8080/polite and http://localhost:8080/impolite
   - Click "Connect" in either tab
   - Observe automatic negotiation

2. **Collision Testing**
   - Enable "Manual Mode" in both tabs
   - Click "Connect" in both tabs quickly (within ~100ms)
   - Observe collision detection and rollback in logs
   - Polite peer rolls back, impolite wins

3. **Offer Rejection**
   - Enable "Manual Mode" in impolite tab
   - Click "Connect" in polite tab
   - Click "Reject Offer" in impolite tab
   - Observe SetRemote(rollback) transition
   - Both peers return to Stable state

### Log Output Examples

**Successful Collision Resolution:**

```
[POLITE] Creating offer...
[IMPOLITE] Creating offer...
[POLITE] Received remote Offer description
[POLITE] Collision detected, rolling back local offer
signaling state changed to stable
[POLITE] Rolled back to Stable
[POLITE] Creating answer
[IMPOLITE] Received remote Answer description
signaling state changed to stable
```

**Offer Rejection:**

```
[POLITE] Sending Offer
[IMPOLITE] Received remote Offer description
[IMPOLITE] Rejecting remote offer via SetRemote(rollback)
signaling state changed to stable
[IMPOLITE] Successfully rejected offer, back to Stable
[IMPOLITE] Sent reject notification to other peer
[POLITE] Received reject notification
[POLITE] Rolling back our local offer via SetLocal(rollback)
signaling state changed to stable
[POLITE] Successfully rolled back to Stable state
```

---

## Design Philosophy: Why Application-Level?

### Why Application-Level?

The `rtc` library implements Perfect Negotiation as an **application-level pattern** rather than built-in library feature. This is intentional:

#### Library Focus: Spec-Compliant Primitives

- ✅ `RTCSdpType::Rollback` type
- ✅ Complete signaling state machine
- ✅ Rollback state transitions
- ✅ `OnNegotiationNeeded` event
- ✅ Sans-I/O event-driven architecture

**No library changes needed** for Perfect Negotiation—the primitives are complete and spec-compliant.

#### Application Flexibility

Different applications have different needs:

- **Signaling architecture**: Some use WebSocket, others use WebRTC data channels, HTTP long-polling, etc.
- **Collision resolution**: Some might want custom policies beyond polite/impolite
- **Role determination**: Connection order, user interaction, server coordination
- **Error handling**: Retry strategies, fallbacks, logging

Keeping Perfect Negotiation at the application level allows full customization.

#### Benefits

1. **Focused library**: Core stays focused on WebRTC protocol
2. **Zero bloat**: Applications only pay for what they use
3. **Testability**: Applications can mock/test their negotiation logic
4. **Flexibility**: Custom collision resolution strategies
5. **Clarity**: Clear separation between protocol and application logic

### Comparison with Browser APIs

Browsers implement rollback **implicitly**:

```javascript
// Browser: implicit rollback when setting remote offer in HaveLocalOffer state
await pc.setRemoteDescription(incomingOffer);
// ^ Automatically rolls back pending local offer
```

Our sans-I/O implementation requires **explicit** rollback:

```rust
// rtc: explicit rollback before accepting remote offer
if collision && polite {
    let rollback = RTCSessionDescription::default();
    rollback.sdp_type = RTCSdpType::Rollback;
    pc.set_local_description(rollback)?;
}
pc.set_remote_description(incoming_offer)?;
```

This explicitness:
- ✅ Makes state transitions visible
- ✅ Gives applications full control
- ✅ Aligns with sans-I/O philosophy
- ✅ Easier to test and debug

---

## Testing Both Rollback Transitions

The example provides comprehensive test coverage for both RFC 8829 rollback transitions.

### Transition 1: HaveLocalOffer → Stable (SetLocal with rollback)

**Use case**: Polite peer rolling back on collision

**Test steps**:
1. Both peers create offers simultaneously
2. Polite peer receives impolite's offer
3. Collision detected (`is_making_offer || state != Stable`)
4. Polite calls `set_local_description(rollback)`
5. State: HaveLocalOffer → Stable
6. Polite accepts impolite's offer
7. Negotiation continues normally

**Verification**:
```rust
assert_eq!(pc.signaling_state(), RTCSignalingState::HaveLocalOffer);
pc.set_local_description(rollback)?;
assert_eq!(pc.signaling_state(), RTCSignalingState::Stable);
```

### Transition 2: HaveRemoteOffer → Stable (SetRemote with rollback)

**Use case**: Rejecting an incoming offer

**Test steps**:
1. Peer A sends offer
2. Peer B receives offer → HaveRemoteOffer
3. User clicks "Reject Offer" button
4. Peer B calls `set_remote_description(rollback)`
5. State: HaveRemoteOffer → Stable
6. Reject notification sent to Peer A
7. Peer A also rolls back to Stable

**Verification**:
```rust
assert_eq!(pc.signaling_state(), RTCSignalingState::HaveRemoteOffer);
pc.set_remote_description(rollback)?;
assert_eq!(pc.signaling_state(), RTCSignalingState::Stable);
```

Both transitions are exercised by the example's Manual Mode feature.

---

## Performance Implications

Perfect Negotiation has **minimal overhead**:

### Memory

```rust
struct PerfectNegotiationHandler {
    pc: RTCPeerConnection,      // Same as before
    polite: bool,                // 1 byte
    is_making_offer: bool,       // 1 byte
    ignore_offer: bool,          // 1 byte
    signaling_state: RTCSignalingState,  // 1 byte
    auto_answer: bool,           // 1 byte
    suppress_negotiation_needed: bool,   // 1 byte
}
// Total added: ~6 bytes (plus alignment)
```

### CPU

Collision detection:

```rust
let offer_collision = description.sdp_type == RTCSdpType::Offer &&
    (self.is_making_offer || self.signaling_state != RTCSignalingState::Stable);
// Two comparisons + one boolean operation
// ~3 CPU cycles on modern hardware
```

### Network

- **No extra signaling messages** in non-collision case
- **Same number of round trips** as traditional offer/answer
- Rollback happens locally (no network traffic)

### Collision Cost

Only overhead during actual collisions:

1. Create rollback description (stack allocation)
2. Call `set_local_description(rollback)` (~microseconds)
3. Continue with normal negotiation

Collisions are rare (requires microsecond-level timing coincidence).

---

## Conclusion

Perfect Negotiation represents the modern, spec-compliant approach to WebRTC negotiation. Key takeaways:

### What We've Covered

1. **Pattern Mechanics**: How Perfect Negotiation eliminates asymmetry
2. **Required Primitives**: Rollback, state machine, negotiation events
3. **Implementation Details**: Complete working code with collision detection
4. **Design Philosophy**: Why application-level is the right approach

### What `rtc` Provides

1. ✅ **Complete WebRTC primitives** for Perfect Negotiation
2. ✅ **Spec-compliant rollback** support
3. ✅ **Working example** demonstrating the pattern
4. ✅ **Comprehensive documentation** of rollback API
5. ✅ **Sans-I/O flexibility** for custom implementations

### For Application Developers

You can now:

1. Copy the `PerfectNegotiationHandler` pattern
2. Customize for your signaling architecture
3. Build symmetric P2P applications
4. Handle bidirectional calling gracefully
5. Test collision scenarios with confidence

### Looking Forward

Perfect Negotiation is now a proven, production-ready pattern in `rtc`. The sans-I/O architecture makes it straightforward to implement while maintaining full control over I/O, threading, and application logic.

As WebRTC evolves toward more peer-to-peer use cases (gaming, collaboration, IoT), patterns like Perfect Negotiation become increasingly important. The `rtc` library provides the foundation—applications build the experience.

---

## References

### Specifications

- [RFC 8829 (JSEP) §5.7: Rollback](https://datatracker.ietf.org/doc/html/rfc8829#section-5.7)
- [W3C WebRTC 1.0: Perfect Negotiation Example](https://www.w3.org/TR/webrtc/#perfect-negotiation-example)
- [W3C WebRTC 1.0 §4.4.1.6: Set the RTCSessionDescription](https://www.w3.org/TR/webrtc/#dom-peerconnection-setlocaldescription)

### Learning Resources

- [MDN: Perfect Negotiation](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Perfect_negotiation)
- [MDN: Signaling and Video Calling](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Signaling_and_video_calling)

### Code

- [webrtc-rs/rtc Repository](https://github.com/webrtc-rs/rtc)
- Example: `examples/examples/perfect-negotiation/`
