# Vigila protocol

This repository is the normative, public specification of the Vigila **pairing**
and **end-to-end push** protocol, together with the shared, cross-language test
vectors that pin every byte of it.

Vigila is a local-first monitoring app for networks running RouterOS: a native
iOS/watchOS client, a self-hostable relay you run on your own server, and a
small, stateless hosted push bridge. The relay polls your routers and evaluates
alert rules; when a rule fires it encrypts the alert end-to-end and hands the
ciphertext to the bridge, which forwards it to Apple Push Notification service
(APNs). The app's notification-service extension decrypts it on the device.

Homepage: <https://vigila.eu>

## What this repository is

A specification, not an implementation. It documents the wire formats a
compatible relay, bridge, or client MUST produce and accept:

- [`PAIRING.md`](PAIRING.md) — the app ↔ relay pairing ceremony: the QR invite,
  TLS with trust-on-first-use (TOFU), the one-shot X25519 key agreement, session
  tokens, and un-pairing.
- [`PUSH.md`](PUSH.md) — the end-to-end push format: key derivation, the AEAD
  envelope, the bridge HTTP API, the APNs payload, and exactly what the bridge
  stores and logs.
- [`RULES.md`](RULES.md) — a short description of the shared alert rule-engine
  state machine and how the vectors drive it.
- [`vectors/`](vectors/) — verbatim snapshots of the two shared oracles,
  `rule-vectors.json` and `e2e-vectors.json`.

Key words "MUST", "SHOULD", and "MAY" are used as in RFC 2119 / RFC 8174.

## Trust model

Three parties, three trust levels:

- **Your routers** — monitored read-only by default. TLS is pinned by TOFU
  (see [`PAIRING.md`](PAIRING.md)); Vigila never installs scripts on them.
- **Your relay** — software you run. It holds router credentials (encrypted at
  rest) and the symmetric push key. It is an *endpoint* of the encrypted
  channel: it can read alert content because it produces it.
- **The bridge** — a stateless service that forwards pushes to APNs. It is
  **blind by construction**: it never receives router credentials (no field for
  them exists), it stores only a device token mapped to a *hash* of a per-device
  authentication key, and it forwards an opaque ciphertext it cannot open.

The bridge and APNs are treated as honest-but-curious infrastructure. They can
observe a push's size, timing, and a single coarse priority bit, and they can
drop or delay a push — but they cannot read, forge, or redirect its content.

## Verifying the "blind bridge" claim

The claim is that the bridge's inability to read alerts is enforced by tests,
not promises. A third party can check this from these documents and vectors
alone:

1. **The channel is real end-to-end encryption.** [`PUSH.md`](PUSH.md) pins the
   KDF, AEAD, nonce, and envelope. Using any standard X25519 / HKDF-SHA256 /
   ChaCha20-Poly1305 library and the values in
   [`vectors/e2e-vectors.json`](vectors/e2e-vectors.json), you can reproduce the
   derived key, the sealed envelope, and every tamper-rejection case
   byte-for-byte. Those vectors were computed by an independent third-party
   oracle, not by the shipping code.
2. **The bridge has no key.** The bridge API in [`PUSH.md`](PUSH.md) carries
   exactly four request fields for a push (`deviceToken`, `ciphertext`,
   `priority`, `pushType`) plus a per-device auth key in a header. None of these
   is the decryption key; the decryption key never leaves the pairing endpoints
   of the relay and the device.
3. **The bridge stores almost nothing.** Its only persistent state is a table of
   `deviceToken → SHA-256(bridgeKey)`. There is no payload column and no
   plaintext, by schema.
4. **The bridge is silent.** It never logs the device token, the auth key, or
   the ciphertext. A shipping bridge enforces this with a log-capture property
   test plus a black-box scan of captured logs for any payload byte.
5. **Redirection fails closed.** The AEAD additional data binds each ciphertext
   to its destination device token, so a bridge that swapped the target device
   would cause an authentication failure on the device, not a silent
   misdelivery.

## Trademark

Vigila is an independent app and is not affiliated with, endorsed by, or
sponsored by SIA Mikrotikls. "RouterOS" is referenced here only descriptively,
to identify the network operating system Vigila interoperates with. All
trademarks are the property of their respective owners.

## License

Licensed under the Apache License, Version 2.0. See [`LICENSE`](LICENSE).
