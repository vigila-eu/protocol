# Pairing

Pairing binds one app instance to one relay. It establishes a TLS trust anchor
(TOFU), agrees a symmetric push key over X25519, and issues a session token for
subsequent authenticated calls. All values are exchanged over HTTPS to the
relay; nothing secret is sent before the relay's certificate fingerprint is
verified.

The relay holds **at most one** pairing. A new pairing atomically replaces any
existing one.

Notation: base64 means RFC 4648 §4 standard alphabet **with padding**. Hex means
lowercase unless stated. "MUST/SHOULD/MAY" per RFC 2119 / RFC 8174.

## 1. The pairing invite (QR)

The relay renders an invite as a compact JSON object, encoded in a QR code (it
also prints the fields as text for manual entry):

```json
{
  "v": 1,
  "relayUrl": "https://host:8443",
  "pairingToken": "XXX-XXX-XXX",
  "relayCertFingerprint": "<64 lowercase hex characters>"
}
```

- `v` MUST be `1`.
- `relayUrl` MUST be an `https` URL with a host. A trailing slash is stripped
  before use. The relay's default TLS port is `8443`.
- `pairingToken` is the display form of the pairing code (see §2).
- `relayCertFingerprint` is the relay's TLS leaf-certificate fingerprint in the
  canonical form of §3.

The field *set* is normative; on decode, field order is irrelevant (it is JSON).

## 2. The pairing token

The token is **9 characters of Crockford base32** (alphabet
`0123456789ABCDEFGHJKMNPQRSTVWXYZ`, which omits I, L, O, U to avoid ambiguity),
displayed grouped as `XXX-XXX-XXX` — about 45 bits of entropy.

- **Normalization** (both sides): strip `-` and whitespace, then uppercase. The
  Crockford alphabet is already dash-free and case-insensitive.
- The relay stores **only** `SHA-256(UTF-8(normalizedToken))` as lowercase hex,
  never the token itself, and compares in constant time.
- The token is **single-use**, has a **15-minute TTL**, and is protected by a
  **5-attempt lockout**: on the 5th failed attempt (or on expiry) the relay
  deletes the token row.

## 3. TLS and trust-on-first-use (TOFU)

The relay presents a self-signed certificate. Trust is pinned to its
fingerprint, which **fully replaces** CA, hostname, and expiry validation.

- **Canonical fingerprint** = `SHA-256` over the **DER-encoded leaf
  certificate** (the exact bytes presented in the TLS handshake), formatted as
  **64 lowercase hexadecimal characters with no separators**. This matches the
  fingerprint RouterOS prints and `openssl x509 -noout -fingerprint -sha256`
  after normalization.
- **Normalization for comparison**: lowercase and strip `:` separators, so an
  `openssl`-style uppercase colon-separated fingerprint still matches. Storage
  and display use the canonical form.
- **First connect** (no stored fingerprint): complete the handshake, capture the
  leaf fingerprint, and pin it only after the user confirms it. When the invite
  arrived by QR, the client MUST verify the presented fingerprint against
  `relayCertFingerprint` **before** sending the pairing token or any other body;
  a mismatch is a hard failure and nothing is sent. For manual entry, the
  user-facing confirmation of the captured fingerprint is the trust gate.
- **On later mismatch**: abort **before any application data is sent** and block
  the connection until the user explicitly accepts the new fingerprint.

## 4. The key agreement

Pairing performs **one-shot ephemeral–ephemeral X25519**. The app and the relay
each generate a fresh X25519 key pair for this exchange, derive the shared
secret once, derive the symmetric push key from it, and then discard the private
keys. Only the 32-byte derived key persists.

- X25519 public keys are exchanged as the **raw 32-byte RFC 7748 little-endian
  u-coordinate** (equivalently, CryptoKit's `rawRepresentation`), base64-encoded
  on the wire.
- Both sides MUST reject an **all-zero shared secret** (a low-order peer point)
  as a hard pairing failure.
- The derived key is `HKDF-SHA256` of the shared secret; the exact salt, info,
  and length are specified in [`PUSH.md`](PUSH.md) §1, because the same key is
  what later decrypts every push.

## 5. `POST /api/v1/pair`

HTTPS-only and unauthenticated — the pairing token *is* the authentication. A
request that reaches a `TlsRequired` endpoint over a non-TLS listener is
rejected with `403 {"error":"tls_required"}`.

Request body:

```json
{
  "v": 1,
  "pairingToken": "XXX-XXX-XXX",
  "deviceName": "Living-room iPhone",
  "apnsToken": "<lowercase hex, or \"\">",
  "devicePubKey": "<base64(32 raw X25519 bytes)>",
  "bridgeKey": "<base64(32 random bytes)>"
}
```

- `deviceName` MUST be non-blank (display only).
- `apnsToken` is the APNs device token as lowercase hex (64–200 chars), or the
  empty string when APNs registration was unavailable at pairing time.
- `devicePubKey` is the app's ephemeral X25519 public key (§4).
- `bridgeKey` is a fresh random 32-byte per-device key the app also registers
  with the bridge (see [`PUSH.md`](PUSH.md) §3). The relay stores it wrapped and
  presents it to the bridge on every push; the relay never sees it decrypt
  anything.

On success the relay: validates the token (TTL / single-use / attempt cap),
generates its ephemeral key pair, derives the push key
(`deriveKey(relayPriv, devicePub, relayId)`), mints a random 32-byte session
token, stores the derived push key and the bridge key **encrypted at rest**
(AES-256-GCM under the relay master key), stores `SHA-256(sessionToken)` and the
device public key, and **atomically replaces** any existing pairing.

Response `200`:

```json
{
  "v": 1,
  "relayId": "<lowercase uuid>",
  "relayPubKey": "<base64(32 raw X25519 bytes)>",
  "sessionToken": "<base64(32 random bytes)>"
}
```

The app derives the **same** push key from `relayPubKey`, its own private key,
and `relayId`, discards its private key, and stores the push key in its Keychain
(shared with the notification-service extension). `relayId` is a lowercase UUID
and is bound into the key derivation (see [`PUSH.md`](PUSH.md) §1), so keys are
domain-separated per relay and per pairing.

Errors:

| Status | Body | Meaning |
|---|---|---|
| `400` | `{"error":"invalid_request"}` | Malformed body, bad field length/encoding, or blank device name. |
| `403` | `{"error":"invalid_pairing_token"}` | Wrong token, or no active token. |
| `403` | `{"error":"expired_pairing_token"}` | Token past its 15-minute TTL. |
| `403` | `{"error":"tls_required"}` | Request arrived over a non-TLS listener. |
| `429` | `{"error":"too_many_attempts"}` | The 5-attempt cap was reached. |

## 6. The session token

After pairing, authenticated relay endpoints require
`Authorization: Bearer <sessionToken>`. The relay compares
`SHA-256(presentedToken)` against the stored hash in constant time; a missing or
invalid token yields `401 {"error":"unauthorized"}`. The session token is used
for the app's configuration pushes (`PUT /api/v1/routers`, `PUT /api/v1/rules`),
incident acknowledgement (`POST /api/v1/ack`), the pairing-time test
notification (`POST /api/v1/test-notification`), and un-pairing.

## 7. Un-pairing

`DELETE /api/v1/pair` with a valid `Bearer` session token deletes the pairing
row on the relay (its wrapped keys, the session-token hash, and the device
public key) and returns `204`. The app deletes its Keychain items in parallel.
Un-pairing is best-effort from the app's side; because the relay holds only one
pairing, a fresh `POST /api/v1/pair` also supersedes the previous one.
