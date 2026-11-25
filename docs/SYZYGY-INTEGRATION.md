# Syzygy integration design

This document governs the additive Syzygy integration in Eclipsejar. It is based on the current `main` source of Eclipsejar and the current `main` source of NotADeveloper4665/Syzygy inspected before implementation.

## Compatibility invariants

- Keep the Moonlight/GameStream protocol as the baseline. Existing Sunshine, Apollo, Rubylight, Vibepollo and GameStream behavior remains available and unchanged.
- Host identity/family is descriptive only. A feature is enabled only by explicit, validated protocol evidence.
- Do not replace PIN pairing or Rubylight QR pairing. Syzygy six-word pairing is an additional method.
- Keep host-specific extensions behind capability checks. Unknown or absent fields must not enable a feature.
- Keep platform-only code behind `target_os` gates. In particular, `cfg(unix)` also matches macOS and must not be used as a synonym for Linux.
- Keep upstream-friendly changes in focused modules and avoid changing existing wire behavior.

## Verified current architecture

The repository's `docs/code-map.md` is the source of truth for ownership:

- `protocol/policy`: host capability evidence and shared decisions.
- `protocol/client`: pairing, host HTTP, identity, TLS pinning, streaming and networking.
- `protocol/desktop`: desktop host workflows and commands.
- `protocol/desktop-ui`: desktop pairing and settings presentation.
- `app/` and `protocol/android`: Android UI/JNI bridge and Rust engine integration.
- `protocol/desktop-media`: platform media and presentation backends.

Current pairing is implemented in `protocol/client/src/http/pairing.rs` (standard PIN/OTP/link handshake), called by `protocol/client/src/hosts/pairing.rs` and surfaced by desktop pairing flows in `protocol/desktop/src/hosts/pairing.rs` and `protocol/desktop-ui/src/ui/dialogs/pair_dialogs.rs`. The policy capability parser is `protocol/policy/src/http/capabilities/parse.rs`; the capability model and availability decisions are in `protocol/policy/src/http/capabilities/types.rs`. Android has a parallel Java model in `app/src/main/java/com/limelight/nvstream/http/HostCapabilities.java`.

## Syzygy protocol facts confirmed from host source

### Discovery/capability evidence

Syzygy currently emits these GameStream `/serverinfo` fields:

- `syzygy_phrase_version` (currently value `2`): six-word pairing protocol availability/version.
- `syzygy_tails_pairing` (0/1): whether same-tailnet pairing is currently available for the request path.

These fields are not the host's family name and must be parsed as explicit feature evidence. A missing field means unknown/not offered.

### Six-word pairing

The six-word phrase is normalized to six lowercase words, accepting spaces or hyphens. Each word is checked against Syzygy's bundled `phrase_words` list. The host derives a 24-byte key with scrypt (N=32768, r=8, p=1, 64 MiB max memory), salt `Syzygy pairing phrase v2`, then represents the key as 48 lowercase hex characters.

The host's `GET /pair` extension uses query parameters:

1. Challenge: `uniqueid`, `syzygyphase=challenge`, `clientcert` (hex-encoded PEM), `devicename`, and optional `syzygytails=1`.
2. Challenge response: host returns hex `challenge`, `authmessage`, `plaincert`, and `paired=1`.
3. Client response: `syzygyphase=response`, same `uniqueid`, `clientcert`, `clientsignature` (client certificate signature over the exact returned auth message), and `syzygyproof` (HMAC-SHA256 proof using the derived host key); optional `syzygytails=1`.
4. Host response: `paired=1` plus `serverproof` (phrase mode) or `serversignature` (same-tailnet mode).

The signed/authenticated message is domain-separated with `Syzygy pairing v1\0`; it contains length-prefixed nonce, host unique ID, and SHA-256 fingerprint of the client certificate. The server confirmation binds the host certificate fingerprint too. Challenges expire after two minutes and are consumed on response. The host rate-limits challenge attempts. Pairing must preserve these bounds and verify the server confirmation before pinning the server certificate.

The host's standard `phrase=getservercert` PIN flow is a separate branch and must not be modified.

### Tailscale

The current host implementation does not trust a caller-supplied header or claimed Tailscale IP. It queries the local `tailscaled` Unix socket and validates peer identity, machine authorization, network-map membership, expiry and sharing status. The host code currently hardcodes `/run/tailscale/tailscaled.sock` and explicitly reports Windows as unsupported. The client must treat the serverinfo field as a current availability hint, not proof of authorization; the host remains authoritative during pairing.

### QUIC and WAN/FEC

The host repository contains a separate aioquic-based transport process and WAN pacing/FEC controls. These are not part of the six-word pairing handshake. Do not advertise or enable them solely because six-word pairing is available. Before implementation, map their ALPN, certificate pinning, ports, stream/datagram mapping, session binding, fallback behavior and capability advertisement to Eclipsejar's existing RTSP/UDP receive pipeline. The existing Moonlight TCP/UDP path remains the fallback.

## Implementation plan and intended file ownership

### Phase A — capability evidence and protocol primitives

- `protocol/policy/src/http/capabilities/types.rs`: add explicit Syzygy pairing/QUIC/Tailscale feature identifiers and evidence fields.
- `protocol/policy/src/http/capabilities/parse.rs`: parse only the documented serverinfo fields; missing or malformed values remain unavailable.
- `protocol/policy/src/http/capabilities.rs` and its tests: derive availability from evidence, not host family.
- `protocol/client/src/http/syzygy_pairing.rs` (new): phrase normalization, word-list validation, scrypt derivation, domain-separated message construction, HMAC proof and confirmation verification.
- `protocol/client/src/http/client_ops.rs` and `protocol/client/src/http/mod.rs`: add a separate Syzygy pairing operation over the existing HTTP transport.
- `protocol/client/src/hosts/pairing.rs`: expose a distinct secret/method while retaining PIN, OTP and link pairing.
- `protocol/client/Cargo.toml`: add only the minimal audited scrypt dependency if the locked workspace does not already provide one.

### Phase B — desktop and Android UI

- `protocol/desktop/src/hosts/pairing.rs`, desktop command/dialog state and `protocol/desktop-ui/src/ui/dialogs/pair_dialogs.rs`: show six-word pairing only when the host advertises it; preserve PIN and QR options.
- `app/src/main/java/com/limelight/nvstream/http/HostCapabilities.java`, Android pairing flow and corresponding tests: mirror capability parsing and provide the same additive pairing path.
- Add subtle `Syzygy only` labels only to controls that are truly exclusive to Syzygy. Shared Moonlight/Sunshine behavior remains unlabelled; Rubylight-only QR stays clearly identified.

### Phase C — transport extensions

- Map the Syzygy QUIC transport and WAN/FEC protocol against the existing `protocol/client/src/session`, `protocol/client/src/video` and `protocol/client/src/control` boundaries before touching the media pipeline.
- Implement only after exact protocol fields, ports, ALPN, authentication and fallback semantics are verified from host source and tests.
- Keep Tailscale discovery separate from authentication; never trust unverified peer metadata.

## Verification

The existing documented narrow tests are:

- `cargo test --locked -p nightjar-policy`
- `cargo test --locked -p nightjar-core`
- desktop UI pairing tests in `protocol/desktop-ui/src/ui/tests/host_checks_and_pairing_tests.rs`
- Android `HostCapabilitiesContractTest` and pairing tests, plus the documented Gradle unit/lint/build checks.

A protocol implementation is not complete until it has tests for correct proof, wrong phrase, malformed/expired challenge, replay, certificate mismatch, same-tailnet refusal, unknown capability fields, and standard PIN/QR regression behavior. QUIC requires host/client interoperability and a real stream test; unit tests alone do not establish working video transport.

## Current status

Discovery is complete. This document records the protocol and the planned files. It does not claim that Syzygy pairing, QUIC, Tailscale authorization or WAN/FEC integration has been implemented or verified yet.
