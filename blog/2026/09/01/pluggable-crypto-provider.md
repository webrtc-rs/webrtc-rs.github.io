# Bring Your Own Crypto Provider

`rtc` 0.21.0-rc.1 routes every cryptographic operation in the stack through one provider-neutral crate, `rtc-crypto`. No protocol crate depends on `ring`, `aws-lc-rs`, or a RustCrypto primitive crate any more. The provider is an ordinary value you pass through configuration, so two peer connections in one process can use different ones, and an application can supply its own — for OpenSSL, a FIPS-validated module, an HSM, a platform API, or a deterministic test double — by implementing three public traits.

This post is about the design principles behind that boundary and how they played out across the workspace. It is the crypto counterpart to [Bring Your Own Async Runtime](/blog/2026/07/30/pluggable-async-runtime): same instinct, that a library should not decide for the application which implementation of a general capability it links against, applied to the other big one.

---

## What we were replacing

Every crate that touched crypto carried this:

```rust
#[cfg(all(feature = "aws-lc-rs", feature = "ring"))]
compile_error!("At most one of the features \"aws-lc-rs\" and \"ring\" can be enabled.");
```

Four copies, in `rtc-dtls`, `rtc-srtp`, `rtc-stun`, and `rtc`. Cargo unifies features across a dependency graph, so a transitive dependency that enabled the other backend broke an otherwise valid build — and the person hitting it had no fix available, because the conflict was between two crates neither of which they controlled. **Non-additive features are a bug, and this was four of them.**

Underneath, the same crates carried:

```rust
#[cfg(feature = "aws-lc-rs")]
extern crate aws_lc_rs as ring;
```

This works, because `aws-lc-rs` deliberately exposes a ring-compatible API. It also means the *entire* extension story was "be shaped exactly like ring". An OpenSSL backend, a platform keystore, or an HSM could not be expressed at all, and protocol code was full of backend conditionals that had nothing to do with DTLS or SRTP.

And the choice was a compile-time global. One process could not run two peer connections against two backends — a real requirement for anyone migrating a fleet, or running a validated module for one tenant and a fast one for another.

Three more things had leaked out of place. `rtc-shared::Error` carried variants holding `sec1`, `p256`, `rcgen`, `aes-gcm`, and `aes` error types, so a crate that only wanted marshaling inherited a crypto dependency tree and its version constraints. `rtc-dtls::crypto::CryptoPrivateKeyKind` was a **public** enum naming concrete backend key types. And `rtc-srtp`'s `openssl` feature replaced exactly one AES-CTR path inside SRTP while being named as though it selected a backend.

---

## Seven design principles

### 1. Two components, one bundle

We took the shape from [OpenMLS](https://github.com/openmls/openmls), which separates a flat operations trait from a randomness trait and exposes both through a provider:

```rust
pub trait RTCCryptoProvider: Send + Sync {
    fn name(&self) -> &'static str;
    fn crypto(&self) -> &dyn RTCCrypto;
    fn random(&self) -> &dyn RTCRandom;
}
```

`RTCCrypto` is the operations half — one flat trait carrying every primitive the stack needs, with no backend-shaped submodules and no per-protocol subtraits:

```rust
pub trait RTCCrypto: Send + Sync {
    fn supports(&self, algorithm: CryptoAlgorithm) -> bool;

    fn hash(&self, algorithm: HashAlgorithm, data: &[u8]) -> Result<Vec<u8>, CryptoError>;
    fn block_encrypt(&self, algorithm: BlockCipherAlgorithm, key: &[u8], block: &mut [u8])
        -> Result<(), CryptoError>;

    // Keyed operations are factories; see principle 4.
    fn new_hmac(&self, algorithm: HmacAlgorithm, key: &[u8]) -> Result<Box<dyn Mac>, CryptoError>;
    fn new_stream_cipher(&self, /* … */) -> Result<Box<dyn StreamCipher>, CryptoError>;
    fn new_aead(&self, /* … */) -> Result<Box<dyn AeadCipher>, CryptoError>;
    fn new_cbc(&self, /* … */) -> Result<Box<dyn CbcCipher>, CryptoError>;
    fn start_key_exchange(&self, /* … */) -> Result<Box<dyn ActiveKeyExchange>, CryptoError>;

    fn generate_signing_key(&self, scheme: SignatureScheme)
        -> Result<Arc<dyn SigningKey>, CryptoError>;
    fn import_signing_key(&self, scheme: SignatureScheme, pkcs8_der: &[u8])
        -> Result<Arc<dyn SigningKey>, CryptoError>;
    fn verify_signature(&self, scheme: SignatureScheme, public_key: PublicKey<'_>,
                        message: &[u8], signature: &[u8]) -> Result<(), CryptoError>;
}
```

Flat is a decision, not an accident. The alternative we rejected was one trait per protocol — `DtlsCrypto`, `SrtpCrypto`, `StunCrypto` — which duplicates primitives across contracts (all three want HMAC-SHA1) and makes a downstream author implement overlapping surfaces. Protocol-specific composition stays in the protocol crates; the provider exposes primitives.

`RTCRandom` is the other half, and it is deliberately tiny:

```rust
pub trait RTCRandom: Send + Sync {
    fn fill(&self, output: &mut [u8]) -> Result<(), CryptoError>;
}
```

Randomness is separate because it has a different consumer set and a different compliance story. A deployment that must draw every byte of protocol entropy from a validated module needs to replace exactly that, without reimplementing AES-GCM. Conversely a deterministic test provider wants a predictable `RTCRandom` and real crypto.

`name()` lives on the bundle rather than on `RTCCrypto`, so the operations trait contains only capabilities and operations. It is a diagnostic label, and the docs say so explicitly: **a provider name is not a validation claim.**

### 2. A provider is a value, not a global

This is the principle we were least willing to compromise on, and where we diverge from most of the ecosystem. The common pattern is a process-wide default installed once at startup — `rustls` works this way — and it is a reasonable fit for an application that owns its whole process. It is a poor one for a library that gets embedded in a process it does not own.

A process-global provider makes tests order-dependent, prevents per-connection choice, and means a library can silently change the crypto of an application that never asked. So:

```rust
setting_engine.set_crypto_provider(Arc::new(MyProvider::new()));
```

There is nothing to install at startup and no registration step. A downstream provider is a struct that implements three traits and gets passed in.

**No library code resolves a default.** `default_provider()` is called in exactly one place in the whole workspace — peer-connection construction, where the application either supplied one or gets the feature-selected built-in:

```rust
let crypto_provider = match setting_engine.crypto_provider.take() {
    Some(crypto_provider) => crypto_provider,
    None => crypto::default_provider().map_err(|error| {
        Error::Crypto(format!(
            "peer connection requires a crypto provider: {error}; \
             configure one with SettingEngineBuilder::with_crypto_provider"
        ))
    })?,
};
```

Everything below that receives an `Arc<dyn RTCCryptoProvider>` from its caller. (Tests, examples and benchmarks are the outside caller and may resolve a default themselves.) It is also why the default-resolving constructors are gone. Each had a `*_with_provider` sibling, and keeping both meant keeping a path that hid the choice — one that, in a `--no-default-features` build, failed deep inside a handshake instead of at configuration time. Where a type had such a pair, the two collapsed into one provider-taking constructor: `Context::new`, `Agent::new`, `Client::new`, `Certificate::generate_self_signed`, `Certificate::from_pem`, `RTCCertificate::from_pem`, and `get_fingerprints`. `MessageIntegrity` is the exception — there the default-resolving constructors were dropped and the explicit `new_raw_integrity_with_provider`, `new_short_term_integrity_with_provider` and `new_long_term_integrity_with_provider` kept their names. `Default` impls that resolved a provider — on `HandshakeConfig`, `rtc_dtls::State`, `Agent`, `RTCDtlsTransport` — were removed rather than reworked, because `Default` has nowhere to accept one.

### 3. Erase the type at the configuration boundary, not through the stack

OpenMLS threads `&impl OpenMlsProvider` generically. That is right for OpenMLS and wrong here: RTC's peer connection and protocol state are long-lived concrete types, and a generic provider parameter would land on most of them, in public signatures, forever.

`Arc<dyn RTCCryptoProvider>` costs a vtable dispatch. We are not going to claim, as some pluggable-backend designs do, that this is free — it is a real indirect call. It is also negligible next to the cryptographic work behind it, and principle 4 exists to make sure it stays that way. The trade we accepted: dynamic dispatch at a boundary crossed once per operation, in exchange for not parameterizing `RTCPeerConnection` on its crypto.

Two smaller decisions fell out of the same reasoning. The traits require `Send + Sync` where shared concurrency needs them, but **not** `Debug` — crypto implementations, cipher objects, key exchange state and platform handles hold sensitive material, and structural debug formatting is not a safe diagnostics contract. And there is no trait-level `'static` bound; storing the `Arc` in owned state imposes that where ownership actually happens.

### 4. Factories, not one-shot calls

The expensive part of keyed cryptography is usually the key schedule, not the operation. An API of stateless `encrypt(key, ...)` calls repeats it per packet and gives a hardware backend nowhere to keep a session. So every keyed operation is a factory returning a mutable object:

```rust
fn new_hmac(&self, algorithm: HmacAlgorithm, key: &[u8]) -> Result<Box<dyn Mac>, CryptoError>;
fn new_stream_cipher(&self, algorithm: StreamCipherAlgorithm, key: &[u8]) -> Result<Box<dyn StreamCipher>, CryptoError>;
fn new_aead(&self, algorithm: AeadAlgorithm, key: &[u8]) -> Result<Box<dyn AeadCipher>, CryptoError>;
fn new_cbc(&self, algorithm: CbcAlgorithm, key: &[u8]) -> Result<Box<dyn CbcCipher>, CryptoError>;
```

An SRTP `Context` creates its cipher once, during construction, for that one-way cryptographic context — not per SSRC and not per packet. A DTLS epoch keys its record cipher once. Provider dispatch is concentrated in setup, and the per-packet path is a direct call on an already-keyed object.

These objects are `Send` and take `&mut self`, deliberately not `Sync`. A software backend can hold an immutable expanded key internally; an HSM needs mutable session state. Requiring `Sync` would force internal locking on implementations that need it, to serve callers — DTLS epochs, one-way SRTP contexts — that already have exclusive access.

**The `Mac` trait was not in the original design, and it is the change the implementation forced.** The design had one-shot `hmac()` and `verify_hmac()` methods. They looked harmless, and they preserved a path that re-derives the HMAC ipad/opad schedule on every call — which measured at roughly **40% of SRTP's per-RTCP-packet time**. They were removed in favour of `new_hmac` returning a keyed `Mac`, so there is exactly one way to compute an HMAC and the keying cost is visible at the call site.

Key exchange gets the same treatment, with an ownership twist:

```rust
pub trait ActiveKeyExchange: Send {
    fn algorithm(&self) -> KeyExchangeAlgorithm;
    fn public_key(&self) -> &[u8];
    fn complete(self: Box<Self>, peer_public_key: &[u8]) -> Result<SecretVec, CryptoError>;
}
```

The object owns the ephemeral private key, exposes only the wire public key, and **consumes itself** deriving the shared secret. Accidental reuse of an ephemeral key becomes a compile error. This replaced a `pub(crate)` enum holding a `p256::ecdh::EphemeralSecret`, a `p384` one, or an `x25519_dalek::StaticSecret`. The alternative shape — `Box<dyn Any>` handles with downcasts — was rejected: it permits provider/key mismatches at runtime and cannot enforce one-shot use.

### 5. Primitives, not policy

The hardest part of a crypto boundary is deciding what is *not* on it. Our line: **the provider answers whether a signature is valid; configuration answers whether a certificate should be trusted.**

So these stayed out of `RTCCrypto`: X.509 and PEM encoding, WebRTC fingerprint formatting and comparison, CA roots, hostname checks, `VerifyPeerCertificateFn`, the DTLS 1.2 PRF composition, SRTP key-derivation labels and packet layout, and the DTLS handshake state machine. A custom provider is not required to implement CA-chain policy, and a provider cannot become the place where application security policy lives.

The clearest case was `KeyingMaterialExporter`, a trait in `rtc-shared` that existed so `rtc-srtp` could call into `rtc-dtls` without depending on it. It looks like a crypto primitive. It is not: RFC 5705 export needs an established session's master secret, client and server randoms, negotiated suite, and handshake state — all of which belong to `rtc-dtls::State`, and none of which a provider owns. We deleted it rather than moving it:

```text
rtc-dtls::State -- export_keying_material() --> SecretVec
                                                    |
                                                    v
             rtc orchestration -- selected profile and endpoint role
                                                    |
                                                    v
rtc-srtp::Config -- set_session_keys_from_keying_material() --> SessionKeys
```

DTLS exports bytes through an inherent method, SRTP consumes bytes, and the top-level `rtc` crate — which is the only thing that knows both the DTLS state and the negotiated SRTP profile — owns the handoff. The RFC 5764 label was promoted from a private `const` in `rtc-srtp` to a `pub const`, so standalone users of the two crates are not retyping a magic string. The blast radius was one call site.

Secrets crossing the boundary use a zeroizing container with redacted `Debug` and no `Display`:

```rust
pub struct SecretVec(zeroize::Zeroizing<Vec<u8>>);
```

### 6. Capabilities are checked before negotiation, not during

A provider that does not implement P-384 should cause a *construction* error, not a handshake that stalls after the peer picks a suite nobody can complete. So configuration intersects four sets before advertising anything:

```text
DTLS suites implemented by rtc-dtls
  ∩ suites whose primitives the provider supports
  ∩ suites compatible with the configured certificate/signing key
  ∩ application-configured suites
  = advertised/accepted suites
```

The same rule applies to named groups, signature schemes, and SRTP protection profiles. `RTCCrypto::supports(CryptoAlgorithm)` is what makes it possible, and the contract is explicit that `supports` is for negotiation and early failure — an operation must still reject an unsupported algorithm even if a provider's `supports` lies.

### 7. Open traits, and prove it in the build

`RTCCrypto` is user-implementable and will not be sealed. That is the whole point, and it has a consequence that is easy to lose by accident: **dyn compatibility**. A generic method, an `impl Trait` argument, or a method returning `Self` silently makes a trait unusable through `dyn` — and a feature-gated one only breaks some builds. Documenting the invariant is not enough, so it is asserted:

```rust
const _: () = {
    fn assert_dyn_compatible(
        _provider: &dyn RTCCryptoProvider,
        _crypto: &dyn RTCCrypto,
        _random: &dyn RTCRandom,
        _mac: &dyn Mac,
        _stream: &dyn StreamCipher,
        _aead: &dyn AeadCipher,
        _cbc: &dyn CbcCipher,
        _exchange: &dyn ActiveKeyExchange,
        _signing_key: &dyn SigningKey,
    ) {
    }
};
```

It compiles under every feature combination in CI, so the moment someone adds a method that breaks object safety, the build says so.

Every operation method has a default body returning `UnsupportedAlgorithm`. Only `name` and `supports` must be implemented. Partial providers are practical, and adding an operation later does not break every downstream implementation.

---

## How it landed across the crates

The provider does not go everywhere. It goes exactly where cryptography happens: one crate defines it, six consume it, and nine of the sixteen have no crypto dependency at all.

| Crate | `RTCCrypto` | `RTCRandom` | How it holds the provider |
|---|---|---|---|
| `rtc-crypto` | defines and implements | defines and implements | owns every backend dependency |
| `rtc-dtls` | direct — transcripts, PRF, AEAD/CBC/ChaCha20, ECDHE, signing, verification | direct — handshake randoms, cookies, record IVs | keeps the `Arc` for the handshake lifetime |
| `rtc-srtp` | direct at construction — KDF AES blocks, cipher factories | none; IVs derive from salts and packet state | borrows `&dyn RTCCrypto`, then stores the cipher objects |
| `rtc-stun` | direct — MD5, HMAC-SHA1, constant-time comparison | none; transaction IDs keep a documented CSPRNG | `MessageIntegrity<'a>` borrows `&'a dyn RTCCrypto` |
| `rtc-ice` | indirect, via authenticated STUN | direct — ufrags, passwords, tie-breakers | holds the `Arc`, forwards it to STUN |
| `rtc-turn` | indirect, via authenticated STUN | — | forwards the provider it holds |
| `rtc-sctp`, `rtc-shared`, `rtc-rtp`, `rtc-rtcp`, `rtc-interceptor`, `rtc-media`, `rtc-sdp`, `rtc-mdns`, `rtc-datachannel` | none | none | no `rtc-crypto` dependency at all |
| `rtc` | direct — certificate keys and fingerprints | — | `SettingEngine` resolves once, clones downstream |

Two things in that table are decisions rather than outcomes.

**`rtc-shared` does not depend on `rtc-crypto`, and never will.** Crypto errors convert at the consuming crate's boundary. The backend error variants are gone, replaced by a neutral `Error::Crypto(String)`. Otherwise every crate in the workspace that wants a marshaling helper acquires a transitive crypto dependency, which is the problem we started with wearing a different hat.

**Not everything random is cryptographic, and not every secure random is worth the plumbing.** This was the most contested judgement in the design, and we came down against uniformity — twice, in two different directions.

RTP sequence-number and timestamp offsets, SSRCs, codec picture IDs, SDP session IDs and media container serials are collision-avoidance values, not secrets. Routing them through a provider would mean threading `Arc<dyn RTCCryptoProvider>` through `rtc-rtp` and `rtc-sdp` for no security benefit and a permanently worse standalone API. They keep ordinary `rand`.

The harder cases are the values that genuinely need unpredictability in crates that do not otherwise need crypto. DTLS randoms, cookies and record IVs go through `RTCRandom` — it is a hard requirement there, and DTLS already holds the provider. ICE went through too: ufrags, passwords and tie-breakers all take `provider.random()`, because the agent already holds an `Arc` for authenticated STUN. But **STUN transaction IDs and SCTP verification tags, association IDs and initial TSNs did not.** They keep a thread-local CSPRNG, which satisfies their security requirement, and they keep it so that `rtc-sctp` and `rtc-stun`'s message layer remain usable standalone without acquiring a provider parameter. The rule we settled on: adopt `RTCRandom` where the provider is already in hand, and do not distort an API to reach it. The reasoning sits in a comment next to each retained call rather than in a document nobody will find:

```rust
// RFC 4960 requires an unpredictable initial TSN. SCTP remains usable without an RTC
// crypto provider, so this deliberately uses `rand`'s thread-local CSPRNG.
```

The shape of provider ownership also differs by crate, on purpose. DTLS holds an `Arc` because it needs algorithms and randomness across flights that span time. SRTP takes `&dyn RTCCrypto` at construction and afterwards holds only the cipher objects it built — it has no later need for the provider. STUN's `MessageIntegrity<'a>` borrows for a lifetime rather than sharing an `Arc`, because it is a short-lived credential used inside one message operation. **`Arc` is the ownership mechanism, not the abstraction**, and where a borrow is honest we borrow.

---

## What the benchmarks found

We benchmarked before believing anything, and the first measurements were bad: DTLS encryption 3–8× slower than the pre-migration baseline. Three separate causes, none of them the vtable dispatch everybody worries about.

**Per-record `SystemRandom`.** DTLS draws fresh randomness per record — the GCM explicit nonce, the CBC record IV. Before the migration that came from a thread-local ChaCha CSPRNG. After it, `RTCRandom`'s built-ins called the backend's `SystemRandom`, which reaches the operating system on every call:

| 8-byte fill | ns/call |
|---|---|
| `rand::fill` (thread-local) | **8.3** |
| `ring::rand::SystemRandom` | 829.1 |
| `aws_lc_rs::rand::SystemRandom` | 2196.5 |

That is 100–250×, and it accounted for the regression almost exactly. Caching the `SystemRandom` handle instead of constructing one per call recovered only ~9% — the OS round trip was the whole cost. The built-in providers now use an OS-seeded, periodically reseeded thread-local CSPRNG, which is what the pre-provider code did and what BoringSSL and OpenSSL do internally. Backend RNGs are still used where the backend owns the operation, for keypair generation and signing. A deployment that needs every byte from a validated module supplies its own `RTCRandom` — that is what the trait is for.

**Per-record HMAC key setup.** `CryptoCbc` passed raw key bytes on every record, re-deriving the key schedule per record. This is the defect that produced the `Mac` trait in principle 4. It now holds two keyed `Mac` objects, keyed once per epoch.

**`ring`'s software SHA-1.** CBC authenticates every record with HMAC-SHA1, and `ring` exposes SHA-1 only as `HMAC_SHA1_FOR_LEGACY_USE_ONLY`, without the ARMv8 SHA-1 instructions — 4469 ns against RustCrypto's 1373 ns over 1212 bytes. The `ring` provider now composes RustCrypto's HMAC-SHA1.

After those three, on an Apple M1 Max with identical criterion settings:

| DTLS record protection | Pre-migration | ring (default) | aws-lc-rs |
|---|---|---|---|
| `Encrypt/AES-128-GCM` | 270.5 ns | 270.1 ns | 245.8 ns |
| `Decrypt/AES-128-GCM` | 280.6 ns | 274.8 ns | 225.6 ns |
| `Encrypt/AES-256-CBC` | 3.419 µs | 3.204 µs | 2.498 µs |
| `Decrypt/AES-256-CBC` | 2.036 µs | 1.882 µs | 1.194 µs |
| `Setup/AES-256-CBC` | 86.5 ns | 755.9 ns | 818.4 ns |

Every per-record path is at parity or better on both providers. `Setup` rose by design: that is the record MAC key schedule moving out of the per-record path, paid once per epoch instead of once per record. SRTP tells the same story — `Encrypt/RTP` at 1.723 µs against a 1.725–1.780 µs baseline on ring, and 1.018 µs on aws-lc-rs.

The benchmark groups are split into `Setup/*` and `Encrypt/*`/`Decrypt/*` precisely so this property is falsifiable: if a per-packet number ever regresses while setup holds steady, the design has broken. The numbers, and the procedure for reproducing them, are in [`rtc-dtls/benches/README.md`](https://github.com/webrtc-rs/rtc/blob/master/rtc-dtls/benches/README.md) and [`rtc-srtp/benches/README.md`](https://github.com/webrtc-rs/rtc/blob/master/rtc-srtp/benches/README.md).

**An honest note on the built-ins:** both are *composite*. `RingProvider` uses ring for SHA-256, AEAD, key exchange and signatures, and RustCrypto for AES-CTR, CCM, CBC, MD5 and HMAC-SHA1. `AwsLcRsProvider` keeps its own SHA-1, which is faster than both. Neither exposes an `is_fips()` method or should be described as FIPS because of its name. A strict provider that advertises only algorithms routed through a validated module is exactly what the trait is for, and it is a downstream artifact.

---

## Features are additive now

```toml
[features]
default = ["crypto-ring"]
crypto-ring = ["crypto/crypto-ring"]
crypto-aws-lc-rs = ["crypto/crypto-aws-lc-rs"]
```

| Build | Result |
|---|---|
| `--features crypto-ring` | ring only |
| `--features crypto-aws-lc-rs` | aws-lc-rs only |
| `--features crypto-ring,crypto-aws-lc-rs` | both compiled; `default_provider()` returns ring |
| `--no-default-features` | no built-in; supply your own |

Enabling both is supported and tested. `default_provider()` prefers ring when both are on, so adding `crypto-aws-lc-rs` adds capability without changing behaviour — you get aws-lc-rs by naming it, not by enabling it. The features are named `crypto-*` rather than `ring`/`aws-lc-rs` because they now select a *provider*, not a dependency, and `rtc-dtls` forwards them to `rustls` and `rcgen` as well.

CI covers default, ring-only, aws-only, both, and a no-built-in build against a test custom provider, for the workspace and for each protocol crate standalone.

---

## Breaking changes

This is a pre-1.0 gate — it changes public construction APIs and would have been far more expensive after the freeze. These are the ones most likely to affect you:

| Item | Status | Replacement |
|---|---|---|
| `rtc_shared::crypto::KeyingMaterialExporter` | removed | `rtc_dtls::State::export_keying_material` |
| `rtc_srtp::Config::extract_session_keys_from_dtls` | removed | `set_session_keys_from_keying_material` |
| `rtc_dtls::crypto::CryptoPrivateKeyKind` | removed | opaque `Arc<dyn SigningKey>` |
| `rtc_dtls::crypto::CustomSigner` | removed | implement `rtc_crypto::SigningKey` |
| `RTCCertificate::from_key_pair` | removed | `RTCCertificate::generate` |
| `rtc_shared::Error::{Sec1, P256, RcGen, AesGcm, Aes}` | removed | `Error::Crypto(String)` |
| `rtc_srtp` `openssl` / `vendored-openssl` | removed | write a complete provider |
| every default-resolving constructor | changed | takes a provider |

Two are worth expanding.

**`CustomSigner` is subsumed, not lost.** An external HSM or KMS signer implements `SigningKey` directly and returns `None` from `to_pkcs8_der`. Non-exportable keys are first-class: nothing in the API requires producing placeholder private-key bytes, and attempting a PEM serialization that would include a private key returns an explicit error rather than an empty block.

**The OpenSSL features were removed rather than kept.** They selected an alternate AES-CTR path inside SRTP only and never implemented the full contract, so keeping the names would have implied a completeness that did not exist. An OpenSSL backend can return as a complete `RTCCryptoProvider`, in an application or a separate crate, and needs no change to `rtc` to do it.

Each protocol crate re-exports the crypto API — `rtc_srtp::crypto`, `rtc_stun::crypto`, `rtc_ice::crypto`, `rtc_turn::crypto`, and `rtc_dtls::crypto_provider` (named differently because `rtc-dtls` already has a `crypto` module) — so a standalone user can name `Arc<dyn RTCCryptoProvider>` without adding and version-matching a direct `rtc-crypto` dependency.

---

## Writing your own

Implement three traits and pass the value in. There is nothing to register.

```rust
use std::sync::Arc;
use rtc_crypto::{RTCCrypto, RTCCryptoProvider, RTCRandom};

struct MyProvider { crypto: MyCrypto, random: MyRandom }

impl RTCCryptoProvider for MyProvider {
    fn name(&self) -> &'static str { "my-provider" }
    fn crypto(&self) -> &dyn RTCCrypto { &self.crypto }
    fn random(&self) -> &dyn RTCRandom { &self.random }
}

setting_engine.set_crypto_provider(Arc::new(MyProvider::new()));
```

Then validate it against the same suite the built-ins pass:

```rust
// Cargo.toml: rtc-crypto = { version = "0.21", features = ["test-support"] }
#[test]
fn my_provider_conforms() {
    rtc_crypto::conformance::assert_provider(&MyProvider::new());
}
```

`assert_provider` runs RFC known-answer vectors, round trips, tag- and nonce-length validation, wrong-key and wrong-AAD failures, ECDHE agreement per group, signature generation and verification, and unsupported-algorithm reporting. Individual sections are public too — `assert_hashes_and_hmac`, `assert_aead`, `assert_cbc`, `assert_block_and_stream_ciphers`, `assert_key_exchange`, `assert_signatures`, `assert_random` — for a provider that implements part of the surface. Publishing the suite rather than keeping it as an internal test module is deliberate: a downstream provider author should not have to copy our tests to find out whether their AEAD nonce handling is right.

`cargo test --package rtc-crypto --no-default-features --features test-support --test custom_provider` builds a complete downstream-style provider with no built-in enabled, so the "bring your own" path is exercised in CI rather than asserted in a README.

One request, in the docs and repeated here: **report unsupported algorithms honestly through `supports`.** Negotiation intersects protocol support, provider capability, key compatibility, and application configuration before advertising anything. An accurate answer turns an unusable combination into a construction-time error. An optimistic one turns it into a handshake that stalls.

---

## What this does not do

- **It is not a FIPS claim.** Neither built-in provider is validated, and neither exposes `is_fips()`. The trait makes a validated provider possible; it does not make one.
- **It does not delegate DTLS.** A provider supplies primitives to our sans-I/O DTLS engine; it cannot replace the engine. Handing a backend the whole handshake is a coherent design — it is what you do when the DTLS state machine is not yours to begin with — but it is not this one. The flights, transcript, suite negotiation, PRF composition and key schedule stay in `rtc-dtls`.
- **It is not a stable ABI.** Providers are Rust trait implementations compiled into your binary, not dynamically loaded modules.
- **No `no_std` yet.** Out of scope for this change.
- **No new algorithms.** HKDF, RSA-PSS and DTLS 1.3 primitives are absent because no caller exists. The enums are `#[non_exhaustive]` and the operations have default bodies, so adding them later breaks nothing.

---

## Try it

```toml
[dependencies]
rtc = "0.21.0-rc.1"
```

The rc is where feedback is still cheap to act on. If you have a provider that does not fit — a platform keystore with an odd public-key encoding, an HSM whose signing session needs state we have not allowed for, a validated module whose RNG can fail in a way `CryptoError::RandomnessFailed` does not capture — the conformance suite is the fastest way to find out, and an issue on [webrtc-rs/rtc](https://github.com/webrtc-rs/rtc/issues) is the fastest way to fix it. After 1.0, this trait is a compatibility commitment.

## Related posts

- [Bring Your Own Async Runtime](/blog/2026/07/30/pluggable-async-runtime)
- [The Path to webrtc 1.0](/blog/2026/08/01/the-path-to-webrtc-1.0)
- [Interceptor Design Principle: Composable RTP/RTCP Processing](/blog/2026/01/09/interceptor-design-principle-sansio)
- [Building WebRTC's Pipeline with sansio::Protocol](/blog/2026/01/04/building-webrtc-pipeline-with-sansio)
