# Rule engine

The relay evaluates alert rules against periodic router snapshots. The engine is
a **pure reducer**:

```
(previousState, snapshot, rules) → (nextState, [events])
```

It is implemented independently in two languages (the production relay, and the
app for local evaluation and previews). Both implementations MUST agree, and
[`vectors/rule-vectors.json`](vectors/rule-vectors.json) is the shared oracle
that pins their agreement.

## State machine

State is tracked per **key** = `(routerId, ruleType, subject)`, where `subject`
identifies the specific object a rule watches (an interface, a probe, a peer,
and so on). Each key has a persisted phase:

- **OK** — the rule predicate is false (or no state exists yet). The implicit
  resting phase.
- **PENDING** — the predicate is true but the debounce condition is not yet
  satisfied.
- **FIRING** — the debounce condition is satisfied; a `firing` event was
  emitted.

`RESOLVED` is a **transient emitted event**, not a persisted phase: when a
firing predicate goes false, the engine emits a `resolved` event and the key
returns to OK.

```
OK ──predicate true──▶ PENDING ──debounce satisfied──▶ FIRING
▲                         │                               │
└── predicate false ──────┴─── predicate false / resolved ┘
```

Transitions are governed by per-rule fields:

- **`debouncePolls`** — the number of consecutive true evaluations required
  before FIRING (default `2`, minimum `1`; some rule types override it). This is
  the debounce that suppresses transient blips.
- **`resendAfterMin`** — while a key is FIRING, re-emit the `firing` event every
  this-many minutes as a reminder (`0` disables reminders).
- **`notifyRecovery`** — whether to emit the `resolved` event when a firing key
  recovers.

## Flap suppression

To avoid notification storms from a rapidly oscillating condition, a key's
notifications are **flap-suppressed within 60 seconds** of its last emitted
notification. A suppressed pulse may be re-emitted at a later boundary crossing;
the "last notified" timestamp survives a resolve so it continues to gate the
next flap.

## Events and acknowledgement

Each event carries a `dedupKey` of the form `"{routerId}/{ruleType}/{subject}"`,
which the app uses to deduplicate and to acknowledge an incident (an
acknowledgement flips an `acknowledged` flag on the matching firing incidents).
The emitted event object is exactly the plaintext that end-to-end push encrypts
(see [`PUSH.md`](PUSH.md) §2).

## The vectors

[`vectors/rule-vectors.json`](vectors/rule-vectors.json) contains **50**
table-driven cases. Each case gives a router, a rule set, a base snapshot, and a
sequence of `steps`; every step pins the expected next state and the events the
engine MUST emit. The cases cover, among others: link state, router
reachability, netwatch, sustained CPU, temperature and memory thresholds, reboot
and uptime detection, PPPoE drops, stale WireGuard handshakes, cold-start level
semantics, the notify-on-recovery gate, 60-second flap suppression, reminder
resends, and the acknowledged flag.

The file is the contract: an implementation that fails any vector is wrong.
