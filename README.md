# Conveyer Protocol

Conveyer is a store-and-forward coordination protocol for continuous, unidirectional
state propagation over shared storage. A producer writes the current state of a system
into a channel slot; each node in the pipeline immediately forwards it to the next hop;
a consumer at the end reads the latest available state. There is no ACK, no backpressure,
and no expectation that any consumer is present. If a newer payload arrives before the
previous one was consumed, the previous one is overwritten and discarded — this is
correct behavior, not a failure condition.

Conveyer is one protocol in the [DropChannel](https://github.com/dropchannel) runtime.
The system-level specification — including the `ChannelProvider` interface, encryption
standard, and protocol dispatch rules — lives in
[`dropchannel/spec`](https://github.com/dropchannel/spec).

---

## Contents

- [Conceptual model](#conceptual-model)
- [ChannelProvider interface](#channelprovider-interface)
- [Propagation protocol](#propagation-protocol)
- [Node lifecycle](#node-lifecycle)
- [Configuration](#configuration)
- [Comparison with Winch](#comparison-with-winch)
- [Version history](#version-history)
- [Out of scope](#out-of-scope)

---

## Conceptual model

### The belt

A Conveyer channel is a one-way belt. The producer places the current state of a system
onto the belt; the belt carries it forward hop by hop; a consumer at the far end picks
up whatever is currently on the belt. The belt does not stop if nobody is at the far end.
It does not wait for acknowledgement that the last item was picked up. It does not hold
position between items. It moves forward continuously, and the value of any item on the
belt is its currency — not its eventual delivery.

### Channel

A Conveyer channel carries unidirectional flow from a single producer to zero or more
consumers. A channel has exactly one physical pipeline. There is no return pipeline and
no bidirectionality requirement — if bidirectional state exchange is needed, two
independent Conveyer channels are used.

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

A Consumer is an endpoint that reads from the tail of the pipeline. It reads the latest
available payload using `read()` (consume-on-read). If no payload is present, it finds
an empty slot and waits. The consumer's `read()` is terminal — it clears the slot and
does not trigger any cascade. There is no confirmation sent back to the producer.

### Node

A Node is a process that forwards blobs from one `ChannelProvider` to the next. Nodes
are crypto-blind: they forward opaque bytes and perform no cryptographic operations. A
node has no `SHARED_SECRET`.

Node behavior is determined by the `conveyer-` channel name prefix. A node operating on
a Conveyer channel applies overwrite-forward semantics rather than the hold-and-cascade
semantics of a Winch node.

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

## ChannelProvider interface

Conveyer operates through the standard `ChannelProvider` interface defined in
[`dropchannel/spec`](https://github.com/dropchannel/spec). All storage operations are
expressed through these five operations:

| Operation | Behavior |
|-----------|----------|
| `write(channel_id, slot, data)` | Deposit blob. Returns `True` on success, `False` if slot occupied. |
| `read(channel_id, slot)` | Retrieve and delete blob (consume-on-read). Used by the consumer endpoint only. |
| `peek(channel_id, slot)` | Retrieve blob without consuming it. Used by nodes. |
| `exists(channel_id, slot)` | Returns `True` if a blob is present, `False` if empty. |
| `delete(channel_id, slot)` | Delete blob. Idempotent. |

Conveyer nodes use `exists()` to poll the recv-slot, `peek()` to read it, and `delete()`

- `write()` to overwrite the send-slot. Nodes never call `read()`. The consumer uses
only `read()`. There is no ACK cascade and no send-slot polling by any participant.

---

## Propagation protocol

### Slot semantics

**Slot presence = current state available. Slot absence = no state yet.**

A blob in a slot is the most recent payload available at that position. It may be
consumed by the downstream participant or overwritten by a newer payload — both outcomes
are valid. There is no notion of a slot being "in-flight" or "held pending ACK."

### Forward pass

When a node finds a payload on its recv-slot, it immediately writes it to its send-slot.
If the send-slot is already occupied (by a prior payload not yet consumed downstream),
the node **overwrites it unconditionally**. The prior payload is discarded. This is the
defining behavior of the Conveyer protocol.

```
Node forward pass:

  blob = peek(recv_slot)   # non-consuming read
  delete(send_slot)        # clear prior value if present; idempotent if empty
  write(send_slot, blob)   # write latest payload forward
```

The node does not check whether the send-slot is occupied before overwriting. `delete()`
is idempotent — it is safe to call on an empty slot. Overwrite is always unconditional.
The window between `delete()` and `write()` is a known limitation — see
[Out of scope](#out-of-scope).

### Consumer read

The consumer calls `read()` on its recv-slot. This is consume-on-read: the blob is
returned to the application and the slot is cleared. No cascade is triggered. The
consumer does not signal the producer or any intermediate node that delivery occurred.

```
Consumer read:

  blob = read(recv_slot)        # consume-on-read; slot cleared
  if blob is None:
      sleep(poll_interval)      # no payload available yet; wait
  else:
      deliver(blob)             # latest state delivered to application
```

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
own cadence regardless of whether prior payloads were consumed.

**No delivery confirmation.** The producer receives no signal that any payload was
consumed. Consumer presence is not observable at the protocol level.

**Latest-wins.** At any point in the pipeline, the value in a slot is the most recent
payload to have reached that position. Older payloads do not queue behind newer ones.

**Crash safety.** If any participant crashes mid-forward, the most recently written
payload remains durably in every slot already written. On restart, each participant
inspects its slots and resumes forwarding. No payload is lost that had already been
written to a slot — though payloads that were in-memory at crash time and not yet written
are not recoverable.

**Consumer absence is not an error.** If no consumer is polling, payloads accumulate at
the tail slot and are overwritten by newer ones. The pipeline continues to operate
correctly.

---

## Node lifecycle

### Startup: inspect recv-slot only

On startup the node calls `peek()` on its recv-slot once to determine starting state.
The send-slot state is irrelevant — the node will overwrite it unconditionally on the
first forward.

| Recv-slot | Action |
|-----------|--------|
| Empty | → Polling loop (idle) |
| Occupied | Forward immediately: `delete()` send → `write()` send → Polling loop |

### Polling loop (single steady state)

There is only one steady state. The node polls its recv-slot continuously and forwards
whatever it finds, overwriting the send-slot each time.

```
Loop:
  exists(recv_slot)?
    No  → sleep(POLL_INTERVAL), repeat
    Yes → blob = peek(recv_slot)
          delete(send_slot)
          write(send_slot, blob)
          repeat
```

The node never watches its send-slot. Whether the prior payload was consumed downstream
is not its concern. It has one job: when something is on the recv-slot, put it on the
send-slot and wait for the next one.

### Polling cost

| Phase | Operations per cycle |
|-------|---------------------|
| Startup | One `peek()` call, once only |
| Polling (idle) | One `exists()` |
| Forward transition | One `peek()` + one `delete()` + one `write()` |

---

## Configuration

### Producer endpoint

```bash
CHANNEL_ID=conveyer-<identifier>
SHARED_SECRET=<64 hex chars = 32 bytes>
CHANNEL_PROVIDER=<gcs|httprelay|dropbox|local>
SEND_SLOT=<slot this endpoint writes to>
POLL_INTERVAL=<seconds>   # cadence for overwrite check; default 5
```

### Consumer endpoint

```bash
CHANNEL_ID=conveyer-<identifier>
SHARED_SECRET=<64 hex chars = 32 bytes>
CHANNEL_PROVIDER=<gcs|httprelay|dropbox|local>
RECV_SLOT=<slot this endpoint reads from>
POLL_INTERVAL=<seconds>   # default 5
```

### Node

```bash
CHANNEL_ID=conveyer-<identifier>
# No SHARED_SECRET — nodes never encrypt or decrypt
RECV_PROVIDER=<gcs|httprelay|dropbox|local>
SEND_PROVIDER=<gcs|httprelay|dropbox|local>
RECV_SLOT=<slot name>
SEND_SLOT=<slot name>
POLL_INTERVAL=<seconds>   # default 5
```

Provider-specific env vars follow the same namespacing convention as Winch nodes
(`RECV_GCS_BUCKET_NAME`, `SEND_RELAY_URL`, etc.). See the system spec for the full table.

---

## Comparison with Winch

| Property | Winch | Conveyer |
|----------|-------|----------|
| Delivery model | Exactly-once, end-to-end confirmed | Best-effort, latest-wins |
| Backpressure | Yes — sender blocked until ACK | None |
| ACK cascade | Yes | No |
| Slot contention | Write guard; no overwrite | Overwrite always |
| Consumer required | Yes — ACK cascade requires it | No |
| Pipeline direction | Bidirectional (two pipelines) | Unidirectional |
| Startup stale send-slot | Impossible (error) | Recoverable (delete and continue) |
| Crash recovery | Full — any state reconstructible | Full — latest written payload survives |
| Intended use | Reliable message passing | Continuous state propagation |

---

## Version history

| Version | Summary |
|---------|---------|
| [v0.1](history/v0.1.md) | Initial protocol: overwrite-forward, no ACK, single-slot pipeline |

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
