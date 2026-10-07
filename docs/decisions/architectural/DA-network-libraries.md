# DA-network-libraries: Akoe builds on the libraries ZeroClaw wraps, not on ZeroClaw

**Status:** Accepted by the owner on 2026-10-07
**Date:** 2026-10-07

## 1. Context

The owner's given for this project was that "ZeroClaw is used as a library …
Nobody writes their own WhatsApp or IMAP client." The founding agora found that
ZeroClaw's channel crate brings its whole agent runtime and drops what ephor
needs, and both participants preferred the libraries ZeroClaw itself wraps. They
disagreed only on whether that reading was already within the given (V-D01).
Ruling V-D01 adopted the libraries on the owner's behalf and flagged that it
departs from the given's literal words; the owner accepted it on 2026-10-07
(agent-grounds/agent-grounds#15).

## 2. Decision

Akoe builds each network adapter directly on the library ZeroClaw wraps for that
network:

| Network | Library |
|---|---|
| Mail | `async-imap`, `lettre` and `mail-parser` |
| WhatsApp | `whatsapp-rust` |
| Signal | the `signal-cli` JSON-RPC daemon |
| Matrix | `matrix-sdk` |
| Slack, Discord | the platforms' own APIs |

Each sits behind Akoe's own internal trait ([§FS-interfaces.6](../../functional-spec/FS-interfaces.md#6-no-library-crate-and-no-mcp-server)) and is pinned to
an exact version with a committed lockfile. The given's purpose holds: nobody
writes a WhatsApp or IMAP client. ZeroClaw stays the reference; the fixes Akoe
needs are offered upstream.

Source: ruling V-D01.

## 3. Why

At 0.8.5, ZeroClaw adds nothing on any network in scope and takes things away:

- **It discards what ephor needs.** WhatsApp Web drops the owner's ordinary own
  messages, which [§FS-record.6.1](../../functional-spec/FS-record.md#61-the-owners-own-messages-from-other-clients) requires; email observer mode restarts at the
  server's next UID on every connection and misses mail that arrived meanwhile,
  which [§FS-record.1.2](../../functional-spec/FS-record.md#12-the-read-is-the-observation) forbids; Signal ignores sync messages; Slack is bot-only
  (V-C05, V-C08).
- **It has no surface for the record or for sending.** The `Channel` trait
  offers a live stream and a `send` that returns nothing: no history, no edit
  events, no send identity, and a listener-health hook no network in scope
  implements (V-C04). The email channel can neither search the mailbox nor learn
  or set a sent mail's Message-ID (V-C06), so [§FS-sending.3](../../functional-spec/FS-sending.md#3-unknown-blocks-the-thread)'s correlation and
  [§FS-record.1](../../functional-spec/FS-record.md#1-mail-is-read-at-refresh)'s read at refresh would each need a patch.
- **It costs three times the dependencies.** An email-only build resolves to 320
  packages and 16 ZeroClaw crates, the agent runtime and providers among them,
  against 99 for the mail libraries alone (V-C02).
- **It moves fast and says so.** A release every one to five weeks, new channels
  rated Experimental with no stability promise, and 0.8.5's notes broke external
  Rust callers (V-C07).

Source: ruling V-D01, V-C02, V-C04, V-C05, V-C06, V-C07, V-C08.

## 4. What would reverse it

ZeroClaw is weighed again when its embeddable runtime (zeroclaw#10993) lands.
It earns a place only if, by then, it no longer pulls the agent runtime into a
channel-only build, and it has fixed each defect that Akoe's specs rule out:

1. a complete read of incoming and Sent mail that marks nothing and exposes
   UIDVALIDITY, UIDs and envelope metadata ([§FS-record.1.2](../../functional-spec/FS-record.md#12-the-read-is-the-observation));
2. ordinary WhatsApp own messages and Signal sync-sent events passed through,
   with native ids, revisions and deletions ([§FS-record.6](../../functional-spec/FS-record.md#6-own-messages-edits-and-deletions));
3. listener health tied to the actual receiving session, its committed
   frontier and its catch-up ([§FS-record.3.2](../../functional-spec/FS-record.md#32-the-vouching-lease));
4. a send id supplied or returned, with acceptance, a definite failure before
   sending, and an ambiguous attempt told apart ([§FS-sending.2](../../functional-spec/FS-sending.md#2-one-attempt-three-outcomes));
5. a durable commit handoff for received messages, unless the adapter passes
   the kill-and-replay proof ([§FS-record.3.5](../../functional-spec/FS-record.md#35-a-kill-loses-no-received-message));
6. Gmail's native thread id ([§FS-names.2](../../functional-spec/FS-names.md#2-threading-generic-mail)).

Source: ruling V-D01, V-A03.

## 5. Rejected: a pinned ZeroClaw fork

The alternative was ZeroClaw itself, as the given's literal words read: every
`zeroclaw-*` crate declared at `=0.8.5`, because a pin on the top crate alone
leaves its siblings at caret versions; `Cargo.lock` committed, and every build
and install `--locked`; and a fork, carried through `[patch.crates-io]` and owned
by Akoe's maintainers, holding the six fixes of [§DA-network-libraries.4](DA-network-libraries.md#4-what-would-reverse-it), each
offered upstream. Each ZeroClaw release would mean rebasing the fork and
rerunning the keeper checks and the kill-and-replay proof before moving the pin.
Mail would still be read at refresh, not kept by a listener; the choice changes
which code reads each network, not the design or the order of work.

It is rejected for the reasons of [§DA-network-libraries.3](DA-network-libraries.md#3-why), and because it
leaves Akoe patching beneath the boundary it pinned.

Source: V-A03, ruling V-D01.

## 6. Consequences

The mail libraries build on Rust 1.91.1. `whatsapp-rust` 0.7.0 requires Rust
1.94 and `matrix-sdk` 0.19.1 requires 1.96, so the toolchain rises with those
networks; it would with ZeroClaw too.

Source: V-C03.
