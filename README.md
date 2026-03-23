# Riverway Protocol

Riverway is a store-and-forward coordination protocol for continuous, unidirectional
state propagation over shared storage. A producer writes the current state of a system
into a channel slot; each node in the pipeline immediately forwards it to the next hop;
a consumer at the end reads the latest available state. There is no ACK, no backpressure,
and no expectation that any consumer is present. If a newer payload arrives before the
previous one was consumed, the previous one is overwritten and discarded — this is
correct behavior, not a failure condition.

Riverway is one protocol in the [DropChannel](https://github.com/dropchannel) runtime.
The system-level specification — including the `ChannelProvider` interface, encryption
standard, and protocol dispatch rules — lives in
[`dropchannel/spec`](https://github.com/dropchannel/spec).

---

## Contents

- [Conceptual model](#conceptual-model)
- [Propagation protocol](#propagation-protocol)
- [Node lifecycle](#node-lifecycle)
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

### Channel

A Riverway channel carries unidirectional flow from a single producer to zero or more
consumers. A channel has exactly one physical pipeline. There is no return pipeline and
no bidirectionality requirement — if bidirectional state exchange is needed, two
independent Riverway channels are used.

```
Producer → [physical pipeline] → Consumer(s)
```

### Physical pipeline

A physical pipeline is a directed sequence of one or more `ChannelProvider` hops. Each
hop holds at most one payload at a time. Nodes forward payloads immediately and
unconditionally — if a newer payload is available on the recv-slot, it overwrites
whatever is currently on the send-slot.

### Producer

A Producer is an endpoint that originates payloads. It writes to its send-slot at
whatever cadence its application requires and does not wait for any signal before writing
the next payload. The producer has no knowledge of pipeline depth, downstream consumers,
or whether prior payloads were consumed.

### Consumer

A Consumer is an endpoint that observes the tail of the pipeline. It calls `peek()` on
its recv-slot to read the latest available payload without clearing it. Multiple
consumers may independently observe the same tail slot. The consumer does not affect
slot state — it is purely observational. There is no signal sent back to the producer.

### Node

A Node is a process that forwards blobs from one `ChannelProvider` to the next. Nodes
are crypto-blind: they forward opaque bytes and perform no cryptographic operations. A
node has no `SHARED_SECRET`.

Node behavior is determined by the `riverway-` channel name prefix. A node operating on
a Riverway channel applies overwrite-forward semantics rather than the hold-and-cascade
semantics of a Tideway node.

### Separation of concerns

| Concern | Owner |
|---------|-------|
| Encryption / decryption | Endpoint only |
| Message semantics | Client application layer |
| Blob transport | ChannelProvider |
| Multi-hop composition | Node configuration |
| Delivery confirmation | Out of scope (no ACK) |
| Backpressure | Out of scope (no hold) |
| Consumer presence | Out of scope (not required) |

---

## Propagation protocol

### Slot semantics

**Slot presence = current state available. Slot absence = no state yet.**

A blob in a slot is the most recent payload to have reached that position. It remains
in the slot until overwritten by a newer payload from upstream. Consumers do not clear
slots — observation is non-destructive. There is no notion of a slot being "in-flight"
or "held pending ACK."

### Forward pass

When a node finds a payload on its recv-slot, it compares it to the last blob it
forwarded. If the content differs (or no prior forward has occurred), it overwrites the
send-slot unconditionally. If the content is identical, the forward is skipped — the
send-slot already holds the correct value. The node then sleeps regardless of outcome.

```
Node forward pass:

  blob = peek(recv_slot)        # non-consuming; None if empty
  if blob is not None:
      h = sha256(blob)          # hash over ciphertext; node stays crypto-blind
      if h != last_forwarded_hash:
          delete(send_slot)     # idempotent
          write(send_slot, blob)
          last_forwarded_hash = h
  sleep(POLL_INTERVAL)          # always sleep
```

The deduplication hash is held in memory only. It is not persisted across restarts. On
restart, the first recv-slot payload is always forwarded unconditionally.

### Consumer observation

The consumer calls `peek()` on its recv-slot — non-destructive. The slot is not cleared.
Multiple consumers may independently observe the same slot. The consumer sleeps
regardless of whether a payload was found.

```
Consumer observation:

  blob = peek(recv_slot)        # non-consuming; slot unchanged
  if blob is not None:
      deliver(blob)             # latest state delivered to application
  sleep(POLL_INTERVAL)          # always sleep
```

Because `peek()` does not clear the slot, the consumer will deliver the same blob on
every cycle until a newer payload propagates through the pipeline. Applications MUST be
prepared to receive the same payload repeatedly. To detect new arrivals, the application
MAY maintain its own hash of the last-seen blob and compare — this is an
application-layer concern.

### Producer write

The producer writes to its send-slot at application-determined cadence. If the send-slot
is still occupied from a prior write, the producer overwrites it unconditionally —
identical semantics to a node's overwrite-forward behavior.

```
Producer write:

  delete(send_slot)        # idempotent; clears prior value if present
  write(send_slot, payload)
```

### Key properties

**No backpressure.** The producer is never blocked by downstream state. It writes at its
own cadence regardless of downstream activity.

**No delivery confirmation.** The producer receives no signal that any consumer has
observed its payload. Consumer presence is not observable at the protocol level.

**Latest-wins.** At any point in the pipeline, a slot holds the most recent payload to
have reached that position. Older payloads do not queue.

**Stable between updates.** Once a payload has propagated to a slot, it remains there
undisturbed until a newer payload arrives. Nodes do not re-forward unchanged content.
Consumers do not clear slots. The belt carries the current value continuously.

**Non-destructive observation.** Any number of consumers may peek the tail slot
independently without interfering with each other or with pipeline operation.

**Crash safety.** If any participant crashes mid-forward, the most recently written
payload remains durably in every slot already written. On restart, each node inspects
its recv-slot and resumes. The in-memory deduplication hash is lost on restart, causing
at most one redundant forward on resumption.

**Consumer absence is not an error.** If no consumer is polling, the tail slot holds the
latest forwarded payload indefinitely, ready for any consumer that arrives.

---

## Node lifecycle

### State

Each node maintains one piece of in-memory state across cycles:

```
last_forwarded_hash: bytes | None   # SHA-256 of last blob written to send-slot
```

Initialized to `None` on startup. Not persisted across restarts.

### Startup: inspect recv-slot only

On startup the node calls `peek()` on its recv-slot once. The send-slot is not
inspected.

| Recv-slot | Action |
|-----------|--------|
| Empty | Set `last_forwarded_hash = None`. → Polling loop. |
| Occupied | Forward unconditionally. Set `last_forwarded_hash = sha256(blob)`. → Polling loop. |

### Polling loop (single steady state)

```
Loop:
  blob = peek(recv_slot)
  if blob is not None:
      h = sha256(blob)
      if h != last_forwarded_hash:
          delete(send_slot)
          write(send_slot, blob)
          last_forwarded_hash = h
  sleep(POLL_INTERVAL)    # unconditional
  repeat
```

The node sleeps at the end of every cycle without exception. It never watches its
send-slot. Its sole job is: when recv content has changed, push it forward.

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
| Slot contention | Write guard; no overwrite | Overwrite on change; deduplication otherwise |
| Consumer operation | `read()` — destructive | `peek()` — non-destructive |
| Multiple consumers | No | Yes |
| Consumer required | Yes — ACK cascade requires it | No |
| Pipeline direction | Bidirectional (two pipelines) | Unidirectional |
| Node state | Stateless between cycles | `last_forwarded_hash` in memory |
| Crash recovery | Full — any state reconstructible | Full — latest written payload survives; hash lost |
| Intended use | Reliable message passing | Continuous state observation |

---

## Version history

| Version | Summary |
|---------|---------|
| [v0.1](history/v0.1.md) | Initial protocol: overwrite-forward, no ACK, single-slot pipeline |
| [v0.2](history/v0.2.md) | Node deduplication via content hash; unconditional sleep; consumer uses `peek()` |

---

## Out of scope

- ACK / delivery confirmation (no protocol mechanism; left to application layer)
- Consumer presence detection (heartbeat protocol to be specified in a future revision)
- Ordered delivery guarantees across multiple payloads
- Multiple producers on a single channel
- Atomic overwrite (delete + write has a race window on all current providers)
- Payload sequencing or versioning
- Retention of prior payloads (history)
- Configurable overwrite policy (e.g. timestamp-gated overwrite)
