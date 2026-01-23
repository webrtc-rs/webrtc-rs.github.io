# Perfect Negotiation in WebRTC: A Deep Dive into webrtc-rs/rtc Implementation

## Introduction

WebRTC's offer/answer negotiation model is inherently asymmetric: one peer must initiate as the "caller" (offerer) while the other responds as the "callee" (answerer). This asymmetry creates complexity in peer-to-peer applications where either side might want to initiate renegotiation, and it becomes particularly problematic when both peers attempt to send offers simultaneously—a scenario known as "glare."

**Perfect Negotiation** is a design pattern introduced by the W3C WebRTC Working Group that elegantly solves these problems. It makes negotiation appear symmetric from the application's perspective while handling edge cases like glare automatically behind the scenes.

In this post, we'll explore:
- What Perfect Negotiation is and why it matters
- How it's typically implemented in browser-based WebRTC
- The current state of Perfect Negotiation support in webrtc-rs/rtc (our sans-I/O Rust implementation)
- Practical recommendations for filling the gaps

## What is Perfect Negotiation?

Perfect Negotiation is a negotiation pattern that:

### 1. **Separates Negotiation Logic from Application Logic**

Instead of your application needing to care about whether it's the caller or callee, Perfect Negotiation handles all the complexity internally. Your application code can be completely symmetric—the same code runs on both peers.

### 2. **Handles Glare Gracefully**

When both peers simultaneously send offers (glare), Perfect Negotiation resolves the collision automatically:
- The **polite peer** uses ICE rollback to withdraw its offer and accept the other peer's offer instead
- The **impolite peer** ignores the incoming colliding offer and continues with its own

### 3. **Assigns Asymmetric Roles**

Despite the symmetric application code, peers are assigned roles:
- **Polite peer**: Backs off during collisions, essentially saying "never mind my offer, I'll accept yours"
- **Impolite peer**: Wins all collisions by ignoring incoming offers

The role assignment is typically arbitrary (e.g., based on connection order) and doesn't affect functionality—it just determines who yields during glare.

## Perfect Negotiation Pattern (Browser Implementation)

Here's the canonical Perfect Negotiation pattern from MDN and the W3C spec:

```javascript
// Configuration
const polite = /* determine which peer is polite */;
let makingOffer = false;
let ignoreOffer = false;
let isSettingRemoteAnswerPending = false;

// Handle negotiation needed
pc.onnegotiationneeded = async () => {
  try {
    makingOffer = true;
    await pc.setLocalDescription();
    signaler.send({ description: pc.localDescription });
  } catch (err) {
    console.error(err);
  } finally {
    makingOffer = false;
  }
};

// Handle incoming signaling messages
signaler.onmessage = async ({ data: { description, candidate } }) => {
  try {
    if (description) {
      // Detect offer collision
      const readyForOffer =
        !makingOffer &&
        (pc.signalingState === "stable" || isSettingRemoteAnswerPending);
      const offerCollision = description.type === "offer" && !readyForOffer;

      // Polite peer backs off, impolite peer ignores
      ignoreOffer = !polite && offerCollision;
      if (ignoreOffer) {
        return;
      }
      
      // Set remote description (triggers rollback if needed)
      isSettingRemoteAnswerPending = description.type === "answer";
      await pc.setRemoteDescription(description);
      isSettingRemoteAnswerPending = false;
      
      // Create answer if we received an offer
      if (description.type === "offer") {
        await pc.setLocalDescription();
        signaler.send({ description: pc.localDescription });
      }
    } else if (candidate) {
      try {
        await pc.addIceCandidate(candidate);
      } catch (err) {
        if (!ignoreOffer) throw err;
      }
    }
  } catch (err) {
    console.error(err);
  }
};
```

### Key Mechanisms

1. **`makingOffer` flag**: Tracks whether we're currently in the process of making an offer
2. **Collision detection**: Checks if incoming offer arrives while we're not ready
3. **Rollback**: The polite peer automatically rolls back its offer when setting a remote offer (implicit in `setRemoteDescription`)
4. **Ignore strategy**: The impolite peer ignores colliding offers and their ICE candidates

## Analyzing webrtc-rs/rtc Implementation

Now let's examine how our sans-I/O Rust implementation stacks up against the Perfect Negotiation pattern.

### ✅ Primitives Present

The good news: **webrtc-rs/rtc has all the necessary low-level primitives** for Perfect Negotiation.

#### 1. Rollback Support

```rust
// From rtc/src/peer_connection/sdp/sdp_type.rs
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

The library explicitly mentions rollback's purpose for handling glare. The signaling state machine properly handles rollback transitions:

```rust
// Rollback transitions
if sdp_type == RTCSdpType::Rollback {
    match cur {
        RTCSignalingState::HaveLocalOffer => {
            return Ok(RTCSignalingState::Stable);
        }
        RTCSignalingState::HaveRemoteOffer => {
            return Ok(RTCSignalingState::Stable);
        }
        RTCSignalingState::Stable => {
            return Err(Error::ErrSignalingStateCannotRollback);
        }
        // ... other states
    }
}
```

#### 2. Signaling State Machine

The library has a complete signaling state machine with proper validation:

```rust
pub fn check_next_signaling_state(
    cur: RTCSignalingState,
    sdp_type: RTCSdpType,
    local_description_set: bool,
    remote_description_set: bool,
) -> Result<RTCSignalingState>
```

All W3C-specified transitions are implemented and validated.

#### 3. Negotiation Needed Events

```rust
// From rtc/src/peer_connection/internal.rs
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
    // ... W3C spec algorithm implementation
}
```

The library follows the W3C specification for negotiation-needed flag management.

#### 4. Sans-I/O Architecture

The sans-I/O design means applications have complete control over:
- When to call `create_offer()` / `create_answer()`
- When to apply descriptions via `set_local_description()` / `set_remote_description()`
- How to handle signaling messages

This flexibility is **perfect** for implementing custom negotiation strategies like Perfect Negotiation.

### ❌ Missing High-Level Abstractions

However, the library **does not provide built-in Perfect Negotiation support**:

#### 1. No Collision Detection

There's no tracking of:
- `makingOffer` state
- `readyForOffer` calculation
- `offerCollision` detection

Applications must implement this logic themselves.

#### 2. No Automatic Rollback on Collision

The library won't automatically rollback the polite peer's offer. Applications must:
1. Detect collisions
2. Manually call `set_local_description()` with `RTCSdpType::Rollback`
3. Then apply the incoming offer

#### 3. No Role Management

There's no concept of "polite" vs "impolite" peers. Applications choose their own strategy.

#### 4. No Helper Methods

Missing convenience methods like:
```rust
// Hypothetical helpers
pub fn is_making_offer(&self) -> bool;
pub fn can_accept_offer(&self) -> bool;
pub fn handle_offer_collision(&mut self, polite: bool) -> Result<()>;
```

### 🔍 Current Examples

Current examples in the repository: **None implement Perfect Negotiation.** They all use the simple offerer/answerer model:

```rust
// Typical pattern in examples (answerer side only)
WsMessage::Offer(offer) => {
    println!("Received offer, creating peer connection");
    
    let mut pc = RTCPeerConnection::new(config)?;
    pc.set_remote_description(offer)?;
    
    let answer = pc.create_answer(None)?;
    pc.set_local_description(answer.clone())?;
    
    // Send answer to browser
    ws.send(Message::Text(serde_json::to_string(&answer)?)).await?;
}
```

There's no:
- Bidirectional calling support
- Collision handling
- Rollback usage
- Symmetric peer code

## Implementing Perfect Negotiation with webrtc-rs/rtc

Despite the missing abstractions, Perfect Negotiation **can be implemented** at the application level. Here's how:

### Application-Level Implementation

```rust
use rtc::peer_connection::{RTCPeerConnection, RTCSessionDescription};
use rtc::peer_connection::sdp::RTCSdpType;
use rtc::peer_connection::state::RTCSignalingState;

pub struct PerfectNegotiationHandler {
    pc: RTCPeerConnection,
    polite: bool,
    making_offer: bool,
    ignore_offer: bool,
    is_setting_remote_answer_pending: bool,
}

impl PerfectNegotiationHandler {
    pub fn new(pc: RTCPeerConnection, polite: bool) -> Self {
        Self {
            pc,
            polite,
            making_offer: false,
            ignore_offer: false,
            is_setting_remote_answer_pending: false,
        }
    }
    
    /// Call this when negotiation is needed
    pub async fn handle_negotiation_needed(
        &mut self,
        send_description: impl Fn(RTCSessionDescription) -> Result<()>,
    ) -> Result<()> {
        self.making_offer = true;
        
        let result = async {
            let offer = self.pc.create_offer(None)?;
            self.pc.set_local_description(offer.clone())?;
            send_description(offer)?;
            Ok(())
        }.await;
        
        self.making_offer = false;
        result
    }
    
    /// Call this when receiving a description from the remote peer
    pub async fn handle_remote_description(
        &mut self,
        description: RTCSessionDescription,
        send_description: impl Fn(RTCSessionDescription) -> Result<()>,
    ) -> Result<()> {
        // Detect offer collision
        let ready_for_offer = !self.making_offer &&
            (self.pc.signaling_state() == RTCSignalingState::Stable ||
             self.is_setting_remote_answer_pending);
        
        let offer_collision = description.sdp_type == RTCSdpType::Offer && 
                             !ready_for_offer;
        
        // Impolite peer ignores colliding offers
        self.ignore_offer = !self.polite && offer_collision;
        if self.ignore_offer {
            return Ok(()); // Ignore this offer
        }
        
        // Polite peer rolls back if needed (implicit in setRemoteDescription)
        // When we're polite and have a collision, setting the remote offer
        // will automatically rollback our pending local offer
        
        self.is_setting_remote_answer_pending = 
            description.sdp_type == RTCSdpType::Answer;
        
        self.pc.set_remote_description(description)?;
        
        self.is_setting_remote_answer_pending = false;
        
        // If we received an offer, create an answer
        if description.sdp_type == RTCSdpType::Offer {
            let answer = self.pc.create_answer(None)?;
            self.pc.set_local_description(answer.clone())?;
            send_description(answer)?;
        }
        
        Ok(())
    }
    
    /// Call this when receiving an ICE candidate from the remote peer
    pub fn handle_remote_candidate(
        &mut self,
        candidate: RTCIceCandidateInit,
    ) -> Result<()> {
        // Ignore candidates if we're ignoring the offer they belong to
        if self.ignore_offer {
            return Ok(());
        }
        
        self.pc.add_remote_candidate(candidate)?;
        Ok(())
    }
    
    /// Get the underlying peer connection
    pub fn peer_connection(&mut self) -> &mut RTCPeerConnection {
        &mut self.pc
    }
}
```

### Usage Example

```rust
#[tokio::main]
async fn main() -> Result<()> {
    // Determine polite/impolite (e.g., based on connection order)
    let polite = determine_politeness().await;
    
    let pc = RTCPeerConnection::new(config)?;
    let mut negotiation = PerfectNegotiationHandler::new(pc, polite);
    
    // Handle negotiation needed event
    while let Some(event) = negotiation.peer_connection().poll_event() {
        match event {
            RTCPeerConnectionEvent::OnNegotiationNeeded => {
                negotiation.handle_negotiation_needed(|desc| {
                    signaling_channel.send(desc)
                }).await?;
            }
            // ... other events
        }
    }
    
    // Handle incoming signaling messages
    while let Some(msg) = signaling_channel.recv().await {
        match msg {
            SignalingMessage::Description(desc) => {
                negotiation.handle_remote_description(desc, |desc| {
                    signaling_channel.send(desc)
                }).await?;
            }
            SignalingMessage::Candidate(candidate) => {
                negotiation.handle_remote_candidate(candidate)?;
            }
        }
    }
    
    Ok(())
}
```

## Recommendations for Filling the Gap

The webrtc-rs/rtc library philosophy is to provide **WebRTC spec-compliant primitives** at the sans-I/O level, leaving higher-level patterns like Perfect Negotiation to the **application layer**. This keeps the library focused, lightweight, and flexible.

### Library Focus: Spec-Compliant Primitives ✅

The library already provides everything needed for Perfect Negotiation:
- ✅ `RTCSdpType::Rollback` support
- ✅ Full signaling state machine
- ✅ `OnNegotiationNeeded` event
- ✅ Proper SDP offer/answer handling
- ✅ ICE candidate handling

**No library changes needed** - the primitives are complete.

### Application-Level Implementation

Perfect Negotiation should be implemented as an **application-level pattern**:

**Why keep it at application level:**
- 🎯 Keeps library focused on WebRTC spec compliance
- 🔧 Applications have different signaling requirements
- 📦 Reduces library bloat and complexity
- 🚀 Allows application-specific optimizations
- 🧪 Easier to test and customize per use case

**How applications should implement it:**

Applications can create their own `PerfectNegotiationHandler` wrapper around `RTCPeerConnection`, as shown in the implementation section above. This gives full control over:
- Politeness determination logic
- Signaling channel integration
- Error handling strategies
- Logging and debugging
- Custom collision resolution policies

### 1. Create Example: `perfect-negotiation`

**Location**: `examples/examples/perfect-negotiation/`

**Features**:
- Bidirectional calling (either peer can initiate)
- Collision handling demonstration
- Role assignment (polite/impolite)
- Same code runs on both peers
- WebSocket signaling with collision simulation

**Key code**:
```rust
// Same code for both peers!
async fn run_peer(polite: bool, signaling: SignalingChannel) -> Result<()> {
    let pc = RTCPeerConnection::new(config)?;
    let mut negotiation = PerfectNegotiationHandler::new(pc, polite);
    
    // Application logic - completely symmetric!
    loop {
        tokio::select! {
            // Handle peer connection events
            Some(event) = negotiation.peer_connection().poll_event() => {
                handle_pc_event(event, &mut negotiation).await?;
            }
            
            // Handle signaling messages
            Some(msg) = signaling.recv() => {
                handle_signaling_message(msg, &mut negotiation).await?;
            }
            
            // UI events (e.g., "call" button pressed)
            Some(ui_event) = ui_events.recv() => {
                if ui_event == UiEvent::Call {
                    negotiation.handle_negotiation_needed(|desc| {
                        signaling.send(desc)
                    }).await?;
                }
            }
        }
    }
}

// Run two peers with opposite politeness
tokio::join!(
    run_peer(true, signaling_a),   // Polite peer
    run_peer(false, signaling_b),  // Impolite peer
);
```

### 2. Document the Pattern

**Location**: `rtc/docs/perfect-negotiation.md` or community blog posts

**Contents**:
- What is Perfect Negotiation and when to use it
- How to implement it using rtc's primitives
- Complete application-level implementation guide
- Code examples showing collision detection
- Comparison with traditional offerer/answerer model
- Performance considerations

### 3. Add API Documentation Notes

Add documentation to `RTCPeerConnection` highlighting the primitives for Perfect Negotiation:

```rust
/// # Implementing Perfect Negotiation
///
/// For bidirectional peer-to-peer applications where either peer can
/// initiate renegotiation, you can implement the Perfect Negotiation
/// pattern at the application level using these primitives:
///
/// - Use `RTCSdpType::Rollback` to recover from offer collisions
/// - Monitor `signaling_state()` to detect collisions
/// - Handle `OnNegotiationNeeded` events for renegotiation
///
/// ## Example
///
/// ```rust
/// // Check for collision
/// let is_making_offer = pc.signaling_state() == RTCSignalingState::HaveLocalOffer;
/// let offer_collision = desc.sdp_type == RTCSdpType::Offer && is_making_offer;
///
/// // Polite peer rolls back on collision
/// if offer_collision && polite {
///     let rollback = RTCSessionDescription {
///         sdp_type: RTCSdpType::Rollback,
///         sdp: String::new(),
///         parsed: None,
///     };
///     pc.set_local_description(rollback)?;
/// }
/// ```
///
/// See the `perfect-negotiation` example for a complete implementation.
```

## Benefits of Perfect Negotiation

Implementing Perfect Negotiation brings several advantages:

### 1. **Simplified Application Logic**

No need for separate offerer/answerer code paths:

```rust
// Before: Two separate code paths
if i_am_offerer {
    let offer = pc.create_offer(None)?;
    // ...
} else {
    // Wait for offer
    let offer = receive_offer().await?;
    // ...
}

// After: One symmetric code path
negotiation.handle_negotiation_needed(|desc| {
    send(desc)
}).await?;
```

### 2. **Robust Collision Handling**

Glare scenarios are handled automatically:
- No deadlocks
- No manual retry logic
- Predictable behavior

### 3. **Easier Testing**

Same code for both peers means:
- Half the test cases
- Easier to reason about
- Fewer edge cases

### 4. **Better User Experience**

Users don't see errors during collisions—the connection just works.

## Performance Implications

Perfect Negotiation has **minimal performance impact**:

- **No extra allocations**: Just a few boolean flags
- **No extra round trips**: Same number of signaling messages
- **Negligible CPU**: Simple boolean checks
- **Zero-cost abstraction**: Compiles to optimal code

The only overhead is during collision scenarios, which are rare (requires microsecond-level timing).

## Current Workarounds

Until built-in support is added, you can:

### Option 1: Simple Offerer/Answerer

Acceptable for applications where one peer always initiates:

```rust
// Server-side (always answers)
match msg {
    SignalingMessage::Offer(offer) => {
        pc.set_remote_description(offer)?;
        let answer = pc.create_answer(None)?;
        pc.set_local_description(answer.clone())?;
        send(answer)?;
    }
    // ...
}
```

**Limitations**:
- Can't handle bidirectional calling
- Renegotiation only from one side
- Must coordinate roles externally

### Option 2: Application-Level Perfect Negotiation

Use the `PerfectNegotiationHandler` implementation shown earlier.

**Benefits**:
- Full Perfect Negotiation support
- Works today with current library
- Can be extracted into helper crate

**Drawbacks**:
- More boilerplate
- Not battle-tested like browser implementations
- Application must implement carefully

### Option 3: Avoid Simultaneous Offers

Use application-level coordination:

```rust
// Only one peer can renegotiate at a time
let negotiation_lock = Arc::new(Mutex::new(()));

// Before creating offer
let _guard = negotiation_lock.lock().await;
let offer = pc.create_offer(None)?;
// ...
drop(_guard);
```

**Drawbacks**:
- Requires out-of-band coordination
- Adds latency
- Not a true solution

## Conclusion

Perfect Negotiation is a powerful pattern that simplifies WebRTC application development by making negotiation appear symmetric and handling edge cases gracefully.

**webrtc-rs/rtc currently**:
- ✅ Has all necessary primitives (rollback, state machine, events)
- ✅ Supports Perfect Negotiation through application-level implementation
- ❌ Doesn't provide built-in Perfect Negotiation abstractions
- ❌ Doesn't include examples demonstrating the pattern

**This is reasonable** for a sans-I/O library focused on protocol implementation. The library provides the tools; applications choose their negotiation strategy.

However, adding a `perfect_negotiation` module and example would:
- Lower the barrier to entry for P2P applications
- Reduce boilerplate in user code
- Promote best practices
- Align with modern WebRTC standards

The good news: **implementing it is straightforward** given the existing primitives. The community can benefit from:
1. A well-tested `PerfectNegotiationHandler` in the core library
2. A comprehensive example showing real-world usage
3. Documentation explaining when and why to use Perfect Negotiation

Perfect Negotiation represents the modern WebRTC way of handling peer-to-peer connections. By adding first-class support, webrtc-rs/rtc can make it even easier for developers to build robust, symmetric peer-to-peer applications in Rust.

## References

- [MDN: Perfect Negotiation](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Perfect_negotiation)
- [W3C WebRTC Spec: Perfect Negotiation Example](https://www.w3.org/TR/webrtc/#perfect-negotiation-example)
- [webrtc-rs/rtc Repository](https://github.com/webrtc-rs/rtc)

## Acknowledgments

Thanks to the W3C WebRTC Working Group for designing the Perfect Negotiation pattern and to the webrtc-rs community for building a high-quality sans-I/O WebRTC implementation in Rust.

---

*Have you implemented Perfect Negotiation in your webrtc-rs application? Share your experiences in the comments below!*
