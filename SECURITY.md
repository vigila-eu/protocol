# Security policy

## Reporting a vulnerability

Please report security vulnerabilities **privately**. Do not open a public issue
for a suspected vulnerability.

Email: **security@vigila.eu**

> If that address does not yet reach us, use **support@vigila.eu** and put
> `SECURITY` in the subject line.

Please include enough detail to reproduce: the affected component (pairing,
push, bridge API, rule engine), the protocol step involved, and a proof of
concept where possible. If you have a suggested fix, include it.

## Scope

This repository specifies a protocol. Reports are welcome for:

- weaknesses in the specified pairing ceremony, key agreement, or push envelope;
- ways a conforming bridge could read, forge, or redirect alert content contrary
  to the guarantees in [`PUSH.md`](PUSH.md);
- errors in the shared test vectors that would let a broken implementation pass.

## Disclosure

We aim to acknowledge a report within a few business days and to agree a
coordinated disclosure timeline with the reporter. Please give us reasonable
time to remediate before any public disclosure.
