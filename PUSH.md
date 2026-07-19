# End-to-end push

An alert travels **relay → bridge → APNs → device**. The relay produces the
plaintext and seals it; only the relay and the device hold the key. The bridge
and APNs forward opaque ciphertext.

Notation as in [`PAIRING.md`](PAIRING.md): base64 is RFC 4648 §4 **with
padding**; hex is lowercase. "MUST/SHOULD/MAY" per RFC 2119 / RFC 8174.

## 1. Key derivation

The 32-byte symmetric push key is derived once, at pairing, on both sides:

```
sharedSecret = X25519(ownPrivateKey, peerPublicKey)          // RFC 7748
key          = HKDF-SHA256(
                 IKM  = sharedSecret,
                 salt = UTF-8("vigila-e2e-salt-v1"),
                 info = UTF-8("vigila-e2e-push-v1|" + relayId),
                 L    = 32)
```

- HKDF is RFC 5869 (HMAC-SHA256 extract-then-expand). An empty salt is replaced
  by 32 zero bytes per RFC 5869 §2.2 (not reached here — the salt above is
  non-empty).
- `relayId` is the lowercase-UUID relay identifier returned by `/api/v1/pair`.
  Binding it into `info` domain-separates keys across relays and pairings.
- The shared secret MUST be rejected if it is all-zero (low-order peer point).

The key persists for the life of the pairing. There is no per-message rekey;
re-pairing is the only rotation mechanism and it replaces the key on both ends.

## 2. Envelope and AEAD

Each push is one AEAD message:

```
envelope = 0x01 || nonce || ciphertextAndTag
```

- **Version**: a single leading byte `0x01`.
- **Nonce**: 12 bytes, drawn fresh from a CSPRNG for **every** message (never a
  counter).
- **AEAD**: ChaCha20-Poly1305 (IETF, 96-bit nonce, 128-bit tag). `nonce` is the
  cipher nonce; the trailing 16 bytes of the envelope are the Poly1305 tag.
- **Additional data (AAD)**: `UTF-8("vigila:push:v1:" + deviceTokenHex)`, where
  `deviceTokenHex` is the destination APNs device token, lowercased. This binds
  a ciphertext to exactly one device: a bridge that re-targets it to another
  registered device causes an authentication failure on receipt.
- **Minimum decoded length**: 30 bytes = version(1) + nonce(12) + ciphertext(≥1)
  + tag(16). The plaintext is never empty (the alert JSON always has content),
  so a conforming envelope is at least 30 bytes; openers and the bridge MUST
  reject anything shorter.
- The envelope is base64-encoded wherever it appears on the wire.

### Plaintext

The plaintext is the canonical alert JSON object (see [`RULES.md`](RULES.md)):
compact, pinned key order `v, event, rule, severity, router{id,name}, subject,
title, body, ts, dedupKey`, serialized as raw UTF-8. It carries no credentials.
The plaintext MUST NOT exceed **2560 UTF-8 bytes**; the relay enforces this at
seal time (truncating an over-long `body` on a character boundary, or dropping
the event if even an empty body would not fit).

### Decrypt rejection order (pinned)

An opener MUST reject in this order, with these distinct outcomes:

1. Length below 30 bytes → *bad envelope*.
2. First byte not `0x01` → *unsupported version*.
3. AEAD tag or AAD mismatch → *authentication failure*.

Seal-side, a plaintext above 2560 bytes → *plaintext too large*; derivation
against an all-zero shared secret → *zero shared secret*. These five outcome
codes are the ones exercised by [`vectors/e2e-vectors.json`](vectors/e2e-vectors.json).

## 3. Bridge HTTP API (v1)

The bridge is stateless apart from a device-registration table (§4). It is
designed so that no information about a device leaks before authentication.

### `POST /v1/register`

Registers, or re-keys, one device.

Request body:

```json
{ "deviceToken": "<lowercase hex>", "bridgeKey": "<base64(32 bytes)>" }
```

- `deviceToken`: hex, 64–200 characters, even length (case-normalized to lower).
- `bridgeKey`: standard base64 of **exactly 32 bytes** — the per-device key
  minted at pairing (see [`PAIRING.md`](PAIRING.md) §5).

The bridge stores `deviceToken → SHA-256(bridgeKey)` as an **idempotent upsert**.
A repeat with the same key refreshes a timestamp; the same token with a new key
replaces the stored hash. Created and updated are indistinguishable in the
response — there is no registration oracle.

| Status | Body | Meaning |
|---|---|---|
| `200` | `{"status":"ok"}` | Stored. |
| `400` | `{"error":"invalid_request"}` | Malformed token or key. |
| `503` | `{"error":"capacity_reached"}` | A global registration cap was reached for a **new** token. Existing tokens may still rotate their key. |

### `POST /v1/push`

Forwards one sealed envelope.

- Header `X-Vigila-Key: <base64 bridgeKey>` — the exact key registered for the
  target device.
- Body:

```json
{
  "deviceToken": "<lowercase hex>",
  "ciphertext":  "<base64 envelope>",
  "priority":    "high" | "normal",
  "pushType":    "alert"
}
```

`pushType` MUST be `"alert"`; `priority` MUST be `"high"` or `"normal"`; the
decoded envelope length MUST be between 30 and 3200 bytes (the upper bound is
headroom over the 2589-byte maximum a 2560-byte plaintext produces).

The bridge processes in a **pinned order** so nothing leaks pre-authentication:

1. Format validation (no lookup).
2. Device lookup.
3. Constant-time comparison of `SHA-256(decoded X-Vigila-Key)` against the
   stored hash.
4. Rate limit.
5. Forward to APNs.

| Status | Body | Meaning |
|---|---|---|
| `200` | `{"status":"ok"}` | APNs accepted the notification. |
| `400` | `{"error":"invalid_request"}` | Malformed field, bad `pushType`/`priority`, or envelope length out of range. |
| `404` | `{"error":"unknown_device"}` | No registration for this device token. |
| `403` | `{"error":"invalid_key"}` | `X-Vigila-Key` does not match the stored hash. |
| `429` | `{"error":"rate_limited"}` + `Retry-After: <seconds>` | Rate limit hit. |
| `502` | `{"error":"apns_rejected","reason":"<APNs reason>"}` | APNs rejected it; `reason` is APNs transport metadata only. |
| `504` | `{"error":"apns_timeout"}` | APNs did not respond in time. |

### Rate limiting

- **Per device**: token bucket, burst **10**, refill **1 token / 60 s**
  (≈ 60 pushes/device/hour). In-memory only — a bridge restart resets buckets.
- **Per source IP**: a smaller bucket (≈ 5 / hour), keyed on the client IP read
  **only** from a configured trusted proxy header (`X-Forwarded-For` by
  default). `POST /v1/register` consumes one token per attempt (empty bucket →
  `429`). `POST /v1/push` debits a token **only on pre-authentication
  rejections** (`400`/`403`/`404`); an IP that has exhausted this rejection
  budget is refused with a uniform `429` before any lookup, so a well-formed,
  authenticated push never spends a per-IP token. The bridge MUST run behind a
  proxy that sets the trusted header to the real client address and overwrites
  any client-supplied value; requests without the header share one fail-closed
  bucket.

Malformed JSON bodies map to `400 {"error":"invalid_request"}` without logging
(a parser error message could echo a token or ciphertext byte).

## 4. What the bridge stores and logs

**Storage.** A single table:

```
device_registration(
  device_token    TEXT PRIMARY KEY,   -- lowercase hex APNs token
  bridge_key_hash TEXT NOT NULL,      -- SHA-256 hex of the 32-byte bridgeKey
  created_at      TEXT NOT NULL,      -- ISO-8601 UTC, whole seconds
  updated_at      TEXT NOT NULL)
```

There is no payload column, no raw key, and no field for router credentials —
their absence is structural. **Retention**: a registration persists until it is
overwritten (re-register) for that device token; there is no time-based
expiry. A global row cap bounds table growth: once reached, new device tokens
are refused with `503` while already-registered tokens keep working and can
rotate their key. Rate-limit buckets live only in memory and are lost on
restart.

**Logging.** The bridge never logs the device token, the bridge key, or the
ciphertext. Where a device must be named in a log line, it uses a short,
non-reversible correlation id (the first 8 hex of `SHA-256(deviceToken)`). This
"no payload byte in logs" rule is enforced by a log-capture property test and a
black-box scan of captured logs in the end-to-end test harness.

## 5. APNs payload

The bridge composes a fixed payload around the opaque envelope. It never reads
or reshapes the ciphertext:

```json
{
  "aps": {
    "mutable-content": 1,
    "alert": { "title": "Vigila", "body": "New alert" },
    "sound": "default",
    "interruption-level": "time-sensitive"
  },
  "vigila": { "e": "<envelope base64>" }
}
```

- `interruption-level` is `"time-sensitive"` when the request `priority` is
  `"high"`, otherwise `"active"`.
- The `alert` title/body are fixed **pre-decryption placeholders**; the device's
  notification-service extension replaces them with the decrypted content.
- The notification is sent with APNs priority 10, push type `alert`, and a
  1-hour expiration. The composed payload MUST be ≤ 4096 bytes; the bridge drops
  an over-size payload rather than forward it (surfaced as
  `apns_rejected` with reason `payload_too_large`).

**On the device**, the extension reads `userInfo["vigila"]["e"]`,
base64-decodes it to the envelope, derives the AAD from its own stored device
token, and opens the envelope with the pairing key (§1–§2). A missing or
malformed `vigila.e`, a missing key, or an authentication failure falls back to
the placeholder notification; no failure path logs payload content.
