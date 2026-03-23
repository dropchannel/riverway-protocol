# CLAUDE.md — current-protocol

## What this repo is

This is the **Current protocol specification** for the [DropChannel](https://github.com/dropchannel) system. It defines the state machine, Waterway semantics, and participant behavior for `current-` prefixed channels.

**Naming note:** This repo is in the process of being renamed. The protocol was previously called *Conveyer* (prefix `conveyer-`) and is now called *Current* (prefix `current-`). The sibling Tide protocol was previously called *Winch*. History files (v0.1.md, v0.2.md) still use the old names — do not update them.

---

## Where this fits in DropChannel

| Repo | Role |
|------|------|
| `dropchannel/spec` | ChannelProvider interface, encryption standard, security model, protocol registry, Agent/Worker runtime |
| **`dropchannel/current-protocol`** | **This repo — Current protocol state machine spec** |
| `dropchannel/tide-protocol` | Tide protocol spec (exactly-once, ACK cascade) |
| `dropchannel/dropchannel-py` | Python reference implementation (all providers + protocols) |
| `dropchannel/dc-monitor` | Topology visualizer (consumes `telemetry-` channel) |

The **spec repo** is authoritative for the ChannelProvider interface, encryption, and protocol dispatch rules. This repo only owns the Current protocol state machine.

---

## Current protocol in brief

Current is a **latest-wins, no-ACK, unidirectional** propagation protocol. It is the right choice when you want to continuously observe a system's current state and don't need delivery confirmation.

**Key semantics:**

- Producer writes at its own cadence; never blocked
- Nodes forward only when recv content has changed (`last_forwarded_hash` deduplication via SHA-256 over ciphertext)
- Consumers use `peek()` — non-destructive; Waterway is never cleared by consumers
- No ACK, no backpressure, no `read()` used anywhere in the protocol
- Multiple consumers may independently observe the same lower-most Dock

**Channel prefix:** `current-<identifier>` (e.g. `current-system-metrics`, `current-gpu-state`)

**Contrast with Tide:** Tide uses `read()` + ACK cascade for exactly-once confirmed delivery with backpressure. Current uses `peek()` for continuous best-effort state observation. Choose Tide when every message must be confirmed before the next can be sent.

---

## ChannelProvider interface summary

Defined in `dropchannel/spec`. Current participants use:

| Operation | Producer | Node | Consumer |
|-----------|----------|------|----------|
| `write()` | ✓ | ✓ | — |
| `peek()` | — | ✓ | ✓ |
| `delete()` | ✓ | ✓ | — |
| `read()` | — | — | — |
| `exists()` | — | — | — |

---

## Current version: v0.2

- v0.1 → v0.2: added node deduplication hash, guaranteed per-cycle sleep, switched consumer from `read()` to `peek()`
- v0.1 and v0.2 are wire-compatible for nodes but consumer-incompatible (v0.1 consumer destructively clears the lower-most Dock)
- History in `history/` — do not edit history files

---

## Key open items / known gaps

- **Non-atomic overwrite:** `delete()` + `write()` has a race window where the Waterway is momentarily empty; atomic replace not yet required by spec
- **No consumer presence detection:** deferred to a future revision (likely heartbeat integration)
- **No payload sequencing:** application-layer concern
- **Single producer only:** no detection or prevention of concurrent producers

---

## Encryption

End-to-end AES-256-GCM between producer and consumer. Nodes forward opaque ciphertext and hold no `SHARED_SECRET`. Node deduplication hash is computed over ciphertext — nodes remain fully crypto-blind. Encryption standard is defined in `dropchannel/spec`.
