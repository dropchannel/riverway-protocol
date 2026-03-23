# Riverway Protocol

Riverway is a store-and-forward coordination protocol for continuous, unidirectional
state propagation over shared storage. A producer writes the current state of a system
into a Waterway; each Raft in the pipeline immediately forwards it to the next hop;
a consumer at the end reads the latest available state. There is no ACK, no backpressure,
and no expectation that any consumer is present. If a newer payload arrives before the
previous one was consumed, the previous one is overwritten and discarded — this is
correct behavior, not a failure condition.

Riverway is one protocol in the [DropChannel](https://github.com/dropchannel) runtime.
The system-level specification — including the `DockProvider` interface, encryption
standard, and protocol dispatch rules — lives in
[`dropchannel/spec`](https://github.com/dropchannel/spec).

---

## Contents

- [Conceptual model](#conceptual-model)
- [Propagation protocol](#propagation-protocol)
- [Raft lifecycle](#raft-lifecycle)
- [Comparison with Tideway](#comparison-with-tideway)
- [Version history](#version-history)
- [Out of scope](#out-of-scope)

---

## Conceptual model

### The belt

A Riverway is a one-way belt. The producer places the current state of a system
onto the belt; the belt carries it forward hop by hop; a consumer at the far end picks
up whatever is currently on the belt. The belt does not stop if nobody is at the far end.
It does not wait for acknowledgement that the last item was picked up. It does not hold
position between items. It moves forward continuously, and the value of any item on the
belt is its currency — not its eventual delivery.

### Channel and Waterway

A **Channel** is the named path between two fixed endpoints. A **Waterway** is a
protocol-typed flow path within a Channel. Riverway Waterways use the `riverway-`
prefix. A Channel may contain Waterways of different protocols simultaneously.

A Riverway Waterway carries unidirectional flow from a single producer to zero or more
consumers, always Upper toward Lower. There is no return flow — if bidirectional state
exchange is needed, two independent Riverway Waterways are used.

```
Producer → [upper_dock Waterway] → Raft → [lower_dock Waterway] → Consumer(s)
```

### Physical pipeline

A physical pipeline is a directed sequence of one or more Dock hops. Each hop holds at
most one payload file at a time. Rafts forward payloads immediately and unconditionally
— if a newer payload is available at the upper_dock, it overwrites whatever is currently
at the lower_dock.

For multi-hop pipelines, the lower_dock of one Raft serves as the upper_dock of the
next. The Waterway name is invariant across all Docks; only the Dock backend changes.

### Producer

A Producer is an endpoint that originates payloads. It writes to its upper_dock Waterway
at whatever cadence its application requires and does not wait for any signal before
writing the next payload. The producer has no knowledge of pipeline depth, downstream
consumers, or whether prior payloads were consumed.

### Consumer

A Consumer is an endpoint that observes the tail of the pipeline. It calls `peek()` on
its lower_dock Waterway to read the latest available payload without clearing it.
Multiple consumers may independently observe the same tail Waterway. The consumer does
not affect Waterway state — it is purely observational. There is no signal sent back to
the producer.

### Raft

A Raft is a process that forwards blobs from one Dock to the next. Rafts are
crypto-blind: they forward opaque bytes and perform no cryptographic operations. A Raft
has no `SHARED_SECRET`.

Raft behavior is determined by the `riverway-` Waterway name prefix. A Raft operating
on a Riverway Waterway applies overwrite-forward semantics rather than the turn-passing
semantics of a Tideway Raft.

### Separation of concerns

| Concern | Owner |
|---------|-------|
| Encryption / decryption | Endpoint only |
| Message semantics | Client application layer |
| Blob transport | Dock |
| Multi-hop composition | Raft configuration |
| Delivery confirmation | Out of scope (no ACK) |
| Backpressure | Out of scope (no hold) |
| Consumer presence | Out of scope (not required) |

---

## Propagation protocol

### Waterway file model

A Riverway Waterway holds exactly one payload file at a time, using the fixed canonical
filename `payload`. All participants address the file as `(channel, waterway, "payload")`
against the appropriate Dock.

**File present = current state available. File absent = no state yet.**

The file in the Waterway is the most recent payload to have reached that position. It
remains there until overwritten by a newer payload from upstream. Consumers do not clear
the file — observation is non-destructive. There is no notion of a file being
"in-flight" or "held pending ACK."

### Forward pass

When a Raft finds a payload at its upper_dock, it compares it to the last blob it
forwarded. If the content differs (or no prior forward has occurred), it overwrites the
lower_dock unconditionally. If the content is identical, the forward is skipped — the
lower_dock already holds the correct value. The Raft then sleeps regardless of outcome.

```
Raft forward pass:

  blob = upper_dock.peek(channel, waterway, "payload")   # non-consuming; None if absent
  if blob is not None:
      h = sha256(blob)              # hash over ciphertext; Raft stays crypto-blind
      if h != last_forwarded_hash:
          lower_dock.delete(channel, waterway, "payload")    # idempotent
          lower_dock.write(channel, waterway, "payload", blob)
          last_forwarded_hash = h
  sleep(POLL_INTERVAL)              # always sleep
```

The deduplication hash is held in memory only. It is not persisted across restarts. On
restart, the first upper_dock payload is always forwarded unconditionally.

### Consumer observation

The consumer calls `peek()` on its lower_dock Waterway — non-destructive. The file is
not cleared. Multiple consumers may independently observe the same tail Waterway. The
consumer sleeps regardless of whether a payload was found.

```
Consumer observation:

  blob = lower_dock.peek(channel, waterway, "payload")   # non-consuming; file unchanged
  if blob is not None:
      deliver(blob)             # latest state delivered to application
  sleep(POLL_INTERVAL)          # always sleep
```

Because `peek()` does not clear the file, the consumer will deliver the same blob on
every cycle until a newer payload propagates through the pipeline. Applications MUST be
prepared to receive the same payload repeatedly. To detect new arrivals, the application
MAY maintain its own hash of the last-seen blob and compare — this is an
application-layer concern.

### Producer write

The producer writes to its upper_dock Waterway at application-determined cadence. If
the file is still occupied from a prior write, the producer overwrites it unconditionally
— identical semantics to a Raft's overwrite-forward behavior.

```
Producer write:

  upper_dock.delete(channel, waterway, "payload")    # idempotent; clears prior value if present
  upper_dock.write(channel, waterway, "payload", payload)
```

### Key properties

**No backpressure.** The producer is never blocked by downstream state. It writes at its
own cadence regardless of downstream activity.

**No delivery confirmation.** The producer receives no signal that any consumer has
observed its payload. Consumer presence is not observable at the protocol level.

**Latest-wins.** At any position in the pipeline, a Waterway holds the most recent
payload to have reached that Dock. Older payloads do not queue.

**Stable between updates.** Once a payload has propagated to a Dock, it remains there
undisturbed until a newer payload arrives. Rafts do not re-forward unchanged content.
Consumers do not clear files. The belt carries the current value continuously.

**Non-destructive observation.** Any number of consumers may peek the tail Waterway
independently without interfering with each other or with pipeline operation.

**Crash safety.** If any participant crashes mid-forward, the most recently written
payload remains durably at every Dock already written. On restart, each Raft inspects
its upper_dock and resumes. The in-memory deduplication hash is lost on restart, causing
at most one redundant forward on resumption.

**Consumer absence is not an error.** If no consumer is polling, the tail Waterway holds
the latest forwarded payload indefinitely, ready for any consumer that arrives.

---

## Raft lifecycle

### State

Each Raft maintains one piece of in-memory state across cycles:

```
last_forwarded_hash: bytes | None   # SHA-256 of last blob written to lower_dock
```

Initialized to `None` on startup. Not persisted across restarts.

### Startup: inspect upper_dock only

On startup the Raft calls `peek()` on its upper_dock Waterway once. The lower_dock is
not inspected.

| upper_dock | Action |
|------------|--------|
| Absent | Set `last_forwarded_hash = None`. → Polling loop. |
| Present | Forward unconditionally. Set `last_forwarded_hash = sha256(blob)`. → Polling loop. |

### Polling loop (single steady state)

```
Loop:
  blob = upper_dock.peek(channel, waterway, "payload")
  if blob is not None:
      h = sha256(blob)
      if h != last_forwarded_hash:
          lower_dock.delete(channel, waterway, "payload")
          lower_dock.write(channel, waterway, "payload", blob)
          last_forwarded_hash = h
  sleep(POLL_INTERVAL)    # unconditional
  repeat
```

The Raft sleeps at the end of every cycle without exception. It never watches its
lower_dock. Its sole job is: when upper_dock content has changed, push it forward.

### Polling cost

| Phase | Operations per cycle |
|-------|---------------------|
| Startup | One `peek()`, once only |
| Polling (no change) | One `peek()` + hash compare |
| Polling (new payload) | One `peek()` + one `delete()` + one `write()` |

---

## Comparison with Tideway

| Property | Tideway | Riverway |
|----------|-------|----------|
| Delivery model | Exactly-once, end-to-end confirmed | Best-effort, latest-wins |
| Backpressure | Yes — sender blocked until ACK | None |
| ACK cascade | Yes | No |
| Waterway contention | Write guard; no overwrite | Overwrite on change; deduplication otherwise |
| Consumer operation | `read()` — destructive | `peek()` — non-destructive |
| Multiple consumers | No | Yes |
| Consumer required | Yes — ACK cascade requires it | No |
| Pipeline direction | Turn-passing (Upper ↔ Lower) | Unidirectional (Upper → Lower only) |
| Raft state | Stateless between cycles | `last_forwarded_hash` in memory |
| Crash recovery | Full — any state reconstructible | Full — latest written payload survives; hash lost |
| Intended use | Reliable message passing | Continuous state observation |

---

## Version history

| Version | Summary |
|---------|---------|
| [v0.1](history/v0.1.md) | Initial protocol: overwrite-forward, no ACK, single-slot pipeline |
| [v0.2](history/v0.2.md) | Node deduplication via content hash; unconditional sleep; consumer uses `peek()` |
| [v0.3](history/v0.3.md) | Vocabulary update: Node→Raft, ChannelProvider→Dock, slot→Waterway file, channel_id→channel; upper_dock/lower_dock replace recv_slot/send_slot; canonical filename `payload` defined |

---

## Out of scope

- ACK / delivery confirmation (no protocol mechanism; left to application layer)
- Consumer presence detection (heartbeat protocol to be specified in a future revision)
- Ordered delivery guarantees across multiple payloads
- Multiple producers on a single channel
- Atomic overwrite (delete + write has a race window on all current Dock backends)
- Payload sequencing or versioning
- Retention of prior payloads (history)
- Configurable overwrite policy (e.g. timestamp-gated overwrite)
