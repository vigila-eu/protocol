# Shared test vectors

These two files are **verbatim snapshots** of the canonical vectors maintained
in the Vigila source repository. They are copied here unchanged so a third party
can verify the protocol from this repository alone.

The Vigila source repository is authoritative. If a value here ever disagrees
with the canonical source, the canonical source wins; changes flow from there to
these snapshots, never the other way. Do not edit these files here.

- **Snapshot date:** 2026-07-19
- **Source revision:** `6aad936`

## Files

| File | Cases | What it pins |
|---|---|---|
| [`rule-vectors.json`](rule-vectors.json) | 50 cases | The rule-engine reducer: for each case's steps, the expected next state and emitted events. Drives both language implementations (see [`../RULES.md`](../RULES.md)). |
| [`e2e-vectors.json`](e2e-vectors.json) | 20 vectors | The end-to-end push crypto: X25519 derivation, HKDF-SHA256, ChaCha20-Poly1305 seal/open, the AAD, the envelope, and the pairing-token / message field shapes (see [`../PUSH.md`](../PUSH.md)). |

## About the crypto vectors

Every expected value in `e2e-vectors.json` was produced by an **independent
oracle** — a standalone generator built on a third-party X25519 /
ChaCha20-Poly1305 library plus a hand-rolled RFC 5869 HKDF — that never runs the
shipping relay or app code. The generator self-checks against the RFC 7748 ECDH
directions and the RFC 5869 Appendix A published outputs before emitting. The
vectors are byte-exact for the `derive`, `hkdf`, `seal`, `open`, and `aad`
groups; the `pairing` group pins field names and the token-hash byte value.

Any implementation of this protocol can load these files and check itself
byte-for-byte, independently of Vigila's own code.
