# CLAUDE.md — current-protocol

This file provides guidance to Claude Code when working with this repository.

## What this repo is

The specification for the **Riverway protocol** — one protocol in the DropChannel
waterway system. No source code, build system, or tests. Markdown specs only.

System-level concerns (DockProvider interface, encryption standard, security model,
Agent/Worker runtime, observability) live in `dropchannel/spec` and are not owned here.

**Naming note:** This repo is being renamed. The protocol was previously called
*Conveyer* (prefix `conveyer-`), then *Current* (prefix `current-`), and is now called
*Riverway* (prefix `riverway-`). History files (v0.1.md, v0.2.md) use old names —
do not update them.

**Links note:** Cross-repo links (to other repos in the DropChannel org) use
<https://github.com/dropchannel/> as the URL root. Intra-repo links use relative URLs.

## Scope

This repo owns exactly one thing: the **Riverway state machine specification**.

### What belongs here

- Participant roles and behaviors (Endpoint, Raft)
- Waterway file model — how many files, naming convention, what presence/absence means
- Forward pass mechanics — which Dock a Raft reads from and writes to, and when
- Deduplication mechanics (`last_forwarded_hash`)
- Consumer behavior (peek semantics, multi-consumer model)
- Protocol-specific error conditions and halt rules
- Version history

### What does not belong here

- DockProvider interface definition — see `dropchannel/spec/channel-provider.md`
- Encryption details — see `dropchannel/spec/encryption.md`
- Security model — see `dropchannel/spec/security-model.md`
- Agent/Worker config schema — see `dropchannel/spec/agent.md`
- Heartbeat/telemetry/observability — see `dropchannel/spec/`
- Dock backend specifics (GCS, Dropbox, local, httprelay)
- Implementation guidance, configuration, or operational concerns

**The clean test:** a Raft implementor should need only this repo and
`dropchannel/spec/channel-provider.md` to implement Riverway correctly.

## Working with this repo

The canonical spec is `README.md`. Version history snapshots live in `history/v0.x.md`.

**Current state:** `README.md` is current through v0.3. `history/v0.1.md`,
`history/v0.2.md`, and `history/v0.3.md` exist. Do not edit history files.

**To propose a protocol change:** Read `README.md`, then produce a new
`history/v0.x.md` and an updated `README.md`.

Typical prompt:
> "Read README.md. I want to spec the following change to Riverway: [description].
> Generate history/v0.x.md and an updated README.md."

**History files record changes only — not the full protocol.** A history file must
contain: the version, what changed from the prior version, the design rationale, and
any compatibility notes. It must not restate unchanged mechanics. The full protocol
definition at any point in time is `README.md`.

**Keeping CLAUDE.md current:** When producing a new `history/v0.x.md`, review
CLAUDE.md and update it to reflect any changes — in particular the current version
in the "Working with this repo" section and any new or retired terms in the
Vocabulary table.

## Vocabulary

This repo uses DropChannel waterway terminology throughout. Do not use retired terms.

| Term | Retired term |
|---|---|
| Raft | Node |
| Dock / DockProvider | ChannelProvider |
| Waterway | slot, reach |
| upper_dock / lower_dock | recv_slot / send_slot |
| Channel | channel_id (as namespace) |
| Riverway | Conveyer, Current |
